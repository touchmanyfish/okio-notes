# AsyncTimeout

timeout+deadline约束范围：

1. 使用enter/exit/timedOut()；当timeout+deadline约束下的超时出现时回调timedOut()
2. 使用sink()方法返回一个wrappedSink;timeout+deadline对wrappedSink的约束参考后续分析
3. 使用source()方法返回一个wrappedSource;timeout+deadline对wrappedSource的约束参考后续分析

## enter/exit/timedOut()调用时机

**enter/exit：**

enter()+一次或者多次exit()

**timedOut:**

当AsyncTimeout中定义的超时出现时回调，它可能出现在：

1. 在enter()之后，exit()之前
2. 在enter()之后

## 数据结构

```text
ta:timeOutAt

object AsyncTimeout:
head(AsyncTimeout()) -> AsyncTimeout1(ta:5) -> AsyncTimeout2(ta:10) -> null
```

* 如果要启动Watchdog，会先让head指向AsyncTimeout();如果Watch要退出，会先先让head指向null
* AsyncTimeout(ta)从head后开始插入，插入后需要保持ta值升序；

这个链表整个系统共享，它由一把全局锁AsyncTimeout.lock保护;目前使用lock锁的地方有3处：

1. scheduleTimeout()
2. cancelScheduledTimeout()
3. Watchdog

为了方便分析，我们将AsyncTimeout1所在位置称为 **“链表前端”**。将下面的情况叫做"链表为空":

```text
head(AsyncTimeout()) -> null
```

## Watchdog

### 启动

调用scheduleTimeout()方法时，如果发现当前没有Watchdog，就会启动它。

### 内部while循环

Watchdog内部有一个while循环，执行可能暂停在如下状态：

1. 等待 lock
2. await(IDLE_TIMEOUT_MILLIS)；此时:head(node) -> null
3. await()；此时:head(node) -> node1(ta) -> ...;ta时间还未到。

### node超时

如果在while循环中发现有node.ta时间到了，则会:

1. 将node从链表移除
2. 调用它的timeOut()
3. 继续while循环

### 退出

如果从await(IDLE_TIMEOUT_MILLIS)中恢复以后，队列依然为空，WatchDog就会执行退出。

### await(IDLE_TIMEOUT_MILLIS)时遇到虚假唤醒

```kotlin
 if (node == null) {
    val startNanos = System.nanoTime()
    // A
    condition.await(IDLE_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)
    return if (head!!.next == null && System.nanoTime() - startNanos >= IDLE_TIMEOUT_NANOS) {
        head // The idle timeout elapsed.
    } else {
        null // The situation has changed.
    }
}
```

**问题流程**：

1. 执行A处代码以后会当前线程会 awiat IDLE_TIMEOUT_MILLIS 时间
2. 时间还未到，由于虚假唤醒 await 结束
3. watchdog 继续执行 awaitTimeout(),如果中间没有 node 插入，则会再次执行1./2.

上述流程中每次执行 await() 时都是等待 IDLE_TIMEOUT_MILLIS 时间，并没有“补觉”。

Okio不“补觉”的原因：

* 虚假唤醒出现概率较低，这种情况很难出现；就算出现了，总睡眠时间就是 IDLE_TIMEOUT_MILLIS * 1.x(x < 1)
* 这样增加实现难度。

### 为front node进行await()时遇到虚假唤醒

```kotlin
var waitNanos = node.remainingNanos(System.nanoTime())

// The head of the queue hasn't timed out yet. Await that.
if (waitNanos > 0) {
    condition.await(waitNanos, TimeUnit.NANOSECONDS)
    return null
}
```

如果这里发生了虚假唤醒，由于外部有while循环，所以代码可能会再次执行到这块，需要睡眠的时间会重新计算，然后await()正确的时间。

## scheduleTimeout(node,timeoutNanos,hasDeadline)

**这个方法负责：**

1. 将node插入链表
2. 启动或者signal Watchdog

### 插入链表

先计算node.timeOutAt,然后将其从head后面插入到链表，插入后链表中需要保持timeOutAt升序。

### 启动或者signal Watchdog

scheduleTimeout()发现Watchdog未启动时，会先启动它。

如果node被插入到了链表前端，表示它的ta时间更小;如果Watchdog处于await()状态，它应从await()中恢复，因为最早的ta发生了改变；它应重新计算是否需重新进入
await()或者返回对应的node；如果是等待lock状态，则不用做任何操作。

Okio选择只要node被插入到了front位置，就调用conditon.signal()；

当Watchdog处于等待lock状态时，conditon.signal()显然起不到任何作用。为什么Okio这里不判断一下这种情况来避免无用signal()
的无用调用呢？gemini说：

* 并发编程中引入额外的状态位会增加复杂度
* 逻辑归一化：用多调用一次signal()的代价换取插入逻辑和Watchdog的启动逻辑的解开耦

### node.timeoutAt的计算防溢出优化

```kotlin
// AsyncTimeout#scheduleTimeout(node,timeoutNanos,hasDeadline)

// A
node.timeoutAt = now + minOf(timeoutNanos, node.deadlineNanoTime() - now)

//为什么不写成如下形式？
// B
node.timeoutAt = minOf(now + timeoutNanos, node.deadlineNanoTime())
```

**分析B:**

B处代码中now + timeoutNanos可能会出现溢出。

**分析A：**

1. 如果 node.deadlineNanoTime() - now 更小

```text
now  + (node.deadlineNanoTime() - now)
V
node.deadlineNanoTime()

结果显然有意义
```

2. 如果timeoutNanos更小

```text
由于imeoutNanos更小，有：
timeoutNanos < deadlineNanoTime() - now

此时我们假设发生溢出 ，于是有：
now + timeoutNanos > LONGMAX

让左边只剩下timeoutNanos，上述不等式变为

timeoutNanos > LONGMAX - now
timeoutNanos < deadlineNanoTime() - now

于是有
LONGMAX - now < deadlineNanoTime() - now

去掉两边的now变为
LONGMAX < deadlineNanoTime()

显然deadlineNanoTime() 必然是一个 <= LONGMAX的值，所以这种情况下不会有溢出出现
```

## cancelScheduledTimeout(node)

这个方法的执行流程：

1. node没插入过链表，返回
2. node在链表中，只将它从链表中移除
3. node没在链表中，返回

### inQueue

inQueue值的修改时机

1. 默认false
2. 入队时修改为true；
3. 超时移除队列以后继续保持true
4. cancelScheduledTimeout()中会将true修改为false

### 移除node但是不唤醒Watchdog

```text
head -> node1(500) ->  node2(10_000) ->  node3(15_000) ->...
```

如果我们要移除node1和node2

**假设移除node时唤醒Watchdog:**

1. Watchdog为了node1进入await()
2. 只从链表中移除node1
3. condition.signal()唤醒Watchdog

4. Watchdog为了node2进入await()
5. 只从链表中移除node2
6. condition.signal()唤醒Watchdog

7. Watchdog为了node3进入await()

**假设移除node时不唤醒Watchdog:**

1. Watchdog为了node1进入await()
2. 只从链表中移除node1
3. Watchdog还处于node1的await()当中

4. 只从链表中移除node2
5. Watchdog还处于node1的await()当中

6. node1对应的await()时间到，Watchdog为node3进入await()

显然，“移除node时不唤醒Watchdog，等到它自然醒，然后在需要时才进入await()”可以减少singal()/await()的调用(gemini说它俩的开销比较昂贵)，从而提升了性能。

### 调用exit()以后还可能执行node.timeOut()

```text
// AsyncTimeout.kt注释

Note that the call to timedOut is asynchronous, and may be called after exit.
```

node.timedOut()方法可能在exit()方法之后被调用

```kotlin
// Watchdog

override fun run() {
    while (true) {
        try {
            var timedOut: AsyncTimeout? = null
            AsyncTimeout.lock.withLock {
                timedOut = awaitTimeout()

                // The queue is completely empty. Let this thread exit and let another watchdog thread
                // get created on the next call to scheduleTimeout().
                if (timedOut === head) {
                    head = null
                    return
                }
            }
            // A

            // time window

            // B
            // Close the timed out node, if one was found.
            timedOut?.timedOut()
        } catch (ignored: InterruptedException) {
        }
    }
}
```

注意上面的时间窗口A-B:

1. 此时lock已经释放掉了
2. 还未执行node.timedOut()

假设worker线程在时间窗口调用exit()方法且在窗口中完成执行：

1. exit()
2. cancelScheduledTimeout(this):获取lock，发现node不在链表中，于是直接返回；

时间窗口A-B结束，timedOut()执行。

**Okio为什么这样设计?:**

如果timedOut()在lock范围内;如果它很耗时，由于获取不到lock，整个系统都没法执行node的插入/移除。

如果timedOut()在lock之外;如果它很耗时，虽然它也会卡住Watchdog线程,但是可以让lock更快的被释放，其他代码就可以更快的获取lock，执行node的插入/移除。

Okio将可能会耗时的操作从lock范围移除，来提高node的插入/删除的性能。

## sink(sink)
```kotlin
// AsyncTimeout.kt

fun sink(sink: Sink): Sink {
    return object : Sink {
        override fun write(source: Buffer, byteCount: Long) {
            checkOffsetAndCount(source.size, 0, byteCount)

            var remaining = byteCount
            while (remaining > 0L) {
                // Count how many bytes to write. This loop guarantees we split on a segment boundary.
                var toWrite = 0L
                var s = source.head!!
                while (toWrite < TIMEOUT_WRITE_SIZE) {
                    val segmentSize = s.limit - s.pos
                    toWrite += segmentSize.toLong()
                    if (toWrite >= remaining) {
                        toWrite = remaining
                        break
                    }
                    s = s.next!!
                }

                // Emit one write. Only this section is subject to the timeout.
                withTimeout { sink.write(source, toWrite) }
                remaining -= toWrite
            }
        }

        override fun flush() {
            withTimeout { sink.flush() }
        }

        override fun close() {
            withTimeout { sink.close() }
        }

        override fun timeout() = this@AsyncTimeout

        override fun toString() = "AsyncTimeout.sink($sink)"
    }
}
```
为了方便分析有如下代码

```kotlin
val wrappedSink = asyncTimeout.sink(innerSink)
```

### withTimeout{}

被withTimeout{}包裹的代码被当前AsyncTimeout(timeout+deadline)约束。

### wrappedSink.flush()

inner.flush()受AsyncTimeout(timeout+deadline)约束

### wrappedSink.close()

inner.close()受AsyncTimeout(timeout+deadline)约束

### wrappedSink.write(source, byteCount)

wrappedSink.write(source,byteCount)时会将source中byteCount总量多数据分为多次，每次不超过TIMEOUT_WRITE_SIZE的innerSink.write()进行写入；

“innerSink.write()”会被包裹到withTimeout{}中，被AsyncTimeout(timeout+deadline)约束。例如:

```kotlin
val asyncTimeout = AsyncTimeout().apply {
    timeout(5_000L, TimeUnit.MILLISECONDS)
    deadline(System.currentTimeMillis() + 50_000L, TimeUnit.MILLISECONDS)
}

val wrappedSink = asyncTimeout.sink(innerSink)
```

1. wrappedSink.write()执行了40s没有发生timeout，因为可能其中执行了10次innerSink.write()每次4s
2. wrappedSink.write()执行了5s还未结束，发生了timeout，因为其中第一次sink.write()执行了5s还未结束。
 
### 和文档的冲突

我最开始的理解是,timeout()应该直接约束通过sink直接访问到的操作。然而wrappedSink却打破了这种规则。导致使用者从Sink接口拿Timeout时总是充满了
不知道如何正确使用的恐惧
```kotlin
// someSink()源码不能查看
val sink:Sink = someSink()

// 这个timeout约束的范围是什么呢？恐惧出现！
val timeout = sink.timeout()
```

### 为什么Okio这么实现



## source

```kotlin
  /**
 * Returns a new source that delegates to [source], using this to implement timeouts. This works
 * best if [timedOut] is overridden to interrupt [source]'s current operation.
 */
fun source(source: Source): Source {
    return object : Source {
        override fun read(sink: Buffer, byteCount: Long): Long {
            return withTimeout { source.read(sink, byteCount) }
        }

        override fun close() {
            withTimeout { source.close() }
        }

        override fun timeout() = this@AsyncTimeout

        override fun toString() = "AsyncTimeout.source($source)"
    }
}
```

这里的read()为什么没有拆开？(OKIO_TODO)



## System.nanoTime() 溢出减法妙用环绕

OKIO_TODO:这里没弄明白。



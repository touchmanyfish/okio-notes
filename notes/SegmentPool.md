# SegmentPool

SegmentPool用于存储Segment,可以对其执行take()和recycle(segment)操作.执行操作时需要先通过firstRef()进行分桶。
如果不同线程在同一个桶中发生了竞争，没有占有LOCK的线程会:

* 如果执行的是take()操作，则直接返回一个新创建的segment，不从桶中取
* 如果执行的是recycle(segment)操作,则跳过放入桶操作，直接返回

## 分桶算法

```kotlin
// SegmentPool.kt

private val HASH_BUCKET_COUNT =
   Integer.highestOneBit(Runtime.getRuntime().availableProcessors() * 2 - 1)

private fun firstRef(): AtomicReference<Segment?> {
    val hashBucket = (Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)).toInt()
    return hashBuckets[hashBucket]
}
```

### 桶数HASH_BUCKET_COUNT

HASH_BUCKET_COUNT为桶的数目，值为小于(core数 x 2 - 1)的最大的2^n。例如：
```text
core:8
8 x 2 - 1 = 15，对应二进制为:1111
highestOneBit(1111) -> 1000 -> 2^3 -> 8
此时HASH_BUCKET_COUNT == 8
```

### hashBucket

根据前面的分析，HASH_BUCKET_COUNT为2^n,它的二进制表示为:
```text
1+n个0
```
HASH_BUCKET_COUNT - 1L的二进制形式为:
```text
n个1
```
所以(Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)实际上是在截取Thread.currentThread().id二进制形式的后n位作为桶的标识hashBucket。

根据重复排列公式：
```text
如果有 n 种位置，每个位置都有 k 种选择，且允许重复选择,那么能表达的编码数量 = k ^ n
```
后n位二进制可表达的数 = 2 ^ n,刚好就是桶数HASH_BUCKET_COUNT。范围显然是[0,n个1]。

结合[使用位运算优化对2的n次方取模](使用位运算优化对2的n次方取模.md)分析可知道:
```
(Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)
V
thread.id and (2^n -1) == thread.id % 2^n
```

gemini说位运算只需要一个时钟周期；而%运算需要几十个。所以使用位运算实现更快。

### 随机性

由于截取时使用的二进制其中包含n个1，所以2 ^ n个桶都有机会被选中。thread.id的概率决定了选中桶的概率。

### 碰撞

由于分桶算法等同于thread.id % 2^n，显然，当thread.id相差2^n时，就会发生碰撞。

### 使用thread.id作为hash的trade-off

优点:

* 计算简单，分桶速度快
* 让同一个线程每次都分到同一个桶，如果cpu有软亲和特性，当下次在同一个core上访问同一个地址时，可以直接从core当缓存中读取，访问速度更快

缺点：

如果thread.id分配有问题，可能会导致碰撞

### 分桶算法总结

**步骤:**

1. 计算“所有小于(core * 2 -1)值的2^n中最大的那个”
2. 截取thread.id二进制后n位作为桶的标识hashBucket
3. 获取桶hashBuckets[hashBucket]

## 分桶优势

分桶的优势有如下几点：

A. 可以缓解LOCK锁竞争，减少getAndSet(LOCK)方法带来的微乎其微的执行卡顿

B. 可以避免只有单一桶时cpu1修改值导致cpu2的同一地址的值缓存失效的问题

C. cpu亲和性：cpu1后续对相同值的操作可以直接读缓存，而不用重新加载

### 优势A

根据firstRef()方法的实现可以看出，不同的的thread可能会进入不同的桶。这显然缓解了LOCK锁的竞争。

当不同线程落在同一个桶中时，它们会调用getAndSet(LOCK)来试图占有LOCK锁。此时先占有LOCK锁的thread执行后续操作，未占有LOCK锁的thread则
跳过对桶的操作(直接返回新创建的Segment或者不将旧的Segment放入桶中)。虽然getAndSet(LOCK)虽然会非常短暂的暂停一下线程执行，但是它并不会改变执行线程的RUNNING状态，所以当不同线程落在同
一个桶中时，不会发生线程级别的阻塞。

当两个 CPU 核心同时执行 getAndSet 修改同一个内存地址时，底层硬件会通过 MESI 协议 或 总线锁 强行让其中一个核心先写，另一个核心后写。过程是纳秒级别的(gemini)。

所以分桶实现实质缓解的就是：缓解了CPU 核心在总线锁/原子指令上的排队等待。

## 优势B

在多核 CPU 中，每个核心（Core）都有自己的 L1 Cache。为了保证所有核心看到的数据是一致的，硬件实现了一个协议（通常是 MESI 协议）。

* **独占（Exclusive**）：只有我这有这个数据，我可以随便改。
* **共享（Shared）**：大家都有，如果要改，必须先通知别人。
* **无效（Invalid）**：别人改了数据，我手里的这块内存缓存失效了，必须重新从主存拉取。

**单一桶场景:**

假设 SegmentPool 只有一个全局唯一的 head 指针，存放在内存地址 0xAAA.

**物理过程：**

1. Core 1 想要执行 take()。它把地址 0xAAA 的数据加载到自己的 L1 缓存，状态为 Shared。
2. Core 2 同时也要执行 take()。它也将 0xAAA 加载到自己的 L1，状态也是 Shared。
3. 核心冲突爆发：
   * Core 1 执行 getAndSet，准备修改 0xAAA。它必须向总线发出信号：“我要改这个地址，请其他核心把你们那里的 0xAAA 设为 Invalid！”
   * Core 2 的 L1 缓存被迫失效。
   * Core 2 还没来得及操作，发现缓存空了，只好重新通过内存总线去拉取最新的 0xAAA。
   * 此时如果 Core 3、Core 4 也在抢，内存总线就会充斥着大量的“失效通知”和“重新读取请求”。

**后果：** 由于大家都在争抢同一个物理地址 0xAAA，CPU 的时间都花在了核心间的“吵架”（通信）上，真正执行逻辑的效率极低。这就是缓存同步风暴。

**多桶场景:**

假设我们有 2 个桶：Bucket[0] 在地址 0x111，Bucket[1] 在地址 0x222。

**物理过程：**
1. Thread A 运行在 Core 1，根据 ID 计算，它去操作 Bucket[0]（地址 0x111）。
2. Thread B 运行在 Core 2，根据 ID 计算，它去操作 Bucket[1]（地址 0x222）。
3. 互不干扰的并行：
   * Core 1 将 0x111 加载到 L1。因为没有其他人读写这个地址，它的状态很快变为 Exclusive。Core 1 可以直接在 L1 里完成修改，不需要发总线信号。
   * Core 2 同样将 0x222 加载到 L1，它也处于 Exclusive 状态，独立完成修改。

**后果：** Core 1 和 Core 2 像是在两个平行的宇宙中工作。它们不需要通过总线互相通知，不需要撤销对方的缓存。这种状态下，多核 CPU 的性能才真正得到了 1+1=2 的线性提升。

### 优势C

线程在恢复执行以后，执行的core可能和上次不同，但是cpu调度一般更倾向于在同一个core上执行（软亲和）。

**优势分析：**

1. 当 ThreadA 在 Core 1 上运行，使用SegmentPool时假设被分到 hashBuckets[0] 桶中，hashBuckets[0] 的数据会被加载到 Core 1 的 L1/L2 缓存中。
2. 当ThreadA再次使用SegmentPool时，由于cpu当软亲和，ThreadA还会在Core1上执行；由于分桶算法(Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)),ThreadA每次都会进入同一个桶hashBuckets[0]；
这时Core1可以直接读取上一次的缓存，速度提升10-100倍。

## 使用(Thread.currentThread().id and mask)来选桶的分析


## hashBuckets[hashBucket]线程安全分析

```kotlin
// SegmentPool.kt
internal actual object SegmentPool {
    //..

    private val hashBuckets: Array<AtomicReference<Segment?>> = Array(HASH_BUCKET_COUNT) {
        AtomicReference<Segment?>() // null value implies an empty bucket
    }

    private fun firstRef(): AtomicReference<Segment?> {
        // Get a value in [0..HASH_BUCKET_COUNT) based on the current thread.
        val hashBucket = (Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)).toInt()
        return hashBuckets[hashBucket]
    }
// ...
}
```

* SegmentPool是object类，这保证了在这个单例初始化阶段是hashBuckets线程安全了。
* "val"关键字保证了任何时候，不同线程都能看到同一个hashBuckets值。
* 虽然Array是可变类型，但是在线程安全的初始化以后，不会进行修改，所以进行访问值的操作(array[index])是安全的
* AtomicReference类保证了当访问数组元素上的方法/字段时的线程安全

## limit

每个桶使用用环状链表保存segment。segment.limit的值为从当前segment开始到尾部的segment为止的segment.data.size总和。所以可以通过查看链表
上第一个segment的limit来获取这条链标上所有segment.data.size的总和。

## 总结

SegmentPool里线程级别的阻塞只会发生在SegmentPool对应实例初始化阶段，其他阶段不会发生任何线程级别的阻塞。

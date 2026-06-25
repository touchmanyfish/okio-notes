# Sink

[doc](https://square.github.io/okio/3.x/okio/okio/okio/-sink/index.html)

```kotlin
actual interface Sink : Closeable, Flushable {
  @Throws(IOException::class)
  actual fun write(source: Buffer, byteCount: Long)

  @Throws(IOException::class)
  actual override fun flush()

  actual fun timeout(): Timeout

  @Throws(IOException::class)
  actual override fun close()
}
```

## write()

调用write()以后，数据将被添加进(appends)sink。Sink接口没有定义此时数据所在的位置:

* 可能在Sink中
* 也可能在Sink的final destnation中

这取决于Sink的实现。

## flush()

```kotlin
// Sink#flush()官方文档
Pushes all buffered bytes to their final destination.
```

### "flush操作"的定义

1. final destination为pipline上距离当前位置最近的下一个数据目的地
2. 调用当前位置的flush操作,会把从pipline当前位置开始到往后的所有数据都push到它们各自对应的final destination

### 理解

假设有如下pipline：

```text
data ->  sink1 -> sink2 -> other place
```

**final destination：**

* sink1 的 **final destination** 为sink2
* sink2 的 **final destination** 为 ohter place。

**调用sink的flush操作:**

1. 调用sink1的flush操作,在这里是flush()方法，
2. sink1会将数据全部写入sink2，然后调用sink2的flush操作，这里是flush()。
3. sink2会将数据全部写入other place，然后调用other place的flush操作。
4. other place会将全部数据写入下一个位置，然后调用下一个位置的flush操作。

flush操作实际干了2件事：1.将当前位置数据写入下一个位置 2.通知下一个位置，让下一个位置继续执行flush操作；

```text
data ->  sink1(data1) -> sink2(data2) -> other place(datao)
V
调用sink1.flush()
V
data ->  sink1() -> sink2(data1+data2) -> other place(datao)
V
data ->  sink1() -> sink2(data1+data2) -> other place(datao)
V
data ->  sink1() -> sink2() -> other place(data1+data2+datao)
V
....
```

注意，只要满足语义的操作就可以叫做flush操作。只是Sink#flush()的定义满足语义而已。

## close()


### “close操作”的定义

1. 执行“flush操作”
2. 释放然后释放所有相关的资源

在Okio的实现中：

* 1./2.操作也是可能的
* 某一个位置的“close操作”中 **可能会** 调用下一个位置的“close操作”；

```kotlin
//ChunkedSink.kt

@Synchronized
override fun close() {
    if (closed) return
    closed = true
    sink.writeUtf8("0\r\n\r\n")
    detachTimeout(timeout)
    state = STATE_READ_RESPONSE_HEADERS
}
```

### 理解









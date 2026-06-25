# inernal_Buffer#commonWrite(source,byteCount)

代码位于

```text
package okio.internal
Buffer.kt
```

这个方法用于将source中byteCount数量多数据写入当前Buffer

## 流程解析

### 处理流程

byteCount跨过了source中的一些segment。

完整跨过的块直接从source取下来插入当前链表的尾部。

部分跨过的segment，如果当前尾部是owner且空间足够，则直接将跨过的数据写入(writeTo())尾部；否则就将跨过的数据split()
出来，然后插入尾部。

上述两种情况中，在将segment插入尾部时如果head不为空则尝试进行compact().

## 使用的优化

## 零拷贝

当byteCount跨了至少一个segment时，被完整跨过的segment通过指针移动来搬运，速度很快。

## split()

当移动segment中被byteCount部分跨过的数据时，使用split()可以加快大数据的搬运。因为如果要拆的数据至少有1kb时，split()内部使用shardCopy(),
它会创建一个新的shared Segment来指向同一个data中被拆出来的部分，这个过程并不发送拷贝操作。

splite()出来的shared Segment会被插入到尾部。整个过程涉及指针移动+Segment对象创建，这比直接考这坨大数据快。

缺点参考[Segment#23split(byteCount)](Segment%23split(byteCount).md)

## compact()碎片整理

当segment插入尾部时，可以进行碎片整理

```text
当前buffer：
segment1(10,owner:true)

split/直接搬运:
segment2(1,shared:false)

插入尾部以后变成
segment1(10) -> segment2(1)

compact()方法发现segment1中空间足够且为owner，于是就将segment2合并进segment1，然后将segment放入SegmentPool。

当前buffer：
segment1(12)

于是，segment2+data(8k)内存得到回收。
```

### 没有使用compact()的场景

```kotlin
//commonWrite(source,byteCount)

// Our existing segments are sufficient. Move bytes from source's head to our tail.
source.head!!.writeTo(tail, byteCount.toInt())
source.size -= byteCount
size += byteCount
return
```
这里可能出现如下情况:
```text
当前buffer:seg(10)
source:seg(7000) -> seg(1000)

//writeTo以后

当前buffer:seg(10+6999)
source:seg(1) -> seg(1000)
```
注意到source的第一个segament只存了1byte的数据，为什么这里不执行类似compact()的操作呢？

gemini说这里通过浪费一点内存来避免执行拷贝操作。

## 总结：Okio 的博弈论

**性能优先级：** 移动引用 > 逻辑拆分 (Split) > 物理拷贝 (ArrayCopy)。

**内存策略：** 宁可承受短期的内存膨胀（Memory Bloat），也要换取高吞吐量的线性写入性能。

**碎片治理：** 依赖 SegmentPool 的循环利用和节点转移时的 compact 来维持长期的平衡。
# Segment#compact()

## 执行时机
```kotlin
//package okio.internal
//Buffer.kt

internal inline fun Buffer.commonWrite(source: Buffer, byteCount: Long) {
   // ..
    if (head == null) {
        head = segmentToMove
        segmentToMove.prev = segmentToMove
        segmentToMove.next = segmentToMove.prev
    } else {
        var tail = head!!.prev
        tail = tail!!.push(segmentToMove)
        // 这里
        tail.compact()
    }
   // ..
}
```

当将segment插入尾部时，会尝试将它和prev进行合并。

## 实现原理
```kotlin
/**
 * Call this when the tail and its predecessor may both be less than half full. This will copy
 * data so that segments can be recycled.
 */
fun compact() {
    check(prev !== this) { "cannot compact" }
    if (!prev!!.owner) return // Cannot compact: prev isn't writable.
    val byteCount = limit - pos
    val availableByteCount = SIZE - prev!!.limit + if (prev!!.shared) 0 else prev!!.pos
    if (byteCount > availableByteCount) return // Cannot compact: not enough writable space.
    writeTo(prev!!, byteCount)
    pop()
    SegmentPool.recycle(this)
}
```

### segment分类

**segment(owner:false):**

segment(owner:false)不具有修改权限，所以不能将其他segment合并进去。

**segment(owner:true,shared:false,pos,limit):**

```text
       p   l
       V   V
[0,0,0,1,1,1,0,0]

剩余空间 = Segment.Size - (l - p) = Segment.Size - l + p

如果从l开始的空间够，就直接从l开始写入；
如果不够则需要先将数据平移到最左边，然后才能从l开始写入数据。

 p   l
 V   V
[1,1,1,0,0,0,0,0]
```
可以直接将数据写入(writeTo())剩余空间

**segment(owner:true,shared:true,pos,limit):**
```text
segment(owner:true,shared:true,pos,limit)
       p   l
       V   V
[0,0,0,1,1,1,0,0]

只能从l开始写入数据
剩余空间 = Segment.Size - l
```

### compact()流程

当前segment刚刚被插入到了链表尾部：
1. 如果prev不能写入，跳过compact
2. 如果prev空间不够，跳过compact
3. 使用writeTo()将当前segment所有数据写入prev
4. 从链上取下当前segment并放入SegmentPool中
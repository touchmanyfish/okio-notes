# Segment#writeTo(sink,byteCount)

Moves byteCount bytes from this segment to sink.

## 要求

shared/非shared sink都必须要是owner才能对其进行写入！

## 写入

为了方便讨论:

* Segment.SIZE假设为8
* 所有包含数据的bits都为1，不包含数据的bits为0

### 写入非shared sink

**写入步骤：**
1. 检查剩余空间是否足够写入byteCount数据
2. 检查当前数据右边空位能否写入byteCount数据;如果不够则，将当前数据移动到最左边侧。
3. 将byteCount数据数据拷贝到当前数据右边

**例子：**
```text
// 将source中 2 bytes写入非shared sink中

source:
   p     l
   V     V
[0,1,1,1,0,0,0,0]

// sink中剩余空间为3 bytes，可以写入
// 但是右边只有1 byte空间
sink:
     p       l
     V       V
[0,0,1,1,1,1,0,0]

// 所以需要先将其中数据移动到最左边，把右边位置腾出来
// 现在右边有3 bytes的空间，足够进行接下来的拷贝
sink:
 p       l
 V       V
[1,1,1,1,0,0,0,0]

// 拷贝完成
// 根据结果调整source/sink的pos/limit值

source:
       p l
       V V
[0,1,1,1,0,0,0,0]

sink:
 p           l
 V           V
[1,1,1,1,1,1,0,0]
```

根据例子可以发现，拷贝结束以后，source.p的左边有2 bytes的旧数据。不用担心，正确的数据位于[p,l),这些旧的数据后续可能会被正确的数据覆盖掉的。

### 写入shared sink

1. 检查剩余空间是否足够写入byteCount数据
2. 检查当前数据右边空位能否写入byteCount数据
3. 将byteCount数据数据拷贝到当前数据右边

shared sink只允许从limit开始写入数据，当右边空间不够时，没法进行写入。
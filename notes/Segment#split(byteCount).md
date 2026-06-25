# Segment#split(byteCount)

split用于从当前Segment中拆byteCount出去，其中有2种拆法：

1. 创建SegmentB，它和原本的SegmentA共享data
2. 创建SegmentB，将需要拆出来的部分从SegmentA中拷贝到SegmentB的data中

## 不同拆法优缺点分析

### 1.共享拆法(sharedCopy())

从segmentA sharedCopy() 出 segmentB

```text
segmentA(owner:true,shared:false,datax)

// sharedCopy以后
// 两个segment都变成了shared了，它们都指向同一个datax

segmentA(owner:true,shared:true,datax)
segmentB(owner:false,shared:true,datax)
```

**shared segment相关内存回收:**

```text
//假设Buffer中segment链表结构如下
//segment1/segment2共享同一个dataX

segment1(shared,dataX) -> segment2(shared,dataX)
```

调用segment2.pop()将segment2从链表中移除；由于SegmentPool只接受非shared的segment对象，
所以不不能将它放入SegmentPool。不过由于此时它不被任何人持有，所以这个对象后续可以被jvm垃圾回收。
不过注意，这里只垃圾回收segment1对象，不包含它指向的data。

```text
segment1(shared,dataX)
```

类似的，调用segment1.pop()，然后segment1对象等待被jvm垃圾回收。另外由于此时data只被segment1对象持有，所以data也会被垃圾回收。

**回收流程总结:**

当一个data被多个segment共享时，可以通过调用segment.pop()的方式来让segment对象被垃圾回收；当指向data当最后一个segment.pop()
调用以后，data
也会被垃圾回收机制回收。

gemini说每当segment.split()使用sharedCopy()时，就是在延长底部data生命周期。这种表达很有意思。

**优点:**

因为是共享内存，所以拷贝极快

**缺点:**

A:sharedCopy()会产生shared segment，它们享受不到compact()方法的合并优化。这会导致链表变长。

B:如果链表中有很多shared segment(被分出来的owner:false)，它们只持有很小的数据，例如segment(1byte,shared,data(8k)),如果这些segment被长期持有，
就会造成大量内存浪费。

C:如果想修改segment中的数据，必须先创建新的segment(非shared)对象，然后将数据拷贝进去才行。

### 2.直接拷贝拆法

从segmentA 直接拷贝 出 segmentB

```text
segmentA(owner:true,shared:false,datax)

// sharedCopy以后
// 两个segment都是非shared都，它们都指向不同的data

segmentA(owner:true,shared:false,datax)
segmentB(owner:true,shared:false,datay)
```

**优点：**

A：调用segment.pop()以后就可以将这个segment以及对于data放入SegmentPool中进行回收。

B: 对其修改可以立即进行。

**缺点:**

Segment越大拷贝越慢(Segment最大8192)

## trade-off:SHARE_MINIMUM()

我们需要一个SHARE_MINIMUM:

* 当size < SHARE_MINIMUM使用“直接拷贝”
* 当size > SHARE_MINIMUM使用“sharedCopy”

### 维度1(gemini):

在Okhttp的场景中，数据通常分为2大类:

**1kb以内（小数据）：大概率是 Metadata（元数据）。**

* 比如 HTTP Header、Multipart 的边界字符串（Boundary）、JSON 中的 Key 名。
* 特性：这些数据被切分出来后，往往紧接着就要被解析、修改或拼接。比如拦截器可能会修改一个 Header。
* 结论：对这种修改频繁的的数据进行 **直接拷贝**，数据量不大，速度不会很慢；直接拷贝出的非shared segment还能享受到compact()
  的合并优化；
  当调用segment.pop()就可以立即将其放入SegmentPool中进行回收。

**1kb以上（大数据）：大概率是 Payload（实体载荷）。**

* 比如图片、视频流、文件块。
* 特性：这些数据就像传送带上的货物，Okio 只负责“搬运”（从 Socket 搬到 File，或者从 File 搬到 Response）。程序很少会去修改一张图片中间的几个字节。
* 结论：这类数据使用shareCopy()，享受快速拷贝的优势；数据量大，内存浪费小；同时由于是只读的，所以不会执行unsharedCopy()
  操作导致segment对象的分配。

所以1k可以作为一个copy方式选择的分界线：

1. 数据小于1kb时使用直接拷贝
2. 数据大于1kb时使用sharedCopy()

1024这个值让我们享受到了每种方式对优点同时避免了其对缺点。

### 维度2(gemini):

JVM 为了加速对象分配，使用了 TLAB (Thread Local Allocation Buffer)。

* 小对象分配快：1kb 以内的对象在 TLAB 中分配几乎是无锁的，非常快。
* 大对象分配慢：如果对象太大，会直接进入堆（Heap）分配，可能触发同步锁甚至 GC 检

将阈值设为 1kb，保证了即使触发了拷贝（创建了新 Segment），这个新对象的分配大概率也能在 TLAB 中完成，不会引发显著的系统性延迟。

类似的1kb，也可以作为一个copy方式选择的分界线。

### 选择1kb

还有很多其他的维度，选择1kb是因为这些维度重叠在1kb的位置(**OKIO_TODO**)。



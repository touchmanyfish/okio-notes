# Timeout

## timeout和deadline区别

**timeout:**

操作开始执行，经过timeout时间以后还未结束，表示超时

**deadline:**

操作开始执行，过了一会当前时刻到达deadline，操作还未结束，表示超时

## Timeout如何检测超时？

### throwIfReached()

Throws an InterruptedIOException if the deadline has been reached or if the current thread has been interrupted. 

### awaitSignal(condition)

循环调用检测

### waitUntilNotified(monitor)

循环调用检测

### intersectWith(other,block)



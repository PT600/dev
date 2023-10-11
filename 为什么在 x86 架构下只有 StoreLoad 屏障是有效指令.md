
##  x86 的 TSO(Total Store Order)[1] 模型
* 首先store buffers被设计成了FIFO的队列，如果线程需要读取内存里的变量，务必优先读取store buffers中的值，如果没有就去内存都
* MFENCE指令用于清空本地store buffer, 将数据写到内存
* 线程执行Lock前缀的指令集时，在释放锁前，会清空本地的store buffer, 和MFENCE逻辑一直。
* store buffer里面的数据，任何时候都有可能写回内存。

上面的 x86-TSO 模型，我们可以推测出 x86 架构下是不需要 StoreStore 屏障的，因为x86 的 store buffer 被设计成了 FIFO，纵然在同一个线程中执行多次写入 buffer 的操作，最终依旧是严格按照 FIFO 顺序 dequeue 并写回到内存里，自然而然，对于其他任何线程而言，所『看到』的该线程的变量写回顺序是和指令序列一致的，因此不会出现重排序。

进一步思考，我们知道**读屏障就是为了解决 invalidate queue 的引入导致数据不一致的问题**，x86-TSO 模型下是没有 invalidate queue 的，因此也不需要读屏障

虽然store buffer 是 FIFO，但整体架构本质依然是最终一致性而非线性一致性。这势必会出现在某个时间节点，不同处理器看到的变量不一致的情况。继续看下面的伪代码：

``` c
x,y=0;
// proc 0
void foo(){
  x=1;
  read y;
}
​
// proc 1
void bar(){
  y=1;
  read x;
}
```
如果遵循线性一致性，我们大可以枚举可能发生的情况，但无论怎么枚举，都不可能是 x=y=0，然而诡异之处在于 x86-TSO 模型下是允许 x=y=0这种情况存在的。
以 foo 方法为例，由于 y=1 的写入有可能还停留在 proc1 的 store buffer 中，foo 方法末尾读到的 y 可能是旧值，同理 bar 方法末尾也有可能读到 x 的旧值，那么读出来自然有可能是 x=y=0。

在 x86-TSO 里提到 MFENCE，这个指令用于强制清空本地 store buffer，并将数据刷到主内存，本质上就是 StoreLoad Barrier。

由于 x86 是遵循 TSO 的最终一致性模型，如若出现 data race 的情况还是需要考虑同步的问题，尤其是在 StoreLoad 的场景。而其余场景由于其 store buffer 的特殊性以及不存在 invalidate queue 的因素，可以不需要考虑重排序的问题，因此在 x86 平台下，除了 StoreLoad Barrier 以外，其余的 Barrier 均为空操作。

基于 CPU 缓存设计上的特点，带来了内存可见性的问题，进一步造成“看似”指令乱序的现象

## Store buffer
CPU修改Cache Line中的变量时，要通过总线发送Invalidate信息给远程CPU，等到远程CPU返回ACK，才能进行继续修改。如果远程的CPU比较繁忙，会带来更大的延迟。
在CPU和L1Cache之间引入Store buffer来对cache line的写操作进行优化。
* 写操作写入Store Buffer后就返回，同时向其它核心发出Invalidate消息。
* 等CPU的Invalidate ACK消息返回后再异步写进Cache Line
* 核心从Cache中读取前都要先扫描自己的Store buffers来确认是否存在目标行。有可能当前核心在这次操作之前曾经写入cache，但该数据还没有被刷入cache(之前的写操作还在 store buffer 中等待)。

[CPU缓存架构到内存屏障](https://blog.chongsheng.art/post/golang/cpu-cache-memory-barrier/)


## memory barrier
上面两小节中可已看出memory barrier的作用，上面提到的smp_mb()是一种full memory barrier，他会将store buffer和invalidate queue都flush一遍。但是就像上面例子中体现的那样有时候我们不用两个都flush，于是硬件设计者引入了read memory barrier和write memory barrier。
* read memory barrier会将invalidate queue flush。
* write memory barrier会将storebuffer flush。

[高并发编程–多处理器编程中的一致性问题(上)](https://zhuanlan.zhihu.com/p/48157076)


## refer
[JSR-133 Review](https://zhuanlan.zhihu.com/p/75509358)
[从 Java 内存模型看内部细节](https://zhuanlan.zhihu.com/p/71589870)
[为什么在 x86 架构下只有 StoreLoad 屏障是有效指令？](https://zhuanlan.zhihu.com/p/81555436)

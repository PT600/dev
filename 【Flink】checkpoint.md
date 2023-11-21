
## checkpoint
* Flink的检查点协调器（Checkpoint Coordinator）触发一次Checkpoint（Trigger Checkpoint），这个请求会发送给Source的各个子任务。
* 各Source算子子任务接收到这个Checkpoint请求之后，会将自己的状态写入到状态后端，生成一次快照，并且会向下游广播Checkpoint Barrier。
    Source算子做完快照后，还会给Checkpoint Coodinator发送一个确认，告知自己已经做完了相应的工作。这个确认中包括了一些元数据，其中就包括刚才备份到State Backend的状态句柄，或者说是指向状态的指针。至此，Source完成了一次Checkpoint。跟Watermark的传播一样，一个算子子任务要把Checkpoint Barrier发送给所连接的所有下游算子子任务。


    ![Checkpoint Barrier Alignment](https://img-blog.csdnimg.cn/img_convert/4db54b4004837378595777e6ebcd2b45.png)

如上图所示，对齐分为四步：

1. 算子子任务在某个输入通道中收到第一个ID为n的Checkpoint Barrier，但是其他输入通道中ID为n的Checkpoint Barrier还未到达，该算子子任务开始准备进行对齐。

2. 算子子任务将第一个输入通道的数据缓存下来，同时继续处理其他输入通道的数据，这个过程被称为对齐。

3. 第二个输入通道的Checkpoint Barrier抵达该算子子任务，该算子子任务执行快照，将状态写入State Backend，然后将ID为n的Checkpoint Barrier向下游所有输出通道广播。

4. 对于这个算子子任务，快照执行结束，继续处理各个通道中新流入数据，包括刚才缓存起来的数据。



# refer
* [Flink Aligned Checkpoint和Unaligned Checkpoint原理详解](https://blog.csdn.net/bigdatakenan/article/details/124261147)


## barrier对齐

* JobManager定时发送Barrier
* 算子需要收到所有依赖的上游的Barrier，开始checkpoint
* 精准一次：先到的数据需要缓存。
* 至少一次：先到的数据无需缓存。

## Unalignment Barrier
核心思想: **只要in-flight的数据也存到状态里，barrier就可以越过所有in-flight的数据继续往下游传递**，
也就是一个快照，下次恢复时，把in-flight的数据重新处理，就可以达到和对齐一样的状态。
当第一个Barrier到达输入线冲区时:
* 直接将barrier放到输出缓冲区末端句下游传递
* 标记第一个barrier越过的输入缓冲区和输出缓冲区的数据
* 标记其他barrier之前的所有数据
* 把标记数据和状态一起保存到checkpint中，从checkpoint恢复时这些数据也会一起恢复到对应位置
* 可以设置alignedCheckpointTimeout参数，优先对齐，当对齐超时后，使用非对齐。

## 为什么输出缓冲区也需要保存？

## barrier提前向后传，导致checkpoint也会提前，会保存更多额外的数据，产生较大的io压力，慎用之！！！



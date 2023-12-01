## stage的划分
在DAG调度的过程中，Stage阶段的划分是根据是否有shuffle过程，也就是存在ShuffleDependency宽依赖的时候，需要进行shuffle,这时候会将作业job划分成多个Stage；

**spark 划分 stage 的整体思路是：从后往前推，遇到宽依赖就断开，划分为一个 stage；遇到窄依赖就将这个 RDD 加入该 stage 中。**

## Task 的类型两种
在 spark 中，Task 的类型分为 2 种：ShuffleMapTask 和 ResultTask
简单来说，DAG 的最后一个阶段会为每个结果的 partition 生成一个 ResultTask，即每个 Stage里面的 Task 的数量是由该 Stage 中最后一个 RDD 的 Partition 的数量所决定的！而其余所有阶段都会生成 ShuffleMapTask；之所以称之为 ShuffleMapTask 是因为它需要将自己的计算结果通过 shuffle 到下一个 stage 中；



## 定义
Shuffle的本意是“洗牌”，在分布式计算环境中，它有两个阶段。一般来说，前一个阶段叫做“Map阶段”，后一个阶段叫做“Reduce阶段”。当然，也有人把它们叫做Shuffle Write阶段和Shuffle Read阶段。

## Map阶段
Map阶段每一个Task的执行流程都是一样的，每个Task最终都会生成一个数据文件和一个索引文件。
中间文件的数量与Map阶段的并行度保持一致。换句话说，有多少个Task，Map阶段就会生产相应数量的数据文件和索引文件。

Map Task会把每条数据记录和它的目标分区，放到一个特殊的数据结构里，这个数据结构叫做“PartitionedPairBuffer”
最理想的情况当然是PartitionedPairBuffer足够大，大到足以容纳Map Task所需处理的所有数据。不过，现实总是很骨感，每个Task分到的内存空间是有限的，PartitionedPairBuffer自然也不能保证能容纳分区中的所有数据。因此，Spark需要一种计算机制，来保障在数据总量超出可用内存的情况下，依然能够完成计算。这种机制就是：**排序、溢出、归并**。 **有点类似LSM中的Level0。**

虽然Map阶段的计算步骤很多，但其中最主要的环节可以归结为4步：

1. 对于分片中的数据记录，逐一计算其目标分区，并将其填充到PartitionedPairBuffer；- 
2. PartitionedPairBuffer填满后，如果分片中还有未处理的数据记录，就对Buffer中的数据记录按（目标分区ID，Key）进行排序，将所有数据溢出到临时文件，同时清空缓存；- 
3. 重复步骤1、2，直到分片中所有的数据记录都被处理；- 
4. 对所有临时文件和PartitionedPairBuffer归并排序，最终生成**数据文件和索引文件**。

## Reduce阶段
### reduceByKey的Map阶段计算
    相比groupByKey有何不同。就Map端的计算步骤来说，reduceByKey与刚刚讲的groupByKey一样，都是先填充内存数据结构，然后排序溢出，最后归并排序。
    区别在于，在计算的过程中，reduceByKey采用一种叫做PartitionedAppendOnlyMap的数据结构来填充数据记录。这个数据结构是一种Map，而Map的Value值是可累加、可更新的。
    因此，PartitionedAppendOnlyMap非常适合聚合类的计算场景，如计数、求和、均值计算、极值计算等等。

### Reduce阶段是如何进行数据分发的？
每个Map Task都会生成如上图所示的中间文件，文件中的分区数与Reduce阶段的并行度一致
换句话说，每个Map Task生成的数据文件，都包含所有Reduce Task所需的部分数据
因此，任何一个Reduce Task要想完成计算，必须先从所有Map Task的中间文件里去拉取属于自己的那部分数据。
**索引文件正是用于帮助判定哪部分数据属于哪个Reduce Task。**
Reduce Task通过网络拉取中间文件的过程，实际上就是不同Stages之间数据分发的过程。


## refer
[11 为什么说Shuffle是一时无两的性能杀手？](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Spark%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98/11%20%E4%B8%BA%E4%BB%80%E4%B9%88%E8%AF%B4Shuffle%E6%98%AF%E4%B8%80%E6%97%B6%E6%97%A0%E4%B8%A4%E7%9A%84%E6%80%A7%E8%83%BD%E6%9D%80%E6%89%8B%EF%BC%9F.md)
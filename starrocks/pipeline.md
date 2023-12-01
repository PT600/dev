
## MPP调度基本概念
* 物理执行计划(ExecPlan)
    物理执行计划是 FE 生成的，由物理算子构成的执行树；SQL 经过 parse、anlyze、rewrite、optimize 等阶段处理，最终生成物理执行计划。

* PlanFragment
    PlanFragment 是物理执行计划的部分。只有当执行计划被 FE 拆分成若干个 PlanFragment 后，才能多机并行执行。PlanFragment 同样由物理算子构成，另外还包含 DataSink，上游的 PlanFragment 通过 DataSink 向下游 PlanFragment的 Exchange 算子发送数据。
    **类似spark的Stage**

* FragmentInstance
    Fragment Instance 是 PlanFragment 的一个执行实例，StarRocks 的 table 经过分区分桶被拆分成若干 tablet，每个 tablet 以多副本的形式存储在计算节点上，可以将 PlanFragment 的实例化成多个 Fragment Instance 处理分布在不同机器上的 tablet，从而实现数据并行计算。
    FE 确定 Fragment Instance 的数量和执行 Fragment Instance 的目标 BE，然后 FE 向 BE投递 Fragment Instance
    **类似spark的task**

* Pipeline
    在 Pipeline 执行引擎中，BE 上的 PipelineBuilder 会把 PlanFragment 进一步拆分成若干 Pipeline，每个 Pipeline 会根据 Pipeline 并行度参数而被实例化成一组 PipelineDriver， PipelineDriver 是 Pipeline 实例，也是 Pipeline 执行引擎所能调度的基本任务。
    **更加细粒度的调度单元coroutine**

** 物理算子(ExecNode)
    物理算子是构成物理执行计划 PlanFragment 的基本元素，例如 OlapScanNode，HashJoinNode 等等。

## FE负责MPP调度
我们以下面的简单 SQL 为例，进一步说明上述概念：
```sql
select A.c0, B.c1  from A, B  where A.c0  = B.c0
```
* 第一步：FE 产生物理计划并且拆分 PlanFragment，如下图所示，物理计划被拆分成三个 PlanFragment，其中 Fragment 1 包含 HashJoinNode，Fragment 0 为 HashJoinNode 的右孩子。


[技术内幕 | StarRocks Pipeline 执行框架（上）](https://mp.weixin.qq.com/s/fgNwk2DyijKM4D9L77LQNA)
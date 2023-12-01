
## 基本概念
* FE: 负责查询解析，查询优化，查询调度和元数据管理
* BE： 负责查询执行和数据存储

## 从 SQL 文本到执行计划
1. SQL Parse: 将SQL文本转换成一个AST
2. SQL Analyzer: 基于AST进行语法和语义分析
3. SQL Logic Plan：将AST转化为逻辑计划
4. SQL Optimize：基于关系代数、统计信息、Cost 模型，对逻辑计划进行重写、转换，选择出 Cost “最低” 的物理执行计划
5. 生成 Plan Fragment：将 Optimizer 选择的物理执行计划转换为 BE 可以直接执行的 Plan Fragment


## SQL Parse
![SQL Parse](https://pic4.zhimg.com/80/v2-0b5075ea8605ce4185474c5bab290b8f_720w.webp)

Query Parse 的输入是 SQL 的 String 字符串，Query Parse 的输出是 Abstract Syntax Tree，**每个节点都是一个 ParseNode** 。

一个查询 SQL Parse 后生成一个 QueryStmt， 由 SelectList, FromClause, wherePredicate, GroupByClause, havingPredicate, OrderByElement, LimitElement 等组成，**基本和 SQL 文本一一对应。**

StarRocks 目前使用的 Parser 是 ANTLR4，语法规则定义的文件可在 GitHub 搜索 StarRocks g4 获取。

## SQL Analyzer
StarRocks 获取到 AST 后，接着会进行语法分析和语义分析，完成下面的工作：
1. 检查并绑定 Database, Table, Column 等元信息
2. SQL 的合法性检查：Where 中不能有 Grouping 操作， HLL 和 Bitmap 列不能 Sum 等
3. Table 和 Column 的别名处理 
4. 函数参数的合法性检测: Sum的参数类型必须是数值类型，Lead 和 Lag 窗口函数第 2 和第 3 个参数必须常量等
5. 类型检查和类型转换：BIGINT 和 DECIMAL 比较，BIGINT 类型需要 Cast 成 DECIMAL

SQL Analyze 的结果是一个有层级结构的 Relation，如图 2 所示，比如一个 From 子句对应一个 TableRelation， 一个子查询对应一个 SubqueryRelation。

## SQL Logical Plan
StarRocks 会将 Relations 转化成一颗 Logical Plan Tree， 如下图所示，可以简单理解为每个集合操作都会对应一个 Logical Node。

![Logic Plan](https://pic2.zhimg.com/80/v2-87a42d178d1ae9ffa4f7ea5c8596c685_720w.webp)


## SQL Optimize

![SQL Optimize](https://pic3.zhimg.com/80/v2-2e7918cfbedb29120b275f05b244dc36_720w.webp)

StarRocks Optimizer 的输入是一棵逻辑计划树，输出是一棵 Cost “最低” 的分布式物理计划树。

一般 SQL 越复杂，Join 的表越多，数据量越大，Optimizer 的意义就越大，因为不同执行方式的性能差别可能有成百上千倍。

StarRocks 优化器完全自研，主要基于 Cascades 和 ORCA 论文实现，并结合 StarRocks 执行器和调度器进行了深度定制，优化和创新。

它完整支持了 TPC-DS 99 条 SQL，实现了公共表达式复用，相关子查询重写，Lateral Join， CTE 复用，Join Rorder，Join 分布式执行策略选择，Global Runtime Filter 下推，低基数字典优化等重要功能和优化。

### Logical Plan Rewrite
![Logical Plan Rewrite](https://pic2.zhimg.com/80/v2-9d96d2df7b13c7ef7e29b166401e2de9_720w.webp)
在正式进入 CBO 之前，StarRocks 会首先进行一系列 Logical Plan 的 Rewrite，Rewrite 阶段的 Rule 我们认为都会生成更优的 Logical Plan，主要的 Rewrite Rule 有下面这些：
* 各种表达式的重写和化简
* 列裁剪
* 谓词下推
* Limit Merge, Limit 下推
* 聚合 Merge
* 等价谓词推导（常量传播）
* Outer Join 转 Inner Join
* 常量折叠
* 公共表达式复用
* 子查询重写
* Lateral Join 化简
* 分区分桶裁剪
* Empty Node 优化
* Empty Union, Intersect, Except 裁剪
* Intersect Reorder
* Count Distinct 相关聚合函数重写

### CBO Transform
我们在 Logical Plan Rewrite 完成后，正式基于 Columbia 论文进行 CBO 优化，主要包括下面的优化：
* 多阶段聚合优化：普通聚合（count, sum, max, min 等）会拆分成两阶段，单个 Count Distinct 查询会拆分成三阶段或是四阶段。
* Join 左右表调整：**StarRocks 始终用右表构建 Hash 表**，所以右表应该是小表，StarRocks 可以基于 cost 自动调整左右表顺序，也会自动把 Left Join 转 Right Join。
* Join 多表 Reorder：多表 Join 如何选择出正确的 Join 顺序，是 CBO 优化器的核心。当 Join 表的数量小于等于 5 时，StarRocks 会基于 Join 交换律和结合律进行 Join Reorder，大于 5 时，StarRocks 会基于贪心算法和动态规划进行 Join Reorder。
* Join 分布式执行选择：StarRocks 支持的分布式 Join 方式有 Broadcast、Shuffle、单边 Shuffle、Colocate、Replicated。StarRocks 会基于 Cost 估算和 Property Enforce 机制选择出 “最佳” 的 Join 分布式执行方式。
* Push Down Aggregate to Join
* 物化视图选择与重写

### 统计信息 和 Cost 估计
CBO 优化器好坏的关键之一是 Cost 估计是否准确，而 Cost 估计是否准确的关键点之一是统计信息是否收集及时准确。
StarRocks 目前**支持表级别和列级别的统计信息**，支持自动收集和手动收集两种方式。无论自动还是手动，**都支持全量和抽样收集两种方式**。

有了统计信息之后， StarRocks 就会基于统计信息进行 Cost 估算。StarRocks 估算 Cost 时会考虑 CPU、内存、网络、IO 等资源因子，每个资源因子会有不同的权重，每个执行算子的 Cost 计算公式都不太一样。

当你使用 StarRocks 发现 Join 左右表不合理、Join 分布式执行策略不合理时，可以参考 StarRocks CBO 使用文档收集统计信息。

## 生成 Plan fragment

![Plan Fragment](https://pic3.zhimg.com/80/v2-e62a5dbc1b5b06c19fe687a59752f976_720w.webp)
StarRocks Optimizer 的输出是一棵分布式物理执行计划树，但并不能直接被 BE 节点执行，所以需要转换成 BE 可以直接执行的 PlanFragment。转换过程基本是个一一映射的过程。

## MPP 多机并行执行
StarRocks 会将一个查询在逻辑上切分为多个 Query Fragment（查询片段），每个 Query Fragment 可以有一个或者多个 Fragment 执行实例，每个 Fragment 执行实例会被调度到集群某个 BE 上执行。一个 Fragment 可以包括一个或者多个 Operator（执行算子），图中的 Fragment 包括了Scan、Filter、Aggregate。每个 Fragment 可以有不同的并行度。

## Pipeline 单机并行执行
StarRocks 在 Fragment 和 Operator 之间引入了 Pipeline 的概念，一个 Pipeline 内的数据没有到达终点前不需要 Materialize，遇到需要 Materialize 的算子（Agg, Sort, Join)，则需要拆分出一个新的 Pipeline，**所以 1 个 Fragment 会对应多个 Pipeline， 一个 Pipeline 由多个 Operator 组成**。

* 类比spark:
    Fragment => Stage
    Pipeline => task

[StarRocks 技术内幕：查询原理浅析 - 知乎](https://zhuanlan.zhihu.com/p/506063323)
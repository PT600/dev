StarRocks 目前 Join 的算法主要是一个 Hash Join，默认使用右表去构建 Hash 表，在这个前提下，我们总结了五个优化方向：

1. 不同 Join 类型的算子，性能是不同的，尽可能使用性能高的 Join 类型，避免使用性能差的 Join 类型。根据 Join 输出的数据量，大致上的性能排序：Semi-Join/Anti-Join > Inner Join > Outer Join > Full Outer Join > Cross Join。

2. Hash Join 实现时，使用小表做 Hash 表，远比用一个大表做 Hash 表高效。

3. 多表 Join 时，优先执行选择度高的 Join，能大幅减少后续 Join 的开销。

4. 尽可能减少参与 Join 的数据量。

5. 尽可能减少分布式 Join 产生的网络成本。


## Join 逻辑优化
### 转换规则一：Cross Join 转换为 Inner Join
当 Cross Join 满足某个约束时，可以将 Cross Join 转为 Inner Join。该约束为：**Join 上至少存在一个表示连接关系的谓词**。例如：
```sql
-- 转换前
Select *  From t1, t2 Where t1.v1 = t2.v1; 

-- 转换后, Where t1.v1 = t2.v1是连接关系谓词
Select *  From t1 Inner Join t2 On t1.v1 = t2.v1;
```

### 转换规则二：Outer Join 转换为 Inner Join
当满足以下约束时，可以将 Outer Join 转为 Inner Join：
1. Left / Right Outer Join 上存在一个 Right / Left 表的相关谓词；
2. 该相关谓词是一个**严格（Restrick Null）谓词**。
    **严格（Restrick Null）谓词**。StarRocks 把一个可以过滤掉 Null 值的谓词叫做严格谓词，例如a > 0；而不能过滤 Null 的谓词，叫做非严格谓词，例如：a IS Null。大部分谓词都是严格谓词，非严格谓词主要是IS Null、IF、CASE WHEN或函数构成的谓词。
例如：
```sql
-- 转换前                
Select *  From t1 Left Outer Join t2 On t1.v1 = t2.v1 Where t2.v1 > 0; 
-- 转换后， t2.v1 > 0 是一个 t2 表上的严格谓词
Select *  From t1 Inner Join t2 On t1.v1 = t2.v1 Where t2.v1 > 0;
```
需要注意的是，在 Outer Join 中，需要根据 On 子句的连接谓词进行补 Null 操作，而不是过滤，所以该转换规则不适用 On 子句中的连接谓词。例如：
```sql

Select *  From t1 Left Outer Join t2 On t1.v1 = t2.v1 And t2.v1 > 1; 
-- 显然，上面的SQL和下面SQL的语义并不等价
Select *  From t1 Inner Join t2  On t1.v1 = t2.v1 And t2.v1 > 1;
```

### 转换规则三：Full Outer Join 转为 Left / Right Outer Join
同样，当满足该约束时，Full Outer Join 可以转为 Left / Right Outer Join：存在一个可以 bind 到 Left / Right 表的严格谓词。例如：
```sql
-- 转换前
Select *  From t1 Full Outer Join t2 On t1.v1 = t2.v1 Where t1.v1 > 0; 
-- 转换后， t1.v1 > 0 是一个左表上的谓词，且是一个严格谓词
Select *  From t1 Left Outer Join t2 On t1.v1 = t2.v1 Where t1.v1 > 0;
```

### 谓词下推
谓词下推是一个 Join 上非常重要，也是很常用的一个优化规则，其主要目的是提前过滤 Join 的输入，从而提升 Join 的性能。
对于 Where 子句，当满足以下约束时，我们可以进行谓词下推，并且伴随着谓词下推，我们可以做 Join 类型转换：
1. 任意 Join 类型；
2. Where 谓词可以 bind 到其中一个输入上。
例如：
```sql
Select *  
From t1 Left Outer Join t2 On t1.v1 = t2.v1 
        Left Outer Join t3 On t2.v2 = t3.v2 
Where t1.v1 = 1 And t2.v1 = 2 And t3.v2 = 3;
```
其谓词下推的流程如下。

* 第一步，分别下推 (t1.v1 = 1 And t2.v1 = 2) 和 (t3.v2 = 3)，由于满足类型转换规则(t1 Left Outer Join t2) Left Outer Join t3 转换为 (t1 Left Outer Join t2) Inner Join t3。
![谓词下推](https://mmbiz.qpic.cn/mmbiz_png/Sq4ia0xXeMC7KibEIPcibwAINIe67P0CKS5KqNC7WpyLQ4Y7ianI0s1R6o89B1nrBNrucKXtl6Fz11J2mGn52F5cxw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

* 第二步，继续下推 (t1.v1 = 1) 和 (t2.v1 = 2)，且 t1 Left Outer Join t2 转换为 t1 Inner Join t2。
![谓词下推](https://mmbiz.qpic.cn/mmbiz_png/Sq4ia0xXeMC7KibEIPcibwAINIe67P0CKS5CPwiaSiadpj6rO63LJYZ1mwuEsHaLndk0PA2UyOQE4m7WT3EG0oehDnA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


## 谓词提取
## 等价推导
## Limit 下推 
```sql
-- 下推前
Select * 
From t1 Left Outer Join t2 On  t1.v1 = t2.v1 
Limit 100; 

-- 下推后
Select * 
From (Select * From t1 Limit 100) t Left Outer Join t2 On t.v1 = t2.v1 
Limit 100;

```

# Join Reorder
Join Reorder 用于推断多表 Join 的执行顺序，数据库需要尽可能地先执行一个**高选择度**的 Join，这样就能减少后续 Join 的输入数据，从而提升性能。
目前业界实现 JoinReorder 的算法有很多种，或者基于不同模型的，例如：

* Heuristic：基于启发式规则的，类似 MemSQL，通过定义维度表中心表排 Join 顺序。
* Left-Deep：左深树模型，搜索空间小，但是不一定最优。
* Bushy：稠密树模型，搜索空间大，包含最优解。其常见的一些 reorder 算法有：
* Exhaustive(Commutativity + Associativity) 
* Greedy
* Simulated annealing
* DP（DPsize, DPsub，DPccp...） 
* Genetic：GreenPlum
* ...... 


# Cost Model
StarRocks 使用这些 Join Reorder 的算法推导出 N 个 Plan，最终会根据 Cost Model 的算法，估算出每个 Join 的 Cost，整个 Cost 的计算公式如下：
```math
Join Cost: CPU * (Row(L) + Row(R)) + Memory * Row(R)
```
其中 Row(L）、Row(R) 分别表示 Join 左右孩子的输出行数，公式主要是考虑 CPU 开销，以及 Hash Join 右表做 Hash 表内存的开销

# 分布式 Join 规划
## MPP并行执行
StarRocks 执行 A Join B 的流程如下：
1. 按照 A 表和 B 表的分布信息分别从不同的机器上读取数据；
2. 按照 Join 的连接谓词，将 A 表和 B 表的数据 Re-Shuffle 到同一批机器上；
3. 单机 Join 执行，输出结果

## 分布式 Join 优化
StarRocks 实际执行中会产生 5 种最基本的分布式 Plan：
* Shuffle Join：分别将 A、B 两表的数据按照连接关系都 Shuffle 到同一批机器上，再进行 Join 操作。
* Broadcast Join：通过将 B 表的数据全量的广播到 A 表的机器上，在 A 表的机器上进行 Join 操作，相比较于 Shuffle Join，节省了 A 表的数据 Shuffle，但是 B 表的数据是全量广播，适合 B 表是个小表的场景。
* Bucket Shuffle Join：在 Broadcast 的基础上进一步优化，将 B 表按照 A 表的分布方式 Shuffle 到 A 表的机器上进行 Join 操作，B 表 Shuffle 的数据量全局只有一份，比 Broadcast 少传输了很多倍数据量。当然，有约束条件限制，Join 的连接关系必须和 A 表的分布一致。
* Colocate Join：通过建表时指定 A 表和 B 表是同一个 Colocate Group，意味着 A、B 表的分布完全一致，那么当 Join 的连接关系和 A、B 表分布一致时，StarRocks 可以直接在 A、B 表的机器上直接 Join，不需要进行数据 Shuffle。
* Replicate Join：StarRocks 的实验性功能，当每一台 A 表的机器上都存在一份完整的 B 表数据时，直接在本地进行 Join 操作，该 Join 的约束条件比较严格，基本上意味着 B 表的副本数需要和整个集群的机器数保持一致，所以实践意义并不理想。
## 探索分布式 Join
## 复杂的分布式 Join
## Global Runtime Filter
StarRocks 的 Hash Join 执行过程如下：
1.  StarRocks 先查询得到全量的右表数据；
2. 将右表的数据构造为一个 Hash 表；
3. 再去拉取左表的数据；
4. 基于 Hash 表来构建 Join 的连接关系；
5. 输出 Join 结果。
Global Runtime Filter 的工作时机就在 Step 2 和 Step 3 之间，StarRocks 在得到右表的数据后，通过这些运行时数据构造出来一个过滤谓词，在拉取左表数据前先将这样一个 Runime 的过滤谓词下发到左表的 Scan 节点，从而帮助左表的 Scan 节点提前过滤数据，最终达到减少 Join 输入的目的。
目前 Global Runtime Filter 支持的过滤方式为：Min / Max、In predicate 和 Bloom Filter。

[StarRocks 技术内幕 | Join 查询优化](https://mp.weixin.qq.com/s/Fv_FaoYDGZuPt4TBHPg-wg)
[Join查询优化&HashJoin算子优化 | StarRocks Hacker Meetup第四期\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1bi4y1r7Td/?vd_source=9654d62f7cce5932a53570602662aa2d)

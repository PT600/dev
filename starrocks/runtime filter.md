
## Sample
 当 Query 执行时，HashJoin 的右表构建 Hash 表，待 Hash 表构建完成后，利用 Hash 表生成 RuntimeFilter，然后 HashJoin 左侧算子（比如ExchangeNode，OlapScanNode）使用 RuntimeFilter 过滤数据，减少行数，从而降低磁盘 IO，网络 IO 和算子工作量，最使 Query 的计算性能显著提高。

## RuntimeFilter的简单介绍
在 HashJoin 中，对于INNER JOIN，下面关系代数恒等式成立：
```sql
A JOIN B = (A LEFT SEMI-JOIN B) JOIN B
```

事实上需要考虑的 HT 的 keySet 的规模，避免 key Set 过大而引入的开销， 而引入了 RuntimeFilter 机制， 在 StarRocks 中， RuntimeFilter 的类型有：

1. 0 < HT.keySet().size() <= 1024，将 keySet 转换为 in filter；
2. 0 < HT.keySet().size() <= 1024000，将keySet转换为bloom filter；
3. Min-max filter，生成 bloom filter 同时，生成 Min-max filter

## Local RuntimeFilter 和 Global RuntimeFilter
* Local RuntimeFilter 是产生和消费 RuntimeFilter 的算子都处于一个 Fragment Instance 内，目前 Local RuntimeFilter 的类型有：in-filter 和 bloom-filter(max-min filter)
* Global RuntimeFilter 是指产生和消费 RuntimeFilter 的算子是可以跨多个 Fragment Instance，需要GRF coordinator 参与 GRF 的分量收集，合并和投递。
    Global RuntimeFilter 会在 Shuffle Join 中构建，Shuffle Join 会把左表和右表数据分成 N 个对应的Partition，每个 HashJoin 算子处理其中一路
    GRF 只有 bloom-filter 类型，是因为右表比较大时并且 HashJoin 的方式是 shuffle join 时，构建 in-filter 需要的代价比较高，而 bloom-filter 的代价较小，bloom-filter需要在字节数约等于 Hash 表 size。

[StarRocks Runtime Filter 源码解析 - 知乎](https://zhuanlan.zhihu.com/p/605085563)
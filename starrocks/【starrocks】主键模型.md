## Primary Key 模型 
    * 让数据可以更好地实时更新，并且具备极速的查询能力

## 一、背景
* TP 系统通过 CDC 实时同步数据 Binlog 到 AP 系统，可以非常方便地实时对业务数据“变更”进行分析，这是最常见的场景。
* 传统的 OLAP 数据流采用 ETL (Extract-Transform-Load) 的形式，数据充分加工后再导入 AP 系统，而随着现代 AP 系统的能力提升，ELT (Extract-Load-Transform) 形式的数据流逐渐流行。即把原始或者粗加工的数据先导入 AP 系统，在 AP 系统内部使用 SQL 对数据进行各种加工转换后再
查询。
    这种方式有很多好处：
    统一使用一套系统开发维护成本更低；
    **相比使用其他计算框架处理 ETL，SQL 易用性更好**；
    最后在同一个 AP 数据库系统内通过事务机制可以更加简单便利地保证数据的一致性。
 
 而要支持实时 ELT 中的各种数据加工处理逻辑，数据库对于实时更新的支持也变成了强需求。

## 二、技术方案
AP 数据库系统中通常使用列存作为底层的存储引擎，**通常一个表包含多个文件（或 Rowset）**，每个文件使用列存格式组织（例如 Parquet），并且是不可修改的。
在这种组织结构的前提下支持更新，常见的技术方案有下面几种：

### Copy-on-Write
当一批更新到来后，需要检查其中每条记录跟原来的文件有无冲突（或者说 Key 相同的记录）。对有冲突的文件，重新写一份新的、包含了更新后数据的。

**这种方式读取时直接读取最新数据文件即可，无需任何合并或者其他操作，查询性能是最优的，但是写入的代价很大**，因此适合 T+1 的不会频繁更新的场景，不适合实时更新场景。

很多数据湖都使用了这种方式实现更新，例如 Delta Lake、Hudi 的 Copy-on-Write 表

![Copy-on-Write](https://cdn-forum.starrocks.com/optimized/2X/3/373a2c7eeec380f8fea3f12592f8135a4c36dbc8_2_634x500.png)

### Merge-on-Read
当一批更新到来后，直接排序后以列存或者行存的形式写入新的文件。
由于数据在写入时没有做去重或者说冲突检查，就需要在读取时通过 Key 的比较进行 Merge 的方式，合并多个版本的数据，仅保留最新版本的数据返回给查询执行层。

**这种方式写入的性能最好，实现也很简单，但是读取的性能很差，不适合对查询性能要求很高的场景。**
采用这种方案的包括 Hudi 的 Merge-on-Read 表，**StarRocks的Unique 模型表，ClickHouse 的实时更新模型等。**

![Merge-on-Read](https://cdn-forum.starrocks.com/optimized/2X/a/a53216799fe3e557e418ff795c3b383dbabab401_2_596x500.png)


### Delta Store
当一批更新到来后，通过主键索引，先找到每条记录原来所在的文件和位置（通常是一个整数的行号）。把位置和所做的修改作为一条 Delta 记录，放到跟原文件对应的一个 Delta Store 中。查询时，需要把原始数据和 Delta Store 中的数据合并（或者说应用 Delta 数据）。
**由于 Delta 数据是按照行号组织的，合并过程直接按照行号进行合并即可，和 Merge-on-Read 的按照 Key 进行合并相比，查询性能好很多**。这种方式由于写入时要查询索引，**写入性能稍差，但是读取的性能要好很多**。相当于牺牲了一些写入性能换来读性能。另外由于引入了主键索引和 Delta Store，复杂性提高。采用这种方案的典型系统就是 Kudu，当然还有一些内存数据库 TP 系统也常用这种方案。

![Delta Store](https://cdn-forum.starrocks.com/optimized/2X/1/141d35356e077ed8bc95b8a15d00ca7b998f43bb_2_423x500.png)

### Delete-and-Insert
当一批更新到来后，通过主键索引，先找到每条记录原来所在的位置，把该条记录标记为删除，然后把最新数据作为新记录写入新文件。读取时，根据删除标记来将旧版本过期数据过滤掉，留下最新更新后的数据。
因为无需像 Merge-on-Read 和 Delta Store 模式下进行 Merge，另外过滤算子可以下推到 Scan 层直接利用各类索引进行过滤减少扫描开销，所以查询性能的提升空间更大。

和 Delta Store 类似，此模式也牺牲了一些写入性能换取读性能，当前业界也有一些应用：比如 SQL Server 的 In-memory 列存等。Delete-and-Insert 模式需要引入主键索引以及存储和管理删除标记，在大规模实时更新和查询场景下实现难度非常大。

当前 StarRocks 的 Primary Key 模型主要采用了 Delete-and-Insert 模式，并且进行了很多新的设计，以支持在大规模实时数据更新时提供极速的查询性能。

![Delete-and-Insert](https://cdn-forum.starrocks.com/optimized/2X/5/50cbb00b8258373a2bf06f62e1cb929d8d1d7c1b_2_423x500.png)


## 主键索引
Commit 阶段的最主要工作是查找和更新主键索引，约占整个 Commit 过程 90% 以上的时间。Commit 的速度主要取决于主键索引的性能，因此主键索引目前使用内存 HashMap 来实现，和计算层一样使用高性能 HashMap 库 phmap。HashMap 的 key 是多列编码后的 Binary 串，Value 是由(rowset_id, rowid)组成的 64bit 整数，可以根据主键查找主键对应的行在哪个 Rowset 的第几行。内存 HashMap 相比 RocksDB 等通用 KV 存储在性能上有很大优势，通常在现代 CPU 上每个操作耗时仅 20ns~200ns，也就是单核可以达到 5M-50Mop/s，能够保证批量导入的大事务快速完成 Commit。但是其最大的劣势是需要消耗内存。我们针对定长和变长类型的主键分别做了一些优化，以降低内存占用，未来会持续优化 HashMap 的内存占用，另外会探索实现基于 SSD/PMEM 的可持久化的主键索引。

主键索引目前是按需加载的，在有写入时会读取主键列构建，在长时间没有写入后会释放以节约内存。这非常适合有冷热数据模式的场景，即一条记录在创建后的一段时间内是热数据经常被更新，过一段时间后逐渐变冷不再更新。数据按天分区后，只有最近几天的分区才会有导入存在，这样最近几天的 Tablet 的主键索引才会被加载，整体内存使用量可控。

**典型的场景比如各种订单场景**，包括电商的订单、打车/共享单车订单等，或者应用或者设备的 Session 等。

主键占内存，可以按日期分区，冷热数据


## Tablet 内部结构

Tablet 是 StarRocks 底层存储引擎的基本单元，每个 Tablet 内部可以看作一个单机列存引擎，负责对数据的读写操作。

![Primary Key的Tablet内部结构](https://cdn-forum.starrocks.com/optimized/2X/9/9cab296d94c046178d4f861e4feacc53917262b2_2_690x298.png)

* Meta: 元数据，保存 Tablet 的版本历史以及每个版本的信息，比如包含哪些 Rowset 等。序列化为 Protobuf 后存储在 RocksDB 中，为了快速访问，也会缓存在内存中。
* Rowset: 一个 Tablet 的所有数据被切分成多个 Rowset，每个 Rowset 以列存文件的形式存储，其内部组织编码方式类似 Parquet 文件，但是是 StarRocks 特有的格式。
* Primary Index: 主键索引，保存主键到该记录所在位置的映射。目前完整维护在内存中，不持久化到磁盘，**在导入进行时按需构建**，一段时间没有持续导入时又会释放以节约内存。
* DelVector: 记录每个 Rowset 中被标记为删除的行，比如一个文件中有 10000 行数据，其中 200 行被标记删除，则可以使用一个 Bitmap 来标记被删除的行。一般都是很稀疏的，使用 RoaringBitmap 可以高效存储和操作，同样保存在 RocksDB 中，但也会缓存在内存中以便能够快速访问。



[📖 StarRocks Primary Key 模型深度解析：实时更新与极速查询如何兼得 - 🙌 StarRocks 技术分享 / 原理解读 - StarRocks中文社区论坛](https://forum.mirrorship.cn/t/topic/2305)


## 局限
1. 内存消耗大
    目前是基于全内存的主键索引，性能好，对写入性能影响小，但是内存消耗非常高，直接限制了使用场景：
    * 总行数相对可控的业务场景
    * 数据有冷热特征的业务场景
解决办法：
    持久化主键索引，引入两层哈希表，第一层在内存中，第二层保存在磁盘，内存使用减低到1/10，性能只有一点下降

2. 高频导入
    starrocks是一个批量(微批)系统，不适合处理太频繁的事务，会导致too many versions、too many pending versions 之类的问题
解决办法:

* 针对高频导入导致的各种问题，比如 too many versions、too many pending versions 的报错导致整个导入失败。我们对 publish、compaction 做了大量的优化，比如对 publish 的 task 并行执行和 rocks DB 元数据提交的并行执行，可以大大缩短 publish 的延迟，从而提升整个事务的吞吐能力。另外针对 compaction，我们添加了劣势的 vertical group 的 compaction 来提升 compaction 的性能。我们还设计了新的 compaction 机制来解决 BE 中当 tablet 数量特别大时 compaction 的调度问题。

* 针对 Flink CDC 同步任务并发导致的事务数量成倍增加的问题，我们新增加了基于 streamload 的事务导入接口。原来每个 Flink Sink 的 task 会单独对应一个导入事务，导致事务数量成倍增加。使用新的事务接口之后，可以将多个导入任务合并成一个事务。在定期 Sink 开始前，开启这个事务，然后并行导入写入数据，最后全部 task 完成数据的传输后，整体再提交这个事务。在上面这个例子中，总事务数就可以从 4 个减少到 1 个。高频导入的问题，本质上是事务数量多的问题，通过降低事务数量，可以避免高频导入带来的一系列问题。

3. 不支持部分更新
目前的解决办法：
    * 使用Flink做流式join，然后导入到StarRocks
    * 使用 TP 系统构造一个大宽表，上游模块一部分以列更新的方式写入 TP 系统，再通过 TP 系统同步给 AP 系统
    * 分模块导入 AP 系统，在 AP 系统中通过 DML 定期地去做 join，刷新到大宽表中，但会牺牲一部分实时性
这三种模式都需要引入新的模块，增加了系统的复杂度，不是一个完美的方案。

[峰会实录 | StarRocks存储引擎近期进展与实时分析实践\_数据库·\_StarRocks\_InfoQ写作社区](https://xie.infoq.cn/article/618154e316757be22fccca899)
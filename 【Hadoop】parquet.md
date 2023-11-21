
## 文件结构

![文件结构](https://pic4.zhimg.com/80/v2-c9be01d41e1807cbce7941c89a8450af_1440w.webp)

## Header
Header的内容很少，只有4个字节，本质是一个magic number，用来指示文件类型。这个magic number目前有两种变体，分别是“PAR1”和“PARE”。其中“PAR1”代表的是普通的Parquet文件，“PARE”代表的是加密过的Parquet文件。

## Index
* Index是Parquet文件的索引块，主要为了支持“谓词下推”（Predicate Pushdown）功能
* 目前Parquet的索引有两种，一种是Max-Min统计信息，一种是BloomFilter。其中Max-Min索引是对每个Page都记录它所含数据的最大值和最小值，这样某个**Page**是否不满足查询条件就可以通过这个Page的max和min值来判断。

## Footer
* Footer是Parquet元数据的大本营，包含了诸如schema，Block的offset和size，Column Chunk的offset和size等所有重要的元数据。
* 另外Footer还承担了整个文件入口的职责，读取Parquet文件的第一步就是读取Footer信息，转换成元数据之后，再根据这些元数据跳转到对应的block和column，读取真正所要的数据。
* 关于Footer还有一个问题，**就是为什么Parquet要把元数据放在文件的末尾而不是开头**？**这主要是为了让文件写入的操作可以在一趟（one pass）内完成**。因为很多元数据的信息需要把文件基本写完以后才知道（例如总行数，各个Block的offset等），如果要写在文件开头，就必须seek回文件的初始位置，大部分文件系统并不支持这种写入操作（例如HDFS）。而如果写在文件末尾，那么整个写入过程就不需要任何回退。

## Row Group
* Row Group是单独的文件，一般是hdfs上的一个block
* Row Group切分为Column Chunk
* Column Chunk有Page组成，Page默认大小1M, Page是Parquet文件的最小读取单位，也是压缩单位，支持单独索引，支持细粒度的查询。


## Repetition Level
* Repetition Level用来表达数组类型字段的长度
* 当嵌套层级变化时，Repetition Level表示变化前的层级
* 当嵌套层级不变时，Repetition Level表示当前的层级

[["a", "b"], ["c", "d", "e"]]对应的Repetition Level为

| Value | Repetition Level      |
| ----- | -----------------     |
|   a   | 0 (嵌套层级变化0 -> 2) |
|   b   | 2                     |
|   c   | 1 (嵌套层级变化1 -> 2) |
|   d   | 2                     |
|   e   | 2                     |

## Definition Level
* Definition Level主要表达null的位置
* Definition level小于嵌套层级的，表示这个值是null
* Definition level的具体的值表示null出现在哪个嵌套层级

## 优化
* 每个值都存储Repetition Level和Definition Level，比较浪费
* 可以对非数组类型的字段，不保存Repetition Level
* 对非null类型的字段，不保存Definition Level
* 使用bit-packing技术(一个byte即可？)

## refer
[详解Parquet文件格式原理 - 知乎](https://zhuanlan.zhihu.com/p/538163356)
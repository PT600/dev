
## StarRocks 全局字典的构建
对于 StarRocks 的全局字典的构建，主要有以下考虑：
* 自适应，不需要用户通过 Schema 指定特定低基数列，而是根据数据特性，自动选择优化策略。
* 尽可能避免单点问题，比如数据导入的时候遇到新的字符串，先通过中心节点更新全局字典。

### 数据存储上的字典优化
StarRocks 的基本存储单元为 Segment，每个 Segment 的存储结构如下图所示：
![Segment](https://pic1.zhimg.com/80/v2-be38288c4b89b44febf946a03b423230_720w.webp)

StarRocks 的存储结构天然为低基数字符串做了字典编码。对于 Segment 上的低基数字符串列会有以下特点：
* Footer 上会存储有这个 Column 特有的字典信息，包括字典码跟原始字符串之间的映射关系；
* Data page 上存储的不是原始字符串，而是整数类型的字典码(整型)。

## 全局字典的构建
StarRocks 支持 CBO 优化器，并且存在一套统计信息机制，那么就可以通过统计信息来收集全局字典。我们通过统计信息，筛选出潜在的低基数列，再**从潜在的低基数列的元数据中读取局部字典信息**，然后做去重/编码操作，就可以收集到全量的字典了。

## 全局字典的正确性保证
对于低基数列来说，那么肯定会出现一种情况，在某次导入中导入了新的 String (这个 String 不在全局字典的集合内)，那么这个时候，原先已经构建的全局字典就没有办法包含所有的字符串的值。因此 StarRocks 需要维护全局字典的有效性。

全局字典可能失效只会出现在导入， StarRocks 支持了很多类型的数据导入方式，而所有的导入都有两个共同点
* 导入产生新的 Segment。
* 通过 Master FE 提交事务

对于低基数列，**所有 Segment 中都必定存在局部字典信息**，那么对于一个新的导入，在产生新的 Segment 时，会有几种情况。
* 如果新生成的 Segment 没有了局部字典，那么说明这个列很可能是一个高基数列，此时不再适合全局字典优化；
* 新生成的 Segment 有局部字典，而且局部字典中的所有 String 是全局字典的子集，这种情况下可以直接使用旧的字典；
* 新生成的 Segment 有局部字典，而且局部字典所有的 String 值，部分不在全局字典里，此时全局字典失效已经生效，需要重新生成全局字典。

无论出现了上面的哪种情况，在向 FE 中心节点提交的时候，带上这个对应的信息，我们就都能保证全局字典的正确性。

## refer
[StarRocks 技术内幕 | 基于全局字典的极速字符串查询 - 知乎](https://zhuanlan.zhihu.com/p/554256193)

## Build global Dictionary
* Collect dictionary with a very low cost
* Only scan dictionary Page
* Build global dictionary by Merge Local Dictionary(When Query)
* Dictionaries are stored in memory

## Use Global Dictionary in SQL
* Optimizer and Executor
    Select count(distinct Key1) From Table1 group by Key2;
* Optimizer Rewrite the Plan

## Use Global Dictionary Optimization in Scan  
* Convert Local Dictionary code to Global Dictionary code (with simd)

## Use Global Dictionary Optimization in Expression
* Rewrite Expression in Optimization
    select substring(key1, 1, 2) From table
* Build Code Convert Mapping
* Process Complex Expression
    select sum(if(season="summer", 1, NULL)) from Table;
    
## refer
[StarRocks 技术内幕：低基数全局字典优化\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1ra411N7g8/?vd_source=9654d62f7cce5932a53570602662aa2d)

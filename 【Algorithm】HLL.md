## 假如需要统计展览馆的唯一访问人数
# Flajolet-Martin Algorithm
1. 可以根据手机号，但是数据量太大
2. 可以根据手机号后6位，假设开始为0的长度L，然后使用 10ᴸ 来估算。
3. 由于电话号码不是均匀分布的，可以使用hash函数，将电话号码转换为均匀分布的二进制数据，然后使用 2ᴸ 来估算。
4. 统计分析发现 2ᴸ 误差较大，可以通过加入校正因子  ϕ ≈ 0.77351 得到公式：2ᴸ / ϕ

# 使用LogLog 算法改进
1. Flajolet-Martin Algorithm容易被异常值影响，可以使用多个独立的hash方法，然后求平均。但是计算量比较大
2. 使用一个hash和分桶方法，使用高m位作为桶的序号，然后将 2^m个桶求平均。

# SuperLogLog
1. 通过去掉最大的30%的异常值，the accuracy is improved from 1.3/√m to 1.05/√m

# HyperLogLog
1. 使用harmonic mean来处理异常值，By using harmonic mean instead of geometric mean used in LogLog and only using 70 percent smallest values in SuperLogLog, HyperLogLog achieve an error rate of 1.04/√m

# Reference
* [HyperLogLog: A Simple but Powerful Algorithm for Data Scientists](https://chengweihu.com/hyperloglog/)
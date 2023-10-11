## Watermark的传递

* Watermark为真实的特殊的数据
* 合并时，取最小的watermaker，通过空闲等待时间控制的上游watermaker的更新
* watermaker通过广播形式传递

## 迟到数据
* 触发计算和关闭窗口是分开的
* 每个迟到数据会触发一次计算
* 关闭窗口后可以使用侧边输出流
## DMA(Direct Memory Access)
* DMA将数据从一个地址空间复制到另外一个地址空间，绕过CPU
* 支持DMA的硬件包括网卡、声卡、硬盘等

## Kafka零拷贝传输过程
* 操作系统将磁盘数据加载到内核空间的Read Buffer(页缓存区)中
* 操作系统将Read Buffer的数据发送到网卡
* 操作系统将数据的描述信息拷贝到Socket Buffer。
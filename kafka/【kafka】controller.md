
## 控制器是如何被选出来的
1. 第一个在 ZooKeeper 中成功创建 /controller 临时节点的 Broker 会被指定为控制器。

2. 其他 Broker 在控制器节点上创建 Zookeeper watch 对象。

3. 如果控制器被关闭或者与 Zookeeper 断开连接，Zookeeper 临时节点就会消失。集群中的其他 Broker 通过 watch 对象得到状态变化的通知，它们会尝试让自己成为新的控制器。

4. 第一个在 Zookeeper 里创建一个临时节点 /controller 的 Broker 成为新控制器。其他 Broker 在新控制器节点上创建 Zookeeper watch 对象。

4. 每个新选出的控制器通过 Zookeeper 的条件递增操作获得一个全新的、数值更大的 controller epoch。其他节点会忽略旧的 epoch 的消息。

5. 当控制器发现一个 Broker 已离开集群，并且这个 Broker 是某些 Partition 的 Leader。此时，控制器会遍历这些 Partition，并用轮询方式确定谁应该成为新 Leader，随后，新 Leader 开始处理生产者和消费者的请求，而 Follower 开始从 Leader 那里复制消息。

简而言之，Kafka 使用 Zookeeper 的临时节点来选举控制器，并在节点加入集群或退出集群时通知控制器。控制器负责在节点加入或离开集群时进行 Partition Leader 选举。控制器使用 epoch 来避免“脑裂”，“脑裂”是指两个节点同时被认为自己是当前的控制器

## 控制器的作用

1. 主题管理（创建、删除、增加分区）
    控制器帮助我们完成对Kafka主题的创建、删除以及分区增加的操作。换句话说，当我们执行kafka-topics脚本时，大部分的后台工作都是控制器来完成的。
2. 分区重分配
    分区重分配主要是指，kafka-reassign-partitions脚本（关于这个脚本，后面我也会介绍）提供的对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。

3. Preferred领导者选举
    Preferred领导者选举主要是Kafka为了避免部分Broker负载过重而提供的一种换Leader的方案

4. 集群成员管理
    集群成员管理，包括自动检测新增 Broker、Broker 主动关闭及被动宕机。这种自动检测是依赖于前面提到的 Watch 功能和 ZooKeeper 临时节点组合实现的。

    比如，控制器组件会利用Watch 机制检查 ZooKeeper 的 /brokers/ids 节点下的子节点数量变更。目前，当有新 Broker 启动后，它会在 /brokers 下创建专属的 znode 节点。一旦创建完毕，ZooKeeper 会通过 Watch 机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增 Broker 作业。

    侦测 Broker 存活性则是依赖于刚刚提到的另一个机制：临时节点。每个 Broker 启动后，会在 /brokers/ids 下创建一个临时 znode。当 Broker 宕机或主动关闭后，该 Broker 与 ZooKeeper 的会话结束，这个 znode 会被自动删除。同理，ZooKeeper 的 Watch 机制将这一变更推送给控制器，这样控制器就能知道有 Broker 关闭或宕机了，从而进行“善后”。

5. 数据服务
    控制器的最后一大类工作，就是向其他 Broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有 Broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。

    控制器中保存了多种数据，比较重要的的数据有：

    * 所有 Topic 信息。包括具体的分区信息，比如领导者副本是谁，ISR 集合中有哪些副本等。
    * 所有 Broker 信息。包括当前都有哪些运行中的 Broker，哪些正在关闭中的 Broker 等。
    * 所有涉及运维任务的分区。包括当前正在进行 Preferred 领导者选举以及分区重分配的分区列表。
    值得注意的是，这些数据其实在 ZooKeeper 中也保存了一份。每当控制器初始化时，它都会从 ZooKeeper 上读取对应的元数据并填充到自己的缓存中。有了这些数据，控制器就能对外提供数据服务了。这里的对外主要是指对其他 Broker 而言，控制器通过向这些 Broker 发送请求的方式将这些数据同步到其他 Broker 上。
    
## refer
[Kafka 集群 | BIGDATA-TUTORIAL](https://dunwu.github.io/bigdata-tutorial/kafka/Kafka%E9%9B%86%E7%BE%A4.html#_2-1-%E5%A6%82%E4%BD%95%E9%80%89%E4%B8%BE%E6%8E%A7%E5%88%B6%E5%99%A8)
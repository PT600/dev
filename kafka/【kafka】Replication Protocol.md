
## Fetch Reqeust includes:
* offset: latest offset
* lastFetchEpoch: latest epoch

## Replication
## follower send fetch request to leader. 
    if not enough bytes(**replica.fetch.min.bytes** default 1) in the response, wait up to **replica.fetch.wait.max.ms**(default 500ms)


## Commiting Partiton Offsets -- High Watermark
* Subsequent fetch request implicitly confirms receipt of previously fetched records.
* Leader commits records once all followers in ISR have confirmed receipt.
* Only committed records are exposed to consumers.

## Advancing the Follower High Watermark
* Subsequent fetch response include new high watermark

## Handling Leader Failure
* New leader elected from ISR and propagated through control plane

[Apache Kafka®'s Replication Protocol](https://www.youtube.com/watch?v=PPDffzAy86I)


## LEO & HW

1. LEO（last end offset）：
    当前replica存的最大的offset的下一个值
2. HW（high watermark）：
    小于 HW 值的offset所对应的消息被认为是“已提交”或“已备份”的消息，才对消费者可见。

![leo & hw](assets/kafka_leo_hw.png)


[Kafka之ISR机制的理解-CSDN博客](https://blog.csdn.net/daima_caigou/article/details/109390705)


## min.insync.replicas
    The lead replica for a partition checks to see if there are enough in-sync replicas for safely writing the message (controlled by the broker setting min.insync.replicas). The request will be stored in a buffer until the leader observes that the follower replicas replicated the message, at which point a successful acknowledgement is sent back to the client.

[Kafka Topic Configuration: Min In-Sync Replicas | Learn Apache Kafka](https://www.conduktor.io/kafka/kafka-topic-configuration-min-insync-replicas/)
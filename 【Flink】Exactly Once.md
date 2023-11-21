

## Sink
* 当收到非barrier的数据时，Sink开始外部事务（如Kafka事务），然后写入数据
* 当收到barrier的数据时，Sink开始pre commit，保存外部事务相关数据(如txid)到checkpoint
* JobManager发送的检查点完成的通知（commit事务），Sink提交外部事务
    1. 当commit失败时，会重新恢复到checkpoint，继续提交事务
    2. 所以要求外部事物的等幂性
* 当有一个precommit失败时，回滚事务

[An Overview of End-to-End Exactly-Once Processing in Apache Flink (with Apache Kafka, too!) | Apache Flink](https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/)
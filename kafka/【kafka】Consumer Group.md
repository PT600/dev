

## Group Coordinator vs Group Leader
* Group Coordinator is one of the brokers, while the group leader is one of the consumer in a consumer group
* Group Coordinator receives heartbeats(or polling for messages)
* Group Leader is responsible for computing the assignments, while the group coordinator is responsible for rebalancing.
* Group Coordinator is responsible for committing or fetching consumer offset by using the __consumer_offsets topic.

## Whay seperate group coordinator and group leader?
* Short answer: More flexible assignment policies without rebooting the broker.
* Long answer: 
    The group leader is responsible for computing the assignments
    It means the assignment strategy can be configured on a consumer(**partition.assignment.strategy** consumer config parameter)
    If handled by group coordinator, it would be impossible to configure a custom assignment strategy without reboot the broker.


 ## [kafka consumer group rebalance](https://medium.com/lydtech-consulting/kafka-consumer-group-rebalance-1-of-2-7a3e00aa3bb4)

## Consumer Groups
* Group membership is managed on the broker side, and partition assignment is managed on the client side. 
* The broker has no knowledge of that the resources are and how they are assigned amongst the consumers.


## Kafka Consumer Fetch Behavior
* Upon calling .poll(), the consumer with fetch data from the kafka partitions. 
* The consumer then process the data in the main thread, and the consumer proceeds to **an optimization of prefetching the next batch of data**.
* important configurations:
    **fetch.min.bytes**: default 1
    **fetch.max.wait.ms**: default 500
    **max.partition.fetch.bytes**: default 1MB
    **fetch.max.bytes**: default 55MB
    **max.poll.records**: default 500

## Kafka Consumer Heartbeat Thread
* heartbeat.interval.ms(default 3 seconds)
* session.timeout.ms(kafka v3.0+: 45 seconds, Kafka up to v2.8: 10 seconds), 

## Kafka Consumer Poll Thread
* max.poll.interval.ms: (default 5 minutes) This setting is particularly relevant for frameworks that can take time to process data such as Apache Spark
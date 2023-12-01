
## Transactional Flow
The following diagram illustrates a Kafka Streams application consuming a batch of events, writing state to the state store, and producing outbound events. The flow is wrapped in a Kafka Transaction, so the application uses the Transaction Coordinator which lives on the broker to coordinate the transaction as it progresses.

![kafka streams transaction](https://miro.medium.com/v2/resize:fit:1100/format:webp/0*ov_BhNk0H8zyORic.png)

1. The Kafka Streams library sends a begin transaction request to the Transaction Coordinator informing it that a transaction has begun.

2. **A batch of events are consumed from the inbound topic** and processed by the source processor to begin processing.

3. The state being captured is written to the local persistent state store (RocksDB by default).

4. Any sink processors in the processor topology write their resulting events to outbound topics.

5. The changelog topic, which backs the state store, is written with the state change.

6. The consumer offset topic is written with the offsets of the processed events, marking their processing as complete.

7. The Kafka Streams library sends a commit transaction request to the Transaction Coordinator.

8. The Transaction Coordinator writes a prepare commit record to the transaction topic. At this point the transaction will complete no matter what failure scenarios may happen.

9. The Transaction Coordinator writes a commit marker to the outbound topic, changelog topic, and consumer offsets topic. Downstream transactional consumers (configured as READ_COMMITTED) will block until the commit marker is written to the topic partition.

10. The Transaction Coordinator writes a complete commit transaction record to the transaction topic. The producer can now begin its next transaction.

[Kafka Streams: Transactions & Exactly-Once Messaging | by Rob Golder | Lydtech Consulting | Medium](https://medium.com/lydtech-consulting/kafka-streams-transactions-exactly-once-messaging-82194b50900a)
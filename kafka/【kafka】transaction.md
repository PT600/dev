
## Data flow
At a high level, the data flow can be broken into four distinct types.

### A: the producer and transaction coordinator interaction
When executing transactions, the producer makes requests to the transaction coordinator at the following points:

1. The initTransactions API registers a transactional.id with the coordinator. At this point, the coordinator closes any pending transactions with that transactional.id and bumps the epoch to fence out zombies. This happens only once per producer session.
2. **When the producer is about to send data to a partition for the first time in a transaction, the partition is registered with the coordinator first**.
3. When the application calls commitTransaction or abortTransaction, a request is sent to the coordinator to begin the two phase commit protocol.

### B: the coordinator and transaction log interaction

* As the transaction progresses, the producer sends the requests above to update the state of the transaction on the coordinator. The transaction coordinator keeps the state of each transaction it owns in memory, and also writes that state to the transaction log (which is replicated three ways and hence is durable).

* The transaction coordinator is the only component to read and write from the transaction log. If a given broker fails, a new coordinator is elected as the leader for the transaction log partitions the dead broker owned, and it reads the messages from the incoming partitions to rebuild its in-memory state for the transactions in those partitions.

### C: the producer writing data to target topic-partitions
After registering new partitions in a transaction with the coordinator, the producer sends data to the actual partitions as normal. This is exactly the same producer.send flow, but with some extra validation to ensure that the producer isn’t fenced.

### D: the coordinator to topic-partition interaction
**After the producer initiates a commit (or an abort), the coordinator begins the two-phase commit protocol.**
* In the first phase, the coordinator updates its internal state to “prepare_commit” and updates this state in the transaction log. **Once this is done the transaction is guaranteed to be committed no matter what.**

* The coordinator then begins phase 2, where it writes transaction commit markers to the topic-partitions which are part of the transaction.

These transaction markers are not exposed to applications, but are used by consumers in read_committed mode to filter out messages from aborted transactions and to not return messages which are part of open transactions (i.e., those which are in the log but don’t have a transaction marker associated with them).

Once the markers are written, the transaction coordinator marks the transaction as “complete” and the producer can start the next transaction.


## Performance of the transactional producer
First, transactions cause only moderate write amplification. The additional writes are due to:
1. For each transaction, we have had additional RPCs to register the partitions with the coordinator. **These are batched**, so we have fewer RPCs than there are partitions in the transaction.
2. When completing a transaction, one transaction marker has to be written to each partition participating in the transaction. Again, the transaction coordinator **batches all markers bound** for the same broker in a single RPC, so we save the RPC overhead there. But we cannot avoid one additional write to each partition in the transaction.
3. Finally, we write state changes to the transaction log. This includes a write for each batch of partitions added to the transaction, the “prepare_commit” state, and the “complete_commit” state.

As we can see the overhead is independent of the number of messages written as part of a transaction. So the key to having higher throughput is to **include a larger number of messages per transaction**.

In practice, for a producer producing 1KB records at maximum throughput, committing messages every 100ms results in only a 3% degradation in throughput. Smaller messages or shorter transaction commit intervals would result in more severe degradation.

The main tradeoff when increasing the transaction duration is that it increases end-to-end latency. Recall that a consumer reading transactional messages will not deliver messages which are part of open transactions. So the longer the interval between commits, the longer consuming applications will have to wait, increasing the end-to-end latency.

## refer
[Transactions in Apache Kafka | Confluent](https://www.confluent.io/blog/transactions-apache-kafka/)

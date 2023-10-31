
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

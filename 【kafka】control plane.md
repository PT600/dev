
## KRaft Metadata Replication
* controller pool 
* The active controller is the lead of metadata topic(__cluster_metadata-0)

## KRaft Metadata Replication vs Kafka Data replication
* Leader election and committing record: quorum instead of ISR
* Records are fsync to disk prior to commit

## Leader Election Step1: Vote Request
* Increase its epoch
* Transitions to candidate
* Votes for itself
* Requests votes from other controllers
```
    VoteRequest:
        candidateEpoch: 2
        lastOffset: 4
        lastOffsetEpoch: 1
```

## Leader Election Step2: Vote Response
* Candidate first vote self
* If a lower epoch is seen, reject
* If already voted on epoch, send same answer
* Otherwise, grant vote only if the candidate's log is at least as "long"


## Refer
[The Apache Kafka® Control Plane – ZooKeeper vs. KRaft](https://www.youtube.com/watch?v=6YL0L4lb9iM)
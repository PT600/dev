
aft算法在具体实现中，将分布式一致性问题分解为了**Leader选举、日志同步和安全性保证**三大子问题

## 请求处理

* 所有操作采用类似**两阶段提交**的方式，Leader 在收到来自客户端的请求后并不会执行，只是将其写入自己的日志列表中，然后将该操作发送给所有的 Follower。Follower 在收到请求后也只是写入自己的日志列表中然后回复 Leader，当有超过半数的结点写入后 Leader 才会提交该操作并返回给客户端，同时通知所有其他结点提交该操作。
* 通过这一流程保证了只要提交过后的操作一定在多数结点上留有记录（在日志列表中），从而保证了该数据不会丢失。

# 子问题1： Leader选举
* 每个follower有个定时器，如果定时器超时（150-300ms之间的随机值），说明没收到Leader发送的消息，则变成Candidate，发送投票请求
* 如果收到半数以上的yes，则成为Leader，并立即发送心跳信息给其他Follower。


### 投票与任期Term
* 每个任期以一次选举开始，当一个节点向其他节点发送投票请求时，会将自己当前的Term加1，表明新的一轮开始。
* 当节点收到比自己更新的Term的投票请求时，**会更新自己的Term**，成为Follower，否则直接拒绝。
* 在一个任期内，节点只能投给一个节点，确保一个任期只会产生一个Leader（Election Safety)。
* 投票由一个称为 RequestVote 的 RPC 调用进行，请求中除了有 Candidate自己的 term 和 id 之外，还要带有自己最后一个日志条目的 index 和 term。

### 投票规则：
* Raft通过比较**最后一条日志的index和term**来决定谁更新一些。如果term不一致则拥有更大的term日志更新，如果term一样，则index更大的日志更新。
* 在一个任期内只可以投票给一个结点
* 首先会判断请求的term是否更大，不是则说明是旧消息，拒绝该请求。
* 如果任期Term相同，则比较index。

# 子问题2：日志同步
    Leader选出后，就开始接收客户端的请求。Leader把请求作为日志条目（Log entries）加入到它的日志中，然后并行的向其他服务器发起 AppendEntries RPC复制日志条目。当这条日志被复制到大多数服务器上，Leader将这条日志应用到它的状态机并向客户端返回执行结果。
![日志同步流程](https://imgs.lfeng.tech/images/2023/04/f7VXgC.png)

    Leader 会给每个 Follower 发送该 RPC 以追加日志，请求中除了当前任期 term、Leader 的 id 和已提交的日志 index，还有将要追加的日志列表（空则成为心跳包），前一个日志的 index 和 term。
![AppendEntries RPC](https://imgs.lfeng.tech/images/2023/04/Vlm2Ub.png)

## 在接到该请求后，会进行如下判断：
* 检查term，如果请求的term比自己小，说明已经过期，直接拒绝请求。
* 如果步骤1通过，则对比先前日志的index和term，如果一致，则就可以从此处更新日志，把所有的日志写入自己的日志列表中，否则返回false。

    这里对步骤2进行展开说明，每个Leader在开始工作时，会维护 nextIndex[] 和 matchIndex[] 两个数组，分别记录了每个 Follower 下一个将要发送的日志 index 和已经匹配上的日志 index。每次成为 Leader 都会初始化这两个数组，前者初始化为 Leader 最后一条日志的 index 加 1，后者初始化为 0，每次发送 RPC 时会发送 nextIndex[i] 及之后的日志。

    在步骤2中，当Leader收到返回成功时，则更新两个数组，否则说明follower上相同位置的数据和Leader不一致，这时候Leader会减小nextIndex[i]的值重试，一直找到follower上两者一致的位置，然后从这个位置开始复制Leader的数据给follower，同时follower后续已有的数据会被清空。


## 在复制的过程中，Raft会保证如下几点：

* Leader 绝不会覆盖或删除自己的日志，只会追加 （Leader Append-Only），成为 Leader 的结点里的日志一定拥有所有已被多数节点拥有的日志条目，所以先前的日志条目很可能已经被提交，因此不可以删除之前的日志。

* 如果两个日志的 index 和 term 相同，那么这两个日志相同 （Log Matching），第二点主要是因为一个任期内只可能出现一个 Leader，而 Leader 只会为一个 index 创建一个日志条目，而且一旦写入就不会修改，因此保证了日志的唯一性。

* 如果两个日志相同，那么他们之前的日志均相同，因为在写入日志时会检查前一个日志是否一致，从而递归的保证了前面的所有日志都一致。从而也保证了当一个日志被提交之后，所有结点在该 index 上提交的内容是一样的（State Machine Safety）。


# 子问题3：安全性保障（核心）
Raft算法中引入了如下两条规则，来确保了
* 已经commit的消息，一定会存在于后续的Leader节点上，并且绝对不会在后续操作中被删除。
* 对于并未commit的消息，可能会丢失。

## 多数投票规则
* 一个candidate必须获得集群中的多数投票，才能被选为Leader；而对于每条commit过的消息，它必须是被复制到了集群中的多数节点，也就是说成为Leader的节点，至少有1个包含了commit消息的节点给它投了票。
* 而在投票的过程中每个节点都会与candidate比较日志的最后index以及相应的term，如果要成为Leader，必须有更大的index或者更新的term，所以Leader上肯定有commit过的消息。
* Leader只对自己任期内的日志条目适用该规则，**先前任期的条目只能由当前任期的提交而间接被提交**。



## refer
* [一文彻底搞懂Raft算法](https://juejin.cn/post/7218915344130359351)
* [Raft论文中文版](https://docs.qq.com/doc/DY0VxSkVGWHFYSlZJ)
* [Raft论文英文版](https://raft.github.io/raft.pdf)
* [Implementing Raft: Part 0 - Introduction](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/)
* [Raft共识算法](https://fanlv.wiki/2022/03/08/raft-introduction/)
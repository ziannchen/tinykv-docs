# Raft

在本项目中我们需要实现一个基于 raft 分布式共识算法的高可用kv存储服务器，它既需要我们实现Raft算法也需要我们知道如何实际使用它。

我们通过3个步骤来达成这一目标：

- 实现基本的Raft算法。
- 在Raft之上构建一个可容错的KV服务。
- 添加对于raft日志垃圾回收以及快照的支持。

## Part A：实现基本的Raft算法

raft层中没有物理时钟，而是使用一个逻辑时钟。上层应用通过调用`RawNode.Tick()`来推动逻辑时钟，进而推动election \ heartbeat timeout的发生，从而推动了raft状态机。

- 在`Raft.tick`函数中，根据`Raft.State`推动election timeout或是 heartbeat timeout。这是通过控制`electionElapsed`和`heartbeatElapsed`进行自增操作实现的，当`electionElapsed >= randomizeElectionTimeout`时触发一次选举，当`heartbeatElapsed >= heartbeatTimeout`时触发一次心跳。触发之后对应地要清零`Elapsed`。


raft中不同peer之间、raft层与上层应用之间收发消息都是异步的。在raft层中只需将希望发出的消息存入`Raft.msg`中，上层应用会在调用`HandleRaftReady`处理消息并转发到目标处。上层应用同时为每个收到的消息调用`Raft.Step`让raft层处理消息。

- raft中定义的不同`Message`有不同的`MsgType`。根据论文，不同的`MsgType`只能拥有特定的raft状态的peer才能处理，如下表所示。每个消息先在`Raft.Step`函数中根据此时raft状态的的不同被路由到对应的不同状态的step函数中处理，再根据消息类型的不同由对应的`handlexxx`函数处理。

|`Msgtype`|`State`|
|--|-----|
|pb.MessageType_MsgHup|L、C、F|
|pb.MessageType_MsgBeat|L|
|pb.MessageType_MsgPropose|L|
|pb.MessageType_MsgAppend|L、C、F|
|pb.MessageType_MsgAppendResponse|L|
|pb.MessageType_MsgRequestVote|L、C、F|
|pb.MessageType_MsgRequestVoteResponse|C|
|pb.MessageType_MsgSnapshot|L、C、F|
|pb.MessageType_MsgHeartbeat|L、C、F|
|pb.MessageType_MsgHeartbeatResponse|L|
|pb.MessageType_MsgTransferLeader|L、C、F|
|pb.MessageType_MsgTimeoutNow|C、F|


newRaft函数，根据传入的一个`Config`参数创建一个raft peer实例。

-   `Raft.RaftLog`需要调用`newLog`函数，需要注意Raft中的entries和Storage中entries数组下标、日志项index、几个特殊的index之间的关系。

- `Raft.Prs`记录了同一个region内各个peer的日志同步情况。 还有一个隐含的作用是记录了同一个region内各个peer的id，也让我们知道了peer的数量，这在有时候很有用。

1.Leader election

首先要实现`becomexxx`函数。根据论文可以很容易实现

- `becomeFollower`：更新State、Vote、Term、Lead，将electionElapsed清零。
- `becomeCandidate`：更新State，将Vote设为自己的id，清空votes、Lead、electionElapsed，设置votes[r.id] = true，增加Term。
- `becomeLeader`：更新State、heartbeatElapsed，将Lead设为自己的id，更新Prs，自增一条空日志并广播（每个leader只会commit本term内的日志，如此可以保证集群的可用性）。需要注意的是如果当前Region中只有自己，则直接commit该条日志。

发起选举的过程是在`startElection`中实现。调用becomeCandidate，并向其他peer广播RequestVote消息。在消息中需要附加自身的最后一条日志的Index和term。需要注意的是，若region中只有自己一个节点，则直接成为leader。

处理candidate发来的RequestVote消息是在`handleRequestVote`函数中实现。
- 如果消息中的Term比自身的Term小，拒绝投票。
- 如果`r.Vote != None && r.Vote != m.From`，这意味着在本次任期中已经为另一个peer投过票，拒绝投票。
- 判断消息发送者的日志是否比自己更up-to-date。若不是，拒绝投票。
- 至此，同意投票，更新自身Vote字段。

处理其他peer回复的RequestVoteResponse是在`handleRequestVoteResponse`函数中实现。
- 如果消息的Term比自己的Term小，拒绝处理该消息。这意味着这个消息是过时的。
- 如果同意投票，更新r.votes。如果同意投票的数量过半，自己成为leader；如果拒绝投票的数量过半，自己退化为follower。

成为leader后要定期广播心跳，通过调用`boardcastHeartbeat`来进行广播，在消息中附加自己的Term。

2. Log replication

leader在Prs中记录同一region中各个peer的日志同步情况，在某个peer日志落后时要进行日志复制，在上层应用向raft层Propose日志后也要进行广播日志复制。

发送日志复制的操作在`sendAppend`中处理。`r.Prs[to].Next`代表需要向`to`复制的第一条日志的index，如果这条日志已经被compact了，则转为发送snapshot。发送日志时还需要附带前一条日志的index和term以及自己的commit。

接收到leader的日志复制消息后，相应的处理在`handleAppendEntries`中。
- 若消息的term比自身小，这表示原leader可能陷入网络分区之类的情况，不知道外部已经产生了term更大的新的leader，拒绝。
- 至此，至少我们可以承认该leader的合法性，调用becomeFollower。
- 如果 `m.Index > r.RaftLog.LastIndex()` 这表示自己没有leader想要复制的日志的前一条日志，拒绝，并设置response消息中的index为自己的lastIndex。
- 如果`m.Index`对应的日志的term冲突，则拒绝，同时在response消息中需要附带上自己的日志中该term对应的第一条日志的index。
- 至此，同意复制日志。将消息中与自己冲突的日志全部附加在自身未冲突的日志项之后，更新`stabled`和`commit`。需要注意避免`commit`回退。

leader接收到其他peer对日志复制的回复后，在`handleAppendEntriesResponse`中处理。

- 若消息的term比自身的小，说明消息过期，拒绝。
- 如果消息类型是拒绝：
  
  - 若m.Index为0，直接返回。
  - 若m.LogTerm为0，说明该peer的日志落后自己太多。若不为0，说明该peer的日志和自己在prevIndex处冲突。调整r.Prs[m.From].Next，重新发送一次日志复制消息。
- 如果消息类型是接受：

- 更新r.Prs，视情况是否更新commit。若commit更新，需要立即广播`AppendEntries`。若该peer日志仍落后，需要再次发送日志复制。

3. Implement the raw node interface

`RawNode`包装了一个`Raft`结构，是与上层应用交互的接口。`Campaign`函数让`RawNode`在raft层直接发起一次选举。`Propose`向raft层中添加一条日志。

`Ready`函数将此刻的`RawNode`状态相对于上一次调用`Ready`时的增量包装为一个Ready结构返回。其中：
- `Ready.Entries`表示未被持久化的日志项。
- `Ready.CommittedEntries`表示已经commit未被apply的日志项。
- `Ready.Snapshot`表示`pendingSnapshot`，此后需要清空`pendingSnapshot`。
- `Ready.Messages`表示需要发给其他peer的消息，此后需要清空`RawNode.Raft.msgs`。
由于需要获取增量，所以`HardState`和`SoftState`需要与上一次获得的进行对比，若不同才能写入`Ready`中。因此，`RawNode`结构中需要记录`prevSoftState`、`prevHardState`。

`hasReady`用于判断`Ready`的增量是否存在。

`Advance`在上层应用处理完`Ready`后调用，更新raft层的stable、applied。

## Part B：构建一个可容错的KV服务

`RaftStorage`类似于`StandaloneStorage`，它处理用户发送来的请求。

它启动一个`Raftstore`来驱动raft层，将请求包装为`RaftCmdRequest`，通过`RaftstoreRouter`发送给region中的leader，同时记录一个callback。raft层在日志复制到半数以上节点后apply，我们从callback中读出response，进行相应处理

1. Implement peer storage

我们需要将`Ready`中的unstable entries持久化，需要注意几点
- 已经转化为快照的日志对应的index不能重复持久化。
- 已经持久化但过时的日志需要删除。

在`SaveReadyState`中我们要保存`Ready`中的状态，保存日志、更新`PeerStorage.raftState`，并持久化到raftDB。

2. Implement Raft ready process

`Raftstore`会启动一个`raftWorker`用于驱动raft层。它在一个循环中不停地接收消息，将一些消息调用`RawNode`的接口传递给raft层，并处理`RawNode`的Ready。
```go
	for _, msg := range msgs {
		peerState := rw.getPeerState(peerStateMap, msg.RegionID)
		if peerState == nil {
			continue
		}
		newPeerMsgHandler(peerState.peer, rw.ctx).HandleMsg(msg)
	}
	for _, peerState := range peerStateMap {
		newPeerMsgHandler(peerState.peer, rw.ctx).HandleRaftReady()
    }
```

首先实现`proposeRaftCommand`函数。它将`RaftCmdRequest`中的每一个`Request`与callback绑定，记录在`peer.proposals`数组中，并调用`RawNode.Propose`函数将请求发送到raft层进行共识。

接着，实现`HandleRaftReady`函数。
- 若该peer已经被destroy，直接返回。
- 若该peer没有pending Ready，直接返回。
- 获取Ready，并调用`SaveReadyState`保存状态。如果自身是leader，调用`HeartbeatScheduler`向PD发送心跳同步状态。
- 调用`Send`发送消息给其他peer，处理Ready中的CommittedEntries，调用Advance。

在处理CommittedEntries时，对每个日志项使用一个WriteBatch来保证原子性。从entry.Data中Unmarshal出RaftCmdRequest结构，将其中的请求应用到状态机。需要注意的是请求类型为`CmdType_Snap`表示一个读事务，需要设置`cb.Txn = d.peerStorage.Engines.Kv.NewTransaction(false)`。

将结果封装成`RaftCmdResponse`，从`proposals`中找到对应的callback返回。此后还需要更新`peerStorage.applyState.AppliedIndex`，一同加入WriteBatch，最后持久化。

寻找对应的callback的过程在`peerMsgHandler.handleProposal`中实现。proposal的index和term和该请求在raft层中日志项的index和term是相同的，因此可以直接遍历proposals数组，找到对应的callback。若index相同但term不同说明这个请求可能未完成共识，应该调用`NotifyStaleReq`返回错误信息。


## Part C：raft日志的垃圾回收以及快照

长时间运行的服务器会保存大量的raft日志，这会消耗大量磁盘空间。而且在许多时候我们只需要记录状态机的最终状态，而不需要保存状态机达到该状态所经历的过程，因此我们需要截断日志，进行日志项的垃圾回收，并通过快照同步到其他节点。

我们需要在raft和raftstore两个部分中分别增加对快照的支持。

1. Implement in Raft

在raft层的日志同步过程中，如果leader发现希望复制给follower的日志项已经被删除，应该转变为发送一个快照，并在此后复制剩余的日志项。

首先我们修改`Raft.sendAppend`函数，leader调用这个函数以向follower复制日志。如果r.Prs[to].Next <= r.RaftLog.truncatedIndex，转为发送快照。调用`r.RaftLog.storage.Snapshot`以获取快照。

接下来是`Raft.handleSnapshot`函数，follower在接收到leader发送的快照后调用这个函数处理。首先需要判断这个快照是否是最新的，如果`meta.Index <= r.RaftLog.committed`表示该快照不是最新，拒绝接收。之后follower接受这个快照，需要将applied、commited、stabled、truncatedIndex都修改为meta.Index，并清空本地的日志，将这个快照保存在RaftLog.pendingSnapshot中。还需要根据Metadate.ConfState.Nodes更新Raft.Prs。

pendingSnapshot会在Ready中被保存到raftstore中，需要实现`PeerStorage.ApplySnapshot`函数。先调用`clearMeta`和`clearExtraData`来清空过时的信息，然后更新raftState和applyState这些metadata，将snapState.StateType设置为SnapState_Applying。最后通过regionSched这个channel向region worker发送一个RegionTaskApply，等待其中的Notifier返回。在`HandleRaftReady`中需要先apply快照，后apply日志。

2. Implement in raftstore

raftstore会检查是否需要对日志项进行垃圾回收，并propose一个CompactLogRequest。

CompactLogRequest在AdminRequest中，我们需要修改`peerMsgHandler.proposeRaftCommand`中增加对AdminRequest的处理。这个请求没有callback，因此不需要记录proposal，直接propose到raft层进行共识。

经过共识的CompactLogRequest会在Ready中被apply，我们修改`HandleRaftReady`，在处理CommittedEntries先判断请求中AdminRequest是否为空，若不为空需要特殊处理。这个分支里目前只需要处理CompactLogRequest，先判断只有在adminReq.CompactLog.CompactIndex > d.peerStorage.applyState.TruncatedState.Index的时候才能compact。如果需要compact，首先更新 TruncatedState ，然后调用ScheduleCompactLog添加一个 raftlog-gc 的任务异步处理。






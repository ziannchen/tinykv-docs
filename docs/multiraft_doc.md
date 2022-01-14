# Multi-Raft  

在本项目中我们需要实现基于多个raft集群的KV服务器，其中的每一个raft集群只负责固定范围内的关键字，从而解决raft算法为了追求一致性而影响了并发性的问题。

为了支持这样的设计，我们需要完成三个部分的内容：

1. 为我们的raft算法添加集群成员变更以及领导权变更的功能。

2. 在raftstore中实现配置变更以及region分裂。

3. 实现一个调度算法用于合理地变化配置，获得更好的服务器调度性能。

## Part A：实现集群成员变更以及领导权变更

### 1. 领导权变更

在这个部分我们需要引入两个新的消息类型。`MsgTransferLeader`使得目前的leader检查其继任者的状态，`MsgTimeoutNow`驱动继任者无视其`ElectionTimeOut`立即发动选举。在leader帮助其更新日志并且其他follower的时钟都为随机设定的条件下，它有相当高的几率成为新的leader，而这也正是我们所期望的。

在`handleTransforLeader`函数中定义了leader面对领导权变更时的行为。对于`MsgTransferLeader`，它的`from`成员应当是上层设定的`transferee`。因此应当排除三种错误情况：

- 和一般的local message一样，`from`成员等于自身id的情况。

- leader的`transferee`已经设定并且等于`from`的情况(说明有已经在运行的领导权变更，无需重复执行)。

- 将要确定的transferee不在leader记录的peer中的情况。

在排除以上三种情况之后，我们需要检查`transferee`的`log`是否是最新的，这样才能不与正常的选举规则冲突。若不是，则应调用`sendAppend`函数向它发送缺少的entry。在transferee的记录已经最新的情况下，leader应当调用`sendTimeoutNow`函数向其发送`MsgTimeoutNow`消息使其立刻发动新的选举。

收到`MsgTransferLeader`消息时，follower将其转发给leader。follower收到`MsgTimeoutNow`之后调用`step`函数，参数为`MsgHup`类型的消息。follower接着就会立即将计时器置零并发动选举。

### 2. 配置变更

与论文中相同的是，为了支持配置变更，我们需要一个特殊的entry来记录配置的参数。因此需要在`raft/rawnode.go`中加入`ProposeConfChange`函数，它将entry的类型设置为`EntryConfChange`并将数据设置为`ConfChange`的相应字节，最后向raft层发送一个带有这样的entry，并且类型为`MsgPropose`的消息。

`MsgPropose`类型的消息只会被leader受理，它将其加入自己的log之中并且尝试向follwer发送并提交消息中的entry，这与我们在项目2中所做的并没有什么区别，只需要在`raft.go`中的`handlePropose`函数内加入设置`PendingConfIndex`的代码，将它设置为第一个配置变更的entry的index并暂时不受理其他的配置变更。

对于Part A，我们只需要在`rawnode.go`中加入`ApplyConfChange`函数，用于处理一个节点的参数变化。为了实现这个功能，我们在raft层中加入了`addNode`与`removeNode`函数，它们都具有相当直观的实现。

`AddNode`函数在raft的peers中添加相应的peer，并且初始化它的`Progress`，最后将raft的`PendingConfIndex`清空，使其能够处理下一个配置变更的消息。

`removeNode`函数则较为复杂一些：

(1) 首先检查需要删除的raft是否在当前配置之中，若不在则直接返回，若在则转(2)。

(2) 检查自身是否为需要删除的peer，若是则清空自己的peers并且返回。若不是则转(3)。

(3) 删除相应的peer，并检查自己的peer数量。若为1，则它需要成为这个集群的leader。

(4) 删除peer后，原先不满足commit条件的entry有可能已经满足，调用`maybeAdvanceCommit`函数进行提交。

(5) 最后清空`PendingConfIndex`。


## Part B: 在raftstore中实现配置变更以及region分裂

### 1. Propose transfer leader

`TransferLeader`直接被发送给集群中的leader，不需要被复制到多个peer上，因此不需要记录在proposal中，直接调用`RawNode.TransferLeader`。

### 2. Implement conf change in raftstore

在raft层需要在`PendingConfIndex`字段中记录类型为`EntryType_EntryConfChange`的日志项的index，当配置变更被apply后落实到raft层后清空这个字段。在propose的时候，如果这个字段不为None则直接返回，表示还有一次配置变更没有完成。之后调用`RawNode.ProposeConfChange`。

`HandleRaftReady`中由于存在配置变更的情况，在应用了Ready中的快照后region可能发生改变，。因此在`SaveReadyState`之后需要修改d.ctx.storeMeta。

处理CommittedEntries的部分代码也需要重构以特殊处理`EntryType_EntryConfChange`类型的日志项。
```go
if len(rd.CommittedEntries) > 0 {
		for _, e := range rd.CommittedEntries {
			kvwb := new(engine_util.WriteBatch)
			switch e.EntryType {
			case eraftpb.EntryType_EntryNormal:
				d.processNormalEntry(e, kvwb)
			case eraftpb.EntryType_EntryConfChange:
				d.processConfChangeEntry(e, kvwb)
			}
			if d.stopped {
				return
			} else {
				d.peerStorage.applyState.AppliedIndex = e.Index
				log.Infof("%s successfully apply entry from raft peer %d [index: %d]", d.Tag, d.PeerId(), e.Index)
				kvwb.SetMeta(meta.ApplyStateKey(d.regionId), d.peerStorage.applyState)
			}
			kvwb.WriteToDB(d.peerStorage.Engines.Kv)

		}
	}
```
在添加或是删除node时，都需要先判断目标node是否已存在当前的region的peers中。配置变更之后，还需要增加`region.RegionEpoch.ConfVer`，修改peerCache。如果删除了自己，需要调用`destroyPeer`并直接返回，此后的日志项也不需要再处理。 新建的peer无需我们关心，他会在收到leader的心跳后被初始化，之后接收快照来达到接近region内其他peer的状态。

### 3. Implement split region in raftstore

为了支持multi-raft，TinyKV对于较大的region进行分裂，有助于均衡集群内各个机器的负载。

在propose的时候需要检查SplitKey是否合法，不合法则直接返回。

region分裂需要将region中的每一个peer都分裂为两个。split消息会被复制到region中的所有peer上，并由每一个peer在apply时将自己分裂，这样就完成了分裂，而且他们共用同一个store。

Split属于AdminRequest，在处理的时候需要先依次检查RegionId、RegionEpoch、SplitKey是否都合法，若检查通过则开始分裂。

首先，原先的region从splitkey处分裂为两个，key range分别为[startKey, splitkey)和[splitkey, endkey)。因此需要新建一个region，其中RegionEpoch中的ConfVer和Version都设为1，在原先的region中增加RegionEpoch.Version。

然后需要修改d.ctx.storeMeta，删除旧的region信息，记录两个新的region信息。调用`WriteRegionState`持久化两个新的region，state要设置为PeerState_Normal。

最后，调用createPeer新建一个peer，并在route中注册，向其发生一个MsgTypeStart信息。不要忘了处理proposal。


## Part C：实现调度器

调度器根据集群的负载选择每个region中每个副本的最佳位置，因此需要获取整个集群的所有关键信息，并定期检查这些信息以做出调度。

### 1. Collect region heartbeat

调度器要求每个region中的leader定期向它发送心跳信息。

我们需要实现`RaftCluster.processRegionHeartbeat`函数，更新本地保存的region信息。

在此之前我们先实现一个`checkRegionEpoch`来判断新的region信息是否合法且是否过期。
```go
func checkRegionEpoch(region *core.RegionInfo, origin *core.RegionInfo) error {
	if region.GetRegionEpoch() == nil || origin.GetRegionEpoch() == nil {
		return errors.Errorf("region is nil")
	}

	if util.IsEpochStale(region.GetRegionEpoch(), origin.GetRegionEpoch()) {
		return ErrRegionIsStale(region.GetMeta(), origin.GetMeta())
	}

	return nil
}
```

我们先需要根据心跳中的regionInfo的regionId先获取本地保存的region信息。若本地存在且新的region过期，直接返回错误。若不存在，遍历本地所有包含了新region的key range的region信息，检查新的region对于它是否过期，若存在过期则直接返回错误。

若新的region未过期，则更新本地存储，包括了region tree和store status。


### 2. Implement region balance scheduler

这一部分需要实现`balanceRegionScheduler.Schedule`函数，它负责进行region的调度，返回一个MovePeerOperator，将某个peer在store之间移动。

首先寻找合适的可移动的region，在此之前需要寻找所有合适的store，即：正在运行并且 downTime 小于 store的最大downTime 的所有store。如果suitableStores数量小等于1，不能进行MovePeer，则停止本次调度返回nil。之后将所有suitableStores按照拥有的region大小从大到小排序。

接着我们遍历suitableStores，在每个store上按照PendingRegionsWithLock -> FollowerWithLock -> LeadersWIthLock的优先级选出第一个满足条件的region，这便是我们想要移动的region。如果没有找到合适的region，或是该region的副本数小于MaxReplicas，则停止本次调度返回nil。

最后我们寻找接收peer的目标store，即拥有的region大小最小且不在suitableStores中的store。如果sourceStore.GetRegionSize()-targetStore.GetRegionSize() <= region.GetApproximateSize()*2，则本次调度不能进行。之后调用`AllocPeer`在targetStore上新建一个peer，调用`CreateMovePeerOperator`生成operator返回。









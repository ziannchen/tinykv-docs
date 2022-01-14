# Project 4: Transactions

&emsp;&emsp;在之前的几个 projects 中，我们已经构建起了基于 multi-raft 的分布式KV数据库。在 project 4 中，我们将构建一个事务系统以应对多个 clinet 的并发请求并保证快照隔离（snapshot isolation）。

&emsp;&emsp;我们将基于`Percolator`的两阶段提交协议来构建我们的事务模型。

## Part A：MVCC
&emsp;&emsp;在part A中我们需要实现MVCC即多版本并发控制（multi-version concurrency control）。`tinykv`的底层存储`Badger`为我们提供了3个`column family`：`CfDefault`、`CfLock`、`CfWrite`，对应了论文中提到的3个列：`data`、`lock`、`write`。

- `CfDefault`：用于暂时存储对应key的value值，将由MVCC机制来决定之后该值是否被commit或者被delete（即回滚）
- `CfLock`：用于存储锁，如果某key存在对应key的lock，说明它正在被某个事务修改。
- `CfWrite`：用来存储key的每个版本value值的提交时间(commit version)
- `WriteKind`：Lock和Write都有一个write kind属性来记录本次对key进行了什么样的修改。有三种，分别是Put、Delete和Rollback。

我们依据论文，使用`Badger`提供的读写api来完成 `transaction.go` 中的`MvccTxn`和它的方法。
```golang
// MvccTxn groups together writes as part of a single transaction. It also provides an abstraction over low-level
// storage, lowering the concepts of timestamps, writes, and locks into plain keys and values.
type MvccTxn struct {
	StartTS uint64
	Reader  storage.StorageReader
	writes  []storage.Modify
}
```
MvccTxn struct中需要包含事务开始的timestamp `StartTS`，底层存储的`Reader`和用来将一系列写操作原子化的`writes`数组。

这里需要注意一下`CfDefault`、`CfLock`、`CfWrite`这三个`column family`的key/value编码方式。
| |`key`|`value`|
|:-:|:-:|:-:|
|`CfDefault`|(key, StartTS)|value|
|`CfLock`|key|lock|
|`CfWrite`|(key, commitTS)|write|

写、删除只需构建一个`storage.Modify`结构添加到`MvccTxn`的`writes`中即可。读操作的实现稍微复杂。

- GetValue
    >GetValue finds the value for key, valid at the start timestamp of this transaction.
I.e., the most recent value committed before the start of this transaction.

    根据注释，这个方法要求我们读出在事务开始之前最晚写入的值。在论文中提到：一个值只有被commit后才对其他事务可见，表现为一个write记录。很容易想到应该去读 `CfWrite` 这个CF。又根据方法EncodeKey的编码规则，key按照userkey升序，userkey相同时按照timestamp降序，我们可以适当编码key并调用seek，使迭代器指向 `CfWrite` 这个CF中事务开始之前最晚写入的值。此后还需要注意两点：得到的write的key可能不等于userkey，需要特判；判断write的类型可能为`WriteKindDelete`，这表示这个userkey已经被删除，应该返回nil。最后根据write中的StartTS从`CfDefault`中读取value。

- CurrentWrite
    
    我们需要搜索：对于key，startTS和事务的开始时间相同的write。

    我们可以直接遍历 `CfWrite` 这个CF从中找到符合条件的write。从中找到由于userkey相同时key的编码按timestamp降序，我们在调用EncodeKey时将timestamp设置为`^uint64(0)`，之后即可遍历对于userkey的所有write。

- MostRecentWrite
    
    我们需要找到对于给定的userkey最迟的write。使用和`CurrentWrite`中相似的编码方式，在 `CfWrite` 中查找即可。

## Part B：KvGet, KvPrewrite, and KvCommit

&emsp;&emsp;在Part B中我们需要实现`KvGet`, `KvPrewrite`, 和`KvCommit`三个request handler。

---

`KvGet` 根据事务的StartTS读出key对应的value，应当判断该key在读事务的StartTS处是否被其他事务上锁，即查找key是否存在锁且锁的startTS是否早于事务的startTS。
```go
	txn := mvcc.NewMvccTxn(reader, req.GetVersion())
	lock, err := txn.GetLock(key)
	if err != nil {
		return nil, err
	}

	// lock's ts <= txn' ts means the key are locked before txn start and not commited yet
	if lock != nil && lock.Ts <= txn.StartTS {
		resp.Error = &kvrpcpb.KeyError{
			Locked: &kvrpcpb.LockInfo{
				PrimaryLock: lock.Primary,
				LockVersion: lock.Ts,
				Key:         key,
				LockTtl:     lock.Ttl,
			},
		}
		return resp, nil
	}
```
若key未被上锁，调用 `GetValue` 读取。

---

`KvPrewrite` 先检查每一个key是否可以被写入，并对每一个key上一个指向primary key的锁 。 `KvCommit` 检查所有锁是否仍有效并commit所有key。它们共同构成了 `Percolator` 事务的两阶段提交。

`KvPrewrite` 需要将value写入每个key的data列，并对该key上锁以防止其他事务的写冲突（即分别写入 `CfDefault` 和 `CfLock`）。在此之前需要判断没有其他事务也对相同的key上锁或写入。我们应循环检查每一个key是否可以被合法的prewirte。

- 首先应该判断是否有其他事务在本事务开始之后提交了相同key的commit，若有则需要记录该冲突并检查下一个key。这里可以调用 `MostRecentWrite` 来找到对于key最近的commit记录，再判断是否冲突（冲突即commitTs > 事务的version）。
- 检查key是否在事务开始之前被加锁。
- 在本地事务中写入value、lock。

循环结束后应判断是否出现冲突，若出现冲突则直接abort本次事务，防止写-写冲突。若没有冲突，调用`storage.Write`提交对于`CfDefaule`和`CfLock`的写入。

`KvCommit` 需要先检查所有的锁的状态。

- 若锁不存在，表明可能原事务rollback或锁过期，则本次commit失败。
- 若锁存在，但startTS不等于本次commit请求的startVersion，表明该锁不属于我们期望commit的原事务，而是被其他事务加的锁，本次commit失败。此时需要在resp.Error中的retryable字段处记录，让client重试。

若锁状态符合要求，删除锁并写入write以提交commit。这两个操作需要加锁latch来保证原子性。

## Part C：KvScan, KvCheckTxnStatus, KvBatchRollback, and KvResolveLock
---

`KvScan` 一次读取 `multiple key/value pairs`。

badger自带的迭代器只支持在指定的CF中按key顺序迭代，而事务要求我们只能读取在startTS之前提交的value，这需要我们对于每一个key先找到符合条件的write记录，再去`CfDefault`列中获取对应的value。

下面我们实现一个支持事务的迭代器。
```go
// NewScanner creates a new scanner ready to read from the snapshot in txn.
func NewScanner(startKey []byte, txn *MvccTxn) *Scanner {
	// Your Code Here (4C).
	scan := Scanner{
		txn:     txn,
		iter:    txn.Reader.IterCF(engine_util.CfWrite),
		lastKey: []byte{},
	}

	scan.iter.Seek(EncodeKey(startKey, txn.StartTS))
	return &scan
}
```
它包含一个`CfWrite`列的迭代器。以及一个 `lastKey` 字段，它记录找到的上一个符合条件的write所对应的Key。

最核心的是`Scanner.Next`函数，它返回下一个符合事务条件的key/value。

首先移动迭代器，使其指向第一个满足 key != lastKey 的write记录。
```go
    var currentUserKey []byte
	for scan.iter.Valid() {
		currentUserKey = DecodeUserKey(scan.iter.Item().KeyCopy(nil))
		if !bytes.Equal(currentUserKey, scan.lastKey) {
			break
		}

		scan.iter.Next()
	}
```
循环使用seek寻找 `EncodeKey(currentUserKey, scan.txn.StartTS)` 或之后的第一个 key。需要注意在 `currentUserKey` 处可能没有在 `txn.StartTS` 之前的commit，这会使迭代器指向下一个userKey最早的commit。
```go
	// now we find a key that userKey != lastKey, seek commitTs < startTs
	for currentCommitTs > scan.txn.StartTS {
		scan.iter.Seek(EncodeKey(currentUserKey, scan.txn.StartTS))
		if !scan.iter.Valid() {
			return nil, nil, nil
		}

		currentUserKey = DecodeUserKey(scan.iter.Item().KeyCopy(nil))
		currentCommitTs = decodeTimestamp(scan.iter.Item().KeyCopy(nil))
	}
```
这样我们便找到了下一个commitTS符合要求的userkey，只需去 `CfDefault` 中读取value即可。

此后我们可以简单的实现 `KvScan` ，只需根据startKey和limit建立事务迭代器实例，并循环调用`Next`获取key / value对。

---

`KvCheckTxnStatus` 报告事务的状态，并回滚过期的锁。

需要注意 `CheckTxnStatusResponse` 中给出的注释。
```go
    // Three kinds of txn status:
    // locked: lock_ttl > 0
    // committed: commit_version > 0
    // rolled back: lock_ttl == 0 && commit_version == 0
```

为了确认事务是否以及被回滚或提交，应当先查找 (primary key, lockTS) 对应的write记录，这里可以使用 `CurrentWrite` 函数，这要求我们使用req.lockTS来初始化本次事务。若存在write记录，设置 `resp.Action = kvrpcpb.Action_NoAction`。还需要判断write的类型，若类型为 `WriteKindRollback` 则需要在resp中的 `CommitVersion` 记录write
的commitTS。

之后需要确认锁的状态。若锁不存在，则需要回滚primary key，并记录 `resp.Action = kvrpcpb.Action_LockNotExistRollback`。若锁存在，判断ttl是否过期。若过期则删除锁和value并回滚，并记录 `resp.Action = kvrpcpb.Action_TTLExpireRollback`。这里判断ttl需要使用 `physical time`。

若锁仍有效，在 `resp.LockTtl` 字段中记录剩余ttl并返回。

---

`KvBatchRollback` 批量回滚事务。检查每个key是否仍被原事务上锁，如果是则删除锁和value，回滚该事务。

循环检查每一个key
- 检查该key是否已经被回滚或提交。若已被回滚则忽略该key，若已被commit则在`resp.Error.Abort` 中记录错误信息并直接返回。可以调用 `MostRecentWrite` 来获取最近提交的write。

- 检查该key是否存在锁。
  - 若不存在，直接回滚该key。
  - 若存在。
    - 锁不属于原事务，视为出错，回滚该事务。
    - 锁仍属于原事务，执行所有回滚步骤，删除锁、value，再回滚。

若循环过程中发生出错则不将txn中的writes写入storage，即一个key出错视为全部出错。

---

`KvResolveLock` 将查找属于具有给定开始时间戳的事务的所有锁，要么将他们全部回滚，要么全部commit。

首先要根据startVersion找到所有的lock，我们遍历`CfLock`找到所有满足 `lock.Ts == req.StartVersion` 的lock。之后根据 `req.CommitVersion` 是否为零决定回滚或是commit，相应操作可以直接调用 `KvBatchRollback` 或者 `KvCommit` 。
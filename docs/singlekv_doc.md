# Standalone KV
 
 在本项目中，我们将在 `column family` 的支持下构建一个独立的 key / value 存储gRPC服务。在 `kv/main.go` 中我们初始化了一个 `gRPC` server，它包含了一个 `tintkv.Server` ，提供名为 `TinyKV` 的 `gRPC` 服务。
 ```go
 	server := server.NewServer(storage)

	var alivePolicy = keepalive.EnforcementPolicy{
		MinTime:             2 * time.Second, // If a client pings more than once every 2 seconds, terminate the connection
		PermitWithoutStream: true,            // Allow pings even when there are no active streams
	}

	grpcServer := grpc.NewServer(
		grpc.KeepaliveEnforcementPolicy(alivePolicy),
		grpc.InitialWindowSize(1<<30),
		grpc.InitialConnWindowSize(1<<30),
		grpc.MaxRecvMsgSize(10*1024*1024),
	)
	tinykvpb.RegisterTinyKvServer(grpcServer, server)
```
其中，`server` 依赖于一个 `storage` ，它是一个接口，可以根据配置文件选择raft存储实现或者单机存储实现。


我们通过两步来完成这个项目：
 1. 实现一个单机存储引擎。
 2. 实现原始的kv服务handler。


## 1. 实现一个单机存储引擎。
我们需要用一个单机存储引擎来实现这个接口：
```go
// Storage represents the internal-facing server part of TinyKV, it handles sending and receiving from other
// TinyKV nodes. As part of that responsibility, it also reads and writes data to disk (or semi-permanent memory).
type Storage interface {
	Start() error
	Stop() error
	Write(ctx *kvrpcpb.Context, batch []Modify) error
	Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```
代码位于 `kv/storage/standalone_storage/standalone_storage.go` 中。我们底层的存储服务使用了 `badger` ，因此实现 `Storage` 接口只需对 `badger` 的api进行封装。
```go
type StandAloneStorage struct {
	// Your Data Here (1).
	db *badger.DB
}
```
`Reader` 方法中直接返回一个 `StandAloneStorageReader` 即可，其中包含了一个 `badger.Txn` 是一个事务接口，提供了读时的快照。

`Write` 方法需要一次性 `batch` 中的所有写入或是删除操作写入数据库。需要使用一个 `WriteBatch` 记录下所有写操作，最后使用 `WriteBatch.WriteToDB` 方法一次性写入 `badger`。`badger` 并不支持 `column family`，因此tinykv在不同 `column family` 上的区分只是为把不同 `column family` 的key在编码时加上不同的前缀。

## 2. 实现原始的kv服务handler。

我们需要使用刚刚实现的单机存储引擎为 `server` 实现 `RawGet/Put/Delete/Scan` 4个方法。

`RawGet` 方法需要使用 `RawGetRequest` 中的 `Context` 初始化一个 `Reader`，此后直接读取。

`RawPut/Delete` 这两个写操作需要使用 `storage.Modify` 先进行封装再调用 `Write` 写入。

`RawScan` 需要初始化一个迭代器 `Reader.IterCF` ，之后直接循环读取。



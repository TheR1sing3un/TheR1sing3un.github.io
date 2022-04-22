# Distributed Txn Lab翻译


# Distributed-txn

# Lab1

## The Design

在lab1中，我们已经完成了Raft日志引擎以及存储引擎。使用Raft日志引擎，事务日志能够"可靠的"持久化，并且可以在故障转移后恢复状态。在这个章节，我们将讨论**分布式事务层**的设计与实现。raft日志引擎已经提供了**持久性(Durability)**以及状态恢复保险，事务层需要保证**原子性(Atomicity)**以及并发正确或者称为**隔离性(Isolation)**。

percolator协议被用于保证**原子性(Atomicity)**，以及使用全局时间戳来对并发事务的执行进行顺序排序。基于这些，可以为客户端提供一个强隔离级别，通常被成为**快照隔离(Snapshot Isolation)**或**可重复读(Repeatable Read)**。percolator协议在`tinysql`和`tinykv`的服务端都被实现了，以及全局事务时间戳分配由`tinyscheduler`服务端完成，以及所有逻辑时间戳都是单调递增的。在这个lab中，我们将要实现`tinykv`中的percolator部分，或者说是分布式事务的参与者部分。

### Implement The Core Interface of  `StandAloneStorage`

尝试去实现`kv/storage/standalone_storage/standalone_storage.go`中的缺失代码，这些代码部分被使用如下注释标记：

```
// YOUR CODE HERE (lab1).
```

完成这些部分之后，运行`make lab1P0`命令来检查是否所有的测试用例都通过了。注意事项：

- 由于[badger](https://github.com/dgraph-io/badger)被用于存储引擎，因此常见的用法可以在其文档和存储库中找到。
- `badger`存储引擎不支持[`column family`](https://en.wikipedia.org/wiki/Standard_column_family)。`percolator`事务模型需要列族，在`tinykv`中，列族相关功能已经被包装在`kv/util/engine_util/util.go`中，当处理`storage.Modify`时，被写入到存储引擎中的键应该使用`KeyWithCF`函数来编码，考虑到它所期望的列族。在`tinykv`中有两种`Modify`类型，检查`kv/storage/modify.go`来获取更多的信息。
- `Scheduler_client.Client`在`standAloneServer`中不会被使用，因此它可以被跳过。
- 考虑下`badger`提供的拥有`txn`特性的读写接口。去`BadgerReader`中获取更多消息。
- 一些测试用例可能有助于理解存储接口的用法。

---

## Implement Percolator In TinyKV

> 实现TinyKV中的percolator

事务处理流程如下：

![image-20220419180000791](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220419180000791.png)

用户发送写请求比如说`Insert`到`tinysql`服务端，该请求被解析以及执行，然后用户数据被从行数据转变为键值对。然后事务模块将负责提交该键值对到存储引擎。由于不同的键可能分布在不同的区域，而这些区域可能又分布在不同的`tinykv`服务器上，因此事务引擎必须确保提交过程最终成功或者什么也不做，这就是percolator协议。

考虑提交一个多键的分布式事务。首先，将被提交的键中有一个被选择作为**主键(Primary Key)**，然后事务状态仅取决于**主键**的状态。换句话说，事务将被认为成功提交只有当**主键(Primary Key)**被成功提交。提交过程被分成两阶段，第一阶段是**Prewrite**，第二阶段是**Commit**。

### Prewrite Phase

>预写阶段

在键值对准备好之后，事务进入到`Prewrite`阶段，在该阶段所有的键将被预写到不同的`tinykv`服务器包含不同的区域领袖。每一个`Prewrite`请求将会被一个`tinykv`服务器处理，`预写锁(Prewrite Lock)`将会被放入存储引擎中的每个键的`锁列族(lock column family)`中。如果任何的预写处理失败那么提交处理将会立即失败，然后任何预写锁都会被清理。

### Commit Phase

> 提交阶段

如果所有的预写处理都成功了，事务将会进入`提交(Commit)`阶段，该阶段中`主键(Primary Key)`将会被第一个提交。`提交(Commit)`处理操作会在存储引擎中提交一个`写记录(Write Record)`到`写列族(Write cloumn family)`

并且解锁`锁列族(lock column family)`中的`预写锁(Prewrite Lock)`。如果主键的提交操作已经成功那么事务将被认为成功提交了并且将发送成功相应到客户端。所有被称为`副键(Secondary Keys)`将会在后台任务中被**异步**

提交。

在一般的情况下，两阶段已经足够了，但是没有做**故障恢复**。在分布式环境下任何地方都有可能失败，比如说`tinysql`服务器可能在完成`两阶段提交(Two Phase Commit)`前发生了失败，`tinykv`服务器也一样。当一个新的区域领袖被选举出来然后开始处理请求，我们怎么**恢复未完成的事务并且确保正确性**？

### Rollback Record

> 回滚记录

一旦事务被决定为失败，它留下的锁应该被清理以及一条回滚记录应该被放入到存储引擎以防止将来可能的预写或者提交(考虑到网络延迟等)。因此如果将回滚记录放在事务的主键上，则事务状态被确定为回滚。它将永远不会被提交，并且必须失败。

### Check The Transaction Status

> 检查事务状态

如上所述，事务的最终状态只取决于`主键(Primary Key)`或者`主锁(Primary Lock)`的状态。如果一些事务状态没有被决定，它需要检查它的`主键(Primary Key)`或者`主锁(Primary Lock)`的状态。如果有主键的提交或者回滚记录，然后我们可以安全的认为事务已经被提交了或者回滚了。如果主锁仍然存在或者没有没有过期，那么事务提交仍是可能的，仍在进行。如果没有锁记录和提交/回滚记录，那么事务状态就是未确定的，我们可以选择等待或者写入一个回滚记录来防止之后的提交，然后该事务就决定回滚了。每一个在两阶段提交中的`预写锁(Prewrite Lock)`将拥有一个`存活时间(Time To Live)`字段，如果锁的ttl已经过期了那么该锁将会被并发命令比如说`CheckTxnSrtatus`来回滚，之后该事务最终一定会失败。

### Process Conflicts And Do Recovery

> 处理冲突以及恢复

不同的事务协调者位于不同的**tinysql**服务端，读和写请求可能在它们之中发生冲突。考虑如下情况，一个事务`txn1`在键`k1`上有一个预写锁，另一个读事务`txn2`尝试读取键`k1`，那么`txn2`该如何处理呢？

读和写请求无法继续因为锁记录阻塞了它们，有如下几种可能性：

- 锁属于的事务已经提交了，同样也可以提交锁，接下来在该键上阻塞的请求可以继续。
- 锁属于的事务已经回滚了，同样也可以回滚锁，接下来在该键上阻塞的请求可以继续。
- 锁属于的事务仍在继续，阻塞的请求必须等待该事务完成。

这些冲突处理在`tinysql/tinykv`集群中被称为`Resolve`。一旦请求被一个别的事务的预写锁阻塞的时候，resolve将用于帮助确定锁的状态然后帮助该锁提交或者回滚，以至于该请求可以继续。`Resolve`操作隐式的协助了**事务恢复**。假设一个情况，事务调度器完成了预写锁然后`tinysql`服务器宕机了以及那些留下来的锁将在别的`tinysql`服务器的并发事务请求中被解决。

---

# Lab2

我们将会在tinykv服务端实现上述接口以及处理逻辑。

## Code

### The `Command` Abstraction

> 命令的抽象

在`kv/transaction/commands/command.go`中，那里有所有事务命令的接口。

```go
// Command is an abstraction which covers the process from receiving a request from gRPC to returning a response.
type Command interface {
	Context() *kvrpcpb.Context
	StartTs() uint64
	// WillWrite returns a list of all keys that might be written by this command. Return nil if the command is readonly.
	WillWrite() [][]byte
	// Read executes a readonly part of the command. Only called if WillWrite returns nil. If the command needs to write
	// to the DB it should return a non-nil set of keys that the command will write.
	Read(txn *mvcc.RoTxn) (interface{}, [][]byte, error)
	// PrepareWrites is for building writes in an mvcc transaction. Commands can also make non-transactional
	// reads and writes using txn. Returning without modifying txn means that no transaction will be executed.
	PrepareWrites(txn *mvcc.MvccTxn) (interface{}, error)
}
```

`WillWrite`创建该请求所需写入的内容，`Read`将执行该命令所需的读取请求。`PrepareWrites`用来为这个命令构建实际的写内容，它是写命令处理的核心部分。由于每个事务都有它唯一的标识符，即`start_ts`也就是分配的全局时间戳，因此可以使用`StartTs`来返回当前命令的值。

尝试理解整个客户端请求处理的过程(事务命令处理和`raftstore`日志提交/应用)。事务命令导致一些写突变，这些突变将会被转为raft命令请求，然后发送到`raftstore`，处理之后，在`raftstore`中提交并应用处理，事务命令被认为时成功的，结果发回给客户端。

这个lab忽略了`raftstore`，对于该兴趣的同学，可以去[TinyKV](https://github.com/tidb-incubator/tinykv)看看。

![image-20220419205656278](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220419205656278.png)

---

### Implement the `Get` Command

> 实现Get命令

`KvGet`用来执行来获取特定键的值，尝试实现`kv/transaction/commands/get.go`中缺失的代码，这些代码部分被使用如下标记：

```
// YOUR CODE HERE (lab1).
```

---

### Implement the `Prewrite` and `Commit` Commands

> 实现`预写(Prewrite)`以及`提交(Commit)`命令

关于我们的事务引擎有两个最重要的接口，尝试去实现`kv/transaction/commands/prewrite.go`和`kv/transaction/commands/commit.go`中缺失的代码。这些代码部分被使用如下标记：

```
// YOUR CODE HERE (lab1).
```

注意事项：

- 可能会有重复的请求，因为这些请求是由tinysql服务端发送的RPC请求。
- `StartTS`只是一个事务开始的标识。
- `CommitTS`是一个记录的预期提交的时间戳。
- 考虑读写处理冲突。

完成这两部分，运行`make lab2P1`来检查是否所有的测试都通过了。

### Implement the `Rollback` and `CheckTxnStatus` Commands

> 实现`回滚(Rollback)`和`检查事务状态(CheckTxnStatus)`两个命令

`回滚(Rollback)`用于解锁键，以及添加一个键的`回滚记录(Rollback Record)`。`检查事务状态(CheckTxnStatus)`用于查询指定事务的主键锁的状态。尝试实现`kv/transaction/commands/rollback.go`和`kv/transaction/commands/checkTxn.go`中缺失的代码。这些代码部分被使用如下标记：

```
// YOUR CODE HERE (lab1).
```

完成这些部分，运行`make lab2P2`来检查是否所有的测试都已经通过。注意事项如下：

- 考虑到查询的锁不存在的情况。
- 可能会有重复的请求，因为这些请求是由tinysql服务端发送的RPC请求。
- 在`CheckTxnStatusResponse`中有三种不同的响应操作。`Action_TTLExpireRollback`意味着目标锁已经被回滚了因为锁已经过期了，以及`Action_LockNotExistRollback`意味着目标锁不存在以及回滚记录已经被写入了。

### Implement the `ResolveLock` Command

> 实现`ResolveLock`命令

`Resolve`用于提交或者回滚锁当它关联的事务的状态已经被确定了。尝试实现`kv/transaction/commands/resolve.go`中缺失的代码。这些代码部分被使用如下标记：

 ```
// YOUR CODE HERE (lab1).
 ```

当完成这些部分时，运行`make lab2P3`来检查是否所有测试都已经被通过了。注意事项如下：

- 事务状态已经在输入请求参数中确定。
- `rollbackKey`和`commitKey`是有用的。

当所有的命令和测试都完成了，运行`make lab2P4`来检查是否别的情况测试也能通过。在下一个lab我们尝试实现`tinysql`服务端中`percolator`协议中的事务协调者部分，以及将被使用的所有命令。



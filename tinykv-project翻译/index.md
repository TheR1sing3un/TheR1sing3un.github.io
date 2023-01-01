# TinyKV Project翻译


# TinyKV-Project翻译

## Project1 StandaloneKV

> 项目1 单机KV存储系统

在这个项目中，您将在列族(column family)的支持下构建一个独立的键/值存储的gRPC服务。单机意味着只有一个节点，不是一个分布式的系统。列族(下列缩写为CF)是一个类似于键命名空间的术语，即在不同的列族中相同的键的值是不同的。你可以简单地将多列族视为单独地迷你数据库群。它用于支持project4的事务模型，你将会了解为什么TinyKV会需要CF的支持。

该服务支持四种基本的操作：Put/Delete/Get/Scan。它维护一个简单的KV键值对数据库。键和值都是字符串类型。`Put`替换数据库中指定CF的特定键的值，`Delete`删除指定CF的键的值，`Get`取得指定的CF中的键的当前值，以及`Scan`取得指定CF中的一系列键的值。

这个项目可以分为两个步骤，包括：

1. 实现一个单机存储引擎。
2. 实现原始键值服务的处理程序。

### The Code

`gRPC`服务端在`kv/main.go`中被初始化以及该服务端还包括一个用来提供叫`TinyKv`的`gRPC`服务的`tinykv.Server`。它是在`proto/proto/tinykvpb.proto`中由protocol-buffer定义，以及rpc请求和响应的细节在`proto/proto/kvrpcpb.proto`中被定义。

总的来说，你不需要更改proto文件因为所有需要的变量已经为你定义好了。但是如果你仍然需要更改，你可以修改proto文件然后使用`make proto`命令来更新在`proto/pkg/xxx/xxx.pb.go`中生成的相关代码。

另外，`Server`依赖于`Storage`，也就是你需要在`kv/storage/standalone_storage/stanalone_storage.go`中实现一个单机存储引擎接口。一旦在`StandaloneStorage`实现了接口`Storage`，你可以使用该接口来给`Server`实现一个原始的键值对服务。

### Implement standalone storage engine

> 实现单机存储引擎

第一个任务就是要去实现badger的键值对包装器。gRPC服务器依赖于`kv/storage/storage.go`中定义的`Storage`。在这个上下文中，单机存储引擎只是一个badger的键值对API包装器，它提供两个方法：

```go
type Storage interface {
    // Other stuffs
    Write(ctx *kvrpcpb.Context, batch []Modify) error
    Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```

`Write`应该提供一种对内部状态应用一系列修改的方法，在这种情况下，内部状态就是一个badger实例。

`Reader`应该返回一个`StorageReader`用来支持在快照上进行键值对的get和scan。

以及你现在不需要考虑`kvrpcpb.Context`，它会在接下来的项目中使用。

> 提示：
>
> - 你应该使用badger.Txn来实现`Reader`函数，因为badger提供的事务处理程序可以提供键和值的一致快照。
> - Badger不支持列族。引擎工具(kv/util/engine_util)通过给键添加前缀来模拟列族。例如：一个属于列族`CF`的键`key`被以`${cf}_${key}`来存储。它包装了`badger`以提供列族的操作，以及同样提供了许多有用的辅助函数。因此，你应该通过`engine_util`提供的方法来执行读/写操作。请阅读`util/engine_util/doc.go`来了解更多。
> - TinyKV使用`badger`的原始版本，所以请只使用`github.com/Connor1996/badger` 而不是`github.com/dgraph-io/badger`。
> - 不要忘记在丢弃前调用badger.Txn的`Discard()`以及关闭所有的迭代器。

---

### Implement service handlers

> 实现服务处理程序

该项目的最后一步是去使用实现的存储引擎来构建原始的键值服务处理程序，包括RawGet/RawScan/RawPut/RawDelete。该处理器已经为你定义了，你只需要完善`kv/server/raw_api.go`中的实现。一旦完成，记得运行`make project1`来通过所有的测试套件。

---

---

## Project2 RaftKV

Raft 是一个易于理解的一致性算法。你可以在[Raft网站](https://raft.github.io/)上阅读关于Raft本身的材料，一个Raft的交互式可视化，和其他资源，包括[扩展Raft论文](https://raft.github.io/raft.pdf)。

在本项目中，您将实现一个基于 Raft 的高可用 kv 服务器，这不仅需要您实现 Raft 算法，而且还需要在实际中使用它，这将给您带来更多的挑战，如使用 `badger` 管理 Raft 的持久状态，为快照消息添加流量控制等。

该项目有三个部分需要你做，包括：

- 实现基本的Raft算法
- 基于Raft构建一个容错的KV服务端
- 增加对Raft日志垃圾回收和快照的支持

### PartA

#### The Code

在这一部分中，您将实现基本的Raft算法。你需要实现的代码在 `raft/`下。在 `raft/`中，有一些框架代码和测试用例等着你。您将在这里实现的 Raft 算法与上层应用程序有一个设计良好的接口。此外，它使用一个逻辑时钟(在这里命名为 tick)来测量选举和心跳超时，而不是物理时钟。也就是说，不要在 Raft 模块本身中设置计时器，上层应用程序通过调用 `RawNode` 负责推进逻辑时钟。除此之外，消息的发送和接收以及其他事情都是异步处理的，什么时候实际执行这些事情也取决于上层应用程序(请参阅下面的更多细节)。例如，Raft 不会阻塞等待任何请求消息的响应。

在实现之前，请先检查这部分的提示。同样的，你应该粗略的看一下proto文件`proto/proto/eraftpb.proto`。Raft发送和接收消息以及相关的定义的结构都在文件中，您将使用他们进行实现。

这个部分可以在三步内完成，包括：

- 领袖选举
- 日志复制
- 原始节点接口

#### Implement the Raft algorithm

> 实现Raft算法

`raft/raft.go`中的`raft.Raft`提供了Raft算法核心包括了消息处理，逻辑时钟驱动，等。对于更多的实现指导，请检查`raft/doc.go`，这里包含了整体的设计和那些`MessageTypes`分别都负责什么。

##### Leader election

> 领导选举

为了实现领导选举，你可能想要从`raft.Raft.tick()`开始，用于将内部逻辑时钟提前一个刻度，从而驱动选举超时或心跳超时。你现在不需要关注消息发送和接收逻辑。如果你需要发送出去一个消息，就将其压入`raft.Raft.msgs`然后通过`raft.Raft.Step()`来传递响应消息。`raft.Raft.Step()`是消息处理的入口，你应该处理像`MsgRequestVote`，`MsgHeartbeat`的消息和他们的请求。同时也请实现测试stub函数并让他们像`raft.Raft.becomeXXX`一样被正确调用，这函数用于在Raft的角色发生变化时更新Raft的内部状态。

你可以运行`make project2aa`来测试实现并在这部分结尾处看到一些提示。

##### Log replication

> 日志复制

为了实现日志复制，你可能想要从处理发送方和接收方的`MsgAppend`和`MsgAppendResponse`开始。检查`raft/log.go`中的`raft.RaftLog`，这个是一个辅助结构用来帮助你管理你的Raft日志，在这里你还需要通过`raft/storage.go`中定义的`Storage`接口与上层应用程序交互来获取持久化数据比如日志条目和快照。

你可以运行`make project2ab`来测试实现并在这部分结尾处看到一些提示。

#### Implement the raw node interface

> 实现原始节点接口

`raft/rawnode.go`中的`raft.RawNode`是我们和上层状态机交互的接口，`raft.RawNode`包括`raft.Raft`以及提供一些包装的函数比如说`RawNode.Tick()`和`RawNode.Step()`。它也提供了`RawNode.Propose()`来让上层应用来提出新的Raft日志。

另一个重要的结构`Ready`也在此定义了。当处理消息或者推进逻辑时钟时，`raft.Raft`可能需要和上层状态机交互，比如：

- 发送消息到其他节点
- 保存日志到持久化存储
- 保存硬状态比如说term，commit index和vote到持久化存储
- 应用提交的日志条目到状态机
- 其他等

但是这些交互不会立即发生，相反，它们被封装在`Ready`中并由`RawNode.Ready()`返回到上层应用。什么时候调用`RawNode.Ready()`和处理它是由上层应用程序决定的。在处理返回的`Ready`时，上层应用程序还需要调用一些函数，比如`RawNode.Advance()`来更新`raft.Raft`的内部状态，比如说已应用的日志下标，以持久化的日志下标等。

你可以运行`make project2ac`来测试实现并运行`make project2a`来测试整个A部分。

>Hints:
>
>- 在`eraftpb.proto`中添加任何你在`raft.Raft`，`raft.RaftLog`，`raft.RawNode`中需要的状态和消息
>- 测试假定你第一次运行raft时任期为0
>- 测试假定最新选举出来的领袖应该在它的任期中追加一条空日志
>- 测试假定一旦领袖推进了commit index，它将通过`MessageType_MsgAppend`来广播这个commit index
>- 测试没有为本地消息，`MessageType_MsgHup`，`MessageType_MsgBeat`和`MessageType_MsgPropose`设置任期
>- 日志条目的追加在领袖和非领袖之间是不一样的，有不同的来源，检查和处理要小心
>- 不要忘记不同的peers之间的选举超时要不同
>- 在`rawnode.go`中的一些包装方法可以使用`raft.Step(local message)`来实现
>- 当开启一个新的raft时，从`Storage`中获取上次的持久化状态来初始化`raft.Raft`和`raft.RaftLog`

### PartB

在这个部分，你将构建一个容错地kv存储服务，使用part A中实现的Raft模块。你的kv服务应该是一个可复制的状态机，由几个使用Raft进行复制的kv服务器组成。您的kv服务应该可以持续的处理客户端的请求，只要大多数服务器是活跃的并且可以通信，尽管有一些其他错误或者网络分区。

在Project1中，你已经实现了一个单机kv服务器，所以你应该熟悉kv服务器api和`Storage`接口。

在介绍代码之前，你应该理解三个术语：`Store`，`Peer`和`Region`，这些在`proto/proto/metapb.proto`中定义。

- Store代表一个tinykv-server实例
- Peer代表一个运行在一个Store上的Raft节点
- Region是Peers的集合，同样也叫做Raft组

![region](https://github.com/DBDreamers/HugeKV/raw/course/doc/imgs/region.png)

简单来讲，对于Project2，一个集群中只有一个Region，每个Store上也只有一个Peer。所以现在你不需要考虑Region的范围。多Region在peoject3中被更深的介绍。

#### The Code

首先，你应该看看`kv/storage/raft_storage/raft_server.go`中`RaftStorage`，它同样也实现了`Storage`接口，不像直接从下层的引擎中进行读写`StandaloneStorage`，它是先发送每一个读写请求到Raft，然后在Raft提交了该请求之后才进行实际的读写到引擎的操作。通过这个方式，它可以保证多Stores之间的一致性。

`RaftStorage`主要创建一个`Raftstore`来驱动Raft。当调用`Reader`或者`Write`功能时，它实际发送了一个`proto/proto/raft_cmdpd.proto`中定义的`RaftCmdRequest`请求，请求携带四个基本命令类型中的一个(Get/Put/Delete/Snap)，通过channel(channel的接收者是`raftWorker`的`raftCh`)传递给raftstore，然后在Raft提交和应用了该命令之后返回响应。`Reader`和`Write`中的`kvrpc.Context`变量同样有用，它携带客户端角度的Region信息，然后作为`RaftCmdRequest`的头部传递。可能消息是错误的或者陈旧的，所以raftstore要检查他们，并决定是否提出这个请求。

然后，这里有一个TinyKV的核心—raftstore。这个结构有一些复杂，你可以读关于TiKV的引用来对该设计有更好的理解：

- https://pingcap.com/blog-cn/the-design-and-implementation-of-multi-raft/#raftstore (中文版)
- https://pingcap.com/blog/2017-08-15-multi-raft/#raftstore (英文版本)

raftstore的入口是`RaftStore`，看`kv/raftstore/raftstore.go`。它启动一些工作和来异步处理特定任务，其中大部分现在还没有使用，所以可以忽略它们。你只需要关注`raftworker`。(`kv/raftstore/raft_worker.go`)

整个流程被分为两部分：raft工人从`raftCh`中拉取来获取消息，消息包括了基本的tick来驱动Raft模块以及Raft命令将被以Raft条目提出；它从Raft模块获取和处理ready，包括发送raft消息，持久化状态，应用提交的日志到状态机。一旦应用，返回响应给客户端。

#### Implement peer storage

> 实现peer strorage

Peer storage是你通过part A中的`Storage`接口进行交互的，但是除了raft日志，peer storage同时也管理其他的持久化元数据，它们对重启后恢复一致性状态十分重要。此外，在`proto/proto/raft_serverpb.proto`中定义了三个重要的状态：

- RaftLocalState：用于存储当前的Raft的HardState以及最近的日志索引。
- RaftApplyState：用于存储上一个Raft应用的日志索引，以及一些截断后的日志信息。
- RegionLocalState：用于存储Region信息以及相应的Peer状态到该存储中。Normal表示该peer是正常的，Tombstone表示该peer已经从Region中移除以及不能加入到Raft组中。

这些状态被存储进两个badger实例：raftdb和kvdb：

- raftdb存储raft日志和`RaftLocalState`
- kvdb存储不同列祖的key-value数据，`RegionLocalState`和`RaftApplyState`。你可以视kvdb为Raft论文中提到的状态机。

格式如下，以及在`kv/raftstore/meta`中有一些辅助函数，你可以使用`writebatch.SetMeta()`来设置它们。

| Key              | KeyFormat                        | Value            | DB   |
| ---------------- | -------------------------------- | ---------------- | ---- |
| raft_log_key     | 0x01 0x02 region_id 0x01 log_idx | Entry            | raft |
| raft_state_key   | 0x01 0x02 region_id 0x02         | RaftLocalState   | raft |
| apply_state_key  | 0x01 0x02 region_id 0x03         | RaftApplyState   | kv   |
| region_state_key | 0x01 0x03 region_id 0x01         | RegionLocalState | kv   |

> 你可能有疑问为什么TinyKV需要两个badger实例。实际上，它也可以只使用一个badger来存储raft日志和状态机数据。分开到两个实际只是为了和TiKV设计保持一致

这些元数据应该在`PeerStorage`中被创建和更新。当创建PeerStorage时，看`kv/raftstore/peer_storage.go`，它初始化了该Peer的RaftLocalState，RaftApplyState，或者从从底层的引擎中获得之前的数据。注意RAFT_INIT_LOG_TERM和RAFT_INIT_LOG_INDEX都是5(只要比1大)而不是0。不设置为0的原因是为了区分配置变更后peer被动创建的情况。你现在可能不是很理解，所以就注意这个就行，细节在当你实现配置变更的project3b中会提及。

在这个部分你需要实现的代码只有一个函数：`PeerStorage.SaveReadyState`，这个函数是用来保存`raft.Ready`中的数据到badger中，包括追加日志条目和保存Raft硬状态。

为了追加日志条目，只需要将`raft.Ready.Entries`中所有的日志条目保存到raftdb中，然后删除任何以前追加的那些永远不会提交的日志条目。另外，更新peer storage的`RaftLocalState`和将其保存到raftdb中。

> Hints:
>
> - 使用`WriteBatch`来一次性保存这些状态。
> - 使用`peer_storage.go`中的其他函数为了如何读写这些状态。
> - 设置环境变量LOG_LEVEL=debug可能会帮助你debug

#### Implement Raft ready process

> 实现Raft准备过程

在project2 part A中，你已经构建了一个基于tick的Raft模块。现在你需要编写程序驱动它的外部进程，大部分代码已经在`kv/raftstore/peer.go`和`kv/raftstore/peer_msg_handler.go`下实现。所以你需要学习代码然后完成`proposeRaftCommand`和`HandleRaftReady`的逻辑。下面是对这个框架的一些解释。

Raft`RawNode`已经使用`PeerStorage`创建存储在`peer`中。在raft的工作者上，你可以看到它用`peer`并且将其使用`peerMsgHandler`包装起来。`peerMsgHandler`主要功能有两个：一个是`HandleMsg`，另一个是`HandleRaftReady`。

`HandleMsg`处理所有从raftCh接收的消息，包括用来调用`RawNode.Tick`来驱动Raft的`MsgTypeTick`，用来包装客户端的请求的`MsgTypeRaftCmd`和用来传输Raft之间的消息的`MsgTyepeRaftMessage`。所有的消息类型在`kv/raftstore/message/msg.go`中被定义。你可以检查它们的细节并且其中部分会被用在之后的模块中。

在消息被处理后，Raft节点应该有一些状态需要更新。所以`HandlerRaftReady`应该从Raft模块中做好准备，并执行相应的操作，比如说持久化日志条目、应用提交的条目以及通过网络向其他peer发送raft消息。

在伪代码中，raftstore使用Raft类似于：

```go
for {
  select {
  case <-s.Ticker:
    Node.Tick()
  default:
    if Node.HasReady() {
      rd := Node.Ready()
      saveToStorage(rd.State, rd.Entries, rd.Snapshot)
      send(rd.Messages)
      for _, entry := range rd.CommittedEntries {
        process(entry)
      }
      s.Node.Advance(rd)
    }
}
```

再次之后，读或写的整个过程将是这样：

- 客户端调用RPC RawGet/RawPut/RawDelete/RawScan
- RPC处理程序调用`RaftStorage`相关方法
- `RaftStorage`发送一个Raft命令请求到raftstore，然后等待响应
- `RaftStore`以一条Raft日志的形式提出Raft命令请求
- Raft模块追加日志，然后通过`PeerStorage`持久化
- Raft模块提交日志
- Raft工人在处理Raft ready时执行Raft命令，并通过回调来返回响应
- `RaftStorage`接收回调的响应并返回给RPC处理程序
- RPC处理程序做出一些行动然后返回RPC响应给客户端

你应该运行`make project2b`来通过所有的测试。整个测试是运行一个仿真的集群包括使用仿真网络的多TinyKV实例。它执行一些读写操作，并检查返回值是否符合预期。

注意，错误处理是通过该测试的重要部分。你可能已经注意到有一些定义在`proto/proto/errorpb.proto`中的错误，该错误是gRPC响应的一个字段。同样，实现`error`接口的相应错误在`kv/raftstore/util/error.go`中定义，因此可以将它们用作函数的返回值。

这些错误主要和Region有关。所以它也是`RaftCmdResponse`的`RaftResponseHeader`字段。当处理一个请求或者应用一个命令时，可能会有一些错误。如果这样，你应该携带着错误来返回raft命令的响应，之后错误将会被深入传递给gRPC响应。在返回携带错误的响应时，你可以使用`kv/raftstore/cmd_resp.go`提供的`BindErrResp`来将这些错误转成`errorpb.proto`中定义的错误类型。

在该步骤，你可能需要考虑这些错误，其他的将会在project3中被处理：

- ErrNotLeader：raft命令在follower处被提出。所以使用这个错误来让客户端尝试其他peers
- ErrStaleCommand：这可能由于领导人变更，一些日志没有被提交，并被新的领导人的日志覆盖。但是客户端并不知道这一点。他们仍在等待回复。因此，您应该返回这个命令，让客户端知道并重试该命令。

>Hints:
>
>- `PeerStorage`实现了Raft模块中的`Storage`接口，你应该使用提供的方法`SaveRaftReady()`来持久化Raft相关状态。
>- 使用`engine_util`中的`WriteBatch`来使批量写原子性，例如，你应该确保应用已提交的日志以及更新applied index在一个写批次中。
>- 使用`Transport`来发送raft消息到其他peers，它在`GlobalContext`中。
>- 当没有大部分的节点同意以及没有最新的数据时，服务端不应该完成一个get的RPC。你可以把get操作放入一个raft日志中，或者实现Raft论文章节8中的read-only操作的优化。
>- 不要忘记更新和持久化状态，当应用日志条目时。
>- 你可以异步方式应用已提交的日志条目，就像TiKV做的那样。这不是必要的，但是是一个可以提升性能的巨大挑战。
>- 在提出日志时记录命令的回调，然后在应用中返回回调。
>- 对于一个snap命令的响应，应该显示地将badger Txn设置为callback。
>- 在2A之后，你应该需要多次运行某些测试来找到一些bug。

### PartC

就目前的代码而言，长时间运行的服务器永远记住整个 Raft 日志是不现实的。相反，服务器将检查 Raft 日志的数量，并不时丢弃超过阈值的日志条目。

在本部分中，您将基于上面两部分的实现来实现快照处理。一般来说，Snapshot 只是像 AppendEntry 一样的Raft消息，用于向follower复制数据，不同之处在于它的大小，Snapshot 在某个时间点包含了整个状态机的数据，而一次构建和发送如此大的消息将消耗大量的资源和时间，这可能会阻碍其他Raft消息的处理，为了缓解这个问题，Snapshot 消息将使用一个独立的连接，并将数据分成块传输。这就是为什么 TinyKV 服务有一个快照 RPCAPI 的原因。如果您对发送和接收的细节感兴趣，请检查 `snapRunner` 和参考 https://pingcap.com/blog-cn/tikv-source-code-reading-10/

#### The Code

所有你需要的变动都基于你part A和part B中写的代码。

#### Implement in Raft

> Raft中的实现

即使我们需要一些不同的快照消息的处理程序，但是从Raft算法角度来看没有什么区别。参见`eraftpb.Snapshot`的定义，`eraftpb.Snapshot`的字段`data`不代表实际的状态机数据，但是上层应用使用的一些元数据你可以暂时忽略。当领导者需要向跟随者发送快照消息时，它会调用`Storage.Snapshot()`来获取`eraftpb.Snapshot`，然后像其他的Raft消息一样发送快照消息。如何构建和发送状态机数据是由raftstore实现的，这将在下一步中介绍。你可以假设一旦`Storage.Snapshot()`返回成功，Raft领袖可以发送快照消息到follower，然后follower应该调用`handleSnapshot()`来处理，即会从`eraftpb.SnapshotMetadata`中恢复raft内部数据比如说任期，commit index和membership信息等，然后完成快照处理程序。

Implement in raftstore

> raftstore中的实现

在该步骤，你需要多学习两个raftstore的工作者—raftlog-gc worker和region worker。

Raftstore基于配置`RaftLogGcCountLimit`来时时检查它是否需要回收日志，参考`onRaftGcLogTick()`。如果需要，它会提出一个raft管理员命令`CompactLogRequest`，这个命令是由`RaftCmdRequest`包装的，就像project2 part B中实现二点四种基本命令类型(Get/Put/Delete/Snap)。然后你需要当管理员命令由Raft提交时处理这个它。但是不像 Get/Put/Delete/Snap 命令时读写状态机数据的，`CompactLogRequest`修改元数据，也就是更新`RaftApplyState`中的`RaftTruncatedState`。之后，你应该通过`ScheduleCompactLog`来分配一个任务给raftlog-gc worker。Raftlog-gc worker将会异步的做实际的日志删除工作。

然后由于日志压缩，Raft模块可能需要发送一个快照。`PeerStorage`实现了`Storage.Snapshot()`。TinyKV在region worker中生成快照和应用快照。当调用`Snapshot()`，它实际会发送一个任务`RegionTaskGen`给region worker。region worker的消息处理程序位于`kv/raftstore/runner/region_task.go`中。它会扫描底层的引擎来生成快照，然后通过channel发送快照的元数据。下一次Raft调用`Snapshot`，它会检查是否快照是否完成生成了。如果是，Raft应该发送快照数据到其他的peers，快照发送和接收工作在`kv/storage/raft_storage/snap_runner.go`中被处理。你不需要深入细节，只需要知道快照消息将在接收到快照之后由`onRaftMsg`处理。

然后快照将反映在下一个Raft ready，所以你需要做的任务就是修改reft ready流程来处理如果是snaphot的情况。当你确保应用了快照时，你可以更新你的peer storage的内存状态比如说`RaftLocalState`，`RaftApplyState`和`RegionLocalState`。同样的，不要忘记持久化这些状态到kvdb和raftdb以及从kvdb和raftdb中移除旧的状态。除此之外，你也需要更新`PeerStorage.snapState`到`snap.SnapState_Applying`然后发送`runner.RegionTaskApply`任务到region worker通过`PeerStorage.regionSched`并等待直到region worker完成。

你应该运行`make project2c`来通过所有的测试。



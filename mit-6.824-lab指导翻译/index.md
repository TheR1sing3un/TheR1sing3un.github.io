# MIT 6.824 Lab指导翻译


# MIT6.824

## Lab1

### Rules

1. 最后文件需要输出nReduce个，文件名格式为`mr-out-X`
2. 输出到文件的格式在`mrsequential.go`中
3. 只用写`worker.go`/`coordinator.go`/`rpc.go`这三个文件
4. worker将中间文件输出到当前文件夹下，之后worker执行reduce任务的时候从中取
5. 需要实现`coordinator.go`中的`Done()`方法，当全部任务被执行完了之后返回true，然后`mrcoordinator`退出
6. 当所有的任务的完成的时候，worker也应该停止。简单的方法就是使用rpc的回调，当返回err，也就是故障的时候，这里可以理解为coordinator已经结束了，所以这时候worker可以退出了。

### Hints

1. 一开始可以先实现worker的worker方法来和coordinator进行rpc调用来获取任务，coordinator回应给他文件名作为一个还未开始的map任务。然后worker读取这些文件然后调用map方法，参考`mrsequential.go`中的方法
2. map和reduce方法是作为插件使用的，记得启动的时候带上参数`wc.so`
3. 没有`wc.so`插件就build一个，`go build -race -buildmode=plugin ../mrapps/wc.go`
4. 中间文件可以命名为`mr-X-Y`，X表示map任务编号，Y表示reduce任务编号。
5. 中间文件使用json方法来存来读取
6. map的worker使用`ihash(key)`方法来获取reduce编号，靠这个存到对应的中间文件中
7. coordinator作为一个rpc服务端，需要对共享资源进行并发保护
8. run的时候使用-race检查一下
9. 只有所有的map任务被执行完了之后才能进行reduce任务的分配。一种方法是worker周期的请求coordinator来获取任务，没获取到就sleep一会再来请求。另一种方法是每个rpc的handler可以循环等待一下，使用`time.Sleep`或者`sync.Cond`
10. coordinator不能可靠的区别那些故障worker，包括那些执行的太慢的节点。最好就是每次分配了任务之后可以等待10秒，如果10秒都没有完成，就可以认为该worker已故障了。需要重新将该任务分配给别的worker。
11. 测试故障恢复可以使用`mrapps/crash.go`插件，会随机在map和reduce中故障
12. 为了不让已经故障的节点的产生文件对作为真正的中间文件，可以使用论文中提到的临时文件的方法，worker写的使用可以使用临时文件，使用`ioutil.TempFile`来创建临时文件，然后完成之后使用`os.rename`来原子性的命名。

---

---

## Lab2

### Lab2a

#### Hints

1. 不可以直接跑你的Raft实现的代码；使用`go test -run 2A -race`。
2. 根据论文的图二。这个部分只需要你来关注发送和接受RequestVote RPCs、那些关系到选举的服务器规则，以及那些关系到选举的状态。
3. 为`Raft`增加领袖选举的相关状态在`raft.go`中，同时你也需要定义一个结构关于每个log entry。
4. 完善`RequestVoteArgs`和`RequestVoteReply`结构。修改`Make()`来创建一个后台协程来开始周期性的发送`RequestVote`来进行领袖选举当没有在超时时间内接受到别的节点的心跳。通过这个方式，一个节点会知道谁是领袖(如果已经有领袖的情况)，或者开启选举让自己成为领袖。实现`RequestVote()`来进行投票。
5. 实现心跳机制，定义一个`AppendEntries`的RPC struct，然后让领袖周期性的发送该请求。写一个`AppendEntries`的RPC方法来重置选举时间，这样服务器就不会在已经当选领袖情况下再次成为被选举。
6. 确保所有的选举不会总是同时开始。
7. 测试要求领袖发送心跳检测不应该超过每秒十次。
8. 测试要求你的Raft在五秒内选举出一个新的领袖当旧的领袖已经失败的时候。选举可能会进行多轮，为了防止分裂投票(当包丢失或者候选人不幸选择了相同的随机退出时间)必须选择足够短的的超时。
9. 论文的第5.2节提到选举超时的范围是150到300毫秒。这个范围只有在领导者发送心跳频率大大超过每150毫秒一次的情况下才有意义。因为测试者限制你每秒10次心跳，你必须使用比报纸上150到300毫秒更大的选举超时，但不能太大，因为那样你可能无法在5秒内选出领导者。
10. 您需要编写定期执行操作或在时间延迟之后执行操作的代码。最简单的方法是创建一个 goroutine，循环调用`time.Sleep()`。
11. 记得完成`Getstate()`
12. 测试会调用你的`rf.Kill()`当它永久的关闭一个实例时，你可以使用`rf.Kill()`来检查`Killed()`是否已经被调用。最好每个循环都来判断一下。

---

### Lab2b

#### Hints

1. 实现ledaer和follower的代码去增加新的日志条目，通过`go test -run 2B -race`来检验正确。
2. 首要目标是通过`TestBasicAgree2B`。从实现`Start()`方法开始，然后编写代码去发送和接收新的日志条目通过`AppendEntries`，以及参考图二(也就是那个经典的图)。
3. 需要实现选举限制(在论文的5.4.1)
4. 在早期的2b 测试中，不能达成一致意见的一个原因是，领导人还活着的情况下，依然重复进行选举。在选举计时器管理中寻找漏洞，或者没有在赢得选举后立即发送心跳。
5. 您的代码可能具有重复检查某些事件的循环。不要让这些循环不停顿地连续执行，因为这会使实现变慢，以至于测试失败。使用 Go 的条件变量，或者使用在每个循环的时候进行10ms睡眠。

---

### Lab2c

一个真正的实现会在每次raft改变的时候讲raft的持久化状态写入到磁盘中，以及当重新启动后在重启期间从磁盘中读取状态。你的实现不会真正的使用磁盘；而是，你会保存和恢复持久化状态从一个`Persister`对象(在`persister.go`中)。无论谁调用`Raft.Make()`都会提供一个拥有最近的初始化状态的`Persister`。Raft应该从这个`Persister`来初始化这些状态，以及应该使用这个`Persister`来每次保存raft的持久化状态每当状态改变的时候。使用`Persister`的`ReadRaftState()`和`SaveRaftState()`方法。

#### Task1

完成函数`persist()`和`readPersist()`通过增加代码来保存和恢复持久化状态。你需要编码(或者说“初始化”)这些状态以byte数组的形式为了能将其传递给`Persister`。使用`labgob`编码器；看一些那些`persist()`和`readPersist()`的注释。`labgob`就像Go语言的`gob`编码器，但是会打印错误信息当你尝试编码一些使用小写字段名的结构体。

#### Task2

在你的实现里那些改变了持久化状态的地方插入对`persist()`的调用。一旦你完成了，你应该会通过剩下的这些test。

#### Hints

1. 2C的许多test涉及到server失效和网络丢失rpc的请求和回复，这些事件是不确定的，可能你会很幸运的通过这些test，即使的代码还有bug。通常需要你多次运行test来暴露出这些bug。
2. 你可能需要优化你的nextIndex的备份通过一次备份多个条目。看一下论文的第七页末尾和第八页开头的灰色方框内的字。但是论文对细节含糊其辞，你需要借助6.824的Raft课程来完善。
3. 2C只要求你实现持久化(persistense)和日志快速恢复(fast log backtracking)，2C的test可能会因为之前的部分导致失败。即使你通过了2A和2B的代码，但是你可能还是会有选举和日志的bug在2C的test中被暴露。

---

### Lab2d

为了支持快照，我们需要在service层和Raft库之间的接口。Raft论文没有指定这个接口，因此很多设计都是可以的。为了一个简单的实现，我们决定使用接下来的接口：

- `Snapshot(index int, snapshot []byte)`
- `CondInstallSnapshot(lastIncludedTerm int, lastIncludedIndex int, snapshot []byte) bool`

service层调用`Snapshot()`将其状态传递给Raft。快照包含所有的信息以及index。这意味着相应的Raft节点不再需要日志。你的Raft实现应该尽可能的调整log。你必须修改你的Raft代码来只操作末尾的log。

正如Raft论文中讨论的那样，Raft的leaders必须通过安装快照的方式来告诉落后的Raft节点来更新它的的状态。你需要实现`InstallSnapshot`的RPC发送和处理为了安装快照。这和`AppendEntries`形成对比，`AppendEntries`发送日志条目然后一个个被service应用。

`InstallSnapshot`的RPC是在Raft节点间调用，而提供的框架函数`Snapshot/CondInstallSnapshot`是service用来和Raft通信的。

当follower接受并处理`InstallSnapshot`的RPC请求时，它必须使用Raft将包含的快照交给service。`InstallSnapshot`处理程序可以使用`applyCh`来发送快照，通过将快照放入到`ApplyMsg`中。service从`applyCh`中读取，然后和快照调用`CondInstallSnapshot`来告诉Raft：service正在切换成传入快照状态，然后Raft应该同时更新自己的日志。(参见`config.go`中的`applierSnap()`来查看tester服务是怎么做这个的)。

`CondInstallSnapshot`应该拒绝安装旧的快照(如果Raft在`lastIncludedTerm/lastIncludedIndex`之后处理了日志条目)。这个因为Raft可能在处理`InstallSnapshot`RPC之后和在service调用`CondInstallSnapshot`之前处理了其他的RPC并在`applyCh`上发消息了。不允许Raft回到旧的快照，因此必须拒绝旧的快照。当你的实现拒绝快照时，`CondInstallSnapshot`应该直接返回`false`使得service知道现在不应该切换这个快照。

如果快照是最近的，Raft应该调整日志，持久化新的状态，返回true，以及service应该切换到这个快照在处理下一个消息到`applyCh`之前。

`CondInstallSnapshot`是一种更新Raft和service的状态的方式；其他接口也是可以的。这个特殊的设计允许你的实现检查一个快照是否必须安装在一个地方，并且原子性地将service和Raft切换到快照。你可以自由地以`CondInstallSnapshot`始终可以返回`true`的方式来实现你的Raft。

#### Task

修改你的Raft代码来支持快照；实现`SnapShot`，`CondInstallSnapshot`以及`InstallSnapshot`的RPC。以及对Raft的更改来支持这些(例如：继续使用修剪后的日志)你的解决方案通过2D测试和所有的Lab2测试的时候，它就完成了。

#### Hints

1. 在一个`InstallSnapshot`中发送整个快照。不要实现图13的分割快照的`offset`机制。
2. Raft必须丢弃旧的日志以允许Go的垃圾回收器来使用和重用内存；这要求对于丢弃的日志没有可达的引用(指针)。
3. Raft的日志不再使用日志条目的位置或日志长度的方式来确定日志的条目索引；您需要使用独立于日志位置的索引方案。
4. 即使日志被修剪，你的实现仍需要在`AppendEntries`正确的发送日志的Term和Index；这可能需要保存和引用最新的快照的`lastIncludedTerm/lastIncludedIndex`(考虑是否应该持久化)
5. Raft必须存储每一个快照到persiser中，通过`SaveStateAndSnapshot()`。
6. 对于整套的Lab2测试，合理的时间消耗应该是8分钟的实际时间和1.5分钟的CPU时间。

---

---

## Lab3

### Introduction

在这个lab中，你将构建一个基于lab的Raft库实现的容错性kv存储服务。你的kv服务将是一个复制的状态机，由几个使用Raft进行复制的kv服务器组成。你的kv服务应该在当大多数服务器处于活动状态并且可以通信的时候继续处理客户机请求，即使出现其他故障或者网络分区。lab3之后，你将实现raft_diagram中的所有模块(Clerk，Service and Raft)。

这个服务支持三个操作：`Put(key, value)`，`Append(key, arg)`以及`Get(key)`。它维护一个简单的kv键值对数据库。键值对都是字符串。`Put()`替换数据库中特定键的值，`Append(key, arg)`将arg附加到key的值上，以及`Get()`获取当前键的值。`Get()`一个不存在的key将返回一个空字符串。`Append()`一个不存在的key就和`Put()`效果相同。每一个客户端和服务用一个`Clerk`的Put/Append/Get方法来交互。一个`Clerk`管理和服务器的RPC交互。

你的服务必须为`Clerk`的Get/Put/Append方法提供强一致性。以下是我们认为的强一致性。如果一次调用一个，那么Get/Put/Append方法应该像系统只有一个副本那样工作，每次调用都应该观察前面的调用序列所暗示的对状态的修改。对于并发调用，返回值和最终状态必须相同，就像操作以某种顺序一次执行一个操作一样。如果调用在时间上有重叠，例如，客户端X调用了`Clerk.Put()`，然后客户端Y调用`Clerk.Append()`，然后X的调用返回结果了。此外，一次调用必须观察在调用开始之前完成的所有调用的效果(所以我们在技术上要求线性化)。

强一致性对于应用程序来说很方便，因此这非正式地意味着，所有客户端看到的状态都是一样的并且看到了都是最新的状态。对于单机来说，提供强一致性相对容易。但是如果服务是有副本的，那就困难了。因为所有的服务器必须为并发请求选择相同的执行顺序，并且必须避免使用不是最新的状态来回复客户端。

这个lab有两个部分。在A部分，你将实现一个不用担心Raft的log会无限增长的服务。在B部分，你将实现一个快照(论文第7节)，也就是让Raft丢弃旧的log。

你应该重新阅读Raft论文，特别是第7和第8节。广阔来看，你可以看一下Chubby、Paxos Made Live、Spanner、Zookeeper、Harp、Viewstamped Replication以及Bolosky et al。

### Getting Started

我们在`src/kvraft`中提供框架代码和测试。你需要修改`kvraft/client.go`，`kvraft/server.go`以及可能也要`kvraft/common.go`

### Lab3a

> 一个没有快照的kv服务

每一个你的kv服务器都需要有一个与之关联的Raft节点。Clerks发送`Put()`，`Append()`和`Get()`的RPC请求到当前Raft中leader的那个kvserver上。kvserver将Put/Append/Get操作提交给Raft，以便Raft保存一系列Put/Append/Get操作。所有的kvserver从Raft的log中按顺序执行操作，并将这些操作应用到他们的kv数据库上。这样做的目的是在所有的服务器上维护相同的kv数据库副本。

一个`Clerk`有时候不知道哪一个kvserver是Raft的leader。如果一个`Clerk`发送一个RPC请求到错误的kvserver上，或者没有到达kvserver上；`Clerk`应该通过发送到不同的kvserver上来进行重试。如果一个kv服务将操作提交到它的Raft日志中(并将操作用用到kv状态机上)，那么leader将通过RPC来回应`Clerk`。如果操作没有成功提交(比如leader被更替了)，这个服务器将报告error，然后`Clerk`将在不同的服务器上重试。

你的kvservers不应该直接进行通信；它们应该只通过他们的Raft来进行通信。

#### Task1

你的首要任务是实现一个当没有消息丢失和服务器失败的解决方案。

你将需要增加`client.go`中关于Clerk的Put/Append/Get方法的RPC发送代码，以及实现`server.go`中的`PutAppend()`和`Get()`的RPC处理程序。这些处理程序应该使用`Start()`来在Raft日志中输入一个`Op`；你应该完善`server.go`里面的`Op`的结构定义以至于它可以描述一个Put/Append/Get操作。每一个服务器应该在Raft提交`Op`命令时执行这些命令，即当它们出现在`applyCh`时。RPC处理程序应该在Raft提交其`Op`时发出通知，然后回复RPC。

当你可靠的通过测试中的第一个测试："One Client"时，你就完成了这个任务。

#### Hints1

- 调用`Start()`之后，你的kvservers将需要等待Raft去完成agreement。已经达成一致的命令道道`applyCh`。你的代码需要保持读取`applyCh`当`PutAppend()`和`Get()`处理程序使用`Start()`提交了命令到Raft日志中的时候。注意kvserver和Raft库之间的死锁问题。
- 你可以向`ApplyMsg`和RPC处理程序比如说`AppendEntries`中添加字段。但是对于大多数实现都是不需要的。
- 一个kvserver不应该完成一个`Get()`的RPC请求当它不是大多数的那一部分。一个简单的解决方案就是在Raft日志中输入每一个`Get()`(以及每一个`Put()`和`Append()`)。你不必实现论文章第八节描述的只读操作的优化。
- 最好一开始就加锁因为避免死锁有时候会影响整个的代码设计。使用`go test -race`来检查的你的代码是不是race-free。

现在你需要修改你的解决方案，以便在网络和服务器出现故障时继续执行。您将面临的一个问题是，`Clerk`可能不得不的发送多次RPC直到它找到了一个积极响应的kvserver时。如果一个leader在向Raft日志中提交了一个条目之后失败了，`Clerk`可能不会收到回应，因此可能需要重发请求到别的leader那里。每次`Clerk.Put()`或者`Clerk.Append()`的调用应该最终只执行一次，因此你必须确保重发不会导致服务器执行两次请求。

#### Task2

添加代码来处理失败，并处理重复的`Clerk`请求，包括这样的情况：`Clerk`在一个任期内发送请求到kvserver的leader，等待回复超时，然后再另一个任期重新向新领导发送请求。请求应该只执行一次。您的代码应该能够通过

`go test -run 3A -race`。

#### Hints2

- 你的解决方案需要处理一个领导者，该领导者已经为Clerk的RPC请求调用了`Start()`，但是在将请求提交到日志之前失去了领导权。在这种情况，你应该安排`Clerk`去重发请求到其他服务器直到发到了新的leader那里。这样做的一个方法是，服务器通过注意`Start()`返回的索引出现了不同的请求，或者Raft的任期已经改变，来检测它是否失去了领导权。如果当前领导者自己分区，那么它不会知道新的领导者；但是同一分区的任何客户端也不能和新领导者交互；所以在这种情况下，服务器和客户端可以无限期的等待，直到分区恢复。
- 你可能必须修改您的Clerk，以便记住最后一个RPC的主机是哪个服务器，并首先将下一个RPC发送到该服务器。这将避免浪费时间在通过RPC来找领导者上，这可能会帮助你快速的通过一些测试。
- 你将需要唯一地标识客户端的操作，以确保kv服务只执行每个操作一次。
- 你的重复检测方案应该能够快速的释放服务器内存，例如，让暗示每一个RPC都看到了前一个RPC的回复。假设客户端只在Clerk调用一次是可以的。

---

### Lab3b

> 使用快照的kv服务

就目前的代码而言，重启服务器将重新加载完整的Raft日志来恢复其状态。然而，对于一个长时间运行的服务器而言，永远保存完整的Raft日志是不现实的。相反，您将修改kvserver来和Raft协作来使用lab2d中Raft的`Snapshot()`和`CondInstallSnapshot()`来节省空间。

tester传递`maxraftstate`到您的`StartKVServer()`。`maxraftstate`以字节表示Raft的持久性状态允许的最大大小(包括日志，但是不包括快照)。你应该将`maxraftstate`和`persister.RaftStateSize()`比较。每当你的kv服务器检测到Raft状态大小接近这个阈值，它应该使用`Snapshot()`来保存一个快照，`Snapshot()`又使用`persister.SaveRaftState()`。如果`maxraftstate`是-1，你不需要进行快照。`maxraftstate`应用于你的Raft传递给`persister.SaveRaftState()`的gob编码字节。

#### Task

修改你的kvserver以便于它检测何时Raft的持久性状态变得太大，然后将快照交给Raft。当kvserver重新启动时，它应该从`persister`中读取快照，并从快照恢复其状态。

#### Hints

- 考虑下kvserver何时应该对其状态进行快照，以及快照中应该包含哪些内容。Raft使用`SaveStateAndSnapshot()`将每个快照以及响应的Raft状态存储在持久化对象中。你可以使用`ReadSnapshot()`来读取最新的快照。
- 你的kvserver必须能够检测到日志中跨检查点的重复操作，因此用于检测它们的任何状态都必须包含在快照中。
- 将快照中存储的所有结构字段大写。
- 您的Raft库可能在该lab中暴露出bug。如果您更改了Raft实现，请确保它能继续通过所有的lab2测试。
- lab3测试的合理时间是400s的实时时间以及700秒的CPU时间。此外，运行`go test -run TestSnapshotSize`应该少于20s的实时时间。

---

## Lab4

### Introduction

在这个实验中，你将构建一个kv存储系统，其中"分片"或分区是一组副本上的键。分片是kv键值对的子集；例如，所有以a开头的键可能是一个分片，所有以b开头的键可能是另一个分片，等等。之所以进行切分是因为性能。每一个副本组只管理几个分片的put和get，并且组之间并行操作；因此总的系统吞吐量(单位时间的put和get)与组的数量成比例增加。

您的kv存储区有两个主要组件。首先，一组副本组。每一个副本组负责分片的一个子集，一个副本由少数服务器组成，这些服务器使用Raft去复制组的分片。第二个组件是"分片控制器"。分片控制器决定那个副本组应该服务于每个分片；这些信息称之为配置。配置随着时间而变化。客户端查询分片控制器以查找该键的副本组，而副本组查询控制器以查找需要服务的分片。整个系统只有一个分片控制器，使用Raft实现容错服务。

一个分片存储系统必须能够在副本组之间移动分片。原因之一是某些组可能比其他组的负载大，因此需要移动分片来平衡负载。另一个原因是副本组可能会加入和离开系统：可能会增加新的副本组来增加容量，或者现有的副本组可能离线修复或者退休。

该lab的一个主要挑战是处理重新配置——在分配分片到组的变化。在单个副本组中，所有的组成员必须在当重新配置关系到客户端的Put/Append/Get请求的时候达成一致。例如，一个Put请求可能与导致副本组停止对保存的Put键的分片负责的重新配置通知到达。组内的所有副本必须就Put是在重新配置之前还是之后发生达成一致。如果是在之前，Put应该生效，分片的新所有者应该看到其效果；如果之后，Put请求不应该生效以及客户端必须在新的所有者处进行重试。建议的方法是去让每个副本组使用Raft去记录Put/Append/Get的顺序以及重新配置的顺序。你将需要确保在任何时候最多只有一个副本组为每一个分片服务。

重新配置也需要副本组之间的交互。例如，在配置10 中的组G1可能负责分片S1，配置11中的组G2可能需要负责分片S1。在10到11的重新配置过程中，G1和G2必须使用RPC将分片S1(键值对内容)从G1移动到G2。

> **Note**
>
> 只有RPC可以用于客户端和服务器之间的交互。例如，不允许服务器的不同实例共享Go变量或者文件。

> **Note**
>
> lab中使用"配置"来指代分片指向副本组。这和Raft集群成员身份的更改不同。你不必实现Raft集群成员变更。

这个lab的总体架构(一个配置服务和一组副本组)遵循和Flat Datacenter Storage，BigTable，Spanner，FAWN，Apache HBase，Rosehud，Spinnaker等等相同的总体模式。这些系统在许多细节上和lab不同，而且也通常更复杂和有能力。例如，lab没有改进每个Raft组中的对等点集合；它的数据和查询模型十分简单，分片的切换速度很慢，不允许并发客户端访问。

> **Note**
>
> 你的lab4的分片服务器，lab4分片控制器以及lab3 kvraft必须都使用相同的Raft实现。我们将重新运行lab2和lab3的测试，你在旧测试中的分数将计入你的lab4总分。这些测试在你的lab4的总分中占了10分。

我们建议你使用`src/shardctrler`和`src/shardkv`的骨架代码和测试。

当你完成了之后，你的实现应该通过所有的`src/shardctrler`目录下的测试，还有`src/shardkv`下的所有。

### Lab4a

> 分片控制器(30分)

首先你需要实现分片控制器，在`shardctrler/server.go`和`client.go`中。当你完成的时候，你应该完成`shardctrler`目录下的所有测试。

shardctrler管理一系列编号的配置。每个配置描述一组副本组以及分片到副本组的分配。每当这个分配需要更改时，分片控制器创建一个有着新的分配的新配置。kv客户端以及服务器如果想知道当前(或者过去)的配置，请和sharfctrler联系。

你的实现必须支持`shardctrler/common.go`中描述的RPC接口，该接口由`Join`，`Leave`，`Move`以及`Query`RPC组成。这些RPC用于允许管理员(以及测试)去控制shardctrler：去增加新的副本组、消除副本组以及在副本组之间移动分片。

`Join`RPC由管理员调用来增加新的副本组。它的参数是一组唯一、非零的副本组的标识符(GID)到服务器列表的映射。Shardctrler应该通过创建新的包括了新副本组的新配置来进行响应。新的配置应该在所有组中尽可能平均地分配分片。并尽可能少地移动碎片来实现这一目标。如果有GID不是当前配置的一部分，那么shardctrler应该允许重用它(例如，允许GID加入，然后离开，然后再加入)。

`Leave`RPC的参数之前加入的副本组的GID列表。shardctrler应该创建一个不包含这些组的配置，并将这些组的分片分配给其他的组。新的配置应该尽可能地在组之间平均分配分片，并且应该尽可能减少移动分片来实现这一目标。

`Move`RPC的参数是一个分片的编号和GID。shardctrler应该创建一个新的配置，将分片分配给副本组。`Move`的目的是允许我们测试你的软件。`Move`后的`Join`和`Leave`可能会取消`Move`，因为`Join`和`Leave`会进行重新平衡。

`Query`的RPC参数是一个配置编号。shardctrler用具有该数字的配置来回复。如果编号是-1或者大于已知的最大配置编号，那么应该回复最新的一个配置。`Query(-1)`的结果应该反应每一个在收到`Query(-1)`之前shardctrler已经完成的`Join`，`Leave`或者`Move`RPC。

第一个配置的编号应该为0。它应该不包含任何组，以及所有的分片都应该分配给GID0(一个无效的GID)。下一个配置(为响应`Join`RPC而创建)应该是编号1。通常会有明显多于组数的分片(每一个组应该服务超过一个分片)，以便能够以相当细的粒度转移负载。

#### Task

你的任务是去实现在`shardctrler/`目录下的`client.go`和`server.go`中指定的接口。你的shardctrler必须是容错的，使用lab2/3中的Raft库。注意，我们将重新运行lab2和lab3当给lab4评级时，所以请确保你不会引入bug到Raft实现中。当你通过所有`shardctrler`中的所有测试时，你就完成了这项任务。

#### Hints

- 从一个精简的kvraft服务器副本开始。
  - 你应该实现对分片控制器的rpc的重复客户端请求检测。shardctrler测试不会对此进行测试，但是shardkv测试之后会在不可靠的网络上使用shardctrler；如果你的shardctrler没有过滤掉重复的rpc请求，你可能很难通过shardkv的测试。
- 执行分片重新平衡的状态机中的代码需要具有确定性。在Go中，map的迭代顺序不确定。
- Go中的map是引用的。如果您将一个一个map类型的变量分配给了另一个变量，那么两个变量将引用同一个map。因此，如果希望基于以前的配置创建一个新的`Config`，你需要创建一个新的map对象(使用`make()`)以及分别复制键值。
- Go race探针(go test -race)将帮助你找到Bug。

---

### Lab4b

> 分片kv服务器

现在你将建立一个分片kv系统，一个基于分片的容错kv存储系统。你将修改你的`Shardkv/client.go`，`shardkv/common.go`以及`shardkv/server.go`。

每一个分片kv服务器作为复制组的一部分进行操作。每一个副本组为一些kv键值对分片提供`Get`，`Put`以及`Append`操作。使用`cleint.go`中的`key2shard()`方法来查找这个键是属于哪一个分片的。多个副本组合作为整套分片服务。一个`shardctrler`服务实例将分片分配给副本组；当这个分配发生变化时，副本组必须交出分片，同时确实客户端不会看到不一致的响应。

你的存储系统必须为使用其客户端接口的应用提供一个线性化接口。也就是，已完成的应用调用`shardkv/client.go`中的`Clerk.Get()`，`Clerk.Put()`和`Clerk.Append()`方法必须表现出以相同的顺序影响了所有的副本。`Clerk.Get()`应该看到由最近的`Put/Append`写到的同一个键的值。即使在作为配置更改时的`Get`和`Put`。

你的每一个分片只有在该分片的Raft副本组中大多数服务器存活并且可以互相通信的时候，才可以取得进展，并且它们可以和大多数的`shardctrler`服务器通信。你的实现必须进行操作(服务请求以及可以按照需要进行重新配置)即使在一些副本组中的少部分服务器死机、暂不可用或者太慢的时候。

一个分片kv服务器只是一个单副本组的成员。给定副本组中的服务器永远不会更改。

我们向你提供`client.go`的代码，该代码将每个RPC发送到负责该key的副本组。如果副本组说它不为该key服务，那么客户端进行重试；在这种情况下，客户端代码向分片控制器获取最新的配置然后重试。你必须修改`client.go`作为处理重复客户端RPC的支持的一部分，就像在kvraft的lab中一样。

当你完成你的代码时你应该通过所有shardkv的测试除了challenge测试。

> **Note**
>
> 你的服务器不应该调用分片控制器的`Join()`请求。测试会在合适的时候调用`Join()`

#### Task1

你的第一个任务是去通过第一个shardkv测试，在这个测试中，只有一个分片分配，所以您的代码应该和lab3中的服务器代码非常相似。最大的修改将是让您的服务器检测什么时候发生配置，并开始接收匹配当前分片的key的请求。

---

现在你的解决方案适用于静态分片的情况，那么现在就开始处理配置更改问题了。您需要让服务器监视配置更改，并在检测到更改时启动分片迁移过程。如果一个复制组丢失了一个分片，他必须立即停止对该分片中的key的请求，并开始将该分片的数据迁移到接管所有者的复制组中。

---

#### Task2

在配置更改期间实现分片迁移。确保复制组中所有服务器在它们执行的操作操作序列的相同点执行迁移，以便它们都接受或拒绝并发客户端请求。在处理后续测试之前，你应该集中经历通过第二个测试("join then leave")。当你完成所有的测试除了`TestDelete`。就完成了这个任务了。

> **Note**
>
> 你的服务器需要定期轮询分片控制器以获取最新的配置。测试期望您的代码大约每100ms轮询一次；更高频率是可以的，但是更少频率可能会导致问题。

> **Note**
>
> 服务器需要互相发送RPC以便能够在配置更改期间传递分片。分片控制器的`Config`结构体包括了服务器的名字，但是你需要`labrpc.ClientEnd`来发送RPC。你应该使用传递给`StartServer()`的`make_end()`函数来将一个服务器的名字转化为`CleintEnd`。`shardkv/client.go`包含这样的代码。

#### Hints

- 向`server.go`中添加代码，定期向分片控制器中获取最新的配置，如果接收组不为客户端的key所属的分片负责，则添加代码来拒绝客户端请求。你应该通过第一个测试。
- 你的服务器使用用一个`ErrWrongGroup`的错误来响应客户端的请求当请求的key不是该server应该服务的时候(即一个没有分配给该副本组的分片的Key)。确保你的`Get`，`Put`以及`Append`处理程序能够在并发的重新配置下能够正确的做出决定。
- 按顺序一次只进行一个重新配置。
- 如果测试失败，检查gob错误(例如："gob": type not registered for interface...)。Go不认为gob错误是致命的，尽管它们在lab中是fetal。
- 考虑一下shardkv客户端和服务端应该如何处理`ErrWrongGroup`。如果客户端接收到`ErrWrongGroup`，它是否应该更改序列号？如果服务器在执行`Get`/`Put`请求时返回了`ErrWrongGroup`，那么它是否应该更新客户端状态？
- 在服务器迁移到新的配置之后，继续存储它不再拥有的分片是可以接受的(尽管在真正系统这会让人遗憾)。这可能可以帮助简化服务器的实现。
- 在配置更改期间G1组需要G2组的分片时，在处理日志条目的G2什么时候将分片发给G1是有关系的吗？
- 你可以在RPC的请求或响应中发送整个映射，这可能会有助于简化分片传输的代码。
- 如果您的RPC处理程序之一在其应答中包含了作为服务器状态的一部分映射(比如kv键值对)，那么你有可能由于race而出bug。RPC系统必须读取到map以便将其发送给调用方，但它没有持续覆盖map的锁。你的服务器，会在RPC系统读取map的时候继续修改该map。解决方案是在RPC处理程序中回复该映射的副本。
- 如果你在Raft日志条目中放置了一个map或者slice，而你的键值对服务之后会看到该日志条目通过`applyCh`，并将对map/slice的引用保存在键值对服务器的状态中。在你的kv服务器修改map/slice以及Raft在持久化时读取该map/slice时会产生了一个竞争。
- 在配置更改期间，一对副本组可能需要在它们之间互相移动分片。如果你看到死锁，这可能是一个来源。

---

## Challenge exercises

如果你要为生产使用构建一个这样的系统，接下来的两个特性是必不可少的。

### Garbage collection of state

> 状态的垃圾回收

当一个副本组失去了分片的所有权时，该副本组应该消除其数据库中应该失去的key。对于它来说，保留这些不再拥有的数据时很浪费的。然而，这给迁移带来一些问题。假设我们有两组，G1和G2，并且有一个新的配置C需要将分片S从G1移动到G2。如果G1从数据库中删除了所有的S中的键值对，那么G2再试图转换到C的时候该如何获取S的数据呢？

#### Challenage

使每个副本组保留旧的碎片的时间不超过一个绝对必要的时间。即使像上面的G1这样的副本组中所有的服务器崩溃并重新启动，您的解决方案也必须正常工作。如果你通过了`TestChallengeDelete`那么你就完成了这个挑战。

### Client requests during configuration changes

> 在配置更改期间的客户端请求

处理配置更改的最简单的方法就是在转换完成之前禁止所有的客户端操作。虽然从概念上将很简单，但是这种方法在生产系统中是不可行的；它会导致在引入或取出机器时对所有的客户机进行长时间的暂停。最好是继续服务不受当前配置更改影响的分片。

#### Challenge1

修改你的解决方案以便于客户端在配置更改期间还可以正常操作不受配置更改影响的分片中的key。当你通过`TestChallenge2Unaffected`时你就完成了这个挑战。

---

虽然上面的优化是好的，但是我们可以做的更改。假设某个副本组G3在转换到C的时候，需要G1的分片S1，以及G2的分片S2。我们真的希望G3可以在接收到必要的状态后立即开始服务分片，即使它仍然在等待其他分片。例如，如果G1宕机，G3在收到来自G2的适当数据之后，仍然可以为S2的请求服务，尽管向C的转换尚未完成。

#### Challenge2

修改你的实现以便于副本组可以在即使配置更改仍在进行的时候，为它们可以服务的分片进行服务。当你通过了`TestChallenge2Partial`的时候你就完成了这个挑战。



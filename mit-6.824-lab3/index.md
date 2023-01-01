# MIT 6.824 Lab3


# MIT-6.824-Lab3

## 前言

这个Lab，我们需要实现一个基于Raft库实现的容错性分布式KV存储服务。

> 实现之前，先仔细阅读官方Lab指导

[官方Lab指导](http://nil.csail.mit.edu/6.824/2021/labs/lab-kvraft.html)

[个人翻译后的官方Lab指导](/mit-6.824-lab指导翻译/)

> 再去了解一下线性一致性读

[关于一致性](https://segmentfault.com/a/1190000022248118)

> 以及官方提供给的一个交互图

[官方交互图](https://pdos.csail.mit.edu/6.824/notes/raft_diagram.pdf)

![image-20220320152754046](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220320152754046.png "各个模块交互流程")

> 并且阅读Raft博士论文中的Figure6.1

[Raft博士论文](https://github.com/ongardie/dissertation/blob/master/stanford.pdf)

### ClientRequest RPC

![image-20220320155254485](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220320155254485.png)

**参数：**

|                 |                                      |
| --------------- | ------------------------------------ |
| **clientId**    | **调用请求的客户端id**               |
| **sequenceNum** | **用于消除重复请求**                 |
| **command**     | **请求状态机的命令，可能会影响状态** |

**结果：**

|                |                                                              |
| -------------- | ------------------------------------------------------------ |
| **status**     | **如果状态机成功应用命令则返回OK**                           |
| **response**   | **如果请求成功，则为状态机的输出**                           |
| **leaderHint** | **最近的领袖的地址，如果服务器知道。(用于不为领袖的服务器接收到之后将正确的领袖地址返回让客户端请求)** |

**接收者实现：**

1. 如果接收者不为领袖，回复`NOT_LEADER`，必要时提供提示。(也就是当该接收者知道最近的领袖是谁的时候，将领袖地址回复给客户端)
2. 追加命令到日志，复制并提交它。
3. 如果没有该客户端id的记录或者该客户端的`sequenceNum`的响应内容已经被丢弃了。
4. 如果客户端请求的序列号`sequenceNum`已经被处理了，那么回复OK并且携带存储的响应。
5. 按照顺序应用日志。
6. 保存状态机的输出和该客户端的序列号，丢弃任何先前的回复。
7. 携带状态机的输出并返回OK。

#### lab实现版本

上述论文提及的`ClientRequest RPC`较为复杂，我们采用简化的版本，即只保存当前最大请求的序列号，而且不提供重定向到领袖的功能。

---

---

## 客户端实现

根据论文以及分析，我们需要实现一个含有id以及还有自增的请求序列号的客户端，当请求失败的时候，不断重试该请求。

> 客户端数据结构

```go
type Clerk struct {
   servers []*labrpc.ClientEnd
   // You will have to modify this struct.
   lastLeader           int        //上一次RPC发现的主机id
   mu                   sync.Mutex //锁
   clientId             int64      //client唯一id
   lastAppliedCommandId int        //Command的唯一id
}
```

> 初始化

```go
func MakeClerk(servers []*labrpc.ClientEnd) *Clerk {
   ck := new(Clerk)
   ck.servers = servers
   // You'll have to add code here.
   ck.lastLeader = 0
   ck.clientId = nrand()
   ck.lastAppliedCommandId = 0
   return ck
}
```

> Get方法

**当正常查询到结果(即使不存在)才会结束该次Get方法，否则继续在该序列号上重试**

```go
func (ck *Clerk) Get(key string) string {

   // You will have to modify this function.
   commandId := ck.lastAppliedCommandId + 1
   args := GetArgs{
      Key:       key,
      ClientId:  ck.clientId,
      CommandId: commandId,
   }
   DPrintf("client[%d]: 开始发送Get RPC;args=[%v]\n", ck.clientId, args)
   //第一个发送的目标server是上一次RPC发现的leader
   serverId := ck.lastLeader
   serverNum := len(ck.servers)
   for ; ; serverId = (serverId + 1) % serverNum {
      var reply GetReply
      DPrintf("client[%d]: 开始发送Get RPC;args=[%v]到server[%d]\n", ck.clientId, args, serverId)
      ok := ck.servers[serverId].Call("KVServer.Get", &args, &reply)
      //当发送失败或者返回不是leader时,则继续到下一个server进行尝试
      if !ok || reply.Err == ErrTimeout || reply.Err == ErrWrongLeader {
         DPrintf("client[%d]: 发送Get RPC;args=[%v]到server[%d]失败,ok = %v,Reply=[%v]\n", ck.clientId, args, serverId, ok, reply)
         continue
      }
      DPrintf("client[%d]: 发送Get RPC;args=[%v]到server[%d]成功,Reply=[%v]\n", ck.clientId, args, serverId, reply)
      //若发送成功,则更新最近发现的leader
      ck.lastLeader = serverId
      ck.lastAppliedCommandId = commandId
      if reply.Err == ErrNoKey {
         return ""
      }
      return reply.Value
   }
}
```

> PutAppend方法

**和Get方法同理**

```go
func (ck *Clerk) PutAppend(key string, value string, op string) {
   //fmt.Println("key=", key, "value=", value, "op=", op)
   // You will have to modify this function.
   commandId := ck.lastAppliedCommandId + 1
   args := PutAppendArgs{
      Key:       key,
      Value:     value,
      Op:        op,
      ClientId:  ck.clientId,
      CommandId: commandId,
   }
   //第一个发送的目标server是上一次RPC发现的leader
   DPrintf("client[%d]: 开始发送PutAppend RPC;args=[%v]\n", ck.clientId, args)
   serverId := ck.lastLeader
   serverNum := len(ck.servers)
   for ; ; serverId = (serverId + 1) % serverNum {
      var reply PutAppendReply
      DPrintf("client[%d]: 开始发送PutAppend RPC;args=[%v]到server[%d]\n", ck.clientId, args, serverId)
      ok := ck.servers[serverId].Call("KVServer.PutAppend", &args, &reply)
      //当发送失败或者返回不是leader时,则继续到下一个server进行尝试
      if !ok || reply.Err == ErrTimeout || reply.Err == ErrWrongLeader {
         DPrintf("client[%d]: 发送PutAppend RPC;args=[%v]到server[%d]失败,ok = %v,Reply=[%v]\n", ck.clientId, args, serverId, ok, reply)
         continue
      }
      DPrintf("client[%d]: 发送PutAppend RPC;args=[%v]到server[%d]成功,Reply=[%v]\n", ck.clientId, args, serverId, reply)
      //若发送成功,则更新最近发现的leader以及commandId
      ck.lastLeader = serverId
      ck.lastAppliedCommandId = commandId
      return
   }
}
```

---

---

## kv服务层数据结构

接下来需要实现本lab最重要的模块，也就是一个kv存储服务，我们暂时不用管真正的持久化操作，只需要写入到内存即可。

首先，我们需要一个用来存放kv数据的结构，那么就使用哈希表来存放。

接下来根据上述提到的客户端交互方法，我们还需要一个数据机构来存放客户端的id和客户端请求序列号，那么也可以采用哈希表，以客户端id为key，以请求的序列号和响应为value。

还需要记录当前状态机最后应用的日志索引。

```go
type KVServer struct {
   mu      sync.Mutex
   me      int
   rf      *raft.Raft
   applyCh chan raft.ApplyMsg
   dead    int32 // set by Kill()

   maxraftstate int // snapshot if log grows this big

   // Your definitions here.
   clientReply    map[int64]CommandContext    //客户端的唯一指令和响应的对应map
   kvDataBase     KvDataBase                  //数据库,可自行定义和更换
   storeInterface store                       //数据库接口
   replyChMap     map[int]chan ApplyNotifyMsg //某index的响应的chan
   lastApplied    int                         //上一条应用的log的index,防止快照导致回退
}

// ApplyNotifyMsg 可表示GetReply和PutAppendReply
type ApplyNotifyMsg struct {
	Err   Err
	Value string //当Put/Append请求时此处为空
	//该被应用的command的term,便于RPC handler判断是否为过期请求(之前为leader并且start了,但是后来没有成功commit就变成了follower,导致一开始Start()得到的index处的命令不一定是之前的那个,所以需要拒绝掉;
	//或者是处于少部分分区的leader接收到命令,后来恢复分区之后,index处的log可能换成了新的leader的commit的log了
	Term int
}

type CommandContext struct {
	Command int            //该client的目前的commandId
	Reply   ApplyNotifyMsg //该command的响应
}
```

---

---

## 接收Raft提交的日志

Raft会通过`applyCh`提交日志到状态机中，让状态机应用命令，因此我们需要一个不断接收`applyCh`中日志的协程，并应用命令。

代码实现如下：

```go
func (kv *KVServer) ReceiveApplyMsg() {
   for !kv.killed() {
      select {
      case applyMsg := <-kv.applyCh:
         DPrintf("kvserver[%d]: 获取到applyCh中新的applyMsg=[%v]\n", kv.me, applyMsg)
         //当为合法命令时
         if applyMsg.CommandValid {
            kv.ApplyCommand(applyMsg)
         } else if applyMsg.SnapshotValid {
            //当为合法快照时
            kv.ApplySnapshot(applyMsg)
         } else {
            //非法消息
            DPrintf("kvserver[%d]: error applyMsg from applyCh: %v\n", kv.me, applyMsg)
         }
      }
   }
}
```

**由于从Raft中接收到的日志不仅有命令日志还有快照日志，因此需要也把快照日志接收了**

---

---

## 发送Get/PutAppend请求

根据上述规则，我们可以总结出以下步骤：

1. 当接收到Get/PutAppend请求时，先判断该客户端的该序列号的请求是否已经被正确处理并响应过了，如果有则返回之前响应的结果。
2. 判断当前是否为领袖。(这里的是否为领袖只是自己认为自己是否是领袖，有可能因为网络分区等原因导致了自己认为自己仍然是领袖，但是实际上已经不为领袖了)。若不为领袖，则返回并告诉客户端自己不为领袖，需要去别的服务器那里请求。
3. 将该命令传递给Raft库去进行一致性复制。
4. 由该命令生成的日志的索引为key，创建一个value为`chan`的通知通道，用于接收该命令被应用过的响应消息。
5. 在该通道上等待，并且设定超时时间，若超时则让客户端重试。
6. 若成功接收到该命令被应用后的响应，则判断此时的响应传来的日志任期和之前传给Raft的时候生成的日志的任期是否相同。(有可能该日志并没有被commit就掉线了，之后别的服务器当选领袖，然后在该索引上提交了一条日志，后来这条日志被应用，但是这个日志就不是我们之前传给Raft的日志，也就是任期肯定不相同)若不相同，则丢弃不处理，否则正常响应给客户端。

> Get请求代码

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
   // Your code here.
   kv.mu.Lock()
   //defer kv.mu.Unlock()
   defer func() {
      DPrintf("kvserver[%d]: 返回Get RPC请求,args=[%v];Reply=[%v]\n", kv.me, args, reply)
   }()
   DPrintf("kvserver[%d]: 接收Get RPC请求,args=[%v]\n", kv.me, args)
   //1.先判断该命令是否已经被执行过了
   if commandContext, ok := kv.clientReply[args.ClientId]; ok {
      if commandContext.Command >= args.CommandId {
         //若当前的请求已经被执行过了,那么直接返回结果
         reply.Err = commandContext.Reply.Err
         reply.Value = commandContext.Reply.Value
         kv.mu.Unlock()
         return
      }
   }
   kv.mu.Unlock()
   //2.若命令未被执行,那么开始生成Op并传递给raft
   op := Op{
      CommandType: GetMethod,
      Key:         args.Key,
      ClientId:    args.ClientId,
      CommandId:   args.CommandId,
   }
   index, term, isLeader := kv.rf.Start(op)
   //3.若不为leader则直接返回Err
   if !isLeader {
      reply.Err = ErrWrongLeader
      //kv.mu.Unlock()
      return
   }
   replyCh := make(chan ApplyNotifyMsg, 1)
   kv.mu.Lock()
   kv.replyChMap[index] = replyCh
   DPrintf("kvserver[%d]: 创建reply通道:index=[%d]\n", kv.me, index)
   kv.mu.Unlock()
   //4.等待应用后返回消息
   select {
   case replyMsg := <-replyCh:
      //当被通知时,返回结果
      DPrintf("kvserver[%d]: 获取到通知结果,index=[%d],replyMsg: %v\n", kv.me, index, replyMsg)
      if term == replyMsg.Term {
         reply.Err = replyMsg.Err
         reply.Value = replyMsg.Value
      } else {
         reply.Err = ErrWrongLeader
      }
   case <-time.After(500 * time.Millisecond):
      DPrintf("kvserver[%d]: 处理请求超时: %v\n", kv.me, op)
      reply.Err = ErrTimeout
   }
   //5.清除chan
   go kv.CloseChan(index)
}
```

> PutAppend请求

```go
func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
   // Your code here.
   kv.mu.Lock()
   //defer kv.mu.Unlock()
   defer func() {
      DPrintf("kvserver[%d]: 返回PutAppend RPC请求,args=[%v];Reply=[%v]\n", kv.me, args, reply)
   }()
   DPrintf("kvserver[%d]: 接收PutAppend RPC请求,args=[%v]\n", kv.me, args)
   //1.先判断该命令是否已经被执行过了
   if commandContext, ok := kv.clientReply[args.ClientId]; ok {
      //若该命令已被执行了,直接返回刚刚返回的结果
      if commandContext.Command == args.CommandId {
         //若当前的请求已经被执行过了,那么直接返回结果
         reply.Err = commandContext.Reply.Err
         DPrintf("kvserver[%d]: CommandId=[%d]==CommandContext.CommandId=[%d] ,直接返回: %v\n", kv.me, args.CommandId, commandContext.Command, reply)
         kv.mu.Unlock()
         return
      }
   }
   kv.mu.Unlock()
   //2.若命令未被执行,那么开始生成Op并传递给raft
   op := Op{
      CommandType: CommandType(args.Op),
      Key:         args.Key,
      Value:       args.Value,
      ClientId:    args.ClientId,
      CommandId:   args.CommandId,
   }
   index, term, isLeader := kv.rf.Start(op)
   //3.若不为leader则直接返回Err
   if !isLeader {
      reply.Err = ErrWrongLeader
      return
   }
   replyCh := make(chan ApplyNotifyMsg, 1)
   kv.mu.Lock()
   kv.replyChMap[index] = replyCh
   DPrintf("kvserver[%d]: 创建reply通道:index=[%d]\n", kv.me, index)
   kv.mu.Unlock()
   //4.等待应用后返回消息
   select {
   case replyMsg := <-replyCh:
      //当被通知时,返回结果
      if term == replyMsg.Term {
         reply.Err = replyMsg.Err
      } else {
         reply.Err = ErrWrongLeader
      }
   case <-time.After(500 * time.Millisecond):
      //超时,返回结果,但是不更新Command -> Reply
      reply.Err = ErrTimeout
      DPrintf("kvserver[%d]: 处理请求超时: %v\n", kv.me, op)
   }
   go kv.CloseChan(index)
}
```

---

---

## 应用日志中的命令

我们之前开启了一个协程不断从`applyCh`中接收提交的日志以及快照，那么我们需要对这些日志和快照进行处理。首先，我们需要处理命令，由于这时候有`Get`/`Put`/`Append`三种命令，因此我们需要分别处理。我们可以对其的实际存储操作就行封装，就留三个操作接口即可。

应用流程如下：

1. 接收到日志，先判断是否已经应用过了。应用过则不处理，等待处理请求的协程超时返回。
2. 判断命令类型，做相应处理。
3. 若需要通知，则通知处理请求的协程。
4. 更新`clientReply`，用于去重。
5. 判断是否需要快照。(后续会提到这个流程)

```go
func (kv *KVServer) ApplyCommand(applyMsg raft.ApplyMsg) {
   kv.mu.Lock()
   defer kv.mu.Unlock()
   var commonReply ApplyNotifyMsg
   op := applyMsg.Command.(Op)
   index := applyMsg.CommandIndex
   //当命令已经被应用过了
   if commandContext, ok := kv.clientReply[op.ClientId]; ok && commandContext.Command >= op.CommandId {
      DPrintf("kvserver[%d]: 该命令已被应用过,applyMsg: %v, commandContext: %v\n", kv.me, applyMsg, commandContext)
      commonReply = commandContext.Reply
      return
   }
   //当命令未被应用过
   if op.CommandType == GetMethod {
      //Get请求时
      if value, ok := kv.storeInterface.Get(op.Key); ok {
         //有该数据时
         commonReply = ApplyNotifyMsg{OK, value, applyMsg.CommandTerm}
      } else {
         //当没有数据时
         commonReply = ApplyNotifyMsg{ErrNoKey, value, applyMsg.CommandTerm}
      }
   } else if op.CommandType == PutMethod {
      //Put请求时
      value := kv.storeInterface.Put(op.Key, op.Value)
      commonReply = ApplyNotifyMsg{OK, value, applyMsg.CommandTerm}
   } else if op.CommandType == AppendMethod {
      //Append请求时
      newValue := kv.storeInterface.Append(op.Key, op.Value)
      commonReply = ApplyNotifyMsg{OK, newValue, applyMsg.CommandTerm}
   }
   //通知handler去响应请求
   if replyCh, ok := kv.replyChMap[index]; ok {
      DPrintf("kvserver[%d]: applyMsg: %v处理完成,通知index = [%d]的channel\n", kv.me, applyMsg, index)
      replyCh <- commonReply
      DPrintf("kvserver[%d]: applyMsg: %v处理完成,通知完成index = [%d]的channel\n", kv.me, applyMsg, index)
   }
   value, _ := kv.storeInterface.Get(op.Key)
   DPrintf("kvserver[%d]: 此时key=[%v],value=[%v]\n", kv.me, op.Key, value)
   //更新clientReply
   kv.clientReply[op.ClientId] = CommandContext{op.CommandId, commonReply}
   DPrintf("kvserver[%d]: 更新ClientId=[%d],CommandId=[%d],Reply=[%v]\n", kv.me, op.ClientId, op.CommandId, commonReply)
   kv.lastApplied = applyMsg.CommandIndex
   //判断是否需要快照
   if kv.needSnapshot() {
      kv.startSnapshot(applyMsg.CommandIndex)
   }
}
```

---

---

## 主动进行快照

上述我们已经实现了一个没有快照的kv服务层，但是我们的日志和数据不能无限制的增长下去，而且为了快速同步落后很多的节点，我们在之前在Raft层已经实现了部分快照的功能，在当前模块，我们需要对是否进行快照进行判断，并且通知Raft去执行快照操作。

我们首先需要完成的是主动进行快照，也就是当状态机接收到新的提交的日志之后，可以判断一下当前Raft必须状态的大小，如果接近于我们设定的阈值或者超过该阈值，则可以主动进行快照。

主动进行快照，就是先将自己的状态机必要状态(必须要同步的数据)编码并生成状态机快照数据，然后通知Raft进行快照，并且将状态机快照数据传递给它。

> 判断是否需要快照

```go
//判断当前是否需要进行snapshot(90%则需要快照)
func (kv *KVServer) needSnapshot() bool {
   if kv.maxraftstate == -1 {
      return false
   }
   var proportion float32
   proportion = float32(kv.rf.GetRaftStateSize() / kv.maxraftstate)
   return proportion > 0.9
}
```

> 创建快照

```go
//生成server的状态的snapshot
func (kv *KVServer) createSnapshot() []byte {
   w := new(bytes.Buffer)
   e := labgob.NewEncoder(w)
   //编码kv数据
   err := e.Encode(kv.kvDataBase)
   if err != nil {
      log.Fatalf("kvserver[%d]: encode kvData error: %v\n", kv.me, err)
   }
   //编码clientReply(为了去重)
   err = e.Encode(kv.clientReply)
   if err != nil {
      log.Fatalf("kvserver[%d]: encode clientReply error: %v\n", kv.me, err)
   }
   snapshotData := w.Bytes()
   return snapshotData
}
```

> 状态机进行快照

```go
//主动开始snapshot(由leader在maxRaftState不为-1,而且目前接近阈值的时候调用)
func (kv *KVServer) startSnapshot(index int) {
   DPrintf("kvserver[%d]: 容量接近阈值,进行快照,rateStateSize=[%d],maxRaftState=[%d]\n", kv.me, kv.rf.GetRaftStateSize(), kv.maxraftstate)
   snapshot := kv.createSnapshot()
   DPrintf("kvserver[%d]: 完成service层快照\n", kv.me)
   //通知Raft进行快照
   go kv.rf.Snapshot(index, snapshot)
}
```

**调用Raft的`Snapshot`方法来通知Raft进行快照**

---

---

## 被动进行快照

当Raft接收到领袖的`InstallSnapshot RPC`时，会先将快照命令通过`applyCh`传递给状态机，然后由状态机来处理，它会对旧的快照进行丢弃，避免回退，然后通知Raft去进行快照，如果Raft快照成功，则状态机再应用该快照的数据到状态机中。

代码如下：

```go
// ApplySnapshot 被动应用snapshot
func (kv *KVServer) ApplySnapshot(msg raft.ApplyMsg) {
   kv.mu.Lock()
   defer kv.mu.Unlock()
   DPrintf("kvserver[%d]: 接收到leader的快照\n", kv.me)
   if msg.SnapshotIndex < kv.lastApplied {
      DPrintf("kvserver[%d]: 接收到旧的日志,snapshotIndex=[%d],状态机的lastApplied=[%d]\n", kv.me, msg.SnapshotIndex, kv.lastApplied)
      return
   }
   if kv.rf.CondInstallSnapshot(msg.SnapshotTerm, msg.SnapshotIndex, msg.Snapshot) {
      kv.lastApplied = msg.SnapshotIndex
      //将快照中的service层数据进行加载
      kv.readSnapshot(msg.Snapshot)
      DPrintf("kvserver[%d]: 完成service层快照\n", kv.me)
   }
}
```

如果是被动进行快照，那么还需要对领袖传来的快照数据进行读取。

读取快照代码如下：

```go
func (kv *KVServer) readSnapshot(snapshot []byte) {
   if snapshot == nil || len(snapshot) < 1 {
      return
   }
   r := bytes.NewBuffer(snapshot)
   d := labgob.NewDecoder(r)
   var kvDataBase KvDataBase
   var clientReply map[int64]CommandContext
   if d.Decode(&kvDataBase) != nil || d.Decode(&clientReply) != nil {
      DPrintf("kvserver[%d]: decode error\n", kv.me)
   } else {
      kv.kvDataBase = kvDataBase
      kv.clientReply = clientReply
      kv.storeInterface = &kvDataBase
   }
}
```

---

---

## 总结

这个lab是基于上一个lab的，其实做到这里，就可以改成一个分布式kv数据库并部署应用了。我自己稍微改一下做了个项目：[bluedis](https://github.com/TheR1sing3un/bluedis)(没有做很多功能，只是基本实现了一些读写操作，后续有时间会继续进行这个项目)。

Lab4的文档后续有时间会继续写，若对于内容有问题的欢迎联系我一起交流讨论！



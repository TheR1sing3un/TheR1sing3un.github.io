# KV层的Read请求优化


# KV层的Read请求优化

## 问题

之前我们为了实现线性化的读写，我们将每一个读写操作都封装成Log打入到Raft中，因为Raft可以保证我们的log在多个节点之间是由共识的，不会乱序，因此可以实现线性化的读写操作。

但是我们发现read操作并不需要在状态机中应用，它只是需要读写到目前的最新值，那么如果我们将其放入log中，然后走Raft流程使其可以被线性化的读取。但是一般的场景都是读多写少的情况，如果我们每一个读操作都走一遍log，会导致性能较低。

---

## 分析

我们可以参考[Raft拓展论文](https://raft.github.io/raft.pdf)中的第八节的客户端交互部分。

![image-20220305163118410](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220305163118410.png)

>只读的操作可以直接处理而不需要记录日志。但是，在不增加任何限制的情况下，这么做可能会冒着返回脏数据的风险，因为响应客户端请求的领导人可能在他不知道的时候已经被新的领导人取代了。线性化的读操作必须不能返回脏数据，Raft 需要使用两个额外的措施在不使用日志的情况下保证这一点。首先，领导人必须有关于被提交日志的最新信息。领导人完全特性保证了领导人一定拥有所有已经被提交的日志条目，但是在他任期开始的时候，他可能不知道哪些是已经被提交的。为了知道这些信息，他需要在他的任期里提交一条日志条目。Raft 中通过领导人在任期开始的时候提交一个空白的没有任何操作的日志条目到日志中去来实现。第二，领导人在处理只读的请求之前必须检查自己是否已经被废黜了（他自己的信息已经变脏了如果一个更新的领导人被选举出来）。Raft 中通过让领导人在响应只读请求之前，先和集群中的大多数节点交换一次心跳信息来处理这个问题。可选的，领导人可以依赖心跳机制来实现一种租约的机制，但是这种方法依赖时间来保证安全性（假设时间误差是有界的）。

**总结一下为如下几点:**

1. 根据Raft算法，Leader一定拥有当前所有的已经被提交的日志，因此请求到的数据一定是准确的（Leader仍在任期内）。因此可以直接进行读操作。
2. 但是Leader可能已经被替代了，但是自己还不知道，若这种情况直接进行读操作，那么是很有可能读到脏数据的，因此Leader需要确定自己仍然是Leader。
3. 新上线的Leader有可能自己并不知道目前哪些日志已经被提交了（选举Leader的时候，该节点拥有最新的日志，但是可能它在接收到这一批新日志之后，原来的Leader宕机了，导致这些日志虽然存在，但是没有被Leader告知可以提交，也就是没有更新`commitIndex`），因此需要自己上线后提交一条空日志，这样可以立马发送新的这个空日志，然后更新自己保存的`nextIndex`数据(在Leader上线时，会更新成`commitIndex+1`，因此`commitindex`若不是最新的，会导致`nextIndex`也不是最新的)，然后我们的`updateCommitIndex`协程会将`commitIndex`更新，这时候我们的`commitIndex`就是最新的了。

---

## ReadIndex Read

若我们需要实现线性化的读。之前的方案是直接写进log里面，即可以保证，该log的index前的日志都可以被读到，但是其实这个log本身是没有作用的，只是为了确保我们可以读到index前的数据。那么我们可以进行优化，我们就不用该log，就直接当读请求来的时候，记录下来当前的`commitIndex`，当该index在apply协程处被应用的时候，代表我们可以读到index前的最新的数据了，这时候执行读操作，然后返回即可。

因此根据论文和上述分析，我们可以总结为如下流程：

1. 当接收到到一个读请求的时候，先判断该请求是否已经被执行过了，若是则直接返回上次读到的结果。

2. 当接收到到一个读请求的时候，先判断自己认为自己还是不是Leader。(这里只是自己认为，因为实际可能因为网络分区等原因，自己已经不是Leader了)
3. 这时，立马发送一轮心跳，当收到大部分节点的对应响应之后。可以确定目前仍是Leader了。
4. 记录下来当前的`commitIndex`和其log的`term`，然后判断该`term`和leader的当前`term`是否一致（若是一个新上线leader，但是它的空日志还没提交成功，这时候`commitIndex`还是旧的，所以就不可以进行ReadIndex Read，需要等到`term`一致的时候，因为我们提交的空日志的`term`一定是最新的，和当前leader的`term`一致），若不一致则继续等待取到一致的，然后进行等待该`commitIndex`的log被应用到状态机。
5. 状态机执行到该index处其以后的日志的时候，则可以进行读操作，并返回给client。

---

## 修改KV层Get/Apply

> 原先的Get操作和Apply操作（以kvraft/server为例）

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
   // Your code here.
   kv.mu.Lock()
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

```go
func (kv *KVServer) applyCommand(applyMsg raft.ApplyMsg) {
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
      replyCh <- commonReply
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

> 更改后

> Get

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
   // Your code here.
   kv.mu.Lock()
   defer func() {
      DPrintf("kvserver[%d]: 返回Get RPC请求,args=[%v];Reply=[%v]\n", kv.me, args, reply)
   }()
   DPrintf("kvserver[%d]: 接收Get RPC请求,args=[%v]\n", kv.me, args)
   //1.先判断该命令是否已经被执行过了
   if request, ok := kv.clientReply[args.ClientId]; ok {
      if request.RequestId >= args.CommandId {
         //若当前的请求已经被执行过了,那么直接返回结果
         reply.Err = request.Err
         reply.Value = request.Value
         DPrintf("kvserver[%d]: 该Get RPC请求为历史请求,args=[%v],reply=[%v]\n", kv.me, args, reply)
         kv.mu.Unlock()
         return
      }
   }
   kv.mu.Unlock()
   //2.发送一轮心跳来检查自己是否还是Leader
   if !kv.rf.CheckIsLeader() {
      //不为Leader则返回
      reply.Err = ErrWrongLeader
      return
   }
   //3.当前仍为Leader,取当前的commitIndex(一定可以取到和节点term相同的log的commitIndex,由Raft的该方法自行保证)
   readIndex := kv.rf.GetCommitIndex()
   DPrintf("kvserver[%d]:获取ReadIndex: %d\n", kv.me, readIndex)
   //等待该readIndex被应用到状态机
   kv.mu.Lock()
   var cond *sync.Cond
   if c, ok := kv.readCon[readIndex]; !ok {
      //若没有该readIndex的con,则新建一个
      lock := &sync.Mutex{}
      cond = sync.NewCond(lock)
      kv.readCon[readIndex] = cond
   } else {
      cond = c
   }
   DPrintf("kvserver[%d]: 开始等待ReadIndex: %d,args=[%v]\n", kv.me, readIndex, args)
   kv.mu.Unlock()
   //在该cond上等待
   cond.L.Lock()
   cond.Wait()
   cond.L.Unlock()
   kv.mu.Lock()
   defer kv.mu.Unlock()
   //执行读请求
   if value, ok := kv.storeInterface.Get(args.Key); !ok {
      reply.Err = ErrNoKey
   } else {
      reply.Err = OK
      reply.Value = value
   }
   //更新clientReply
   kv.clientReply[args.ClientId] = RequestResult{args.CommandId, reply.Err, reply.Value}
}
```

> Apply

```go
func (kv *KVServer) applyCommand(applyMsg raft.ApplyMsg) {
   kv.mu.Lock()
   defer kv.mu.Unlock()
   var commonReply ApplyNotifyMsg
   index := applyMsg.CommandIndex
   if index <= kv.lastApplied {
      return
   }
   kv.lastApplied = index
   if applyMsg.Command != nil {
      op := applyMsg.Command.(Op)
      //当命令已经被应用过了
      if request, ok := kv.clientReply[op.ClientId]; ok && request.RequestId >= op.CommandId {
         DPrintf("kvserver[%d]: 该命令已被应用过,applyMsg: %v, requestReply: %v\n", kv.me, applyMsg, request)
         commonReply.Err = request.Err
         commonReply.Value = request.Value
         commonReply.Term = applyMsg.CommandTerm
         return
      }
      //当命令未被应用过
      if op.CommandType == PutMethod {
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
         replyCh <- commonReply
      }
      value, _ := kv.storeInterface.Get(op.Key)
      DPrintf("kvserver[%d]: 此时key=[%v],value=[%v]\n", kv.me, op.Key, value)
      //更新clientReply
      kv.clientReply[op.ClientId] = RequestResult{op.CommandId, commonReply.Err, commonReply.Value}
      DPrintf("kvserver[%d]: 更新ClientId=[%d],CommandId=[%d],Reply=[%v]\n", kv.me, op.ClientId, op.CommandId, commonReply)
   }
   //判断是否需要快照
   if kv.needSnapshot() {
      kv.startSnapshot(applyMsg.CommandIndex)
   }
}
```

---

## 增加CheckReadIndex方法

我们需要等到状态机至少应用到`readIndex`的位置的时候才能进行目标读取操作，因此我们可以单独使用一个协程来完成。定期去检查所有的`readCon`中的`readIndex`是否小于等于`lastApplied`，若是则可以直接唤醒所有在该`readIndex`处等待的Get的handler。

```go
func (kv *KVServer) checkReadIndex() {
   for !kv.killed() {
      //检查是否有可以返回的readIndex了
      kv.mu.Lock()
      for readIndex, cond := range kv.readCon {
         if readIndex <= kv.lastApplied {
            DPrintf("kvserver[%d]: 检查到ReadIndex: %v,lastApplied: %v,因为通知更新返回get结果\n", kv.me, readIndex, kv.lastApplied)
            cond.Broadcast()
         }
      }
      kv.mu.Unlock()
      time.Sleep(100 * time.Millisecond)
   }
}
```

---

## Raft改动

我们为了实现ReadIndex特性，需要对之前我们的raft模块进行更改。主要是为了实现一个可以检验当前是否仍然是leader的方法以及一个获取当前最新的`commitIndex`方法，以及leader一上线立马追加一个空日志用于及时更新到最新的`commitIndex`。

### 上线Start一个空日志

我们为了领袖能够迅速将自己的`commitIndex`更新为目前集群中最新的`commitIndex`，因此需要一上线start一条空日志。为什么会出现领袖并没有当前的最新的`commitIndex`呢，其实可以举个例子。

> 一开始A节点为领袖(当前所有节点的commitIndex都为3)

| 服务器Id  | Index/Term |      |      |
| --------- | ---------- | ---- | ---- |
| A(leader) | 1/1        | 2/1  | 3/1  |
| B         | 1/1        | 2/1  | 3/1  |
| C         | 1/1        | 2/1  | 3/1  |
| D         | 1/1        | 2/1  | 3/1  |
| E         | 1/1        | 2/1  | 3/1  |

> 然后现在领袖A接收到两条新日志，并且将其发送给其他节点，但是这时候A宕机了。导致当前集群中超过一半的节点拥有索引为4和5的日志，但是`commitIndex`并没有更新。(`commitIndex`是需要领袖去通知跟随者更新的)

| 服务器Id          | Index/Term |      |      |      |      |
| ----------------- | ---------- | ---- | ---- | ---- | ---- |
| A(leader，已宕机) | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| B                 | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| C                 | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| D                 | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| E                 | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |

> 此时大家的`commitIndex`仍为3，假如B选举当选了新的领袖

| 服务器Id  | Index/Term |      |      |      |      |
| --------- | ---------- | ---- | ---- | ---- | ---- |
| A(已宕机) | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| B(leader) | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| C         | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| D         | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |
| E         | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |

> 那么这时候领袖的`commitIndex`就不是该集群中实际的可被提交的最高日志索引了，因此需要快速更新`commitIndex`的话，可以一上线提交一条空日志。

> 空日志索引为6任期为2，在一轮`AppendEntries`后C D E都成功复制了该日志，此时B根据规则可以更新`commitIndex`为6，并且此时该`commitIndex`处的日志的任期也和当前领袖的任期相同。

| 服务器Id  | Index/Term |      |      |      |      |      |
| --------- | ---------- | ---- | ---- | ---- | ---- | ---- |
| A(已宕机) | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  |      |
| B(leader) | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  | 6/2  |
| C         | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  | 6/2  |
| D         | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  | 6/2  |
| E         | 1/1        | 2/1  | 3/1  | 4/1  | 5/1  | 6/2  |

> 在下一轮`AppendEntires`后，根据规则，`commitIndex = min(leaderCommit, last log index)`，C D E也正确更新`commitIndex`为6了

**这就是为什么需要新领袖一上线立马提交一条空白日志，因为这样可以快速的将每个服务器的`commitIndex`更新到最新**

---

### 检验自己仍是Leader

该函数由KV层调用，用于在请求读的时候检验自己是否仍然是leader。通过发送一批appendEntries，并且判断是否过半的follower仍然认为我是leader

> CheckIsLeader方法

```go
// CheckIsLeader 检查当前是否仍是leader
func (rf *Raft) CheckIsLeader() (isLeader bool) {
   //发送一轮广播
   rf.mu.Lock()
   defer rf.mu.Unlock()
   defer func() {
      DPrintf("id[%d].state[%v].term[%d]: 检查目前是否为Leader: %v\n", rf.me, rf.state, rf.currentTerm, isLeader)
   }()
   DPrintf("id[%d].state[%v].term[%d]: 开始检查自己是否为leader\n", rf.me, rf.state, rf.currentTerm)
   if rf.state != LEADER {
      return false
   }
   cond := sync.NewCond(&rf.mu)
   ch := make(chan bool, 1)
   go rf.BoardCastOneRound(cond, ch)
   cond.Wait()
   DPrintf("id[%d].state[%v].term[%d]: 检查Leader协程被唤醒: %v\n", rf.me, rf.state, rf.currentTerm, isLeader)
   return <-ch
}
```

> BroadcastOneRound

该函数发送一轮appendEntries来判断是否过半节点认为自己仍是leader。

```go
// BroadcastOneRound 发起广播发送AppendEntries RPC
func (rf *Raft) BroadcastOneRound(cond *sync.Cond, ch chan bool) {
   rf.mu.Lock()
   wg := &sync.WaitGroup{}
   var successNums int64
   successNums = 1
   if rf.state == LEADER {
      DPrintf("id[%d].state[%v].term[%d]: 开始一轮检验leader广播\n", rf.me, rf.state, rf.currentTerm)
      for i := range rf.peers {
         if i != rf.me {
            wg.Add(1)
            go func(server int) {
               DPrintf("server = %v\n", server)
               if rf.HandleAppendEntries(server) {
                  atomic.AddInt64(&successNums, 1)
                  DPrintf("id[%d].state[%v].term[%d]: 节点%v 同意本节点仍为leader\n", rf.me, rf.state, rf.currentTerm, server)
               }
               wg.Done()
            }(i)
         }
      }
   }
   rf.mu.Unlock()
   //等待所有的返回
   wg.Wait()
   //广播完,通知正在等待的CheckIsLeader协程
   if cond != nil {
      DPrintf("id[%d].state[%v].term[%d]: 通知checkIsLeader协程,successNums: %v\n", rf.me, rf.state, rf.currentTerm, successNums)
      cond.Signal()
      ch <- successNums > int64(len(rf.peers)/2)
   }
}
```

---

### 获取最新的CommitIndex

由处理读请求的协程来调用，返回当前leader的最新`commitIndex`。因为有可能leader刚上线，这时候的`commitIndex`不是最新的，而且发的空日志也还没完成追加，导致这时候的`commitIndex`是旧的，因此我们需要等待，直到我们取到的`commitIndex`处的log的任期和当前相同即可返回。

```go
// GetCommitIndex 返回最新的commitIndex
func (rf *Raft) GetCommitIndex() int {
   rf.mu.Lock()
   defer rf.mu.Unlock()
   for rf.index(rf.commitIndex).Term != rf.currentTerm {
      rf.applyCond.Wait()
   }
   DPrintf("id[%d].state[%v].term[%d]: 当前的commitIndex: %d\n", rf.me, rf.state, rf.currentTerm, rf.commitIndex)
   return rf.commitIndex
}
```

---

## 总结

其实还有继续优化的空间，比如说PingCap提到的`LeaseRead`，使用租约的形式来简化读请求，因为我们可以设定一个租约，它是比选举超时的时间要小的，既可以保证在这个租约内，不会发送leader变更，因此可以直接省略掉向集群内节点发送appendEntries来确定自己是否是leader的步骤。

由于这个实现会较为复杂，而且该lab的架构不太适合进行这个优化，因此我只实现了ReadIndex的优化策略。



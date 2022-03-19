# MIT 6.824 Lab2


# MIT-6.824-Lab2

## 前言

这个Lab我们主要是为了实现Raft论文中的功能，包括：选举、日志、持久化以及快照。

> 强烈建议把Raft论文多看几遍，并且可以自己做一些总结。

**重点看懂如下内容：**

[Raft论文]( https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)论文中的**Figure2**

### State

![image-20220317203458927](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220317203458927.png "Raft的状态")

**所有服务器的持久化状态：**

> 在响应RPC之前在稳定的存储上进行更新

|                 |                                                              |
| --------------- | ------------------------------------------------------------ |
| **currentTerm** | **服务器已知的最新任期(在第一次启动时初始化为0，单调递增)**  |
| **voteFor**     | **当前任期投给的候选人的Id(如果为null就是没有投票)**         |
| **log[]**       | **日志项；每一个日志条目包含一个给状态机的命令，以及当它被leader接收的时候所处的任期(第一个索引是1)** |

**所有服务器的易失状态：**

|                 |                                                       |
| --------------- | ----------------------------------------------------- |
| **commitIndex** | **已知被提交的最高日志的索引(初始化为0，单调递增)**   |
| **lastApplied** | **被状态机应用的最高日志的索引(初始化为0，单调递增)** |

**领袖节点的易失状态：**

> 选举后重新初始化

|                  |                                                              |
| ---------------- | ------------------------------------------------------------ |
| **nextIndex[]**  | **对于每一个服务器，下一个应该被发送给他们的日志的索引(初始化为领袖的最近一个日志的index的下一位)** |
| **matchIndex[]** | **对于每一个服务器，领袖已知的这些节点分别复制到的最高日志的索引(初始化为0，单调递增)** |

---

### AppendEntries RPC

![image-20220317205429607](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220317205429607.png "日志复制请求")

> 由领袖调用去将日志条目复制到节点；也用其作为心跳。

**参数：**

|                  |                                                              |
| ---------------- | ------------------------------------------------------------ |
| **term**         | **领袖的当前任期**                                           |
| **leaderId**     | **领袖的Id，可以用于跟随者记录下来，直接重定向客户端的请求** |
| **prevLogIndex** | **最新的日志的前一个日志的索引**                             |
| **prevLogTerm**  | **preLogIndex处的日志的所属任期**                            |
| **entries[]**    | **发送给跟随者去存储的日志条目(空的表示当前的RPC为心跳包；可以发送不知一个用于提效率)** |
| **ledaerCommit** | **领袖的commitIndex(用于跟随者更新自己的commitIndex)**       |

**结果：**

|             |                                                              |
| ----------- | ------------------------------------------------------------ |
| **term**    | **服务器当前的任期，用于领袖更新自己的任期**                 |
| **success** | **如果跟随着包含匹配preLogIndex和preLogTerm的日志就返回true** |

**接收者实现：**

1. **当`term` < `currentTerm`，返回false。(即发送`AppendEntries`的服务器肯定不是现在的领袖，那么返回false表示不接收这个日志复制请求)**
2. **如果接收者在`prevLogIndex`处的日志的任期没有和`prevLogTerm`匹配，那么返回false。(表示当前追加的日志条目开始的位置不对)**
3. **如果服务器存在一个日志和新的需要复制的日志冲突了(相同的索引但是属于不同的任期)，那么删除掉这个日志条目以及所有处于它后面的日志。**
4. **追加所有当前服务器没有的新日志。**
5. **如果领袖的`leaderCommit` > 当前服务器的`commitIndex`，设置`commitIndex` = `min(leaderCommit, index of last new Entry)`。**

#### **接收者实现细节**

##### **规则3**

**有如下情况：**

**当A节点当选领袖后，它的任期为3，并且接收到了两条新的日志，它将其发送给别的跟随着B,C,D,E。	但是刚发送给B，B成功复制，但是还没有发送给CDE或者发送给CDE的消息因为网络原因丢失了。那么就出现了如下情况。**

> **日志情况如下**

| **服务器名称** | **日志的索引和任期(index/term)** |         |         |         |         |
| -------------- | -------------------------------- | ------- | ------- | ------- | ------- |
| **A**          | **1/1**                          | **2/1** | **3/2** | **4/3** | **5/3** |
| **B**          | **1/1**                          | **2/1** | **3/2** | **4/3** | **5/3** |
| **C**          | **1/1**                          | **2/1** | **3/2** |         |         |
| **D**          | **1/1**                          | **2/1** | **3/2** |         |         |
| **E**          | **1/1**                          | **2/1** | **3/2** |         |         |

**此时日志4和5由于领袖没有成功复制到大多数节点，因此并没有被提交**

**这时候C/D/E可以通过选举得到除了A/B以外的节点同意，因此可以成为新的领袖，这时候由于日志4/3和5/3没有成功被提交，因此领袖可以对其进行删除后再追加新的日志，这也就是规则3需要应对的情况。**

> **此时日志情况如下**

| **服务器名称** | **日志的索引和任期(index/term)** |         |         |         |         |
| -------------- | -------------------------------- | ------- | ------- | ------- | ------- |
| **A**          | **1/1**                          | **2/1** | **3/2** | **4/4** | **5/4** |
| **B**          | **1/1**                          | **2/1** | **3/2** | **4/4** | **5/4** |
| **C**          | **1/1**                          | **2/1** | **3/2** | **4/4** | **5/4** |
| **D**          | **1/1**                          | **2/1** | **3/2** | **4/4** | **5/4** |
| **E**          | **1/1**                          | **2/1** | **3/2** | **4/4** | **5/4** |

---

##### **规则5**

**跟随者成功接收了领袖的`AppendEntire RPC`时，设置跟随者自己的`commitIndex` = `min(leaderCommit, index of last new entry)`。因为有可能领袖的发送给该跟随者的日志最大索引也比当前领袖的`commitIndex`要小，所以要在`leaderCommit`和自己当前最新接收的日志中找那个更小的作为更新的数据。可以防止更新了`commitIndex`但是现在跟随者的日志并没有更新到该索引，导致错误。**

---

### **RequestVote RPC**

**![image-20220317210920159](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220317210920159.png "请求投票RPC方法")**

> **由候选人调用来收集选票**

**参数：**

|                  |                                        |
| ---------------- | -------------------------------------- |
| **term**         | **候选人的任期**                       |
| **candidateId**  | **请求投票的候选人Id**                 |
| **lastLogIndex** | **候选人的最近一个日志条目的索引**     |
| **lastLogTerm**  | **候选人的最近一个日志条目的所属任期** |

**结果：**

|                 |                                                      |
| --------------- | ---------------------------------------------------- |
| **term**        | **服务器当前的任期，用于候选人更新自己**             |
| **voteGranted** | **当候选人符合条件的时候，返回true表示成功投票给它** |

**接收者实现：**

1. **如果`term` < `currentTerm`返回false。(即候选人的任期比当前服务器的任期还小，自然不可能成为领袖，因此返回false)**
2. **当该服务器的`voteFor`是空或者就是当前请求投票的这个候选人，以及候选人的日志最少和接受者的日志一样新，那么就给它投票。**

#### **接收者实现细节**

##### **规则2**

**当服务器的`voteFor`为空或者就是当前请求投票的这个候选人的id，而且需要候选人的日志最少和它一样新，才会给它投票。**

**首先，为了防止出现脑裂的情况，我们一个任期只能有一个领袖，因此如果当前任期已经投票了，就不要再投了，除非是当前已经投给的那个候选人又发了一次`RequestVote RPC`。而且只有当候选人的日志最少和该服务器的日志一样新的时候，才能投票。**

**这里的最少一样新，可以理解为，最新日志的任期更大，或者相同任期但是索引更大。**

---

### **服务器的规则**

**![image-20220318215512320](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220318215512320.png "服务器需要遵守的规则")**

**对于所有服务器**

- **如果`commitIndex` > `lastApplied`：自增`lastApplied`，并且将索引为`lastApplied`的日志应用到状态机中。**
- **如果RPC请求或者响应中的参数`term` > `currentTerm`：那么更新`currentTerm`和`term`相等，然后转化为跟随者。(也就是当前自己任期比别人的小的时候，自己当前一定只能为跟随者，并且需要更新到相同的任期)**

**对于跟随者**

- **响应候选人和领袖发来的RPC。**
- **如果选举计时器到期了还没有收到正确的领袖发来的`AppendEntries`，或者没有投票给候选人，那么就变成了候选人。**

**对于候选人**

- **转换为候选人的时候，开始选举。**
  - **自增当前的任期号`currentTerm`**
  - **给自己投票(防止又给别人投票，出现脑裂)**
  - **重置选举计时器**
  - **给所有其他的服务器发送`RequestVote RPC`**
- **如果从大多数服务器收到了选票，那么成为当前任期的领袖。**
- **从一个新的领袖那里接收到`AppendEntries RPC`，就转换为跟随者。**
- **如果选举计时器到期，开始一轮选举。**

**对于领袖**

- **选举后：发送初始化的`AppendEntries RPC`空包(心跳包)到每一个服务器；并在空闲时期不断发送来防止跟随者选举计时器到期。(为了维护自己的领袖地位)**
- **如果从客户端接收到一个命令：追加日志到本机日志，然后当状态机成功应用该日志之后再响应客户端的请求。**
- **如果最新的日志的索引>=该跟随者的`nextIndex`：发送携带从`nextIndex`开始的日志条目的`AppendEntries RPC`请求到跟随者。**
  - **如果成功了：更新该跟随者的`nextIndex`和`matchIndex`。(这里一定要注意并发问题，后续会提到)**
  - **如果是因为日志不一致而导致的失败：递减`nextIndex`然后重试。(我们采取优化的Fast Backup，可以快速的进行日志的同步)**
- **如果存在一个大于`commitIndex`的N，大多数跟随者的`matchIndex[i] > N`，而且索引为N的日志的任期等于当前任期，那么设置`commitIndex = N`。**

---

---

## **选举**

### **Raft服务器参数**

**我们需要如下参数：**

```go
type Raft struct {
   mu        sync.Mutex          // Lock to protect shared access to this peer's state
   peers     []*labrpc.ClientEnd // RPC end points of all peers
   persister *Persister          // Object to hold this peer's persisted state
   me        int                 // this peer's index into peers[]
   dead      int32               // set by Kill()

   // Your data here (2A, 2B, 2C).
   // Look at the paper's Figure 2 for a description of what
   // state a Raft server must maintain.

   //persistent state
   currentTerm int        //当前任期
   voteFor     int        //当前任期投给的候选人id(为-1时代表没有投票)
   logEntries  []LogEntry //日志条目
   commitIndex int        //当前log中的最高索引(从0开始,递增)
   lastApplied int        //当前被用到状态机中的日志最高索引(从0开始,递增)
   //volatile state on leader
   nextIndex  []int //发送给每台服务器的下一条日志目录索引(初始值为leader的commitIndex + 1)
   matchIndex []int //每台服务器已知的已被复制的最高日志条目索引
   //volatile state on all servers
   state State //当前raft状态

   timerElect       *time.Timer   //选举计时器
   timerHeartBeat   *time.Timer   //心跳计时器
   timeoutHeartBeat int           //心跳频率/ms
   timeoutElect     int           //选举频率/ms
   applyCh          chan ApplyMsg //命令应用通道
   applyCond        *sync.Cond    //命令应用cond
   //最近快照的数据
   snapshotData []byte //最近快照的数据
}

// LogEntry 日志条目
type LogEntry struct {
   Command interface{} //日志记录的命令(用于应用服务的命令)
   Index   int         //该日志的索引
   Term    int         //该日志被接收的时候的Leader任期
}
```

---

### **计时器**

**首先，我们是需要周期性的进行选举的，因此肯定是需要实现一个选举计时器的，那么我们可以直接实现如下代码，使用`time.Timer`来作为定时器。**

> **实现`ticker`方法，该方法不断进行超时判断**

```go
func (rf *Raft) ticker() {
   for rf.killed() == false {

      // Your code here to check if a leader election should
      // be started and to randomize sleeping time using
      // time.Sleep().
      select {
      case <-rf.timerElect.C:
         if rf.killed() {
            break
         }
         rf.mu.Lock()
         DPrintf("id[%d].state[%v].term[%d]: 选举计时器到期\n", rf.me, rf.state, rf.currentTerm)
         if rf.state != LEADER {
            //当不为leader时,也就是超时了,那么转变为Candidate
            rf.startElection()
         }
         //重置选举计时器
         rf.resetElectTimer()
         rf.mu.Unlock()

      case <-rf.timerHeartBeat.C:
         if rf.killed() {
            break
         }
         rf.mu.Lock()
         if rf.state == LEADER {
            //当心跳计时器到时间后,如果是Leader就开启心跳检测
            go rf.Broadcast()
         }
         //重置心跳计时器
         rf.timerHeartBeat.Reset(time.Duration(rf.timeoutHeartBeat) * time.Millisecond)
         rf.mu.Unlock()
      }
   }
}
```

---

### **实现RequestVote**

**现在需要完成的就是`RequestVote RPC`。结合上述规则，可得代码如下(相信我的注释应该很详细的~)**

```go
// RequestVote
// example RequestVote RPC handler.
//
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
   // Your code here (2A, 2B).
   //无论如何,返回参数中的term应修改为自己的term
   rf.mu.Lock()
   defer rf.mu.Unlock()
   defer rf.persist()
   DPrintf("id[%d].state[%v].term[%d]: 接收到[%d]的选举申请\n", rf.me, rf.state, rf.currentTerm, args.CandidateId)
   defer func() {
      DPrintf("id[%d].state[%v].term[%d]: 给[%d]的选举申请返回%v\n", rf.me, rf.state, rf.currentTerm, args.CandidateId, reply.VoteGranted)
   }()
   defer func() {
      reply.Term = rf.currentTerm
   }()
   reply.VoteGranted = false
   //1.如果Term<currentTerm或者已经投过票了,则之直接返回拒绝
   if args.Term < rf.currentTerm || (args.Term == rf.currentTerm && rf.voteFor != -1 && rf.voteFor != args.CandidateId) {
      return
   }
   //2.如果t > currentTerm,则更新currentTerm,并切换为follower
   if args.Term > rf.currentTerm {
      rf.currentTerm = args.Term
      rf.toFollower()
      rf.voteFor = -1
   }
   //3.判断候选人的日志是否最少一样新
   //如果两份日志最后的条目的任期号不同,那么任期号大的日志更加新;如果两份日志最后的条目任期号相同,那么日志比较长的那个就更加新
   if rf.lastLog().Term == -1 || args.LastLogTerm > rf.lastLog().Term || (args.LastLogTerm == rf.lastLog().Term && args.LastLogIndex >= rf.lastLog().Index) {
      //重置选举时间
      rf.resetElectTimer()
      //投票给候选人
      rf.voteFor = args.CandidateId
      //投赞成
      reply.VoteGranted = true
   }
}

// AppendEntriesArgs 日志追加RPC的请求参数
type AppendEntriesArgs struct {
   Term         int        //当前leader的任期
   LeaderId     int        //leader的id,follower可以将client错发给它的请求转发给leader
   PrevLogIndex int        //最新日志前的那一条日志条目的索引
   PrevLogTerm  int        //最新日志前的那一条日志条目的任期
   Entries      []LogEntry //需要被保存的日志条目(为空则为心跳包)
   LeaderCommit int        //leader的commitIndex
}

// AppendEntriesReply 日志追加的RPC的返回值
type AppendEntriesReply struct {
   Term    int  //接收者的currentTerm
   Success bool //如果prevLogIndex和prevLogTerm和follower的匹配则返回true
   XTerm   int  //若follower和leader的日志冲突,则记载的是follower的log在preLogIndex处的term,若preLogIndex处无日志,返回-1
   XIndex  int  //follower中的log里term为XTerm的第一条log的index
   XLen    int  //当XTerm为-1时,此时XLen记录follower的日志长度(不包含初始占位日志)
}
```

---

### **实现选举**

**上述的`ticker`方法，我们会调用一个`rf.startElection()`方法来进行选举，代码实现如下。**

```go
// StartElection 发起选举
func (rf *Raft) startElection() {
   rf.toCandidate()
   defer rf.persist()
   voteNums := 1
   for i := range rf.peers {
      if i == rf.me {
         continue
      }
      args := &RequestVoteArgs{
         Term:         rf.currentTerm,
         CandidateId:  rf.me,
         LastLogIndex: rf.lastLog().Index,
         LastLogTerm:  rf.lastLog().Term,
      }
      go func(i int) {
         reply := &RequestVoteReply{}
         ok := rf.sendRequestVote(i, args, reply)
         if ok {
            rf.mu.Lock()
            defer rf.mu.Unlock()
            if rf.currentTerm == args.Term && rf.state == CANDIDATE {
               if reply.VoteGranted {
                  voteNums++
                  if voteNums > len(rf.peers)/2 {
                     go rf.toLeader()
                  }
               }
            } else if reply.Term > rf.currentTerm {
               rf.currentTerm = reply.Term
               rf.toFollower()
               rf.voteFor = -1
            }
         }
      }(i)
   }
}
```

**当获得大多数选票的时候，就成为了当前任期的领袖。**

---

### **状态转换**

#### **成为领袖**

**上述选举后，若成功的当选领袖，那么就需要转化为领袖的操作，代码实现如下，全部基于解析的论文中的`Figure2`中的规则：**

```go
// toLeader 转变为leader
func (rf *Raft) toLeader() {
   rf.mu.Lock()
   defer rf.mu.Unlock()
   DPrintf("id[%d].state[%v].term[%d]: 成为Leader\n", rf.me, rf.state, rf.currentTerm)
   rf.state = LEADER
   //1.初始化volatile state on leader
   rf.nextIndex = make([]int, len(rf.peers))
   rf.matchIndex = make([]int, len(rf.peers))
   //初始化nextIndex为commitIndex+1
   for i := range rf.nextIndex {
      rf.nextIndex[i] = rf.commitIndex + 1
   }
   //初始化matchIndex为0(实例化的时候已经赋值0了,不需要自己再赋值一次了)
   //当为leader时,开始启动协程来实时更新commitIndex
   go rf.updateCommitIndex()
   //追加一条空日志,用于更新到最新的commitIndex
   go rf.Start(nil)
   //立马开始一轮心跳
   rf.timerHeartBeat.Reset(0)
}
```

**当成为领袖之后，需要启动一个协程来更新自己的`commitIndex`(上述解析中提到，领袖的`commitIndex`需要再合适的时候进行更新)。**

**还需要立马开始一轮心跳，也就是发送一轮`AppendEntries RPC`。**

**这里的代码中还有一个立马进行保存一个空日志的逻辑，这个后续会讲到，属于是优化的内容。**

#### **成为候选人**

**当我们选举计时器到期后，需要转化为候选人并进行选举。根据论文中规则，需要自增自己的任期，并且给自己投票。代码实现如下：**

```go
//转变为候选人
func (rf *Raft) toCandidate() {
   //切换状态
   rf.state = CANDIDATE
   //自增任期号
   rf.currentTerm++
   //给自己投票
   rf.voteFor = rf.me
   DPrintf("id[%d].state[%v].term[%d]: 变成Candidate\n", rf.me, rf.state, rf.currentTerm)
}
```

#### **成为跟随者**

**当服务器接收到比自己任期大的服务器发来的请求或者响应的时候，或者投出选票的时候，需要转化为跟随者。**

**代码实现如下：**

```go
//转变为follower
func (rf *Raft) toFollower() {
   if rf.state == FOLLOWER {
      return
   }
   rf.state = FOLLOWER
   DPrintf("id[%d].state[%v].term[%d]: 变成Follower\n", rf.me, rf.state, rf.currentTerm)
}
```

---

### **广播**

**当服务器成为领袖之后，为了维护自己的领袖地位，需要周期性的发送`AppendEntries RPC`到跟随者，为了重置他们的选举计时器，也是为了进行日志复制。代码实现如下：**

```go
// Broadcast 发起广播发送AppendEntries RPC
func (rf *Raft) Broadcast() {
   rf.mu.Lock()
   defer rf.mu.Unlock()
   if rf.state == LEADER {
      DPrintf("id[%d].state[%v].term[%d]: 开始一轮广播\n", rf.me, rf.state, rf.currentTerm)
      for i := range rf.peers {
         if i != rf.me {
            go rf.HandleAppendEntries(i)
         }
      }
   }
}
```

**这里的`rf.HandleAppendEntries()`方法在选举模块只需要实现发送空的`AppendEntries RPC`即可，发送日志在后续实现**

---

---

## **日志**

**这一模块需要实现Raft之间的日志复制，重点是实现`AppendEntries RPC`以及发送`AppendEntries RPC`的方法`HandleAppendEntries`。**

### **日志的存储**

**我们将日志存在Raft的变量`logEntries`中，是由`LonEntry`数组构成，并且`LogEntry`中存有命令、索引和任期。**



**![image-20220319112044841](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220319112044841.png)**

> **`LogEntry`数据结构**

```go
// LogEntry 日志条目
type LogEntry struct {
   Command interface{} //日志记录的命令(用于应用服务的命令)
   Index   int         //该日志的索引
   Term    int         //该日志被接收的时候的Leader任期
}
```

**我们将索引存在该数据结构里面而不是以数组的下标为索引，原因是：**

- **更容易操作，数组反而涉及到很多下标变化之类的问题。**
- **后续我们的日志会进行快照，也就是该数组中第一个日志并不是索引为1了。**

---

### **实现AppendEntries**

**根据上述分析的论文中的规则，可以实现代码如下：**

```go
// AppendEntriesArgs 日志追加RPC的请求参数
type AppendEntriesArgs struct {
	Term         int        //当前leader的任期
	LeaderId     int        //leader的id,follower可以将client错发给它的请求转发给leader
	PrevLogIndex int        //最新日志前的那一条日志条目的索引
	PrevLogTerm  int        //最新日志前的那一条日志条目的任期
	Entries      []LogEntry //需要被保存的日志条目(为空则为心跳包)
	LeaderCommit int        //leader的commitIndex
}

// AppendEntriesReply 日志追加的RPC的返回值
type AppendEntriesReply struct {
	Term    int  //接收者的currentTerm
	Success bool //如果prevLogIndex和prevLogTerm和follower的匹配则返回true
	XTerm   int  //若follower和leader的日志冲突,则记载的是follower的log在preLogIndex处的term,若preLogIndex处无日志,返回-1
	XIndex  int  //follower中的log里term为XTerm的第一条log的index
	XLen    int  //当XTerm为-1时,此时XLen记录follower的日志长度(不包含初始占位日志)
}

// AppendEntries 日志追加的RPC handler
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
   rf.mu.Lock()
   defer rf.mu.Unlock()
   defer rf.persist()
   //将自己的term返回
   defer func() {
      reply.Term = rf.currentTerm
   }()
   reply.Success = true
   DPrintf("id[%d].state[%v].term[%d]: 接收到[%d],term[%d]的日志追加,preLogIndex = [%d], preLogTerm = [%d],entries = [%v]\n", rf.me, rf.state, rf.currentTerm, args.LeaderId, args.Term, args.PrevLogIndex, args.PrevLogTerm, args.Entries)
   //DPrintf("id[%d].state[%v].term[%d]: 此时已有的log=[%v]\n", rf.me, rf.state, rf.currentTerm, rf.logEntries)
   //判断term是否小于当前任期
   if args.Term < rf.currentTerm {
      DPrintf("id[%d].state[%v].term[%d]: 追加日志的任期%d小于当前任期%d\n", rf.me, rf.state, rf.currentTerm, args.Term, rf.currentTerm)
      reply.Success = false
      return
   }
   //若请求的term大于该server的term,则更新term并且将voteFor置为未投票
   if args.Term > rf.currentTerm {
      rf.currentTerm = args.Term
      rf.voteFor = -1
   }
   //重置选举时间
   rf.resetElectTimer()
   //转变为follower
   rf.toFollower()
   //进行日志一致性判断(快速恢复)
   //若leader在preLogIndex处没有日志
   if rf.lastLog().Index < args.PrevLogIndex {
      reply.Term = 0
      reply.Success = false
      //preLogIndex处无日志,记录XTerm为-1
      reply.XTerm = -1
      //记录XLen为当前最新日志的index
      reply.XLen = rf.lastLog().Index
      DPrintf("id[%d].state[%v].term[%d]: 追加日志的和现在的日志不匹配\n", rf.me, rf.state, rf.currentTerm)
      return
   }
   if args.PrevLogIndex < rf.logEntries[0].Index {
      reply.XTerm = -1
      reply.Term = 0
      reply.XLen = rf.logEntries[0].Index
      reply.Success = false
      return
   }
   //若preLogIndex处的日志的term和preLogTerm不相等(或者)
   if rf.logEntries[0].Index <= args.PrevLogIndex && rf.index(args.PrevLogIndex).Term != args.PrevLogTerm {
      reply.Success = false
      //更新XTerm为冲突的Term
      reply.XTerm = rf.index(args.PrevLogIndex).Term
      //更新XIndex为XTerm在本机log中第一个Index位置
      reply.XIndex = rf.binaryFindFirstIndexByTerm(reply.XTerm)
      DPrintf("id[%d].state[%v].term[%d]: 追加日志的和现在的日志不匹配\n", rf.me, rf.state, rf.currentTerm)
      return
   }
   //追加
   for i, logEntry := range args.Entries {
      index := args.PrevLogIndex + i + 1
      if index > rf.lastLog().Index {
         rf.logEntries = append(rf.logEntries, logEntry)
      } else if index <= rf.logEntries[0].Index {
         //当追加的日志处于快照部分,那么直接跳过不处理该日志
         continue
      } else {
         if rf.index(index).Term != logEntry.Term {
            rf.logEntries = rf.logEntries[:rf.binaryFindRealIndexInArrayByIndex(index)] // 删除当前以及后续所有log
            rf.logEntries = append(rf.logEntries, logEntry)                             // 把新log加入进来
         }
         // term一样啥也不用做，继续向后比对Log
      }
   }
   if len(args.Entries) > 0 {
      DPrintf("id[%d].state[%v].term[%d]: 追加后的的log=[%v]\n", rf.me, rf.state, rf.currentTerm, rf.logEntries)
   }
   //更新follower的commitIndex
   rf.updateCommitIndexForFollower(args.LeaderCommit)
}
```

**但是这里大家会发现返回值的参数有一些不同，是因为我们使用了快速恢复进行优化**

---

### **快速恢复**

**上述提到的快速恢复，也就是Fast Backup，是用来快速的使跟随者复制到和领袖一样的日志。论文中提到的如果`AppendEntries RPC`因为日志不一致而导致失败，那么就需要将`nextIndex`自减，然后再次发送请求。**

**![image-20220319124309763](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220319124309763.png "论文中的发送失败后的规则")**

**那么如果出现一个情况，领袖这时候的最新日志的索引已经达到了100万，然后有一个服务器因为网络原因，一直没有被正确的接收到领袖的`AppendEnrties RPC`，导致现在该跟随者最新的日志索引为1。这时候网络恢复正常了，就需要同步几万条，但是这时候如果领袖是新当选的，它会初始化`nextIndex`为最新的日志索引+1，也就是100万零1，那么如果我们每次自减一次，就需要进行100万次`AppendEntries RPC`才能正确的发送日志。会严重影响到效率。**

**因此可以采用快速恢复的方法。**

**快速恢复原理**

**可以在`AppendEntries RPC`的回复参数中加上三个参数：**

- **XTerm：这个是跟随者中与领袖冲突的日志对应的任期。如果跟随者在请求的参数中的`prevLogIndex`处的日志任期号和参数中的`prevLogTerm`不匹配，它会拒绝领袖的`AppendEntries`消息，并将自己的任期号放在XTerm中。如果跟随者在对应位置没有日志，那么这里会返回 -1。**
- **XIndex：这个是跟随者中，对应任期号为XTerm的第一条日志条目的槽位号。(也就代表着从该处开始冲突，也就需要从该处开始复制日志)**
- **XLen：如果跟随者在`prevLogIndex`处没有日志，那么XTerm会返回-1，XLen表示最新的日志。**

**那么这时候只需要一次`AppendEntries RPC`就可以让领袖知道我需要给跟随者发送从哪里开始的日志，也就达到了快速将跟随者的日志和领袖保持一致了。**

---

### **发送AppendEntries**

**也就是实现上面提到的`HandleAppendEntries`方法，由于我已经全部完成了，所以下面代码中还包括了快照模块的代码，自行忽略即可。**

```go
// HandleAppendEntries handle对AppendEntries的发送和返回处理(这里返回值表示这次请求目标follower是否仍认为自己是leader)
func (rf *Raft) HandleAppendEntries(server int) (success bool) {
   success = false
   rf.mu.Lock()
   if rf.state != LEADER {
      rf.mu.Unlock()
      return
   }
   //DPrintf("id[%d].state[%v].term[%d]: leader此时的log=[%v]\n", rf.me, rf.state, rf.currentTerm, rf.logEntries)
   DPrintf("id[%d].state[%v].term[%d]: server[%d]的nextIndex=[%d],matchIndex=[%d],lastIncludedIndex=[%d]\n", rf.me, rf.state, rf.currentTerm, server, rf.nextIndex[server], rf.matchIndex[server], rf.logEntries[0].Index)
   //检查此时是否传的日志存在于快照中
   if rf.nextIndex[server] <= rf.logEntries[0].Index {
      args := InstallSnapshotArgs{
         Term:              rf.currentTerm,
         LeaderId:          rf.me,
         LastIncludedIndex: rf.logEntries[0].Index,
         LastIncludedTerm:  rf.logEntries[0].Term,
         Data:              rf.snapshotData,
      }
      reply := InstallSnapshotReply{}
      DPrintf("id[%d].state[%v].term[%d]: 发送installSnapshot to [%d];lastIncludedIndex=[%d],lastIncludedTerm=[%d]\n", rf.me, rf.state, rf.currentTerm, server, rf.logEntries[0].Index, rf.logEntries[0].Term)
      rf.mu.Unlock()
      ok := rf.sendInstallSnapshot(server, &args, &reply)
      rf.mu.Lock()
      defer rf.mu.Unlock()
      //过期的请求直接结束
      if rf.state != LEADER || args.Term != rf.currentTerm {
         return
      }
      if !ok {
         DPrintf("id[%d].state[%v].term[%d]: 发送installSnapshot to [%d] error\n", rf.me, rf.state, rf.currentTerm, server)
         return
      }
      if reply.Term > rf.currentTerm {
         rf.currentTerm = reply.Term
         rf.toFollower()
         rf.voteFor = -1
         rf.persist()
         DPrintf("id[%d].state[%v].term[%d]: 发送installSnapshot to [%d] 过期,转变为follower\n", rf.me, rf.state, rf.currentTerm, server)
         return
      }
      success = true
      //若安装成功,则更新nextIndex和matchIndex
      rf.matchIndex[server] = args.LastIncludedIndex
      rf.nextIndex[server] = rf.matchIndex[server] + 1
      DPrintf("id[%d].state[%v].term[%d]: 发送installSnapshot to [%d] 成功,更新nextIndex->[%d];matchIndex->[%d]\n", rf.me, rf.state, rf.currentTerm, server, rf.nextIndex[server], rf.matchIndex[server])
      return
   }
   //若不存在于快照中,则正常appendEntries
   args := AppendEntriesArgs{
      Term:         rf.currentTerm,
      LeaderId:     rf.me,
      PrevLogIndex: rf.nextIndex[server] - 1,
      PrevLogTerm:  rf.index(rf.nextIndex[server] - 1).Term,
      LeaderCommit: rf.commitIndex,
   }
   //添加需要发送的日志
   args.Entries = make([]LogEntry, len(rf.logEntries)-rf.binaryFindRealIndexInArrayByIndex(rf.nextIndex[server]))
   copy(args.Entries, rf.logEntries[rf.binaryFindRealIndexInArrayByIndex(rf.nextIndex[server]):])
   reply := AppendEntriesReply{}
   DPrintf("id[%d].state[%v].term[%d]: 发送appendEntries to [%d];PrevLogIndex=[%d];Entries=[%v]\n", rf.me, rf.state, rf.currentTerm, server, args.PrevLogIndex, args.Entries)
   rf.mu.Unlock()
   ok := rf.sendAppendEntries(server, &args, &reply)
   rf.mu.Lock()
   defer rf.mu.Unlock()
   //过期的请求直接结束
   if rf.state != LEADER || args.Term != rf.currentTerm {
      return
   }
   if !ok {
      DPrintf("id[%d].state[%v].term[%d]: 发送ae to [%d] error\n", rf.me, rf.state, rf.currentTerm, server)
      return
   }
   //判断是否任期更大,更新自身状态
   if reply.Term > rf.currentTerm {
      //修改term
      rf.currentTerm = reply.Term
      //转变为follower
      rf.toFollower()
      //更新为未投票
      rf.voteFor = -1
      rf.persist()
      DPrintf("id[%d].state[%v].term[%d]: 发送ae to [%d] 过期,转变为follower\n", rf.me, rf.state, rf.currentTerm, server)
      return
   }
   DPrintf("id[%d].state[%v].term[%d]: follower仍认为自己是leader\n", rf.me, rf.state, rf.currentTerm, server)
   success = true
   //若返回失败
   if !reply.Success {
      //更新nextIndex
      //当follower的preLogIndex处无日志时
      if reply.XTerm == -1 {
         //更新nextIndex为follower的最后一条日志的下一个位置
         rf.nextIndex[server] = reply.XLen + 1
      } else {
         //当preLogIndex处的日志任期冲突时
         //更新nextIndex为该冲突任期的第一条日志的位置,为了直接覆盖冲突的任期的所有的日志
         rf.nextIndex[server] = reply.XIndex
      }
      DPrintf("id[%d].state[%v].term[%d]: 追加日志到server[%d]失败,更新nextIndex->[%d],matchIndex->[%d]\n", rf.me, rf.state, rf.currentTerm, server, rf.nextIndex[server], rf.matchIndex[server])
      return
   }
   //若成功
   rf.matchIndex[server] = args.PrevLogIndex + len(args.Entries)
   rf.nextIndex[server] = rf.matchIndex[server] + 1
   //只有发送了不为空的日志(也就不是心跳包的时候)才真正的更新了,心跳包相当于没更新
   if len(args.Entries) > 0 {
      DPrintf("id[%d].state[%v].term[%d]: 追加日志到server[%d]成功,更新nextIndex->[%d],matchIndex->[%d]\n", rf.me, rf.state, rf.currentTerm, server, rf.nextIndex[server], rf.matchIndex[server])
   }
   return
}
```

---

### **更新commitIndex**

**根据论文中提到的规则，我们需要不断更新领袖的`commitIndex`，那么我们就可以写一个方法，周期性的检查从最新的日志一直到当前的`commitIndex+1`处的日志，是否有超过一半的`matchIndex`达到了其中的日志索引处，若有则更新`commitIndex`为那个索引。之所以从最新的开始往前检查，是因为这样可以快速定位到目标索引。**

**代码实现如下：**

```go
// updateCommitIndex 检查更新commitIndex
func (rf *Raft) updateCommitIndex() {
   for !rf.killed() {
      time.Sleep(5 * time.Millisecond)
      rf.mu.Lock()
      if rf.state != LEADER {
         rf.mu.Unlock()
         return
      }
      //从lastLog开始
      for i := rf.lastLog().Index; i > rf.commitIndex; i-- {
         updateConNum := len(rf.peers) / 2
         num := 0
         for j := range rf.peers {
            if j == rf.me {
               continue
            }
            //若match[j] >= i 而且log[i].Term == currentTerm则该server符合更新要求
            if rf.matchIndex[j] >= i && rf.index(i).Term == rf.currentTerm {
               num++
            }
         }
         //若过半数则更新commitIndex
         if num >= updateConNum {
            rf.commitIndex = i
            DPrintf("id[%d].state[%v].term[%d]: n = %d, 过半节点的matchIndex >= n而且log[n].Term == currentTerm,则更新commitIndex = %d\n", rf.me, rf.state, rf.currentTerm, i, i)
            //唤醒ApplyCommand routine
            rf.applyCond.Broadcast()
            break
         }
      }
      rf.mu.Unlock()
   }
}
```

---

### **应用日志到状态机**

**我们在检查到有可以提交的日志的时候，即可以将其传递给状态机应用，那么就可以开启一个协程用来检查并更新`lastApplied`以及将其应用到状态机中。而且当当前没有可以应用的日志的时候，就在cond上等待，所有更新`commitIndex`的代码后面都会唤醒在该`cond`上等待的协程，也就是`ApplyCommand`协程。**

**代码实现如下：**

```go
// ApplyCommand 检查是否 commitIndex > lastApplied,若是则lastApplied递增,并将log[lastApplied]应用到状态机
func (rf *Raft) ApplyCommand() {
   for !rf.killed() {
      rf.mu.Lock()
      //不符合条件时放弃锁进行等待
      for rf.lastApplied >= rf.commitIndex {
         rf.applyCond.Wait()
      }
      //被唤醒而且符合条件
      //当前的commitIndex
      commitIndex := rf.commitIndex
      lastApplied := rf.lastApplied
      DPrintf("id[%d].state[%v].term[%d]: apply command [%d,%d]\n", rf.me, rf.state, rf.currentTerm, lastApplied+1, commitIndex)
      var applyEntries = make([]LogEntry, rf.commitIndex-rf.lastApplied, rf.commitIndex-rf.lastApplied)
      copy(applyEntries, rf.logEntries[rf.binaryFindRealIndexInArrayByIndex(lastApplied+1):rf.binaryFindRealIndexInArrayByIndex(commitIndex+1)])
      rf.mu.Unlock()
      //解锁后进行apply
      for _, entry := range applyEntries {
         rf.applyCh <- ApplyMsg{
            CommandValid: true,
            Command:      entry.Command,
            CommandIndex: entry.Index,
            CommandTerm:  entry.Term,
         }
      }
      rf.mu.Lock()
      //更新lastApplied,由于在apply过程中进行了解锁,因此不能使用现在的commitIndex,而是之前情况的commitIndex
      //(若在解锁过程中,进行了新的log的apply导致lastApplied更新至比该次更新目标的commitIndex还大,那么保持不变,因此这里的更新需要一个Max()来辅助)
      rf.lastApplied = Max(rf.lastApplied, commitIndex)
      rf.mu.Unlock()
   }
}
```

**由于该写成是每一个服务器都需要的，和服务器状态无关，那么就需要在初始化Raft的时候开启(`ticker`也同理)**

```go
func Make(peers []*labrpc.ClientEnd, me int,
   persister *Persister, applyCh chan ApplyMsg) *Raft {
   rf := &Raft{}
   rf.peers = peers
   rf.persister = persister
   rf.me = me
   // Your initialization code here (2A, 2B, 2C).
   rf.currentTerm = 0
   rf.voteFor = -1
   rf.logEntries = make([]LogEntry, 0)
   rf.logEntries = append(rf.logEntries, LogEntry{-1, -1, 0})
   rf.commitIndex = 0
   rf.lastApplied = 0
   rf.state = FOLLOWER
   rf.nextIndex = make([]int, len(peers))
   rf.matchIndex = make([]int, len(peers))
   rf.timeoutHeartBeat = 150
   rf.timeoutElect = 300
   rf.timerHeartBeat = time.NewTimer(time.Duration(rf.timeoutHeartBeat) * time.Millisecond)
   rf.timerElect = time.NewTimer(time.Duration(rf.timeoutElect+rand.Intn(1000)) * time.Millisecond)
   rf.applyCh = applyCh
   rf.applyCond = sync.NewCond(&rf.mu)
   DPrintf("id[%d].state[%v].term[%d]: finish init\n", rf.me, rf.state, rf.currentTerm)
   // initialize from state persisted before a crash
   rf.readPersist(persister.ReadRaftState())
   rf.snapshotData = persister.snapshot
   rf.lastApplied = rf.logEntries[0].Index
   rf.commitIndex = rf.logEntries[0].Index
   // start ticker goroutine to start elections
   go rf.ticker()
   go rf.ApplyCommand()
   return rf
}
```

---

---

## **持久化**

**由于我们现在都是保存在内存中的，那么断电即失，因此我们肯定是需要持久化保存起来的，比如说写入磁盘中。由于lab测试方便，官方提供的是一个类`Persister`来模拟持久化存储的容器，实际上这部分可以换成直接对磁盘的写入或者通过`RockDB`之类的进行持久化。**

### **持久化数据**

```go
func (rf *Raft) persist() {
   // Your code here (2C).
   // Example:
   // w := new(bytes.Buffer)
   // e := labgob.NewEncoder(w)
   // e.Encode(rf.xxx)
   // e.Encode(rf.yyy)
   // data := w.Bytes()
   // rf.persister.SaveRaftState(data)
   w := new(bytes.Buffer)
   e := labgob.NewEncoder(w)
   //编码currentTerm
   err := e.Encode(rf.currentTerm)
   if err != nil {
      DPrintf("id[%d].state[%v].term[%d]: encode currentTerm error: %v\n", rf.me, rf.state, rf.currentTerm, err)
      return
   }
   //编码voteFor
   err = e.Encode(rf.voteFor)
   if err != nil {
      DPrintf("id[%d].state[%v].term[%d]: encode voteFor error: %v\n", rf.me, rf.state, rf.currentTerm, err)
      return
   }
   //编码log[]
   err = e.Encode(rf.logEntries)
   if err != nil {
      DPrintf("id[%d].state[%v].term[%d]: encode logEntries[] error: %v\n", rf.me, rf.state, rf.currentTerm, err)
      return
   }
   data := w.Bytes()
   //保存持久化状态
   rf.persister.SaveRaftState(data)
}
```

**在我们对如上的数据进行更改的时候，都需要进行一次持久化**

---

### **读取持久化数据**

```go
func (rf *Raft) readPersist(data []byte) {
   if data == nil || len(data) < 1 { // bootstrap without any state?
      return
   }
   // Your code here (2C).
   // Example:
   // r := bytes.NewBuffer(data)
   // d := labgob.NewDecoder(r)
   // var xxx
   // var yyy
   // if d.Decode(&xxx) != nil ||
   //    d.Decode(&yyy) != nil {
   //   error...
   // } else {
   //   rf.xxx = xxx
   //   rf.yyy = yyy
   // }
   r := bytes.NewBuffer(data)
   d := labgob.NewDecoder(r)
   var currentTerm int
   var voteFor int
   var logEntries []LogEntry
   if d.Decode(&currentTerm) != nil || d.Decode(&voteFor) != nil || d.Decode(&logEntries) != nil {
      DPrintf("id[%d].state[%v].term[%d]: decode error\n", rf.me, rf.state, rf.currentTerm)
   } else {
      rf.currentTerm = currentTerm
      rf.voteFor = voteFor
      rf.logEntries = logEntries
   }
}
```

**我们初始化的时候，需要从persister中读取持久化数据**

---

---

## **快照**

**我们的日志肯定是不可以持续的增长下去的，因为当我们日志数量达到很大的时候，比如说我们的日志数据已经达到了几千万条的时候，我们和一个还没有多少数据的跟随者进行同步的话，需要将这些日志全部发送，其实是十分浪费资源和时间的。**

**那么我们其实可以使用快照，也就是对领袖某一个时刻它的状态机的数据进行保存，然后将这个快照发送给那些很落后的节点进行快速的同步，同时由于快照已经记录此时的所有必要数据，那么我们可以将这些日志删除，避免日志无限度的增长下去。**

### **论文解析**

**论文中的`Figure 13`是安装快照的RPC的参数和实现。**

**![image-20220319210923396](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220319210923396.png "安装快照RPC")**

> **由领袖调用，用于发送一个快照的分块给跟随者。领袖领袖按照顺序发送分块**

**参数：**

|                       |                                                |
| --------------------- | ---------------------------------------------- |
| **term**              | **领袖的任期**                                 |
| **leaderId**          | **领袖的id，便于跟随者用于重定向客户端的请求** |
| **lastIncludedIndex** | **快照取代的所有的日志中最后一个日志的索引**   |
| **lastIncludedTerm**  | **lastIncludedIndex处的日志的任期**            |
| **offset**            | **该分块在快照文件中的字节偏移量**             |
| **data[]**            | **从offset开始的分块的纯字节数据**             |
| **done**              | **如果是最后一个分块则为true**                 |

**结果：**

|          |                                                 |
| -------- | ----------------------------------------------- |
| **term** | **服务器的currentTerm，用于领袖更新自己的任期** |

**接收者实现：**

1. **如果`term` < `currentTerm`则立马回复。**
2. **如果是第一个分块则创建一个新的快照文件。(`offset`为0)**
3. **在给定的`offset`处开始写入数据。**
4. **如果`done`不为true，那么回复然后等待更多的数据分块传来。**
5. **保存快照文件，丢弃任何比`lastIncludedIndex`小的快照或者部分快照。**
6. **如果存在一个日志和快照最后包含的日志有着一样的索引和任期，那么保留这个日志以及其以后的日志，并回复。**
7. **丢弃所有日志。**
8. **使用快照的内容重置状态机。(以及加载快照的集群配置)**

---

### **实现快照**

**但是我们lab不需要实现这么复杂的快照，因此对其进行了简化。**

**我们一次直接发送一整个快照过去。**

**调用流程是：**

- **状态机发现自己的目前的存储数据过大，那么就保存当前的状态机必须状态以及日志和Raft的必须状态到快照中。然后通知Raft对自己的日志进行丢弃，也就是调用Raft的`Snapshot()`。(日志数组第一位要么为空占位日志，也就是一次快照都没进行的时候日志数组下标为0位置的日志，要么为快照后索引为`lastIncludeIndex`的日志)**
- **当领袖发送`ApppendEntries RPC`的时候，发现需要跟随者的`nextIndex` <= 日志数组中第一个日志的索引的时候，也就是需要发送的日志已经被丢弃了，那么就调用`InstallSnapshot()`来安装快照。**
- **当跟随者接收到领袖发来的快照的时候，若快照是正确的，那么就接收，并通过`applyCh`传递给状态机。**
- **状态机接收到安装快照的请求，进行快照数据的应用，并且通知Raft去更新到该快照。也就是调用Raft的`CondInstallSnapshot()`。**
- **Raft被调用`CondInstallSnapshot()`之后，对响应的日志进行丢弃。**

**![image-20220319213012992](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220319213012992.png "交互流程图")**

---

### **实现InstallSnapshot**

```go
// InstallSnapshotArgs 快照安装RPC的参数
type InstallSnapshotArgs struct {
   Term              int    //leader的任期
   LeaderId          int    //leader的id
   LastIncludedIndex int    //快照中包含的最后一个日志条目的index
   LastIncludedTerm  int    //快照中包含的最后一个日志条目的term
   Data              []byte //快照数据
}

// InstallSnapshotReply 快照安装的返回值
type InstallSnapshotReply struct {
   Term int //接收者的currentTerm
}
```

```go
// InstallSnapshot 快照安装的RPC
func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
   rf.mu.Lock()
   defer rf.mu.Unlock()
   defer func() {
      reply.Term = rf.currentTerm
   }()
   //1.判断参数中的term是否小于currentTerm
   if args.Term < rf.currentTerm {
      //该快照为旧的,直接丢弃并返回
      return
   }
   DPrintf("id[%d].state[%v].term[%d]: 接收到leader[%d]的快照:lastLogIndex[%d],lastLogTerm[%d]\n", rf.me, rf.state, rf.currentTerm, args.LeaderId, args.LastIncludedIndex, args.LastIncludedTerm)
   //2.若参数中term大于currentTerm
   if args.Term > rf.currentTerm {
      rf.currentTerm = args.Term
      rf.voteFor = -1
      rf.persist()
   }
   //3.重置选举时间
   rf.resetElectTimer()
   //4.转变为follower
   rf.toFollower()
   //5.若快照过期
   if args.LastIncludedIndex <= rf.commitIndex {
      DPrintf("id[%d].state[%v].term[%d]: leader[%d]的快照:lastLogIndex=[%d],lastLogTerm=[%d]已过期,commitIndex=[%d]\n", rf.me, rf.state, rf.currentTerm, args.LeaderId, args.LastIncludedIndex, args.LastIncludedTerm, rf.commitIndex)
      return
   }
   //5.通过applyCh传至service
   applyMsg := ApplyMsg{
      SnapshotValid: true,
      Snapshot:      args.Data,
      SnapshotIndex: args.LastIncludedIndex,
      SnapshotTerm:  args.LastIncludedTerm,
   }
   go func(msg ApplyMsg) {
      rf.applyCh <- msg
   }(applyMsg)
}
```

**异步传递msg，避免持有锁的时候阻塞，导致死锁了**

---

### **实现Snapshot**

```go
func (rf *Raft) Snapshot(index int, snapshot []byte) {
   // Your code here (2D).
   rf.mu.Lock()
   defer rf.mu.Unlock()
   if index > rf.lastLog().Index || index < rf.logEntries[0].Index || index > rf.commitIndex {
      return
   }
   //1.获取需要压缩末尾日志的数组内索引
   realIndex := rf.binaryFindRealIndexInArrayByIndex(index)
   lastLogEntry := rf.logEntries[realIndex]
   DPrintf("id[%d].state[%v].term[%d]: 安装snapshot:lastIncludedIndex=[%d],lastIncludedTerm=[%d];commitIndex=[%d]\n", rf.me, rf.state, rf.currentTerm, lastLogEntry.Index, lastLogEntry.Term, rf.commitIndex)
   //2.清除log中[1,realIndex]之间的数据
   rf.logEntries = append(rf.logEntries[:1], rf.logEntries[realIndex+1:]...)
   //3.保存三项快照数据
   rf.snapshotData = snapshot
   //4.更改日志占位节点
   rf.logEntries[0].Index = lastLogEntry.Index
   rf.logEntries[0].Term = lastLogEntry.Term
   //5.持久化
   rf.persistStateAndSnapshot()
   DPrintf("id[%d].state[%v].term[%d]: 安装snapshot:lastIncludedIndex=[%d],lastIncludedTerm=[%d] 成功\n", rf.me, rf.state, rf.currentTerm, lastLogEntry.Index, lastLogEntry.Term)
}
```

---

### **实现CondInstallSnapshot**

```go
func (rf *Raft) CondInstallSnapshot(lastIncludedTerm int, lastIncludedIndex int, snapshot []byte) bool {

   // Your code here (2D).
   rf.mu.Lock()
   defer rf.mu.Unlock()
   DPrintf("id[%d].state[%v].term[%d]: 安装snapshot:lastIncludedIndex=[%d],lastIncludedTerm=[%d];commitIndex=[%d]\n", rf.me, rf.state, rf.currentTerm, lastIncludedIndex, lastIncludedTerm, rf.commitIndex)
   //1.判断快照是否过期
   if lastIncludedIndex <= rf.commitIndex {
      DPrintf("id[%d].state[%v].term[%d]:安装 snapshot:lastIncludedIndex=[%d],lastIncludedTerm=[%d]已过期,安装失败\n", rf.me, rf.state, rf.currentTerm, lastIncludedIndex, lastIncludedTerm)
      return false
   }
   if rf.lastLog().Index < lastIncludedIndex {
      //若快照的最后一个log比当前最新的log还晚,那么清空log中除了0位的log
      rf.logEntries = rf.logEntries[:1]
   } else {
      //清除log中[1,realIndex]之间的数据
      realIndex := rf.binaryFindRealIndexInArrayByIndex(lastIncludedIndex)
      rf.logEntries = append(rf.logEntries[:1], rf.logEntries[realIndex+1:]...)
   }
   //3.保存快照数据
   rf.snapshotData = snapshot
   //4.更改日志占位节点
   rf.logEntries[0].Index = lastIncludedIndex
   rf.logEntries[0].Term = lastIncludedTerm
   //5.更新commitIndex和lastAppliedIndex
   rf.commitIndex = lastIncludedIndex
   rf.lastApplied = lastIncludedIndex
   //6.持久化
   rf.persistStateAndSnapshot()
   DPrintf("id[%d].state[%v].term[%d]: 安装snapshot:lastIncludedIndex=[%d],lastIncludedTerm=[%d] 成功;commitIndex=[%d]\n", rf.me, rf.state, rf.currentTerm, lastIncludedIndex, lastIncludedTerm, rf.commitIndex)
   return true
}
```

---

### **实现快照持久化**

```go
//保存raft状态和snapshot
func (rf *Raft) persistStateAndSnapshot() {
   w := new(bytes.Buffer)
   e := labgob.NewEncoder(w)
   //编码currentTerm
   err := e.Encode(rf.currentTerm)
   if err != nil {
      DPrintf("id[%d].state[%v].term[%d]: encode currentTerm error: %v\n", rf.me, rf.state, rf.currentTerm, err)
      return
   }
   //编码voteFor
   err = e.Encode(rf.voteFor)
   if err != nil {
      DPrintf("id[%d].state[%v].term[%d]: encode voteFor error: %v\n", rf.me, rf.state, rf.currentTerm, err)
      return
   }
   //编码log[]
   err = e.Encode(rf.logEntries)
   if err != nil {
      DPrintf("id[%d].state[%v].term[%d]: encode logEntries[] error: %v\n", rf.me, rf.state, rf.currentTerm, err)
      return
   }
   data := w.Bytes()
   rf.persister.SaveStateAndSnapshot(data, rf.snapshotData)
}
```

---

### **实现发送InstallSnap**

**这一步就是在日志模块中的`HandleAppendEntries|()`中已经实现，只需要发送的时候判断一下是否需要发送快照即可。**

---

## **总结**

**Lab2算是该课程中最难的一个Lab了，个人前前后后做了由半个月多才达到bugfree(自测2000次无fail)。接下来会继续更新Lab3以及Lab4的实现文档。**



# Multi DLedger


# Multi-DLedger

## DLedger的Multi-Raft架构

> 整体结构如下

![image-20220502152826885](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220502152826885.png)

> DLedgerProxy具体架构如下

![image-20220502165732144](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220502165732144.png)

针对上图的几大组件，做如下详细解释。

### DLedgerProxy

我们之前在DLedger中，是直接使用了DLedgerServer作为接收处理请求的载体。若需要升级为Multi-Raft架构，则需要将对于收发请求的部分和处理请求的部分进行解耦。也就是我们可以使用一个类似容器的思想，使用一层代理来代理下层的Raft节点，我们可以多个DLedgerServer公用一个RPCService来进行请求处理，那么这个DLedgerProxy就是将之前实现的handle请求等方法进行包装，并且可以根据配置进行路由。也就是我们ip+port不能唯一标识一个Raft节点了，因为ip+port也可以表现为一个Multi-Raft结构对外服务，那么这时候，我们就需要一个`selfID`来对该Multi-Raft中的不同的Raft进行区分。此时我们的所有DLedger通信都需要带上`selfID`来进行定位。并且我们在DLedgerProxy中加入多个模块，分别是ConfigManager和DLedgerManager。这两个模块分别用来管理Raft-Group的配置和本机DLedgerServer实例的配置以及处理逻辑。

> 为什么使用ip:port+`selfID`来区分？

我们之前是一个ip+port就可以定位一个Raft实例，但是现在我们ip+port定位到的进程是一个Multi-Raft，因此需要根据`selfID`来进行在进程中的定位。

> 为什么要合用一个RPCService？

如果我们给一个DLedger就实例化一个RPCService来进行请求响应的话。

首先对于实例化RPCService是有一定的开销，如果开多个RPCService就需要绑定多个端口，对于程序和系统都是一定的消耗。

其次，我们不仅处理DLedgerServer的请求响应，同样的对于后续若需要进行配置更改等操作，我们可以统一在一个RPCService处接收处理。而且可以将请求响应与实际逻辑处理解耦合，有更强的拓展性。

最后，我们使用同一个RPCService进行网络请求处理，可以做许多优化，比如说对于发往同一个DLedger的请求进行合并，减少网络io消耗；对于同一个DLedger，共享一个心跳计时器，减少网络io。

### ConfigManager

ConfigManager是DLedger的配置管理模块。后续如果使用配置中心等方案，可以动态的对于Raft-Group负责的Region进行在线更改。此时ConfigManager就可以进行相应的请求和响应。因为我们多个DLedgerServer当然是可以公用一个RegionConfig的，因为RegionConfig是对于所有该集群中的Raft都是一样的。

同时可以管理和配置本机的DLedger实例的配置，可以在启动时根据启动参数进行DLedger实例化，也可以实时的根据网络请求来对本机的DLedger实例的配置文件进行更改，从而管理本机的DLedger实例。

### DLedgerManager

DLedgerManager是对DLedgerServer进行管理的模块，功能包括，对于针对DLedgerServer的网络请求，可以根据配置来进行路由到某个DLedgerServer；对于后续的优化，比如上述提到的请求合并，可以在此模块统一处理。

### Load balance

我们需要Multi-Raft架构来进行集群的负载均衡。通过使用Multi-Raft，让一个进程中开启多个DLedgerServer，并且这个DLedgerServer处于不同的RaftGroup，而且各个Group的Leader在集群中是均衡分配的。也就是若为三RaftGroup三物理节点，每个节点开启一个Multi-Raft DLedger，并且分别配置三个Raft节点，那么就是每个物理节点上实际上是有一个RaftGroup的Leader的，那么可以一定程度上让各个物理节点的负载均衡。

> 怎么实现Leader均匀分配？

使用perferLeader来在一开始的配置中分配好Leader即可。DLedger已经实现该功能。

>怎么知道是否均衡分配了？

留一个查询本Multi-Raft进程中所有Raft的信息，根据信息可以知道哪些节点是否成为Leader。

### MultiStorage

我们同样也需要给每个DLedger一个存储空间，并且对不同的DLedger的存储之间进行隔离。这里我们只需要对多个Storage实例进行管理即可。

### 开发任务

1. 完成将DLedgerProxy和DLedgerServer的分离。兼容单DLedgerServer的初始化。
2. 增加Multi-Raft配置，可以根据配置进行多DLedgerServer实例化。
3. 增加ConfigManager模块。
4. 增加DLedgerManager模块。
5. 对MultiStorage进行适配。
6. 对PerferLeader的Multi-Raft结构进行适配。

### 优化任务

1. 动态监听更改进程的DLedger实例配置。
2. ConfigManager的RegionConfig的在线更新。
3. 针对同一个DLedger的请求合并。

---

## 如何以Multi-DLedger启动

若需要以Multi-DLedger启动，需要使用配置文件启动

简易配置包括：

```yaml
DLedgerConfigs:
  - 
   	  group: g0
      id: n0
      peers: n0-127.0.0.1:10000;n1-127.0.0.1:10001;n2-127.0.0.1:10002
      preferredLeaderId: n0
  - DLedgerConfig:
      group: g1
      id: a0
      peers: a0-127.0.0.1:10000;a1-127.0.0.1:10001;a2-127.0.0.1:10002
      preferredLeaderId: a1
  - DLedgerConfig:
      group: g2
      id: b0
      peers: b0-127.0.0.1:10000;b1-127.0.0.1:10001;b2-127.0.0.1:10002
      preferredLeaderId: b2
```

根据上述配置进行多个DLedgerConfig的实例化，然后根据DLedgerConfig来进行DLedgerServer来实例化。我们可以看到我们现在一个端口用来处理多个Raft节点的请求，使用`id`来进行表示。

### 需要完成的要点

1. 对配置启动参数进行更新，添加yaml配置文件启动的方式，先对单Raft节点的进行启动适配。
2. 对多DLedgerServer的配置文件进行管理，使用上述提到的ConfigManager来进行管理。

---

## 接收请求和响应的流程

### 之前的请求响应流程

![image-20220503172615624](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220503172615624.png)

由于只有一个DLedgerServer，因此可以直接调用DLedgerServer的Handle等方法。

### 现在的请求响应流程

![image-20220503172553882](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220503172553882.png)

现在由于我们有多个DLedgerServer实例，那么需要根据请求的id来具体路由到某个DLedgerServer上。因此我们需要让DLedgerProxy中的DLedgerManager来根据`selfID->DLedgerServer`来找到具体的DLedgerServer来处理该请求。

### 需要完成的要点

1. 将原来的DLedgerRpcNettyService接收请求后直接调用DLedgerServer改成调用DLedgerProxy。
2. 现在的DLedgerRpcNettyService的注册是根据一个DLedgerServer来的。但是我们现在需要注册多个进去。并且对于同一个ip+port的remoteClient来进行合并，不需要创建多个，造成资源浪费。

---

## 在线增删DLedger节点

我们可以预留两种请求响应，称为`addDLedgerRequest`和`deleteDLedgerRequest`来让一个Multi-DLedger组来增删DLedgerServer。当然这些新增的不能是新加入某个Raft-Group的。该Raft可以是处于一个新的Raft-Group；也可以是该Raft之前被从该Multi-DLedger中删去了，后来又恢复了。这两种情况是允许的。由于目前没有做RaftGroup成员变更功能，所以只能支持上述两种情况。

### 增加DLedger请求流程

![image-20220503180827652](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220503180827652.png)

我们在接收到增加DLedger的请求后，由DLedgerProxy来做处理，首先它会对该请求进行判断，是否符合要求（比如说是否已有该DLedger）。接下来它会让DLedgerManager来根据配置进行初始化新的DLedger并启动，同时注册进RPC，同时也让ConfigManager来更新管理的DLedgerConfig。

### 删除DLedger请求流程

![image-20220503181347439](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220503181347439.png)

删除DLedger请求，也是由DLedgerProxy来做处理，先对删除的DLedgerId进行判断（是否存在）。接下来让DLedgeManager来删除该DLedgerServer的配置，同时让DLedgerManager来关闭并移除DLedgerServer。也需要对RPC中不再需要的数据进行删除。

### 需要完成的点

1. 对于增删的接口的实现，以及RPC注册。
2. 完成DLedgerProxy的对DLedgerManager和ConfigManager以及DLedgerRpcNettyService的调度。
3. 完成DLedgerManager中增删节点的方法。
4. 完成ConfigManager中增删节点的方法。
5. 完成DLedgerRpcNettyService中增删节点的方法。



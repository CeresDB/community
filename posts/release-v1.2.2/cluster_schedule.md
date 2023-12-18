- GitHub 仓库：https://github.com/CeresDB/ceresdb
- CeresDB 文档：https://docs.ceresdb.io
- Release v1.2.2: https://github.com/CeresDB/ceresdb/releases/tag/v1.2.2

## 背景
在 [Shared Nothing](./../release-v1.2/sharded_nothing.md) 文章中介绍了 CeresDB 的集群拓扑正确性的理论性保障，它从理论上解决了数据可能损坏的可能性，但是并没有就上层具体的实现做过多的说明，我们将在本文和大家分享 CeresDB 分布式调度的设计和实践。

## 问题
作为一个分布式时序数据库，我们在设计时主要面临的问题有：

1. Shard 的调度，作为 CeresDB 集群调度和管理数据的基本单元，一旦一个 Shard 在多个计算节点上打开且写入，就会导致数据损坏。这个问题在上一篇 [Shared Nothing](./../release-v1.2/sharded_nothing.md) 已经得到了解决，本文主要介绍如何使用这种机制来完成具体的集群调度。
2. 分布式调度的的容错性以及异常重试，以 Shard 分裂为例：
```
由于热点问题，CeresMeta 发起一次 Shard 分裂，将 Shard 中的部分热点表分裂到新 Shard 中，并将这个新 Shard 在其它 CeresDB 计算节点上打开。
1. 首先 CeresMeta 创建出新 Shard 的元数据，并修改表和 Shard 的映射关系。
2. CeresMeta 向 CeresDB 计算节点发起请求，关闭掉需要分裂的表。
3. CeresMeta 挑选出 CeresDB 空闲计算节点，发起请求，在节点上打开新的 Shard。
```
这里面任何一步都可能失败和出错，我们需要保证在极端情况下，这次分裂也可以重试并最终成功。
3. Shard 调度的并发问题，我们需要以某种方式来组织和管理 DDL 以及 Shard 调度的顺序，避免因为 DDL 和 Shard 在并发情况下执行出错的情况。

## 设计
带着对以上问题的思考，我们分别调研学习了 HBase 和 TiDB 的分布式调度方案，并根据我们的实践经验，从中借鉴了一些设计。

### 容错性
```
┌───────────────────┐                                         
│                   │                                         
│                   │                ┌─────────────────┐      
│                   │                │                 │      
│                   │  Submit & Run  │Procedure Manager│      
│                   │───────────────▶│                 │      
│                   │                └────────▲────────┘      
│     Procedure     │                         │               
│                   │                         │Load & Retry   
│                   │                ┌────────┴────────┐      
├───────────────────┤   Persistence  │                 │      
│                   │───────────────▶│ Procedure Store │      
│        FSM        │                │                 │      
│                   │                └─────────────────┘      
└───────────────────┘                                         
```

我们参考了 HBase 的 [Procedure V2](https://hbase.apache.org/book.html#pv2) 框架的设计，完成了类似的 Procedure 实现。
这个框架的核心思路是， 构造一个状态会持久化的状态机，并且这个状态机中的每一个步骤都是**幂等且可回滚**的。
对于任意一个 DDL 或者是 Region Operation，HBase 都会将其包装为一个 Procedure，Procedure 内部包含一个状态机，并且执行过程中会当自己当前最新的状态持久化，在执行失败时尝试从状态机当前的最新状态重试，当多次失败确认这个 Procedure 已经无法被执行成功时，则会触发回滚，回收这个过程中创建的资源，避免资源泄露。
以上面问题中描述的 Shard 分裂为例，我们会为每一次 Shard 分裂创建一个包含了以上步骤状态机的 Procedure，并在下列异常情况下正常完成 Shard 分裂。
- 当 CeresMeta 要求 CeresDB 计算节点打开 Shard， 由于某种原因，导致 Shard Recover 失败，状态机的状态停留在这个步骤，并不停地重试，直到 Shard 成功打开或者重试次数达到阈值，终止并回滚此次分裂。
- 当 CeresMeta 的 Leader 节点在 Shard 分裂过程中宕机，由于 Procedure 已经被持久化，当 CeresMeta 选出新的 Leader 节点时，我们也可以从状态机最新的状态开始重试。


### 并发管理
```
┌─────────┐  ┌─────────┐  ┌─────────┐            
│ Shard0  │  │ Shard1  │  │ Shard2  │            
└────▲────┘  └────▲────┘  └────▲────┘            
     │            │            │ Start Procdure  
     │            │            │                 
┌────┴────────────┴────────────┴────┐            
│            Entry Lock             ├──┐         
│                                   │  │         
└────────────────▲──────────────────┘  │         
                 │                     │         
                 │    Try Lock         │Push     
┌────────────────┴──────────────────┐  │         
│                                   │  │         
│                                   │  │         
│           Waiting Queue           │  │         
├───────────┐┌──────────┐┌──────────┤◀─┘         
│           ││          ││          │            
│Procedure0 ││Procedure1││Procedure2│            
│           ││          ││          │            
└───────────┴┴──────────┴┴──────────┘            
```

为了避免并发操作资源导致的问题，我们将 CeresDB 目前的并发模型设计为 **Shard 级别串行，集群级别并行**，并通过以下设计来实现这个模型
- Procedure 调度管理
    - 将所有的 DDL 和 Shard Operation 封装为 Procedure ，它们要提交给 ProcedureManager 来管理并协调运行。
    - ProcedureManager 中为每个 Shard 维护一把 Entry Lock，Procedure 在执行前必须持有对应 Shard 的 EntryLock，保证在同一时刻每个 Shard 上只能有一个 Procedure 运行。
    - ProcedureManager 中维护了一个具有优先级的延迟队列，所有的 Procedure 会被提交到这个延迟队列中，并在 Shard 空闲时被取出执行，如果在尝试执行时失败，那么它会被丢回这个队列并在指定的延迟之后尝试再次执行。
- Version 正确性校验
    - 我们定义了 Version 相关的概念：
    1. ShardVersion，它代表了 Shard 与它对应的表的映射关系，当这个映射关系发生变化时，ShardVersion 就会增长。
    2. ClusterVersion，它代表了节点与 Shard 的映射关系，类似的，当这个映射关系发生变化时，ClusterVersion 就会增长。
- 为 CeresDB 与 CeresMeta 的每个请求附带上这两个 Version，只有 Version 的校验通过，一次操作才能正常的执行成功，这样的校验能保证我们的每次操作都是基于最新的拓扑，避免我们在过期的数据上做出错误的调度。
  举一个具体的例子：
```
1. 在 t0 时刻，由于负载均衡策略，CeresMeta 基于当前的集群拓扑，决定将负载高的 node0 上的 shard0 在负载低的 node1 上打开，由于 Procedure 的排队机制，此时这个 Procedure 并未执行。
2. 在 t1 时刻，用户此前提交的删表请求执行成功， shard0 上的大表 table0 被删除，此时 node0 负载降低。
3. 在 t2 时刻，t0 时刻生成的 Procedure 开始执行，此时集群的拓扑已经和 t0 时刻不同，node0 的负载比 node1 更低，由于 Version 的校验机制，我们发现了拓扑的变化，并直接取消此次调度。
```

### 集群拓扑管理
我们参考了 PD 中的设计，并在 ShardLock 的基础上实现了集群拓扑的反向同步。关于这个设计的讨论已经在 [CeresMeta 状态同步](https://docs.ceresdb.io/cn/design/shared_nothing.html#ceresmeta-%E7%8A%B6%E6%80%81%E5%90%8C%E6%AD%A5)  中详细描述，本文中不再重复。


### 集群调度策略
```
┌────────────────┐                                 
│                │                                 
│TopologyManager │                                 
│                │                                 
└────────┬───────┘                                 
         │                                         
         │ClusterTopology                          
         │                                         
┌────────▼───────┐                                 
│                │        ┌───────────────────────┐
│                ├───────▶│ AssignShardScheduler  │
│   Scheduler    │        └───────────────────────┘
│    Manager     │ GenerateProdure                 
│                │        ┌───────────────────────┐
│                ├───────▶│RebalanceShardScheduler│
│                │        └───────────────────────┘
└───────┬────────┘                                 
        │                                          
        │Submit                                    
 ┌──────▼──────┐                                   
 │             │                                   
 │             │                                   
 │  Procedure  │                                   
 │   Manager   │                                   
 │             │                                   
 │             │                                   
 └─────────────┘                                                            
```

我们通过一个 SchedulerManager 来管理所有的 Scheduler ，它会以触发式以及定时调度的形式将集群的最新拓扑下发到所有 Scheduler 中，各个 Scheduler 接收到拓扑信息后，根据自身的调度策略，生成对应的 Procedure ，并提交给 ProcedureManager 来执行。
Procedure 本身的并发安全以及容错等问题由上文中描述的机制来保证，Scheduler 只是一个很轻量的策略生成器，它接收当前集群的拓扑，并根据自己的调度策略来生成一个 Procedure，通过这个 Procedure，来将集群转向我们期望的状态。
我们通过底层的容错和并发管理、Shard Lock 等机制来保证了调度的安全性，因此 Scheduler 本身不再需要关注生成的调度策略能否被**安全正确**的执行完毕，它只需要关注调度本身的逻辑就足够了，这样就让我们上层的调度变的**简单且易于扩展**，我们可以在上层完成各种各样具备不同策略的调度器。
以集群的节点 failover 为例，来描述一下这个流程：
```
1. CeresDB 的 Node0 宕机，在 Node0 上分配的所有 Shard 在 lease 到期后释放 Shard Lock，CeresMeta 通过 ETCD Watch 发现 Shard Lock 释放。
2. 根据上一步获取到的信息，CeresMeta 清理掉内存中宕机节点与 Shard 的映射关系，产生新的集群拓扑，此时原来在 Node0 上的所有 Shard 为未分配状态。
3. SchedulerManager 将最新的集群拓扑下发到所有 Scheduler 中，负责 failover 的 AssignShardScheduler 发现此时集群中存在未分配 Shard，尝试将其分配到当前集群可用的节点上，生成对应的 Procedure，并将其提交到 ProcedureManager。
4. ProcedureManager 在合适的时间执行上一步中提交的 Procedure，错误的调度会被我们底层的机制拦截，因此我们可以放心大胆的生成期望的调度目标，最终会正确的完成 failover。
```


## 总结
以上就是当前 CeresDB 的分布式方案的一些关键性设计，我们尝试在底层通过一些机制来解决分布式系统中的一些难题，从而让上层的调度变得简单且可靠。目前我们的分布式设计中还存在很多问题，在下个阶段，我们着重想解决的是：
1. 基于机器的实际负载实现更复杂的调度机制，目前我们的 Shard 分配是基于一致性哈希完成的，而 Table 的分配是均匀随机的，我们期望在后续基于机器的实际负载（CPU、内存的利用率）来完成更精细化的调度，做到真正的负载均衡。
2. 细化 Procedure 的并发管理机制，目前我们将单个 Shard 上的所有调度和 DDL 全部设为串行，这会影响 DDL 的执行效率，我们期望在后续能够设计更好的，完成并行的 DDL。
3. Procedure 的设计目前无法很好的解决分布式 DDL 的问题，例如分区表删除时外部用户可能会看到部分子表存在的情况，不过在我们当前的使用场景下，这种问题是可以容忍的。
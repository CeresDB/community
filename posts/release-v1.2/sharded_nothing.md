> CeresDB 在最近发布了 v1.2.0，这个是一个比较大的版本，在这个版本中，我们对于 CeresDB 的可观测性、写入性能、InfluxQL 支持、Proxy 模块都做了相当大的增强，此外，最重要的是重构了集群的 failover 和动态调度的实现机制，让后续的集群特性开发变得更加简洁、优雅，本文将就这一点进行一些经验的分享，希望对于读者有所启发。

- GitHub 仓库：https://github.com/CeresDB/ceresdb
- CeresDB 文档：https://docs.ceresdb.io
- Release v1.2.0: https://github.com/CeresDB/ceresdb/releases/tag/v1.2.0

# Shared Nothing 架构

## 背景

在 [集群](./clustering.md) 文章中介绍了 CeresDB 的集群方案，简单总结一下就是：

- 计算存储分离；
- 由中心化的元数据中心，管理整个集群；

然而计算存储分离的架构下，有个重要的问题在于：在集群调度的过程中，如何保证在共享的存储层中的数据不会因为不同的计算节点访问导致数据损坏。一个简单的例子就是如果同一块数据块被多个计算节点同时更新，可能就会出现数据损坏。

而 CeresDB 的解决方案是通过特定的机制，在共享存储的情况下达到了类似 [Shared-Nothing 架构](https://en.wikipedia.org/wiki/Shared-nothing_architecture) 的效果，也就是说存储层的数据经过一定规则的划分，**可以保证在任何时刻最多只有一个 CeresDB 实例可以对其进行更新**，本文中，将这个特性定义成**集群拓扑的正确性**，如果这个正确性得到保证的话，那么数据就不会因为集群的灵活调度而受到损坏。

本文对于 Shared Nothing 架构的优劣不做赘述，主要分享一下，CeresDB 集群方案是如何在计算存储分离的方案下，达到 Shared Nothing 的效果（即如何保证 **集群拓扑的正确性**）。

## 数据划分

为了达到 Shared Nothing 的效果，首先需要将数据在共享的存储层上面进行好逻辑和物理的划分。在 [此前的集群介绍文章](./clustering.md#shard) 中介绍了 Shard 的基本作用，作为集群的基本调度单元，实际上也是数据分布的划分单元，也就是说不同的 Shard 在存储层的数据是隔离的：

- 在 WAL 中，写入的 Table 数据会按照 Shard 组织起来，按照 Shard 写入到 WAL 的不同区域中，不同的 Shard 在 WAL 中的数据是隔离开的；
- 在 Object Storage 中，数据的管理是按照 Table 来划分的，而 Shard 和 Table 之间的关系是一对多的关系，也就说，任何一个 Table 只属于一个 Shard，因此在 Object Storage 中，Shard 之间的数据也是隔离的；

## Shard Lock

在数据划分好之后，需要保证的就是在任何时刻，同一时刻最多只有一个 CeresDB 实例能够更新 Shard 的数据。那么要如何保证这一点的呢？很自然地，通过锁可以达到互斥的效果，不过在分布式集群中，我们需要的是分布式锁。通过分布式锁，每一个 Shard 被分配给 CeresDB 实例时，CeresDB 必须先获取到相应的 Shard Lock，才能完成 Shard 的打开操作，对应地，当 Shard 关闭后，CeresDB 实例也需要主动释放 Shard Lock。

CeresDB 集群的元数据服务 CeresMeta 是基于 ETCD 构建的，而基于 ETCD 实现分布式的 Shard Lock 是非常方便的，因此我们选择基于现有的 ETCD 实现 Shard Lock，具体逻辑如下：

- 以 Shard ID 作为 ETCD 的 Key，获取到 Shard Lock 等价于创建出这个 Key；
- 对应的 Value，可以把 CeresDB 的地址编码进去（用于提供给 CeresMeta 调度）；
- Shard Lock 获取到了之后，CeresDB 实例需要通过 ETCD 提供的接口对其进行续租，保证 Shard Lock 不会被释放；

CeresMeta 暴露了 ETCD 的服务提供给 CeresDB 集群来构建 Shard Lock，下图展示了 Shard Lock 的工作流程，图中的两个 CeresDB 实例都尝试打开 Shard 1，但是由于 Shard Lock 的存在，最终只有一个 CeresDB 实例可以完成 Shard 1 的打开：

```
             ┌────────────────────┐
             │                    │
             │                    │
             ├───────┐            │
   ┌─────┬──▶│ ETCD  │            │
   │     │   └───────┴────CeresMeta
   │     │       ▲
   │     │       └──────┬─────┐
   │     │          Rejected  │
   │     │              │     │
┌─────┬─────┐        ┌─────┬─────┐
│Shard│Shard│        │Shard│Shard│
│  0  │  1  │        │  1  │  2  │
├─────┴─────┤        ├─────┴─────┤
└─────CeresDB        └─────CeresDB
```

## 其他方案

Shard Lock 的方案本质上是 CeresDB 通过 ETCD 来保证在集群中**对于任何一个 Shard 在任何时刻最多只有一个 CeresDB 实例可以对其进行更新操作**，也就是保证了在任何时刻 **集群拓扑的正确性**，需要注意，这个保证实际上成为了 CeresDB 实例提供的能力（虽然是利用 ETCD 来实现的），而 CeresMeta 无需保证这一点，下面的对比中，这一点是一个非常大的优势。

除此 Shard Lock 的方案，我们还考虑过这样的两种方案：

### CeresMeta 状态同步

CeresMeta 规划并存储集群的拓扑状态，保证其**正确性**，并将这个正确的拓扑状态同步到 CeresDB，而 CeresDB 本身无权决定 Shard 是否可以打开，只有在得到 CeresMeta 的通知后，才能打开指定的 Shard。此外，CeresDB 需要不停地向 CeresMeta 发送心跳，一方面汇报自身的负载信息，另一方面让 CeresMeta 知道该节点仍然在线，用以计算最新的正确的拓扑状态。

该方案也是 CeresDB 一开始采用的方案，该方案的思路简洁，但是在实现过程中却是很难做好的，其难点在于，CeresMeta 在执行调度的时候，需要基于最新的拓扑状态，决策出一个新的变更，并且应用到 CeresDB 集群，但是这个变更到达某个具体的 CeresDB 实例时，即将产生效果的时候，该方案无法简单地保证此刻的集群状态仍然是和做出该变更决策时基于的那个集群状态是一致的。

让我们用更精确的语言描述一下：

```
t0: 集群状态是 S0，CeresMeta 据此计算出变更 U；
t1: CeresMeta 将 U 发送到某个 CeresDB 实例，让其执行变更；
t2: 集群状态变成 S1；
t3: CeresDB 接收到 U，准备进行变更；
```

上述的例子的问题在于，t3 时刻，CeresDB 执行变更 U 是否正确呢？执行这个变更 U，是否会让数据遭到损坏？这个正确性，需要 CeresMeta 完成相当复杂的逻辑来保证即使在集群状态为 S1 的情况下，执行 U 变更也不会出现问题，除此之外，状态的回滚也是一个非常麻烦的过程。

举一个例子，就可以发现这个方案的处理比较麻烦：

```
t0: CeresMeta 尝试在 CeresDB0 打开 Shard0，然而 CeresDB0 打开 Shard0
    遇到一些问题，Hang 住了，CeresMeta 只能在超时之后，认为打开失败；
t1: CeresMeta 计算出新的拓扑结构，尝试在 CeresDB1 继续打开 Shard0；
t2: CeresDB0 和 CeresDB1 可能会同时打开 Shard0；
```

自然是有一些办法来避免掉 t2 时刻的事情，比如在 t0 时刻失败了之后，需要等待一个心跳周期，来获知 CeresDB0 是否仍然在尝试打开 Shard0，来避免 t1 时刻发出的命令，但是这样的逻辑比较繁琐，难以维护。

对比 Shard Lock 的方案，可以发现，该方案尝试获得更强的一致性，即尝试保证集群的拓扑状态在任何时刻都需要和 CeresMeta 中的集群状态保持一致，显然，这样的一致性肯定是能够保证**集群拓扑的正确性**的，但也正因为如此实现起来才会更加复杂，而基于 Shard Lock 的方案放弃了这样一个**已知**的、正确的集群拓扑状态，转而只需要保证集群状态是正确的即可，而不需要知道这个状态究竟是什么。更重要的是，从另外一个角度来看，保证**集群拓扑的正确性**这部分逻辑和集群的调度完成了解耦，因此 CeresMeta 的逻辑大大简化，只需要专注于完成负载均衡的集群调度工作即可，而集群拓扑的正确性由 CeresDB 本身来保证。

## CeresDB 提供一致性协议

该方案是参考 TiDB 的元数据服务 [PD](https://github.com/tikv/pd) 的，PD 管理着所有的 TiKV 数据节点，但是 PD 不需要维护一致性的集群状态，并且应用到 TiKV 节点上面去，因为 TiKV 集群中，每一个 Raft Group 都能够达到一致性，也就是说 TiKV 无需借助 PD，本身就具备了让整个集群拓扑正确的能力（一个 Raft Group 不会出现两个 Leader）。

参考该方案，实际上我们也可以在 CeresDB 实例之间实现一致性协议，让其本身也具备这样的能力，不过在 CeresDB 之间引入一致性协议，似乎把事情变得更加复杂了，而且目前也没有更多的数据需要同步，通过外部的服务（ETCD）依然可以同样的效果，从 CeresMeta 看，就等价于 CeresDB 本身获得了让集群一致的能力。

因此 Shard Lock 的方案，可以看作是该方案的一个变种，是一种取巧但是很实用的实现。

## 总结

CeresDB 分布式方案的最终目标自然不是保证集群拓扑正确就够了，但是保持正确性是后续特性的重要基石，一旦这部分的逻辑简洁明了，有充分的理论保证，就可以让后续的特性实现也同样的优雅、简洁。例如，为了使得 CeresDB 集群中的各个节点达到负载均衡的效果，CeresMeta 就必须根据 CeresDB 实例上报的消息，对集群中的节点进行调度，而调度的单位必然是 Shard，然而任何一次 Shard 的变动都可能造成数据的损坏（一个 Shard 被两个实例同时打开），在有了 Shard Lock 的保证后，CeresMeta 就可以放心地生成调度计划，根据上报的负载信息，计算出当前状态的最佳调度结果，然后发送到涉及的 CeresDB 实例让其执行，即使计算的前提可能是错误的（即集群状态已经和计算出调度结果时的状态不一样了），也不用担心集群拓扑的正确性遭到破坏，因而 CeresMeta 的调度逻辑，变得简洁而优雅（只需要生成、执行，不需要考虑失败处理）。
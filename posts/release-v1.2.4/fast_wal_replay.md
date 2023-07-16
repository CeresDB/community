- GitHub 仓库：https://github.com/CeresDB/ceresdb
- CeresDB 文档：https://docs.ceresdb.io
- Release v1.2.4: https://github.com/CeresDB/ceresdb/releases/tag/v1.2.4

# WAL 日志快速回放

## 背景

在上一篇 [CeresDB 分布式调度](https://github.com/CeresDB/community/blob/pr-1.2.4/posts/release-v1.2.2/cluster_schedule.md) 中，提到了 CeresDB 集群的 failover 过程，其可以简单概括为：

- CeresMeta 感知到节点宕机，然后将宕机节点的 shard 调度到仍然存活的其他节点上。
- 被选取的 CeresDB 节点接收到请求，然后打开相应 shard。

关于 shard 的定义以及作用，可以参考 [集群](https://github.com/CeresDB/docs/blob/main/docs/src/cn/design/clustering.md)，可以简单理解为表的集合。
在实践中发现，failover 性能的瓶颈在于 CeresDB 节点打开 shard 这一步骤，而其中主要的流程为 wal 日志的 回放，因此问题转换为了如何提高 wal 日志回放性能，下文将分享一下 CeresDB 在这方面踩过的坑和得出的经验。

## WAL 日志存储方式

WAL 日志回放分为读取日志 (IO 操作)，写 memtable (内存操作) 两部分，明显瓶颈为前者，而日志读取方式的性能高低，和数据存储方式是高度相关的，因此这里先介绍一下 CeresDB 中 WAL 日志的存储方式。

在 [Shared Nothing 架构](https://github.com/CeresDB/community/blob/pr-1.2.4/posts/release-v1.2/sharded_nothing.md) 中提到，CeresDB 的 WAL 日志是按照 shard 进行存储的，意思就是 shard 内的所有表共享同一个 WAL 日志区域（可以简单理解为共享一个 wal 日志“文件”）。类似于 HBase 单个 region server 内多个 region 共享一个 HLog，以避免 region 过多时产生大量对磁盘的随机写入从而影响写入性能，CeresDB 将 WAL 按照 shard 进行组织，主要也是出于对写入性能的考虑。

但类似于 HBase 在 failover 时，是按 region 进行 WAL 日志回放的，CeresDB 在 release 1.2.4 之前也是只支持按表串行进行回放。个人认为，这种设计从直觉上就已经比较别扭，数据存储的粒度和读取的粒度是不一致的。事实上因为这种不一致，也导致了 HBase 在回放前额外的 [Log Splitting](https://dirtysalt.github.io/html/hbase-log-splitting.html) 步骤，而针对这个额外步骤的性能问题，HBase 方面也付出了不少精力进行优化。

CeresDB 也在这方面踩了坑，下面将从本地 WAL 和分布式 WAL 两方面分别介绍，按表回放方案所遇到的问题。

## 本地 WAL 遇到的问题

当前 CeresDB 的本地 WAL 是基于 RocksDB 进行实现的，单机上所有 WAL 日志写入到同一个 RocksDB 实例中（只使用了 default column family）。在按表级别回放时，采用 iterator 的方式读取单表日志，而考虑到 WAL 日志顺序写入的特性，以及 RocksDB iterator 的读取原理（此处不展开，具体可以参考官方 wiki），回放时读取文件的情况可以简单梳理成下图：

<img src="./rockswal.drawio.svg?sanitize=true">

这种读取方式，存在两个明显的问题：

- 需要进行 n * m 次磁盘寻址，当 n 和 m 的值较大时 (表多时会导致 n 变大，RocksDB compaction 不够及时会导致 m 变大），这方面的性能损耗将会变得明显。
- 可能不能充分利用磁盘带宽，当表分布在单个文件的日志量不大的情况下，每次寻址后能读取的数据量可能远小于磁盘带宽，造成很大浪费。举个简单例子，磁盘带宽为 3，有 3 个表每个表数据量为 1，当一次读取 3 个表的数据，那么只需要读 1 次磁盘；如果逐个表读取，那么就不能充分利用带宽，需要读 3 次磁盘。

事实上在表特别多的业务场景下，确实也遇到了上述原因造成的严重性能问题，在某次宕机恢复的过程中发现，对 100G WAL 日志进行回放足足花费了 1 小时以上，对服务的可用性造成了巨大影响。

## 分布式 WAL 遇到的问题

当前 CeresDB 集群主要采用基于 Kafka 实现的分布式 WAL，如上文所说 WAL 区域是按 shard 进行划分的，而一个 WAL 区域则被映射为一个 Kafka topic（只使用了单个 partition）。Kafka partition 内的记录并不会按 Key 进行排序，并且 Kafka 作为消息队列只支持对于 partition 内数据的顺序读取，在这种情况下，按表恢复方案的实现方式基本只有如下两种：

- 类似 HBase，先拉取整个 WAL 区域的日志，然后再拆分到表级别存储到本地磁盘中，之后按表进行日志回放。
- WAL 区域内的每个表，都拉去整个区域对应 partition 内的日志，然后回放属于自己的部分日志。

明显可以看出，由于读取方式对存储方式本身就不够友好，上述实现都会造成大量额外的 IO 操作，性能都不会好。最后 CeresDB 选取了第二种方式进行实现，从而导致每个表在回放时都需要拉取整个 WAL 区域所有日志，可想而知这造成了严重的回放性能问题。

<img src="./kafkawal.drawio.svg?sanitize=true">

## WAL日志回放优化

### 按 shard 进行日志回放

总结下来，导致以上性能问题的本质原因在于：按表的回放方式，对按 shard 的存储方式很不友好。所以与其针对具体的 WAL 实现打补丁，不如解决根本问题，于是 CeresDB 在 release 1.2.4 中实现了按 shard 进行的日志回放。
在按 shard 回放的情况下，RocksDB 版 WAL 只需要顺序读取所有相关的文件（单机场景下所有表都会被划分到默认 shard 中），而在 Kafka 版 WAL 中，则只需要每个 shard 统一拉取一次 WAL 区域对应 partition 中的日志，具体如下图所示。
RocksDB 版 WAL：

<img src="./rocksshard.drawio.svg?sanitize=true">

`Kafka 版 WAL`：

<img src="./kafkashard.drawio.svg?sanitize=true">

很明显，对两种 WAL 实现来说，采用按 shard 的日志回放方式，都能减少大量多余的 IO 操作，从而提高回放性能。
为了验证效果，基于 RocksDB 版 WAL ，对按表与按 shard 两种 WAL 日志回放方式的耗时进行了简单测试：

- 测试步骤：在测试环境中，创建了 200 张测试表，并引入测试流量生成了 2.8G WAL 日志，分别按表和按 shard 进行回放，并记录耗时。
- 测试结果：按表回放耗时 1256.96s，按 shard 回放耗时 51.36s，按 shard 回放性能提高到按表回放的 24.48 倍。

而可以预见，对 Kafka 版 WAL 也会有较大的性能提升（对于单个 shard 的回放，直接减少掉表数量次的多余读取）。

### 减少 WAL 日志大小

显而易见，WAL 日志大小越小，回放时需要读取的内容就越少，回放的耗时自然就越短。在讨论如何减小 WAL 日志大小前，我们需要先搞清楚一个问题：什么情况下 WAL 日志会很大？
WAL 主要的作用是，负责存储在 memtable 中数据（未被 flush 到 sst 中的数据）的持久化，而 CeresDB 中每个表会拥有独立的 memtable，因此可以得出使 WAL 日志变得很大的原因：

- 表的数量较多。
- 表的数据长期停留在 memtable 中，未能触发 flush。

这种情况在某些业务场景下是可能出现的，例如表多 + 单表写入量不大（flush 一般在 memtable 超过大小限制时被触发）的场景。
针对这个问题，CeresDB 会定期检查每个表的 flush 情况，如发现该表数据在 memtable 中停留过久（一般阈值为 1 小时），会强行触发该表的 flush。当然单靠这种机制来控制 WAL 日志大小，可能仍然过于简单，在未来可以考虑加入 WAL 日志大小的统计信息，以及根据统计信息来触发表的 flush 的机制。

## 总结

以上就是 CeresDB 在 WAL 日志回放设计上踩过的坑，和针对这些坑所进行的优化，其实在这过程中我们也总结出了更广泛通用的设计原则：在设计数据的读写方式时，必须着重考虑数据的存储方式，尽量做到读写方式对存储方式友好。
回到具体问题上，在 WAL 日志回放方面，仍存在以下的一些已知的优化点：

- 读取日志阶段的性能仍不够好，可能需要加入对日志数据的压缩，以减少 IO 传输的数据量，提高性能。
- 如上述所说，flush 策略仍过于简单，可能需要加入由 WAL 日志大小触发的策略，以更好地控制日志大小。

在之后的开发中，CeresDB 会继续进行这方面的优化工作，降低集群 failover 耗时，以求提供更高的集群可用性。

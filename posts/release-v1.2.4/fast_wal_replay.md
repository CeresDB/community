# WAL 日志快速回放

## 背景

在上一篇 [CeresDB 分布式调度](https://github.com/CeresDB/community/blob/pr-1.2.4/posts/release-v1.2.2/cluster_schedule.md) 中，提到了 CeresDB 集群的 failover 过程，其可以简单概括为：

- CeresMeta 感知到节点宕机，然后将宕机节点上的 shard 调度到仍然存活的其他节点上。
- 被选取的 CeresDB 节点接收到打开 shard 请求，然后打开相应 shard。

关于 shard 的定义以及作用，参考集群，可以简单理解为表的集合。
在实践中发现，failover 性能的瓶颈在于 CeresDB 节点打开 shard 这一步骤，而其中主要的流程为 WAL 日志的 回放，因此问题转换为了如何提高 WAL 日志回放性能，下文将分享一下 CeresDB 在这方面踩过的坑和得出的经验。

## WAL 日志存储方式

WAL 日志回放分为读取日志 (IO 操作)，写 memtable (内存操作) 两部分，明显瓶颈为前者，而日志读取方式的性能高低，和数据存储方式是高度相关的，因此这里先介绍一下 CeresDB 中 WAL 日志的存储方式。

在 [Shared Nothing 架构](https://github.com/CeresDB/community/blob/pr-1.2.4/posts/release-v1.2/sharded_nothing.md) 中提到，CeresDB 的 WAL 日志是按 shard 进行存储的，意思就是 shard 内的所有表共享同一个 WAL 日志区域（可以简单理解为共享一个 WAL 日志“文件”）。类似于 HBase 单个 region server 内多个 region 共享一个 HLog，以避免 region 过多时产生大量对磁盘的随机写入从而影响写入性能，CeresDB 将 WAL 按照 shard 进行组织，主要也是出于类似的考虑。
而 CeresDB 在 release 1.2.4 之前的 WAL 日志回放方式，也是主要参考了 HBase 的按 region 回放，采用了按表回放，其大致过程为：

```rust
// 串行对每个表进行回放。
for table in tables {
    let table_wal_logs_iter = read_wal_logs_of_table(table);
    // 按 batch 大小读取表的 WAL 日志。
    for table_wal_log_batch in table_wal_logs_iter {
        // 将读取到的 WAL 日志写入到 memtable。
        apply_log_batch_to_memtable(table_wal_log_batch);
    }
}
```

按 shard（表的集合） 存储，但按表回放？细心的读者可能已经发现，这似乎不太一致。而 CeresDB 确实也在这种不一致上踩了坑（个人认为，方案的参考来源 HBase 也是踩了坑的，详见 HLog splitting 以及其优化历史），下面将从本地 WAL 和分布式 WAL 两方面，分别介绍具体遇到的问题。

## 本地 WAL 遇到的问题

当前 CeresDB 的本地 WAL 是基于 RocksDB 进行实现的，单机上所有表的 WAL 日志写入到同一个 RocksDB 实例中（只使用了 default column family）。在按表别回放时，考虑到 WAL 日志顺序写入的特性，以及 RocksDB iterator 的读取原理（此处不展开，具体可以参考官方 wiki），读取文件的情况可以简单梳理成下图：

<img src="https://raw.githubusercontent.com/Rachelint/drawio-store/main/rockswal.drawio.svg?sanitize=true">

这种读取方式，存在两个明显的问题：

- 需要进行 n * m 次磁盘寻址，当 n 和 m 的值较大时 (表多时会导致 n 变大，RocksDB compaction 不够及时会导致 m 变大），这方面的性能损耗将会变得明显。
- 可能不能充分利用磁盘带宽，当表分布在单个文件的日志量不大的情况下，每次寻址后能读取的数据量可能远小于磁盘带宽，造成很大浪费。举个简单例子，磁盘带宽为 3，有 3 个表每个表数据量为 1，当一次读取 3 个表的数据，那么只需要读 1 次磁盘；如果逐个表读取，那么就不能充分利用带宽，需要读 3 次磁盘。

事实上在表特别多的业务场景下，确实也遇到了上述原因造成的严重性能问题，在某次宕机恢复的过程中发现，对 100G WAL 日志进行回放足足花费了 1 小时以上，对服务的可用性造成了巨大影响。

## 分布式 WAL 遇到的问题

当前 CeresDB 集群主要采用基于 Kafka 实现的分布式 WAL，如上文所说 WAL 是按 shard 存储的（称为 WAL 区域），而一个 WAL 区域则被映射为一个 Kafka topic（只使用了单个 partition）。Kafka partition 采用 append 的方式写入记录，并且 Kafka 作为消息队列只支持对 partition 内数据的顺序读取，因此在按表回放的方式下，每个表在读取阶段，只能先读取整个 partition 内的日志然后提取自己的部分(做了一些优化，每个表的读取起点 offset 为尚未被 flush 日志的最小 offset)，如下图所示：

<img src="https://github.com/Rachelint/drawio-store/blob/main/kafkawal.drawio.svg?sanitize=true">

明显可以看出，这中读取方式会产生大量额外的 IO 操作，而这也确实造成了严重的性能问题。总的来看，无论本地还是分布式 WAL 回放，性能问题的根因，其实都在于**日志读取方式对存储方式本身不够友好**。

## WAL日志回放优化

### 按 shard 进行日志回放

正如上文所说，导致以上性能问题的本质原因在于：按表的回放方式，对按 shard 的存储方式很不友好（不友好的主要是作为瓶颈的日志读取部分）。那么与其针对具体的 WAL 实现打补丁，不如先解决根本问题，于是 CeresDB 在 release 1.2.4 中实现了按 shard 进行的日志回放，其大致流程为：

```rust
let shard_wal_logs_iter = read_wal_logs_of_shard(shard);
// 按 batch 大小统一读取 shard 的 WAL 日志。
for shard_wal_log_batch in shard_wal_logs_iter {
    // 将 table 日志从 shard 日志中提取出来，
    // 写入到具体 table 的 memtable 中。
    for table_wal_log_batch in shard_wal_log_batch {
        apply_log_batch_to_memtable(table_wal_log_batch);
    }
}
```

在改用按 shard 的日志回放方式后，在日志读取阶段，RocksDB 版 WAL 只需要顺序读取所有相关的文件（单机场景下所有表都会被划分到默认 shard 中），而在 Kafka 版 WAL 中，则只需要每个 shard 统一拉取一次 WAL 区域对应 partition 内的日志，具体如下图所示：

**RocksDB 版 WAL：**

<img src="https://github.com/Rachelint/drawio-store/blob/main/rocksshard.drawio.svg?sanitize=true">

**Kafka 版 WAL：**

<img src="https://github.com/Rachelint/drawio-store/blob/main/kafkashard.drawio.svg?sanitize=true">

很明显，对两种 WAL 实现来说，采用按 shard 的日志回放方式，都能减少掉量额外的 IO 操作，从而提高回放性能。

### 优化效果测试

为了验证效果，别采用 RocksDB 版 WAL 和 Kafka 版 WAL ，对按表与按 shard 两种 WAL 日志回放方式的耗时进行了简单测试。

**RocksDB 版 WAL：**

- 测试步骤：在测试环境中，创建了 443 张测试表，并引入测试流量生成了 1.1G WAL 日志，分别按表和按 shard 进行回放，并记录耗时。
- 测试结果：按表回放耗时 82.99s，按 shard 回放耗时 52.28s，按 shard 回放性能提高到按表回放的 1.59 倍。

**Kafka 版 WAL：**

- 测试步骤：在测试环境中，创建了 443 张测试表，并引入测试流量生成了 2.4G WAL 日志（由于与 RocksDB 版 WAL 实现上差异较大，并且两种 WAL 实现之间并不作性能对比，因此没有刻意控制数据大小相等），分别按表和按 shard 进行回放，并记录耗时。
- 测试结果：按表回放耗时 4508.23s，按 shard 回放耗时 102.36s，按 shard 回放性能提高到按表回放的 44.04 倍。

## 总结

以上就是 CeresDB 在 WAL 日志回放设计上踩过的坑，以及针对性优化，其实在这过程中我们也总结出了更广泛通用的设计原则：**在设计数据的读写方式时，必须先着重考虑数据的存储方式，尽量做到读写方式对存储方式友好**。
回到具体问题上，在 WAL 日志回放方面，仍存在以下的一些已知的优化点：

- 读取日志阶段的性能仍不够好，可能需要加入对日志数据的压缩，以减少 IO 传输的数据量，提高性能。
- Flush 策略仍过于简单，可能需要加入由 WAL 日志大小触发的策略，以更好地控制日志大小。

在之后的开发中，CeresDB 会继续进行这方面的优化工作，降低集群 failover 耗时，以求提供更高的集群可用性。

## 关于 CeresDB

- GitHub 仓库：https://github.com/CeresDB/ceresdb
- CeresDB 文档：https://docs.ceresdb.io
- Release v1.2.4: https://github.com/CeresDB/ceresdb/releases/tag/v1.2.4

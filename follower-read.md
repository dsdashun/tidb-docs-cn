---
title: Follower Read
summary: 了解 Follower Read 的使用与实现。
aliases: ['/docs-cn/dev/follower-read/','/docs-cn/dev/reference/performance/follower-read/']
---

# Follower Read

当系统中存在读取热点 Region 导致 leader 资源紧张成为整个系统读取瓶颈时，启用 Follower Read 功能可明显降低 leader 的负担，并且通过在多个 follower 之间均衡负载，显著地提升整体系统的吞吐能力。本文主要介绍 Follower Read 的使用方法与实现机制。

## 概述

Follower Read 功能是指在强一致性读的前提下使用 Region 的 follower 副本来承载数据读取的任务，从而提升 TiDB 集群的吞吐能力并降低 leader 负载。Follower Read 包含一系列将 TiKV 读取负载从 Region 的 leader 副本上 offload 到 follower 副本的负载均衡机制。TiKV 的 Follower Read 可以保证数据读取的一致性，可以为用户提供强一致的数据读取能力。

> **注意：**
>
> 为了获得强一致读取的能力，在当前的实现中，follower 节点需要向 leader 节点询问当前的执行进度（即 `ReadIndex`），这会产生一次额外的网络请求开销，因此目前 Follower Read 的主要优势是处理隔离集群的读写请求以及提升整体读取吞吐。

## 使用方式

要开启 TiDB 的 Follower Read 功能，将变量 `tidb_replica_read` 的值设置为 `follower` 或 `leader-and-follower` 即可：

{{< copyable "sql" >}}

```sql
set [session | global] tidb_replica_read = '<目标值>';
```

作用域：SESSION | GLOBAL

默认值：leader

该变量用于设置期待的数据读取方式。

- 当设置为默认值 `leader` 或者空字符串时，TiDB 会维持原有行为方式，将所有的读取操作都发送给 leader 副本处理。
- 当设置为 `follower` 时，TiDB 会选择 Region 的 follower 副本完成所有的数据读取操作。
- 当设置为 `leader-and-follower` 时，TiDB 可以选择任意副本来执行读取操作，此时读请求会在 leader 和 follower 之间负载均衡。
- 当设置为 `closest-replicas` 时，TiDB 会优先选择分布在同一区域的副本执行读取操作，对应的副本可以是 leader 或 follower。如果同一区域内没有副本分布，则会从 leader 执行读取。
- 当设置为 `closest-adaptive` 时，当一个读请求的预估返回结果大于或等于变量 [`tidb_adaptive_closest_read_threshold`](/system-variables.md#tidb_adaptive_closest_read_threshold-从-v630-版本开始引入) 的值时，TiDB 会优先选择分布在同一区域的副本执行读取操作，否则选择 leader 副本进行处理。此时，为了防止读流量在各个区域分布不均衡，TiDB 会动态检测当前在线的所有 TiDB 和 TiKV 的区域分布是否均衡。如果发现在某一个区域内仅包含 TiDB 或 TiKV 的节点，则会强制使用 leader 读取。例如，如果某个区域的 TiDB 全部无法连接，此时其他在线的 TiDB 将会降级至 leader 读取。当此区域有至少 1 个 TiDB 节点重新上线，则会恢复优先从同一区域的副本读取。

> **注意：**
>
> 当设置为 `closest-replicas` 或 `closest-adaptive` 时，你需要配置集群以确保副本按照指定的设置分布在各个区域。请参考[通过拓扑 label 进行副本调度](/schedule-replicas-by-topology-labels.md)为 PD 配置 `location-labels` 并为 TiDB 和 TiKV 设置正确的 `labels`。TiDB 依赖 `zone` 标签匹配位于同一区域的 TiKV，因此请**务必**在 PD 的 `location-labels` 配置中包含 `zone` 并确保每个 TiDB 和 TiKV 节点的 `labels` 配置中包含 `zone`。如果是使用 TiDB Operator 部署的集群，请参考[数据的高可用](https://docs.pingcap.com/zh/tidb-in-kubernetes/v1.4/configure-a-tidb-cluster#%E6%95%B0%E6%8D%AE%E7%9A%84%E9%AB%98%E5%8F%AF%E7%94%A8)进行配置。

## 实现机制

在 Follower Read 功能出现之前，TiDB 采用 strong leader 策略将所有的读写操作全部提交到 Region 的 leader 节点上完成。虽然 TiKV 能够很均匀地将 Region 分散到多个物理节点上，但是对于每一个 Region 来说，只有 leader 副本能够对外提供服务，另外的 follower 除了时刻同步数据准备着 failover 时投票切换成为 leader 外，没有办法对 TiDB 的请求提供任何帮助。

为了允许在 TiKV 的 follower 节点进行数据读取，同时又不破坏线性一致性和 Snapshot Isolation 的事务隔离，Region 的 follower 节点需要使用 Raft `ReadIndex` 协议确保当前读请求可以读到当前 leader 上已经 commit 的最新数据。在 TiDB 层面，Follower Read 只需根据负载均衡策略将某个 Region 的读取请求发送到 follower 节点。

### Follower 强一致读

TiKV follower 节点处理读取请求时，首先使用 Raft `ReadIndex` 协议与 Region 当前的 leader 进行一次交互，来获取当前 Raft group 最新的 commit index。本地 apply 到所获取的 leader 最新 commit index 后，便可以开始正常的读取请求处理流程。

### Follower 副本选择策略

由于 TiKV 的 Follower Read 不会破坏 TiDB 的 Snapshot Isolation 事务隔离级别，因此 TiDB 选择 follower 的策略可以采用 round robin 的方式。目前，对于 Coprocessor 请求，Follower Read 负载均衡策略粒度是连接级别的，对于一个 TiDB 的客户端连接在某个具体的 Region 上会固定使用同一个 follower，只有在选中的 follower 发生故障或者因调度策略发生调整的情况下才会进行切换。而对于非 Coprocessor 请求（点查等），Follower Read 负载均衡策略粒度是事务级别的，对于一个 TiDB 的事务在某个具体的 Region 上会固定使用同一个 follower，同样在 follower 发生故障或者因调度策略发生调整的情况下才会进行切换。

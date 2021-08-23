
### 什么是状态？
在一条数据流中，很多操作一次只处理一个事件，比如事件解析器，就是简单的 ETL 操作。但是某些操作需要记住已经处理过的事件的信息，比如流式处理中每小时的请求量，也就是窗口操作，我们管这些操作叫作有状态的，已经发生处理过的事件会影响后续的统计。
下面是一些有状态的操作的例子：

- 当应用程序需要按照某些模式搜索事件时，需要记录下来已经发生过的事件。
- 当按照日期聚合事件时，需要记录下来该日期内发生的事件当做状态。
- 当训练机器学习模型时，需要用状态记录下来当前的模型版本，参数等信息。
- 当需要处理历史事件时，需要用状态记录过去发生的事件，这样才有权限访问统计。

Flink 使用 Checkpoints 和 Savepoints 来做容错管理和精确处理一次的处理语义，因此需要管理状态，记录各个操作的状态。

需要注意的是 Flink 重启并且并行度改变时，状态也需要分发至不同节点，并重新分配。这点在 failover 时会讲到。

[Queryable state](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/datastream/fault-tolerance/queryable_state/)允许你在 flink 程序运行时管理状态，即 state processer API，这些在任务状态初始化，状态错误修改场景下比较有用。

当处理状态时，先了解 [Flink’s state backends](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/ops/state/state_backends/)非常有用，这会告诉我们 Flink 支持哪几种状态后端，状态以何种方式存在哪里。

### Keyed State
Keyed State 可以被认为是以键值对方式存储的状态。举个例子，统计当天每个商品的售出，在某一时刻，商品 A 售出了 10 件，那么 `A->10` 就是此刻的状态，当下次再接收到 A 卖出一件的事件时，状态就变成了 `A->11`，checkpoint 中存储的状态就是截止某一时间点前某个 key 的当前状态。
Keyed State 与读取该 key 的流一起被分发，因此，只能在 keyd stream 上操作键值对状态。理解一下，也就是 keyed state 与下游该 key 的 operator 是绑定在一起的，这个 operator 可以更新插入新的 keyed state，并且在状态和 operator 对齐情况下都是本地操作，没有事务开销，保证一致性。这种对齐还允许 Flink 重新分发状态和调整流分区。拿官网上这张图举例子来说，Keys(A,B,Y) 被分发到了 task1 上，A,B,Y 的后续状态会在 task1 上进行管理，管理了三个 key 的 state。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/state_partitioning.svg" width="60%" height="60%"></div>

Keyed State 被进一步组织成 Keyed Groups，这是 Flink 重新分发 Keyed State 的最小单位，Keyed Groups 的数量与设置的最大并行度一致。
在执行过程中，Keyed Operator 的每个并行实例都与一个或多个 Key Groups 的键一起工作。


### 状态持久化
Flink 使用流重放和 Checkpoints 来实现容错。每个 checkpoint 都是输入流中一个特定点以及这个特定点时间当时各个 operator 的状态。一个实时数据流可以从上次成功的 checkpoint 恢复，恢复各个 operator 的状态，并重放数据，重新处理上次 checkpoint 后来的数据。checkpoint 的间隔需要衡量容错开销与恢复时间（需要重新处理的记录数）。

这种容错机制会源源不断的产生分布式快照，对于状态比较小的流式处理程序，即使 checkpoint 比较频繁，这种方式也比较轻量且对性能没有太大影响。checkpoint 被存储在配置的位置，往往是一个分布式文件系统中。

在程序由于机器，网络，处理逻辑问题挂掉后，flink 会将程序重置到上次成功的 checkpoint 处，重启后的 flink 程序并不会影响以前产生的 checkpoint。

>> checkpoint 默认是关闭的，可以阅读 [Checkpointing ](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/datastream/fault-tolerance/checkpointing/) 查看如何配置开启。

>> 为了保证 checkpoint 机制正常使用，数据源（例如消息队列）需要支持数据重放。Kafka 具有这种能力，并且对应的 Flink connector 也支持。可以去 [Fault Tolerance Guarantees of Data Sources and Sinks ](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/connectors/datastream/guarantees/) 查看更多信息。

>> 因为 Flink 的 checkpoint 使用分布式快照实现，所以我们经常用快照表示 checkpoint 或者 savepoint。

### Checkpointing
Flink 容错机制的核心就是产生数据流和对应 operator 状态的分布式一致性快照。这些快照充当一致性检查点，以便在任务失败时可以回退重启。需要注意的是，checkpoint 是异步完成的。状态的分布式一致性快照。这些快照充当一致性检查点，以便在任务失败时可以回退重启。需要注意的是，checkpoint barriers 不会对操作加锁，因此 operator 可以异步持久化他们的状态。

从 Flink 1.11 开始，checkpoints 可以选择对齐或者不对齐，下面我们首先介绍对齐情况下的 checkpoints。
#### Barriers
Flink 分布式快照中的核心是 stream barriers，暂且叫它流式屏障吧。这些 barriers 被注入到数据流中随着数据一起流动，barriers 永远不会超过最新记录，并且是严格顺序的。一个 barrier 将记录分成进入当前快照和进入下一个快照的记录。每个 barrier 携带者快照 id。barriers 不会中断数据流的处理，因此非常轻量。在同一时间数据流中可能存在多个 barrier，这意味着不同的快照在同时并行发生。

<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/stream_barriers.svg" width="60%" height="60%"></div>

stream barriers 从数据源读取开始被注入到并行数据流中。

#### Snapshotting Operator State

#### Recovery

### Unaligned Checkpointing 

#### Unaligned Recovery 

### State Backends

### Savepoints 

### Exactly Once vs. At Least Once

### State and Fault Tolerance in Batch Programs

### 思考
1. Keyed State 中的对齐是指？
2. Keyed Groups 是和什么最大并行度保持一致？



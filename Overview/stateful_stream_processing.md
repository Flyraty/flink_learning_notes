
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

> checkpoint 默认是关闭的，可以阅读 [Checkpointing ](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/datastream/fault-tolerance/checkpointing/) 查看如何配置开启。

> 为了保证 checkpoint 机制正常使用，数据源（例如消息队列）需要支持数据重放。Kafka 具有这种能力，并且对应的 Flink connector 也支持。可以去 [Fault Tolerance Guarantees of Data Sources and Sinks ](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/connectors/datastream/guarantees/) 查看更多信息。

> 因为 Flink 的 checkpoint 使用分布式快照实现，所以我们经常用快照表示 checkpoint 或者 savepoint。

### Checkpointing
Flink 容错机制的核心就是产生数据流和对应 operator 状态的分布式一致性快照。这些快照充当一致性检查点，以便在任务失败时可以回退重启。需要注意的是，checkpoint 是异步完成的。状态的分布式一致性快照。这些快照充当一致性检查点，以便在任务失败时可以回退重启。需要注意的是，checkpoint barriers 不会对操作加锁，因此 operator 可以异步持久化他们的状态。

从 Flink 1.11 开始，checkpoints 可以选择对齐或者不对齐，下面我们首先介绍对齐情况下的 checkpoints。
#### Barriers
Flink 分布式快照中的核心是 stream barriers，暂且叫它流式屏障吧。这些 barriers 被注入到数据流中随着数据一起流动，barriers 永远不会超过最新记录，并且是严格顺序的。一个 barrier 将记录分成进入当前快照和进入下一个快照的记录。每个 barrier 携带者快照 id。barriers 不会中断数据流的处理，因此非常轻量。在同一时间数据流中可能存在多个 barrier，这意味着不同的快照在同时并行发生。

<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/stream_barriers.svg" width="60%" height="60%"></div>

stream barriers 从读取数据源开始，就被注入到数据流中，随着数据源一起流动，在某一时刻，barrier 流动到点 n，这意味着快照 n 将会覆盖数据源 n 位置之前的所有数据。比如，在 Kafka 中，这个位置是分区 offset。这个位置 n 会被提交给 JM 上的 coordinator 进程，用于后续协调整个作业的 checkpoint 生成。

barrier 接着会流往下游，当下游中间的算子接受到所有代表 n 位置的 barrier 时，它继续将 barrier n 发往下游，如此往复，直到 sink operator 从其上游所有输入流中接收到 barrier n 。此时会通知 JM 上的 checkpoint 管理进程该 sink operator 的快照 n 已经准备好了，当所有的 sink operator 都接收到快照 n 时，checkpoint n 的产生也随之完成。一旦快照 n 完成，Flink 应用程序将不会再访问 barrier n 之前的数据。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/stream_aligning.svg" width="60%" height="60%"></div>

对于有多个输入流的 operator，需要对齐 barrier。上图说明了这一点
- 下游 operator 接受到一条上游数据输入流发过来的 barrier n ，直到它的所有输入流都接受到 barrier n，这个 operator 才可以处理位置 n 之后的数据。否则的话，快照 n 里将混合位置 n 和 n+1 之后的数据。（这样来想，如果 barrier n 不等待对齐，而是继续发往下游，处理新的数据，最终快的 barrier 先到达 sink 处，并继续处理数据，等较慢的 barrier 到达 sink 时，请求 JM 生成快照 n，此时生成的快照 n 还是只处理的位置 n 之前的数据吗？已经不是了，快的 task 已经处理了位置 n 之后的数据，这样的检查点在任务重启时，肯定会重复处理数据，也就无法做到精确一次处理。）
- 一旦某个并发的 operator 最后一条流收到了所有的 barrier n，operator 就会继续处理后续的数据，并且发送 barrier n 到下游。
- 接下来，这个 operator 会继续处理输入流中的数据，在真正读取流数据前，会先从输入缓冲区中读取数据。
- 最后，operator 会将自己的状态异步写入状态后端中。

需要注意的是对齐操作适用于多条输入流的 operator，比如多分区的 kafka topic 数据源、发生 shuffle 过程后，需要读取上游数据的下游 operator。

#### Snapshotting Operator State
当 operator 本身也包含状态时，这种状态也必须是快照的一部分。
Operators 接收到来自所有输入流的 barrier 后，在发送 barrier 到下游数据前快照自己的状态。在此时，barrier 之前的数据都已经被处理，状态也已经记录。因为快照一般比较大，所以会存储到配置的状态后端（比如 HDFS），默认情况下是使用 JM 的内存。当状态被存储下来后，operator 会通知检查点，并且会向下游输出流发送 barrier。

生成的快照现在包含
- 对于每一个并行的数据源，快照对应的起始读取位置。
- 对于每一个 operator，存储指向对应状态的指针。此处猜想一下，checkpoint 存储应该分为 metadata 和 operator state 两大部分，在存储 operator state 的过程中，将元数据提交给 JM coordinator 进程，缓存元数据，最后在 sink operator 完成后，元数据完全写入状态后端，最终形成一个完整的 checkpoint。
![](https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/checkpointing.svg)

#### Recovery
这种机制下的任务恢复比较简单，当任务失败重启时，Flink 选择最新成功的 checkpoint 并应用。程序会重新提交 job grpgh，并将检查点的 operator state 应用到对应的 operator 上，source operator 也会被重置到上次 checkpoint 记录的位置，对于 kafka 来说，就是重置 offset。

如果快照是增量更新的话，operator 会从最新的完整状态的快照开始，并逐步应用最新的增量快照。

可以查看 [Restart Strategies](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/execution/task_failure_recovery/#restart-strategies)获取更多信息。

### Unaligned Checkpointing 
非对齐的 checkpoint。
checkpoint 允许不对齐 barrier，其基本核心思想是当某个 operator 的某条输入流的 barrier 到来时，不在等待未到达 barrier 的输入流，而是立即像下游发送 barrier，并通知 JM 生成 operator 的局部快照，这样做会造成 checkpoint 不对齐，因而需要在这个快照中保存较慢输入流未处理的数据，从而在 failover 时能重放状态，也就是让程序知道每个 operator 的每条输入流处理到了什么位置，咋听起来，这种方式更复杂。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/stream_unaligning.svg) 
官网的图感觉不太形象，没太理解。在这里，个人理解下上面的流程
- operator 首先接收到上游的第一个 barrier，及 3 之前的数据，并放在了 input buffer 中。
- 接下来把 3 这个 barrier 立即发往下游，并放在 output buffer 的末端。
- operator 会被超越的过的数据，及相对 3 这个 barrire 来说，其他较慢输入流之前的位置的数据进行缓存并生成快照。

因此，非对齐的 checkpoint 仅仅是缓存数据并且转发 barrier，并不需要过多的对齐时间，这点对于吞吐量是有提升的。

需要注意的是 savepoint 往往是对齐的。


#### Unaligned Recovery
非对齐状态的恢复首先是要恢复 operator 正在处理的数据，其他和对齐 checkpoint 的步骤是一样的。 

### State Backends
状态存储的数据结构取决于使用哪种状态后端，一种状态后端是内存中的 hashmap，也可以使用 rocksdb 的键值对存储。状态后端可以在不改变应用逻辑的情况下进行配置。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/checkpoints.svg" width="60%" height="60%"></div>


### Savepoints 
checkpoint 是自动的，savepoint 是手动触发的，并且不会自动过期，一般用于手动停止程序后的恢复操作。

### Exactly Once vs. At Least Once
barrier 的对齐会增加实时流程序处理的延迟，通常，这些延迟在几毫秒内，但是在某些情况下，对齐延迟可能会在分钟甚至小时级别。如果你确实需要超低延时的的流处理，Flink 提供了非对齐的 checkpoint，barrier 会尽可能快的流向 sink，而不需要等待慢的输入流。

在跳过对齐操作后，operator 会持续处理输入，产生的 checkpoint 也是非对齐的，因此在任务 failover，重放数据时，有可能造成数据重复，及至少一次的处理语义。

### State and Fault Tolerance in Batch Programs
Flink 将批任务运行当做流任务的特例。因此，上述状态的管理程序容错机制同样适用于批处理，不过有以下几点不同：
- 批处理程序没有检查点，在 failover 时直接重放所有数据，将性能压力放在了 source 端，去除了数据处理时的 checkpoint 消耗。
- Dataset 中状态的处理使用内存数据结构，而不是键值对。

### 思考
**1.Keyed State 中的对齐是指？**

这里没太理解，keyed state 需要对齐的是什么信息呢？

**2.Keyed Groups 是和什么最大并行度保持一致，如何计算出来的呢？**

个人感觉是和 operator 的并行度保持一致（这点需要在后续阅读中验证，也可以查看源码）。在并行度改变时，以 key groups 为单位重新分配 keyed state 到对应的 subtask。
此处可以参考 [Flink状态的缩放（rescale）与键组（Key Group）设计](https://cloud.tencent.com/developer/article/1697402)

**3.在 shuffle 产生数据倾斜时，会不会影响 operator barrier 的对齐过程？**

感觉会影响，下游在读取上游数据时，处理大 key 的 task 肯定比较慢，成为 barrier 对齐等待的一方

**4.怎么描述整个 checkpoint 的过程？**

从读取 source 端开始，barrier 被注入到数据流中，下游 operator 在接收到其所有输入流的 barrier 后（即对齐过程），通知 JM checkpoint coordinator 存储状态生成局部快照，记录元数据，并将 barrier 传递给下游，如此往复，直到 sink operator，最终生成全局快照。

**5.对齐和不对齐的 checkpoint 的有什么区别，对于性能的影响和考量又在哪里？**

此处可以参考 [Flink 1.11 新特性详解:【非对齐】Unaligned Checkpoint 优化高反压](https://cloud.tencent.com/developer/article/1663055)
- 对齐会阻塞数据的数据的处理，在 input buffer 被填满后，还没等到对齐时，会对性能产生影响，增大整个数据处理的延时，降低了程序吞吐量，尤其是在存在反压的时候。但是逻辑清晰，以算子快照为界限分隔。本质上是在最后一个 barrier 到达时触发 checkpoint。
- 非对齐提高了程序吞吐量，但是由于每个 operator state 缓存当前被 barrier 越过的数据，快照的大小会显著增加，IO 压力会增大。本质上是在第一个 barrier 到达后就触发 checkpoint。

**6.checkpoint 相关的配置参数？**

代码
	```java
	StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

	// start a checkpoint every 1000 ms
	env.enableCheckpointing(1000);

	// advanced options:

	// set mode to exactly-once (this is the default)
	env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

	// checkpoints have to complete within one minute, or are discarded
	env.getCheckpointConfig().setCheckpointTimeout(60000);

	// make sure 500 ms of progress happen between checkpoints
	env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

	// allow only one checkpoint to be in progress at the same time
	env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

	// enable externalized checkpoints which are retained after job cancellation
	env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

	// This determines if a task will be failed if an error occurs in the execution of the task’s checkpoint procedure.
	env.getCheckpointConfig().setFailOnCheckpointingErrors(true);

	```

flink-conf.yml

	```yml
	# 使用何种状态后端
	state.backends: filesystme
	# 检查点的存储目录
	state.checkpoint.dir: 
    # 保存点的存储目录
    state.savepoints.dir
    # 是否开启增量快照，默认 false
    state.backend.incremental: false

	```
**7.为什么需要增量快照？** 
对于 TB 级别的作业，状态太大，每次做全量快照耗时太长，影响整个链路的处理时延，所以需要增量快照。类似于 Mac 的 TimeMachine，在做完一次全量快照后，剩下的都是增量快照，只处理变化的数据。
那么 Flink 是如何实现增量快照的呢？可以参考 [Apache Flink 管理大型状态之增量 Checkpoint 详解](https://cloud.tencent.com/developer/article/1506196)







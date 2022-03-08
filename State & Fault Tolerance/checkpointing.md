### Checkpointing
Flink 中的每个函数或者 operator 都可以是有状态的，有状态的函数可以暂时理解为在数据流接收计算过程中存储数据，比如计算五分钟之内观看直播的人次，我们就要缓存该五分钟之内进入的用户id。在类似这样的场景中，状态已然成为了程序功能组成的重要部分。

为了保证状态容错，Flink 需要缓存状态，即 Checkpoint。Checkpoint 使得 Flink 在任务失败时，可以恢复到失败时的状态和数据流的位置，以实现相同的处理语义。

Flink Checkpoint 机制需要和存储做交互（数据源存储和状态存储，不管是内存还是外部数据库），因此有几个必需点：

- 一个可以支持重放数据的持久化高可用的数据源，比如很多消息队列（Kafka，RabbitMQ 等）或者外部文件存储系统（HDFS，S3等）
- 一个可以持久化状态的存储，具有代表性的是分布式文件系统（比如 HDFS, S3, GFS, NFS, Ceph, …）

### Checkpointing 的开启和配置
Checkpoint 默认是不开启的，需要调用 StreamExecutionEnvironment 上的 enableCheckpointing(n) 方法来启用，其中 n 代表的两次 Checkpoint 之间的 ms 间隔。

其他 checkpointing 相关的参数如下：

- *checkpoint storage（checkpoint 的存储位置）*：用来设置 checkpoint 快照的存储位置。默认情况下使用 JobManager 的堆内存。在生产环境中一般建议使用分布式文件系统。可以查看 [checkpoint storage](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/ops/state/checkpoints/#checkpoint-storage) 获取更多作业或者集群方面的配置。

- exactly-once vs at-least-once（精确一次处理 vs 至少一次处理）: 可以通过 enableCheckpointing 方法来设置处理语义，exactly-once 在大多数情况下性能是良好的。但是如果需要极低延时的场景下，也可以考虑使用 at-least-once。性能和准确性从来都是需要平衡的问题，需要结合自己的业务场景，比如只是简单的 ETL 处理，sink 侧可以做到幂等保证，而且需要 ms 级别的延迟保障，那么就可以使用 at-least-once。

- checkpoint timeout（checkpoint 的超时时间）：在达到某个时间阈值后，checkponit 仍然没有做完，那么就终止本次 checkpoint。单位是 ms。

- minimum time between checkpoints（两次 checkpoint 间隔的最小时间）：为了保证 checkpoint 已经运行了一段时间，或者说是完成，我们需要设置两次 checkpoint 之间的最小间隔。假如设置成 5000ms，则下次 checkpoint 必须在上次 checkpoint 完成 5s 后才会触发，此时与 checkpoint 间隔和运行时长无关。需要注意该参数必须小于 checkpoint interval。

- tolerable checkpoint failure number（可容忍的 checkpoint 失败数）：其实就是整个作业在多少次 checkpoint 失败后可以被标记为失败。默认值是 0，即一旦有 checkpoint 失败，则整个作业就会挂掉。

- number of concurrent checkpoints（可并行的 checkpoint 数量）：默认情况下，在上一个 checkpoint 未完成的情况下，不会触发下一个 checkpoint，这样做的原因主要是保证流数据处理的速度，不会被过多并发的 checkpoint 所影响。

- externalized checkpoints（物化 checkpoints）：可以设置 checkpoint 写入至外部存储系统，这样在任务失败的时候就不会被清理掉，可以用来恢复数据。

- unaligned checkpoints（非对齐的 checkpoints）：Flink 通过 input buffers 缓存数据和 barrier 的对齐来保证处理过程中的 exactly-once，但是在产生背压的情况下会极大拉长处理时间。非对齐的 checkpoint 可以用来解决此问题，降低处理时延，但是可能会破坏精确一次处理语义。只适用于 checkpoint 并发为 1 的情况。

- checkpoints with finished tasks（对已经完成的 task 做 checkpoint）: 目前这是一个实验性的功能，当 DAG 某一部分已经完成并且处理完了所有数据（比如流批 join 的场景），后续的 checkpoint 依然会记录已完成的 task 的状态。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// only two consecutive checkpoint failures are tolerated
env.getCheckpointConfig().setTolerableCheckpointFailureNumber(2);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained
// after job cancellation
env.getCheckpointConfig().enableExternalizedCheckpoints(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

// enables the unaligned checkpoints
env.getCheckpointConfig().enableUnalignedCheckpoints();

// sets the checkpoint storage where checkpoint snapshots will be written
env.getCheckpointConfig().setCheckpointStorage("hdfs:///my/checkpoint/dir")

// enable checkpointing with finished tasks
Configuration config = new Configuration();
config.set(ExecutionCheckpointingOptions.ENABLE_CHECKPOINTS_AFTER_TASKS_FINISH, true);
env.configure(config); 
```
### Checkpointing 可选的配置项
可以通过 conf/flink-conf.yaml 设置更多参数或者查看对应的默认值。

|参数	|默认值	|类型| 描述 |
|-------|----------|---------|
| state.backend.incremental | false | Boolean | 是否启用增量快照。对于增量快照来说，只需要存储和上次完成的 checkpoint 相比变化的数据，而不是该次全部快照，这对于状态数据非常庞大的场景是非常有用的。一旦启用该功能，flink web ui 上展示的 checkpoint 大小只会是该次变化的大小，而不是整个完整 checkpoint 的大小。并不是所有的状态后端都会支持 增量 checkpoint，目前只有 rocksDB 支持（得益于 LSM ）。|
| state.backend.local-recovery | false | Boolean | 状态的本地恢复，也可以叫作任务的本地恢复。本质上是尽可能的保证失败的 task 被重新调度回原来的 slot（taskManager 粒度），进而从本地文件状态恢复，不需要在跨网络分配传输。默认情况下，本地恢复被禁用。当前本地恢复只支持 KV 类型的状态后端，基于内存的状态存储不支持本地恢复。|
| state.checkpoint-storage | (none) | string |  | 
| state.checkpoints.dir | (none) | string | 用于存储状态数据及其元数据的文件目录，该目录必须允许所有节点可以访问到（即所有的 TaskManager 和 JobManager）| 
| state.checkpoints.num-retained | 1 | integer | 允许同时保留的已完成的 checkpoint 的最大个数|
| state.savepoints.dir| (none) | string | savepoints 的默认存储目录 | 
| state.storage.fs.memory-threshold | 20kb | MemorySize | 状态数据文件的最小大小，小于这个大小的状态文件将被合并存储在 checkpoint 元数据文件中。该配置项最大可设置为 1M。 |
| state.storage.fs.write-buffer-size | 4096 | Integer| checkpoint 数据写入文件系统时的写入缓冲区大小。|
| taskmanager.state.local.root-dirs| (none) | String |  用于本地恢复的状态数据存储的根目录，只支持 keyedStateBackend。MemoryStateBackend 不支持状态本地恢复，可以忽略此选项|

### Checkpoint 存储的选择
默认情况下，checkpoint 存储在 JobManager 的内存当中。另外对于大状态的存储，flink 也提供了不同的状态后端，可以通过 StreamExecutionEnvironment.getCheckpointConfig().setCheckpointStorage(…) 来进行配置。建议在生产环境中使用分布式文件存储系统作为状态后端。


### 迭代流中的 State Checkpoint
Flink 目前只支持非迭代数据流的状态处理。在迭代流中启用 checkpoint 会抛出异常。如果必须使用的话，可以强制开启，通过 env.enableCheckpointing(interval, CheckpointingMode.EXACTLY_ONCE, force = true) 进行配置。

### 对 DAG 中已完成的节点进行 Checkpoint（BETA）
Flink 自 1.14 开始，允许对 DAG 中已完成的节点（也可以说是 task）进行 checkpoint ，一般涉及到有界数据源。需要通过如下代码开启此功能

```java
Configuration config = new Configuration();
config.set(ExecutionCheckpointingOptions.ENABLE_CHECKPOINTS_AFTER_TASKS_FINISH, true);
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(config);
```
为了支持部分任务结束后的 Checkpoint 操作，我们调整了 任务的生命周期 并且引入了 StreamOperator#finish 方法。 在这一方法中，用户需要写出所有缓冲区中的数据。在 finish 方法调用后的 checkpoint 中，这一任务一定不能再有缓冲区中的数据，因为在 finish() 后没有办法来输出这些数据。 在大部分情况下，finish() 后这一任务的状态为空，唯一的例外是如果其中某些算子中包含外部系统事务的句柄（例如为了实现恰好一次语义）， 在这种情况下，在 finish() 后进行的 checkpoint 操作应该保留这些句柄，并且在结束 checkpoint（即任务退出前所等待的 checkpoint）时提交。 一个可以参考的例子是满足恰好一次语义的 sink 接口与 TwoPhaseCommitSinkFunction。

#### 对 operator state 的影响
在部分 Task 结束后的 checkpoint 中，Flink 对 UnionListState 进行了特殊的处理。 UnionListState 一般用于实现对外部系统读取位置的一个全局视图（例如，用于记录所有 Kafka 分区的读取偏移）。 如果我们在算子的某个并发调用 close() 方法后丢弃它的状态，我们就会丢失它所分配的分区的偏移量信息。 为了解决这一问题，对于使用 UnionListState 的算子我们只允许在它的并发都在运行或都已结束的时候才能进行 checkpoint 操作。

ListState 一般不会用于类似的场景，但是用户仍然需要注意在调用 close() 方法后进行的 checkpoint 会丢弃算子的状态并且 这些状态在算子重启后不可用。

任何支持并发修改操作的算子也可以支持部分并发实例结束后的恢复操作。从这种类型的快照中恢复等价于将算子的并发改为正在运行的并发实例数。

### 思考
**1. Flink 状态存储的整个过程是什么样子的，在 stateful_stream_processing 中有涉及到，但是缺失了很多细节，需要了解下。**

**2. 对已经完成的 task 做 checkpoint 有什么用？**





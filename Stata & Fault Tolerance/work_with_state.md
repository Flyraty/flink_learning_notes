### Keyed DataStream
如果你想使用 keyed state，那么就需要指定一个 key 字段用于状态与数据的分区。可以通过 `keyBy(KeySelector)` 方法来指定 key，这会返回 keyDataStream，用来后续在数据流上使用 keyed state。

KeySelector 接口的实现就是将每条记录当做输入，然后返回该条记录的 key，可以理解为业务处理逻辑上数据唯一标识。返回的 Key 可以是任意类型，但是一定是确定的，即输入相同的记录，得到的 key 应该是一样的。

Flink 的数据模型并不是基于键值对的，所以没必要将数据物理的处理成 key-value 的格式。Key 的概念是虚拟的，只是通过某些方法提取出记录的 key 作用于某些 operator 上（比如 group by 这些）。

下面是 KeySelector 的示例 
```java
// some ordinary POJO
public class WC {
  public String word;
  public int count;

  public String getWord() { return word; }
}
DataStream<WC> words = // [...]
KeyedStream<WC> keyed = words
  .keyBy(WC::getWord);
```

#### Tuple Keys and Expression Keys

Flink 还有两种声明 key 的方式，但是在新版本中已经不经常使用，使用 KeySelector 对于大多数场景更简单并且提供了更好的性能保证。
- Tuple 数据的下标指定，比如很多示例中常见的 `keyBy(0)`。
- 表达式指定，即 expression。

### Using Keyed State 
Keyed State 接口提供了对不同类型状态的访问。目前，Flink 提供了了以下几种状态原语：

- ValueState<T>：该状态保存一个可以被更新和读取的值（作用域为输入元素的键，读取不同键值返回的结果是不一样的）。通过 update(T) 更新，T value() 读取。

- ListState<T>：该状态保存一个元素数组，可以添加元素和通过迭代器读取元素数组。通过 add(T) 或者 ``addAll(List<T>)` 添加元素、Iterable<T> get() 读取、update(List<T>) 覆盖更新已经存在的状态。

- ReducingState<T>：该状态保存一个值，含义是该 key 下所有状态值的聚合。接口类似于 ListState，但是是通过 add(T) 添加元素，ReduceFunction 实现聚合计算。

- AggregatingState<IN, OUT>：该状态保存一个值，含义是该 key 下所有状态值的聚合。与 ReducingState 不同的是，聚合值的类型可能和输入元素的类型不一样（比如输入一堆事件，通过事件的某些标记进行聚合）。通过 add(IN) 添加元素，AggregateFunction 实现聚合计算。

- MapState<UK, UV>：该状态保存一个 mapping 数组。可以向该状态中添加键值对和通过迭代器读取已经存在的 mapping。通过  put(UK, UV) 或者 `putAll(Map<UK, UV>)` 添加元素、get(UK) 获取指定 key 的元素。可以分别使用 entry()、keys() 和 values() 访问迭代器检索键值对、键和值。使用 isEmpty() 检测状态是否为空。

所有类型的状态都有 clear() 方法，用来清空当前 key 的所有状态。

需要重点注意的是，状态操作暴露出来的接口仅用于与状态交互，与状态存储无关，状态可以存储再内部或者外部系统上。keyed state 的取值仅与调用时传入的 key 值有关，因此不同的 key 取回来的状态是不一样的。

要获得状态处理的句柄，需要声明 StateDescriptor，包含状态的名称（下面会讲到，存储的状态拥有一个唯一名称，可以理解为命名空间，可以通过该命名操作所有状态）、状态的类型，可选的用户自定义方法（比如 ReducingFunction）。根据你想取回来的状态类型，可以声明 ValueStateDescriptor、 ListStateDescriptor、AggregatingStateDescriptor、 ReducingStateDescriptor、 MapStateDescriptor。

状态通过 RuntimeContext 访问，所以只能应用于 *rich functions*。RichFunction 中的 RuntimeContext 通过以下方法访问状态：
- ValueState<T> getState(ValueStateDescriptor<T>)
- ReducingState<T> getReducingState(ReducingStateDescriptor<T>)
- ListState<T> getListState(ListStateDescriptor<T>)
- AggregatingState<IN, OUT> getAggregatingState(AggregatingStateDescriptor<IN, ACC, OUT>)
- MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)

下面展示了一个示例，当某个 key 的数据量到达两个时，就计算平均值，并清空当前的状态，如此往复。
```java
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(f)
        .keyBy(value -> value.f0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
```

#### State TTL
状态不可能无限期保存下去，因此有了 TTL 的概念，过期的状态将会通过一些策略进行删除。集合类型的状态的每个元素都可以设置 TTL，这意味着 list 和 mapping 类型的状态中每个元素的清理是独立的。

使用 TTL 需要定义 StateTtlConfig，下面是示例：
```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```
StateTtlConfig 后面跟着几个必选项和可选项，newBuilder 是必选的，其参数定义了 TTL 的时间。

可选的 setUpdateType 定义了 TTL 的触发策略，提供了两个选择，默认是 OnCreateAndWrite
- StateTtlConfig.UpdateType.OnCreateAndWrite 在创建和写入时就会触发。
- StateTtlConfig.UpdateType.OnReadAndWrite 在创建，写入，读取时都会触发。

setStateVisibility 用来设置到达 TTL 的状态但是未被清除的时候，读取是否要返回值。
- StateTtlConfig.StateVisibility.NeverReturnExpired 不会返回过期状态
- StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp 状态虽然过期但是未被清除，读取仍然会返回值。

Notes：
- 为了实现 TTL ，状态后端需要存储每个值修改的时间戳，这样会增加存储压力。Heap 状态后端会存储额外的 java 对象，包含对状态的引用和内存中的原始 long 值。RocksDB 会为每个状态元素添加 8 字节用于存储时间。
- TTL 目前仅仅基于处理时间。
- 如果是从状态中恢复任务的话，需要关闭状态的 TTL，否则会引发任务失败和一些未知错误。
- 只要状态值的序列化支持 null，设置 TTL 的 map 类型的状态也可以支持处理 null 值。否则会抛出 NullableSerializer。
- Python DataStream API 目前不支持 State TTL。

#### 过期状态的清除策略
默认配置下，程序是不会读取到过期的状态的（不管此时状态是逻辑删除还是物理删除），如果状态后端支持的话，会启动一个后台进程定期收集过期状态进行删除。通过 StateTtlConfig 可以关闭该后台进程
```java
import org.apache.flink.api.common.state.StateTtlConfig;
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .disableCleanupInBackground()
    .build();
```
下面会介绍一些更细粒度的状态清理策略。一般 Heap 状态后端会进行增量更新，RocksDB 会通过压实策略进行清理。

**Cleanup in full snapshot**

在生成状态快照的时候触发状态的清理，这样做可以减少快照的大小。在当前实现下，本地状态不会被清理，但是前一个快照中过期状态已经被清理了。可以通过 StateTtlConfig 配置。
```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot()
    .build();
```
这种清理策略不适用于 RocksDB 上的增量快照。

**Incremental cleanup**

增量触发状态的清理，可以在访问状态或者处理记录时触发。基本实现是维持一个全局的状态元素迭代器，当清理被触发后，调用迭代器，遍历所有状态元素然后清理掉过期状态。同样是通过 StateTtlConfig 配置。
```java
import org.apache.flink.api.common.state.StateTtlConfig;
 StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupIncrementally(10, true)
    .build();
```
这里有两个参数，第一个参数代表每次触发清理时检查的状态条数，在访问状态的时候就会触发检查。第二个参数代表每次处理数据时是否需要触发清理策略。Heap Backend 默认是在状态访问时触发检查5条状态是否过期。

Notes：
- 如果没有状态访问或者数据流的处理，过期的状态会永远保存下去。
- 增量清理会增大处理时延。
- 目前仅对 Heap Backend 进行增量清理，不支持 RocksDB。 
- 如果配置的 Heap Backend 是同步快照，则全局的迭代器会保留所有 key 的副本用来清理，因为其实现不支持并发修改。异步则没有此问题。
- 对于已经运行的 job，可以通过 StateTtlConfig 开启清理，例如从 savepoint 恢复作业时。

**Cleanup during RocksDB compaction**

使用 RocksDB 状态后端的话，会调用 Flink compaction filter 用于后台状态清理。compaction 意为压实，在大数据存储中经常会接触到，目的是对各种摄入任务的文件碎片进行合并压缩，减小碎片数量，提高存储查询的性能。这里的压实也是一样的，在对状态做合并压缩的过程中顺便对状态进行清理。 Flink compaction filter 会检查状态的时间并清除过期策略。
```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupInRocksdbCompactFilter(1000)
    .build();
```
这种策略会在每次处理一定数量的状态元素后从 Flink 查询当前时间戳，用于检查过期时间。检查条数的配置就是 cleanupInRocksdbCompactFilter 的参数。RocksDB 采用这种策略的默认配置是每处理 1000 条数据就查询更新时间戳。emmn，我此处描述的其实非常不清晰。。

Notes：
- 显而易见，在 compaction 期间进行状态的清理会降低 compaction 的效率。程序必须获取上次查询的时间戳对存储的每个 key 的状态判断是否过期。对于 list，map 类型的状态，还要对状态存储中的每个元素进行检查。

### Operator State
*Operator State* 可以认为是绑定并行 operator 中的一个实例的状态。最明显的例子是 Kafka，在从 Kafka 源读取数据时，一般会根据 paratition 的数量设置并行度，这意味着读取数据源的 opertor 是并行多个实例的，这里的 Operator State 指的就是每个实例读取的 partition 及其 offset。

Operator State 接口支持在并行度发生改变时重新分配状态。分配策略有几种不同的方案，这个后面会讲到。

在典型的有状态数据流处理中，一般不需要 Operator State。Operator State 仅仅是一种特殊的状态类型，在 source 或者 sink 中实现，用于没有明确分区 key 的场景。

### Broadcast State
*Broadcast State* 是一种特殊的 Operator State。主要用于一条数据流中的数据需要广播到下游所有 task 的场景，即每个 task 保留的状态是一致的。一个例子是关于事件模式的匹配，吞吐量比较小的事件规则流作为状态广播到事件流，事件流中处理每条事件筛选出符合规则的用户行为。

Broadcast State 与 Operator State 主要有以下几点不同
- it has a map format,
- it is only available to specific operators that have as inputs a broadcasted stream and a non-broadcasted one, and
- such an operator can have multiple broadcast states with different names.

### Using Operator State
继承 CheckpointedFunction 来使用 Operator State。

#### CheckpointedFunction
CheckpointedFunction 提供非 keyed state 状态的访问，需要实现以下两种方法
```java
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
```
当执行 checkpoint 时，snapshotState 就被调用。对应的是，在每次执行用户自定义函数或者从上个检查点恢复用户自定义函数执行时，会调用 initializeState()。这样看来，initializeState 不仅仅是初始化状态的地方，也可以包含恢复逻辑。

当前，支持 list 类型的 operator state。这种状态应该是一个可序列化对象的列表，彼此独立，这也是可以重新分配状态的基础，换句话说，这些可序列化对象是重新分配的最小粒度。基于此，定义了以下两中重分配策略：

- **均匀分配**：每个 operator 拥有一个状态元素列表，整个状态可以看做是所有 operator 列表的组合。在重新分配状态时，该列表被平均分为与并行运算符一样多的子列表。每个 operator 都得到一个子列表，可能是空，也可能包含一个或者多个元素。例如，如果并行度为 1，operator state 中包含元素 element1 和 element2，当并行度增加到 2 时，element1 可能会在 operator 实例 0 中结束，而 element2 将转到 operator 实例 1。

- **合并分配**：每个 operator 拥有一个状态元素列表，整个状态可以看做是所有 operator 列表的组合。在重新分配状态时，每个 operator 获取状态的所有元素。当状态是高基数时，不建议使用，因为 checkpoint 元数据将存储每个列表条目的偏移量，这可能导致 RPC 帧大小或内存溢出。

下面是一个均匀分配的例子，先缓存一批数据在 sink 出去。在创建 checkpoint 时保存 bufferedElements 的状态。在从 checkpoint恢复时，重放 bufferedElements。
```java
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value, Context contex) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
```
#### Stateful Source Functions
source 相比其它 operator 更需要注意，一般需要锁机制做到原子更新保证精确一次处理语义。
```java
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements CheckpointedFunction {

    /**  current offset for exactly once semantics */
    private Long offset = 0L;

    /** flag for job cancellation */
    private volatile boolean isRunning = true;
    
    /** Our state object. */
    private ListState<Long> state;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        state = context.getOperatorStateStore().getListState(new ListStateDescriptor<>(
                "state",
                LongSerializer.INSTANCE));
                
        // restore any state that we might already have to our fields, initialize state
        // is also called in case of restore.
        for (Long l : state.get()) {
            offset = l;
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        state.clear();
        state.add(offset);
    }
}
```


### 思考
**1.自定义状态可以用来做什么？**

实现一些内置聚合函数无法做到的逻辑。实现程序恢复。



### Queryable State

> 查询状态数据的 API 正处于不断发展更新中，对于暴露的接口不提供稳定性保障。在后续版本中，这些 API 可能会有较大的改变

简而言之，该功能暴露了 Flink 对 key state 的管理功能，用户可以在外部查询 Flink 程序运行过程中产生的状态。对于某些场景，状态可查询消除了与外部存储系统间的交互，这在某些情况下，往往是程序性能的瓶颈。除此之外，该功能对于程序调试也是非常有用的。

> 当查询状态对象时，该对象是并发访问的，无须同步和复制。这是一个设计方面的考虑，可以避免增大整个作业的延迟。由于使用 JVM heap 的状态后端，例如 HashMapStateBackend，不知道副本操作。当检索值而不是直接引用存储的值时，读-修改-写模式是不安全的，并且可能导致可查询状态服务器由于并发修改而失败。EmbeddedRocksDBStateBackend 可以避免这些问题。

### 架构设计
在开始展示如何查询状态时，简要描述组成它的实体很有用。可查询状态的功能主要由以下三个部分组成

- QueryableStateClient，（可能）在 Flink 集群外部运行并提交用户状态查询。
- QueryableStateClientProxy，在每一个 taskManager 上运行，负责接收客户端提交的查询，获取对应 taskManager 上的状态并返回给客户端。
- QueryableStateServer，在每一个 taskManager 上运行，对上面存储的状态进行管理服务。

客户端可以连接到任意其中一个代理，并发送请求查询某些 key （比如 k）关联的状态。正如 working_with_state 章节所说的一样，key state 是通过 key group 进行组织并分配在各个 taskManager 上的。为了获取 k 所在的 key group，代理会访问 jobManager，基于 jobManager 的响应。代理会查询对应 taskManager 上的 QueryableStateServer 获取 k 的状态，最后返回响应给客户端。

### 启动 Queryable State
在 Flink 集群中启动 Queryable State 功能，需要如下步骤：

1. 复制 Flink opt/ 目录下的 flink-queryable-state-runtime_2.11-1.14.3.jar 到 lib/。
2. 设置 queryable-state.enable 属性为 true。

如果 Flink 集群 taskManager 日志中存在 "Started the Queryable State Proxy Server @ ..." 日志，则说明启动成功。

### 如何使状态可查询
Flink 集群中已经启动了该功能，那么该如何使用呢？为了使状态对外可见，需要通过以下方式来声明：

- QueryableStateStream，接收上游传进来的状态值，生成可查询状态流，供后续查询。
- stateDescriptor.setQueryable(String queryableStateName)，这使得状态描述符表示的 Keyed State 可查询。

下面的章节都是介绍如何使用这两种方法。

#### Queryable State Stream
在 KeyedStream 上调用 .asQueryableState(stateName, stateDescriptor) 方法可以生成 QueryableStateStream 供状态查询使用，根据不同类型的状态，需要使用不同的 asQueryableState 方法。
```java
// ValueState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ValueStateDescriptor stateDescriptor)

// Shortcut for explicit ValueStateDescriptor variant
QueryableStateStream asQueryableState(String queryableStateName)

// ReducingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ReducingStateDescriptor stateDescriptor)
```
> 这里没有支持可查询的 ListState，因为该类型的状态一直在增长，list 中的元素得不到清理，会造成极大的内存资源消耗。

QueryableStateStream 可以被看做一个 sink，并且不能再次被转换。在内部，QueryableStateStream 被转换为一个 operator，该 operator 使用所有传入记录来更新可查询状态实例。更新逻辑取决于调用 asQueryableState() 时传入的 stateDescriptor 类型。比如像下面这样的程序中，状态就可以通过 ValueState.update(value) 来进行更新。

```java
stream.keyBy(value -> value.f0).asQueryableState("query-name")
```
这类似于 scala API 中的 flatMapWithState 方法。

#### Managed Keyed State
可以通过 StateDescriptor.setQueryable(String queryableStateName) 使 StateDescriptor 可查询，进而使该 operator 的 keyed state 可查询，如下例所示：

```java
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average", // the state name
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
descriptor.setQueryable("query-name"); // queryable state name
```

> queryableStateName 参数可以任意选择，仅用于查询。不必与状态名称相同。

这种方式对使用哪种状态类型没有限制，可以应用于  ValueState、 ReduceState、ListState、MapState、和 AggregatingState。


### 查询状态
到目前为止，Flink 集群已经可以提供状态查询服务，状态也已经变的可查询，接下来就可以了解如何查询状态。

为此，我们需要使用 QueryableStateClient 类，需要在 pom.xml 中引入此依赖。

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-core</artifactId>
  <version>1.14.3</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-queryable-state-client-java</artifactId>
  <version>1.14.3</version>
</dependency>
```

QueryableStateClient 会提交查询请求到内部的 QueryableStateClientProxy ，用于查询状态并返回最终结果。唯一要
注意的点是我们需要给客户端指定 taskManager 的 hostname （上面已经提到过每个 taskManager 都会启动运行对应的代理服务）和代理监听的端口。关于如何配置该代理服务及其端口可以查看 [Configuration Section](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/fault-tolerance/queryable_state/#configuration)

```java 
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);
```

客户端就绪后，比如查询 k 关联的的类型为 v 的状态，可以调用该方法：
```java
CompletableFuture<S> getKvState(
    JobID jobId,
    String queryableStateName,
    K key,
    TypeInformation<K> keyTypeInfo,
    StateDescriptor<S, V> stateDescriptor)
```
最终返回的 CompletableFuture 包含要查询的状态值。传入的 jobId 参数用来标识该状态属于哪个作业。参数 key 是待查询的 key 值，keyTypeInfo 用来确定 key 的类型以及 Flink 该如何序列化/反序列化该 key。stateDescriptor 中包含状态的类型（Value，Reduce 等等）以及该如何序列化/反序列化的信息。

细心的读者会注意到，返回的 CompletableFuture 是一个类型为 S 的值：即包含实际状态值的 State Object。可以是 Flink 支持的任意状态类型：ValueState, ReduceState, ListState, MapState, and AggregatingState。

> Note: 返回的状态对象中包含的状态不允许被修改。我们可以使用该对象获取真正的状态值，例如 `ValueState.get()`，或者返回包含 <K,V> 状态对的可迭代对象，例如 `mapState.entries()`。我们只是不能修改这些状态，举个例子，在返回的 list state 对象上调用 add() 方法会抛出 UnsupportedOperationException。

> Note: 状态客户端是异步的，且可以被多个线程共享。在想要关闭的时候必须使用 QueryableStateClient.shutdown() 方法来释放资源。

### 示例
下面是一个是状态可查询的例子
```java
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    private transient ValueState<Tuple2<Long, Long>> sum; // a tuple containing the count and the sum

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
        Tuple2<Long, Long> currentSum = sum.value();
        currentSum.f0 += 1;
        currentSum.f1 += input.f1;
        sum.update(currentSum);

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
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
        descriptor.setQueryable("query-name");
        sum = getRuntimeContext().getState(descriptor);
    }
}

```
当包含以上逻辑的作业启动后，就可以通过 jobId 查询 sum 关联的状态。

```java 
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);

// the state descriptor of the state to be fetched.
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
          "average",
          TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

CompletableFuture<ValueState<Tuple2<Long, Long>>> resultFuture =
        client.getKvState(jobId, "query-name", key, BasicTypeInfo.LONG_TYPE_INFO, descriptor);

// now handle the returned value
resultFuture.thenAccept(response -> {
        try {
            Tuple2<Long, Long> res = response.get();
        } catch (Exception e) {
            e.printStackTrace();
        }
});
```
### 配置
以下是 state server 和 state client 相关的一些配置，可以通过 QueryableStateOptions 进行定义。

#### State Server

- queryable-state.server.ports：state server 的端口范围。如果多个 taskManager 运行在同一台机器上，可以用来避免端口冲突。这个参数值可以被设置为多种形式，比如一个端口号："9123"，或者端口号的范围："50100-50200"，或者端口列表：“50100-50200,50300-50400,51234”。默认端口是 "9067"。
- queryable-state.server.network-threads：state server 处理查询请求的网络线程数。
- queryable-state.server.query-threads：state server 处理查询请求的查询线程数。

#### Proxy

- queryable-state.proxy.ports：state proxey 的端口范围。如果多个 taskManager 运行在同一台机器上，可以用来避免端口冲突。这个参数值可以被设置为多种形式，比如一个端口号："9123"，或者端口号的范围："50100-50200"，或者端口列表：“50100-50200,50300-50400,51234”。默认端口是 "9069"。
- queryable-state.proxy.network-threads：state proxey 处理查询请求的网络线程数。
- queryable-state.proxy.query-threads：state proxey 处理查询请求的查询线程数。


### 局限

- 可查询状态的生命周期和运行的作业是绑定在一起的，在 task 启动时注册，task 完成时销毁。在未来的版本中，期望是在某个 task 结束后仍然可以查询其绑定的状态，并通过状态副本机制加速作业的恢复。
- 当天 KvState 的通知（个人理解是和 server，proxey 等的交互）比较简单。未来期望建立更健壮的反馈机制。
- 服务器和客户端跟踪查询的统计信息功能在默认情况下是被禁用的，期望在未来可以通过 metrics 进行暴露。


### 思考
暂无
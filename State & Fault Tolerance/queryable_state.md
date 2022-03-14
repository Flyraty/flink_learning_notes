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
最终返回的 CompletableFuture 包含要查询的状态值。传入的 jobId 参数

> Note:   

> Note:

### 示例

### 配置

### 局限

### 思考
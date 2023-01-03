### Overview
Flink 常常用于实时流处理，其特点是长时间运行。和其它长时间运行的服务类似，我们也需要更新程序以应对不断变化的需求。对存在于应用程序中的 schema 信息也是如此，其会随着应用程序的更新而演化发展。

本章节主要讲解如何更新升级状态的 schema 信息。对于不同的状态结构和类型，其实现会有所差异。需要注意的是仅当您使用由 Flink 自己的类型序列化框架生成的状态序列化程序时，此页面上的信息才相关。也就是说，在声明状态时，提供的状态描述符未配置为使用特定的 TypeSerializer 或 TypeInformation，在这种情况下，Flink 会推断有关状态类型的信息：

```java 
ListStateDescriptor<MyPojoType> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        MyPojoType.class);

checkpointedState = getRuntimeContext().getListState(descriptor);
```
本质上，状态 schema 是否可以自动推断取决于用于读取/写入持久化状态的程序。简单来说，一个状态的 schema 只有在使用正确的序列化时才可以自动推断演化。这由 Flink 的类型序列化框架生成的序列化程序透明地处理。

如果您打算为您的状态类型实现自定义 TypeSerializer，并且想了解如何实现序列化器以支持状态模式演化，可以查看[Custom State Serialization](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/datastream/fault-tolerance/serialization/custom_serialization/)

### 状态 schema 的演化




### 思考

**1. 状态的演化是啥含义？状态的 schema 为啥会发生改变？**
### 状态后端
Flink 提供给了不同的状态后端用于指定状态的存储方式和位置。

状态可以存储在 Java 的堆内或者堆外内存当中。根据所使用的状态后端，Flink 可以对应管理程序的状态，这意味着 Flink 可以进行内存管理（在内存放不下时可以溢写的磁盘），以保证内存中可以放下较大的状态。默认情况下，flink_conf.yml 文件中的配置决定了所有作业该使用何种状态后端。

当然，每个作业的配置可以覆盖掉全局的配置，就像下面这样。

如果想要了解更多可用的状态后端，它们的优点，局限性，可用的配置参数等等，可以查看 [Deployment & Operations](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/ops/state/state_backends/) 章节。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(...);
```
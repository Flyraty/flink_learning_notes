### 基础概念
Flink 是什么？Flink 是一种支持有状态计算的流式数据处理引擎，并且提供了批数据的处理能力。

### Flink’s APIs

Flink 针对流/批程序的处理提供了不同层次的 API 抽象。

<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/levels_of_abstraction.svg" width="50%" height="50%"></div>

- 最底层的 api 抽象是 **stateful and timely stream processing**，通过 DataStream API 中的的 process function 接口向外暴露。它允许用户自由处理来自一个或多个流的事件，并且提供一致性，高容错的状态管理。除此之外，用户可以注册事件事件并且处理基于时间的事件回调，从而实现更复杂的数据计算。

- 在实践中，许多应用程序并用不到上述描述的底层 api，而是采用更上层的 DataStream API 和 DataSet API 分别处理有界和无界数据流。这些 API 提供了通用的接口来处理数据，像用户指定的转换、连接、聚合、窗口、状态等操作，和 Spark 一样，在该层，可以比较轻便的编写一个数据处理的 pipeline。
```java
datastream
	.flatMap(new FlatMapFunction<String, Tuple2<String, Integer>>() {...})
	.keyBy(0)
	.reduce(...)
``` 
其中需要注意的是数据类型的处理，在后面的 Data Types 中会讲到。由于 process function 已经和 DataStream API 集成，因此可以比较方便的使用底层 api 处理，比如状态管理与注册。对于批处理，Dataset API 也提供了更多的原语操作，比如 loop/iterations。

- Table API 是基于表格的声明式 DSL，类似于 Spark 的 DataSet API。它遵循关系模型，每张表格都有自己的 schema，并且和关系数据库一样，提供了 select、join、group by 等基本查询。Table API 并不实现具体代码的逻辑，而是告诉程序该做什么，实际上就是屏蔽掉了 map，reduce function 的实现，只声明查询，聚合等操作。尽管 Table API 提供了 udf 的能力，使用起来也相对简洁，但是抽象和表现能力不如  DataStream API。除此之外，Table API 在真正分布到机器上去执行时，会经过解析，优化，将 RBO 规则应用到语法树上，从而得到相对最优的执行计划。
Table API 和 DataStream API 之间是可以互相转化的。

- 最高层 API 就是 SQL 了，在语义和表达式解析上都同 Table API 一样，但是不是 DSL，而是完全的 SQL 查询，就像下面这样：

	```sql
	CREATE TABLE employee_information (
	    emp_id INT,
	    name VARCHAR,
	    dept_id INT
	) WITH ( 
	    'connector' = 'filesystem',
	    'path' = '/path/to/something.csv',
	    'format' = 'csv'
	);

	select * form employee_information limit 1
	```


### 思考
**1.以 api 来看， Flink 和 Spark 有啥区别？**

单看 Flink 提供的 API 的话，和 Spark 是异曲同工。Flink 对 Dataflow Model 的实现更好，状态和检查点的管理集成的更好，但是流批 api 不统一，一般在做数据初始化的时候要开发两套程序，对于现在仍在大多数场景下使用的 lambda 架构不是很友好。Spark 贵在 api 统一，2.x 之后的 structed streaming 对于流处理的支持更上一层，但是状态的管理和自动 checkpoint 的支持不太好，并且目前 Spark 的开发重心在 Spark ML。

**2.Flink 和 Spark 分别适用于什么场景？**

就生产上来看的话，对于无状态的数据流处理，峰值打到 10000qps 的情况下。Flink 和 Spark streaming 的处理都是 hold 住的，时延都在可接受范围内，并且由于链路比较长，binlog 产生到收集就大概 15s，程序处理大概 5s，整个链路基本上控制在 30s 内。所以对于一些 etl 而非聚合指标任务来说，采用这两个都是没问题的。
对于有状态的数据流处理，Flink 显然集成更好，并且容错机制更完善。
目前国内在推 Flink，所以各家基本上都在从 Spark 切换到 Flink，并且基于 Flink 搭建内部实时开发平台。

**3.用的最多的 api 是什么？**

个人理解是 SQL，SQL 作为一种早就普及的标准化语言，使用起来成本低，SQL+UDF一般也能覆盖到 %80 的业务场景，大多数解析计算都是无状态的，使用 SQL API 开发更便捷，更高效，尤其是内部基建比较好的公司，通过统一的实时开发平台开发这种 etl 程序，熟悉之后，几分钟之内就可以上线一个实时流任务，将 kafka 中数据写入到下游。


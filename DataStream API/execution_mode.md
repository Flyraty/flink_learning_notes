### 执行模式（流/批）
Datastream API 支持流/批两种运行模式。最常见的就是实时数据流处理，即 STREAMING mode。除此之外，Flink 在 Datastream API 上也实现了 BATCH mode，类似 MR，Spark 的批处理，这适用于有界数据流处理，常用于状态初始化。

在 Datastream API 上统一流/批处理意味着对于相同的有界输入，不管使用何种模式，程序返回的最终结果是一致的，只不过可能底层执行过程有所不同。

使用 BATCH mode 时，Flink 对于有界数据流的处理会采用不同的执行策略（比如 join ），相比流处理，批处理的调度和任务恢复会更简单高效，这里可以联想下 Spark 的批处理。

### 何时使用 Datastream 的批执行模式
BATCH mode 只能用于有界数据流处理。而 STREAMING mode 可以处理有界和无界数据流。通常，我们使用批模式来处理有界数据，因为这种方式更加高效。

有些时候，我们需要初始化无界数据流的状态。此时，可以先采用 STREAMING mode 处理有界数据，生成 savepoint。在通过 savepoint 启动无界数据流任务，从而完成任务状态的衔接。

### 配置批执行模式
执行模式可以通过 execution.runtime-mode 设置进行配置，存在以下三种取值：
- STREAMING：Datastream 默认模式
- BATCH - Datastream 批模式
- AUTOMATIC - 根据数据源的有界性自动判断使用何种运行模式。

也可以通过 bin/flink run ... 的命令行参数进行配置，或者在创建 StreamExecutionEnvironment 以编程方式进行配置。

命令行提交任务时配置
```sh
$ bin/flink run -Dexecution.runtime-mode=BATCH examples/streaming/WordCount.jar
```
在 env 中配置
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setRuntimeMode(RuntimeExecutionMode.BATCH);
```

> 建议在命令行中配置，因为这样程序拓展性更好，避免了同一套逻辑在流/批之间的切换。

### 流/批下的 Execution Behavior
本小节主要讲下 Datastream API 在流/批模式下的一些不同。

#### 任务调度和 shuffle
Flink 程序包含多个 operator，这些 operator 从前往后按照某种模式连接起来就构成了数据处理的 pipeline。那么如何在多个进程乃至机器之间调度这些 operator？各个 operator 间又是怎样进行数据交换的呢？

在执行过程中，许多 operator 可以 chain 在一起，旨在降低资源的过多使用，提高性能。一个 operator 或者多个 chain 在一起的 operator组成 Flink 程序的最小调度单元。

任务调度和网络数据交换在流/批模式下是不同的，对于有界数据流的处理，可以使用更多的数据结构和策略，因而相比 STREAMING mode 处理有界数据流更高效。下面会使用一个例子说明这些不同点。
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStreamSource<String> source = env.fromElements(...);

source.name("source")
	.map(...).name("map1")
	.map(...).name("map2")
	.rebalance()
	.map(...).name("map3")
	.map(...).name("map4")
	.keyBy((value) -> value)
	.map(...).name("map5")
	.map(...).name("map6")
	.sinkTo(...).name("sink");
```
对于一对一连接的 operator，像 map()、filter() 这些可以直接将数据发往下游 operator，因此这些 operator 可以 chain 在一起。这也意味着这些 operator 间不会产生 shuffle。而像 keyBy()、reblance() 这些 operator，往往需要将数据发往不到不同的节点（这取决于任务的本地化级别），从而会产生网络数据交换。

上面的示例会生成三个 task
- Task1: source、map1、map2
- Task2: map3、 map4
- Task3: map5、map6、sink

在 task1 和 taks2，task2 和 task3 之间会产生网络 shuffle。下面是该 job 逻辑图的简单描述

<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/datastream-example-job-graph.svg" width="80%" height="80%"></div>

##### 流模式
在流模式下，所有的任务都要一直运行，这使得 Flink 程序一接到数据就可以流经整条 pipeline 快速处理。这也意味着 TaskManager 需要分配足够的资源保证同时启动所有的 task 并可以正常运行。
数据交换也是基于 pipeline 的，数据会被立刻发往下游，只不过在网络侧会存在缓冲区，旨在提高吞吐量。在流式处理中，shuffle 的中间结果是不会落盘的。

##### 批模式
在批模式下，任务会被划分成几个阶段，从前往后逐一实行（当然，毫不相关的前置任务也是可以并行执行的）。我们能做到这些是因为数据流是有界的，我们可以处理完一个阶段的所有数据后在接着处理。像上面的例子会被划分成三个 stage，这里 stage 的划分和 Spark 是一样的，碰到 shuffle dependency 就断开 pipeline。

不同于流模式下直接发送数据，批模式会先 shuffle write 将数据落盘，下一阶段在进行 shuffle read 读取上游落盘后的数据。这虽然会增大处理延迟，但是对于批处理是有益的。
- 任务失败时，不必重启所有的 task，而是从上次缓存的中间结果处开始。
- 更少的资源使用，不必一开始就启动所有的 task。

TaskManager 会保留中间结果，直到下游 task 开始读取。并且 TM 会在空间允许的情况下一直保留该中间结果，方便某个阶段任务失败重启。

#### 状态管理
在流模式下，Flink 通过状态后端来控制状态的存储和 checkpoint 的生成。

在批模式下，配置的状态后端被忽略。数据流可以按照 key 被顺序处理，因此同一时刻只有一个 key 的状态，在处理下一个 key 时，上一个 key 的状态会被清理掉。

#### 处理顺序
在流模式下，不考虑数据的顺序，在数据到达后就会根据程序逻辑尽可能快的处理。

在批模式下，有些 operator 的操作是可以保证顺序的，这取决于系统执行某些操作时的策略，也可能是 shuffle，任务调度的副作用。

一般分为三种输入，批模式下会按照如下顺序处理
- broadcast input: input from a broadcast stream。
- regular input: input that is neither broadcast nor keyed。
- keyed input: input from a KeyedStream。

#### 事件时间/ watermark
Flink 使用 watermark 机制来处理无序数据流。在批模式下，数据流是有界的，我们可以确定某些时间前的数据都已经到达了，因此可以直接指定一个最大的 watermark 值，计算会在最后被触发，注册的 WatermarkAssigners 和 WatermarkGenerators 会被忽略。

#### 处理时间
处理时间是程序处理数据那一刻的机器时间，基于此机制的计算是无法保证幂等的，因此同样的数据在两次处理过程中的处理时间可能是不同的。

尽管如此，在流模式下使用处理时间仍然是有些场景的。在数据延迟乱序可控的情况下，处理时间的一小时相当于事件时间的一小时，使用处理时间计算会更快的被触发。

在批模式下，和事件时间的处理是一样的，在最后的输入后触发整体的计算。

#### 任务恢复
在流模式下，Flink 使用 checkpoint 实现容错管理，任务失败后，会从 checkpoint 恢复并重新启动调度所有的 task 。 

在批模式下，任务失败后，Flink 会尝试回溯到上一个阶段重新开始，默认只有失败的上游 task 会被重启，从而减少任务恢复时间。

### 思考

**1. Datastream 在批模式下和 Dataset 有啥区别呢？**




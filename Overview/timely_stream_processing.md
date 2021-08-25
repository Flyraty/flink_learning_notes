### 即席流处理
这里翻译成了即席流处理，借鉴了即席查询的概念，意味着 Flink 会尽可能快的，以低延迟的方式来处理源源不断的数据。引入时间概念的话，就是 Flink 会按照给定的时间，在触发给定的规则（比如窗口）后尽可能快的处理数据。
emmn，好像跟下面讲的没啥关系。

### 事件时间 vs 处理时间 
当接触到流数据处理时，其中涉及到的时间概念是不同的（比如说定义窗口的时间）。
- **Processing time：** 处理时间是指正在执行相应操作的机器上的系统时间。当程序使用处理时间时，所有基于时间处理的操作（例如时间窗口）都将基于各个 operator 运行所在的机器的系统时钟。比如，一个小时级别的窗口，程序在 9:15 分开始运行，则该时间窗口内，将会处理 9:15~10:00 间到达 operator 的数据，下一个时间窗口将会处理 10:00~11:00 间的数据。处理时间是最简单的时间概念，不需要数据流与机器之间的协调，它提供了更高的性能和低延时的保证，但是在一个分布式执行环境中，这样带来了幂等性问题，同一批数据经过相同程序计算后的结果是不一样的，并且结果容易受到处理速度及 task 调度的影响。

- **Event Time：**事件时间是事件真实产生的时间，它经常会直接嵌套在记录中进入 Flink 系统，并且每条记录都可以被提取出事件产生的时间戳供 Flink 使用。使用事件时间时，数据流的处理取决于数据本身，与外部时钟无关。在基于事件时间数据流中，程序必须指明如何生成 watermark，从而触发计算。

	在理想情况下，基于 Event Time 的流处理不管数据何时到达还是乱序进入，都能保证一致确定的结果。然后，在真实世界中，往往存在乱序数据，在等待乱序数据的过程中就会产生延迟，而程序又不能无限等待，由此造成的结果是数据的处理并不是完全正确的。

	假设数据都到达了，基于事件时间的操作会如期进行，即使存在迟到数据，重新回溯历史数据的情况，程序也能产生正确幂等的结果。例如，一个基于事件时间的小时窗口中会包含所有 event timestamp 在这个小时内的数据，不管数据乱序还是处理时间超出该小时范围。

	需要注意的是，有时候在使用 Event Time 处理实时流时，也会引入 Processing Time 的逻辑用来保证产生的数据可以被及时处理。
    <div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/event_processing_time.svg" width="70%" height="70%"></div>


### 事件时间和 watermark  
*Note：关于 event time 和 watermark 想了解更多的话，可以去查看 DataFlow Model 论文*  

基于 *event time* 的事件处理程序需要一种机制来监测事件的进度，从而知道何时该触发计算，何时该丢弃延迟的数据，何时当前时间窗口内不会在接收到新的数据。*event time* 是独立于处理时间的，例如，一条流中处理时间会大于事件时间，但是他们的速度是相同的，不存在互相等待。另外，一条流也可以在短时间回溯大量历史数据。

Flink 中管理事件进度的机制叫 **Watermarks**。Watermarks 作为数据流的一部分，携带者事件时间戳 t。*Watermark(t)* 代表 t 时间前的数据都已经到达，这意味着不会再有  t' <= t 的事件出现。

下图说明了一条流中 watermark 的产生，该流的数据是顺序到达的，意味着 watermark 只是简单的定时从事件时间中产生。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/stream_watermark_in_order.svg" width="70%" height="70%"></div>

Watermarks 在面对无序数据流时表现的很奇怪，时常会让我们感到疑惑，就像下图一样，事件乱序到达。一般来说，watermark 代表小于等于该时间的数据都已经到达，watermark 流到 operator 时，该 operator 就会更新自己的内部时钟到 watermark 处，不再接受其之前的数据。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/stream_watermark_out_of_order.svg" width="70%" height="70%"></div>

#### 并行数据流中的 watermark
Watermarks 在数据源或之后的一些操作生成。Source 端并行的 subtask 各自生成自己的 watermark，这些 watermark 定义了源数据中的事件时间。随着 watermark 在数据流中的流动，当到达下游 operator 时，该 operator 就会更新自己的内部时钟，并且生成一个新的 watermark 到下游。

对于存在多个输入流的 operator，需要进行 watermark 的对齐，operator 总会使用最小的 watermark 作为更新其内部时钟的基准。下图展示了 watermark 在并行数据流中的流动。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/parallel_streams_watermarks.svg" width="70%" height="70%"></div>


### 延迟  
Watermarks 的条件并不是完全正确的，在现实世界中，总会存在延迟数据，这意味着总会有 t' <= watermark 的事件在 watermark 被触发后到达，而通常由于整个数据链路的延迟原因，数据流又不可能无限的等待延迟数据。如果此时程序又要求极高准确性的话，那么就要处理延迟数据，可以阅读 [ Allowed Lateness](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/datastream/operators/windows/#allowed-lateness) 了解更多信息。

### 窗口
实时数据处理与批处理在聚合操作上的处理是完全不同的，因为实时流是无界的，不可能等待所有的数据都到达后在做聚合，因此 Flink 引入了窗口的概念，比如聚合刚过去的 5min 发生的事件。

窗口可以由时间驱动和事件驱动，常见的窗口类型有 固定窗口（没有重叠）, 滑动窗口（存在重叠）, 会话窗口 (活跃时间间隔划分）.

### 思考  
**1.barriers 和 watermark 都随着数据流移动，这些是怎么实现的呢？**

**2.Watermarks 的主要作用？**

触发计算，限制状态的无限增长。事件不可能无限的等待下去，需要在准确性和延时下做好衡量。

**3.Watermarks 如何配置？**

[这一次带你彻底搞懂 Flink Watermark](https://cloud.tencent.com/developer/article/1629585)

**4.延迟数据的处理？** 

对于无状态的计算，比较容易，可以先旁路输出到延迟队列，在做处理。对于有状态的计算呢？

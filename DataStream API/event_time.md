### WatermarkStrategy
为了程序能够基于事件时间处理，Flink 需要准确知道事件发生的时间戳，这意味着数据流中的每个元素需要注册自己的事件时间戳。一般会使用 TimestampAssigner 从数据中的某些字段来提取事件时间。

事件时间的提取影响 watermark 的生成（watermark 代表着事件处理的进度）。我们可以使用 WatermarkGenerator 来生成 watermark。

emmn，貌似直接看上面两句话根本不知道再讲啥。但是 WatermarkStrategy 就是由 TimestampAssigner 和 WatermarkGenerator 组成的。watermark 策略暴露出来的接口就是实现以上两个方法。Flink 提供了内置的 WatermarkStrategy，也允许自定义。

```java
public interface WatermarkStrategy<T> 
    extends TimestampAssignerSupplier<T>,
            WatermarkGeneratorSupplier<T>{

    /**
     * Instantiates a {@link TimestampAssigner} for assigning timestamps according to this
     * strategy.
     */
    @Override
    TimestampAssigner<T> createTimestampAssigner(TimestampAssignerSupplier.Context context);

    /**
     * Instantiates a WatermarkGenerator that generates watermarks according to this strategy.
     */
    @Override
    WatermarkGenerator<T> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context);
}

```
正如上面提到的，很多时候我们不需要自己实现这个接口，只需要调用内部策略。比如下面这样，使用有界无序的 watermark + 基于 lambda 函数实现的时间注册器。

```java
WatermarkStrategy
        .<Tuple2<Long, String>>forBoundedOutOfOrderness(Duration.ofSeconds(20))
        .withTimestampAssigner((event, timestamp) -> event.f0);
```

watermark 可以在读取数据源或者进行后续某些操作后注册。通常我们会选择在读取数据源时注册，这样产生的 watermark 更加准确，也会考虑到初始分区，分片的影响。直接在源端指定 WatermarkStrategy 有时候需要在数据源实现一些特殊的接口，比如 Kafka，这个在下面 WatermarkStrategy && Kafka Connector 会提到。


### 处理空闲数据源
在实时数据流处理过程中，数据源往往是经过分区分片的，比如 Kafka，某个分区在一段时间内没有写入数据，这个分区就是一个空闲数据源，读取该分区的 task 在这段时间内是空跑的。空闲数据源带来一个问题，就是 watermark 也不会随之更新。此时，如果其他分区仍然有数据写入，对于下游有多个输入的 operator 来说，watermark 仍然停留在没有数据写入的分区最后一次产生的时间，不会处理其它分区后续写入的数据。

为了解决上述问题，我们需要将某段时间内没有写入的数据源分区标记为空闲，在 watermark 传往下游时，空闲数据源的 watermark 暂时不参与计算。WatermarkStrategy 提供了方便的方法来做这件事情。

```java
// 如果超过 1min 没有写入数据，就标记为空闲
WatermarkStrategy
        .<Tuple2<Long, String>>forBoundedOutOfOrderness(Duration.ofSeconds(20))
        .withIdleness(Duration.ofMinutes(1));
```

### WatermarkGenerators
TimestampAssigner 方法比较简单，就是从事件数据某些字段中提取转换出事件发生的时间戳，我们往往不需要关注太多。WatermarkGenerator 相对比较复杂，下面是其定义的接口

```java
/**
 * The {@code WatermarkGenerator} generates watermarks either based on events or
 * periodically (in a fixed interval).
 *
 * <p><b>Note:</b> This WatermarkGenerator subsumes the previous distinction between the
 * {@code AssignerWithPunctuatedWatermarks} and the {@code AssignerWithPeriodicWatermarks}.
 */
@Public
public interface WatermarkGenerator<T> {

    /**
     * Called for every event, allows the watermark generator to examine 
     * and remember the event timestamps, or to emit a watermark based on
     * the event itself.
     */
    void onEvent(T event, long eventTimestamp, WatermarkOutput output);

    /**
     * Called periodically, and might emit a new watermark, or not.
     *
     * <p>The interval in which this method is called and Watermarks 
     * are generated depends on {@link ExecutionConfig#getAutoWatermarkInterval()}.
     */
    void onPeriodicEmit(WatermarkOutput output);
}
```
watermark 生成有两种方式：
- periodic watermark 基于定时，定时生成会通过 onEvent() 获取每条数据的时间作为待选的 watermark。在框架调用 onPeriodicEmit() 时生成最终的 watermark。
- punctuated watermark 基于规则（某些特殊事件触发），规则生成也是通过 onEvent() ，在获取到某些特殊事件后，直接生成 watermark，而不通过 onPeriodicEmit()。


#### 基于定时的实现
定时生成器会监听到来的事件，定时触发 watermark 的生成（基于事件时间或者单纯的处理时间都可以）。定时触发的时间间隔可以通过 ExecutionConfig.setAutoWatermarkInterval(...) 来设置，到达时间触发时会调用 onPeriodicEmit() 方法，只要这次的 watermark 不为 null 并且大于上次生成的 watermark，就会成功生成。下面是两个例子：

```java
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
public class BoundedOutOfOrdernessGenerator implements WatermarkGenerator<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTimestamp);
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // emit the watermark as current highest timestamp minus the out-of-orderness bound
        output.emitWatermark(new Watermark(currentMaxTimestamp - maxOutOfOrderness - 1));
    }

}

/**
 * This generator generates watermarks that are lagging behind processing time 
 * by a fixed amount. It assumes that elements arrive in Flink after a bounded delay.
 */
public class TimeLagWatermarkGenerator implements WatermarkGenerator<MyEvent> {

    private final long maxTimeLag = 5000; // 5 seconds

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        // don't need to do anything because we work on processing time
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        output.emitWatermark(new Watermark(System.currentTimeMillis() - maxTimeLag));
    }
}
```

#### 基于规则的实现
基于规则的生成就是当遇到某些特殊事件时才会生成 watermark。
```java
public class PunctuatedAssigner implements WatermarkGenerator<MyEvent> {

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        if (event.hasWatermarkMarker()) {
            output.emitWatermark(new Watermark(event.getWatermarkTimestamp()));
        }
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // don't need to do anything because we emit in reaction to events above
    }
}
```
使用规则生成需要主要，有些场景下，连续到来的每个事件可能都会触发 watermark，由此产生过多的 watermark，造成下游的频繁计算，降低性能。

### WatermarkStrategy && Kafka Connector

Kafak 是分区内有序，全局乱序的，并且一般都是多线程消费。所以 watermark 的流转和 shuffle 是一样的，可以直接看下面的例子
```java
FlinkKafkaConsumer<MyType> kafkaSource = new FlinkKafkaConsumer<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(
        WatermarkStrategy.
                .forBoundedOutOfOrderness(Duration.ofSeconds(20)));

DataStream<MyType> stream = env.addSource(kafkaSource);
```

<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/parallel_kafka_watermarks.svg" width="70%" height="70%"></div>

### operator 是如何处理 watermark 的
operator 在处理完由 watermark 触发的所有计算后，才会将 watermark 继续发往下游。同样的规则适用于 TwoInputStreamOperator，只不过此时发出去的 watermark 是多个输入中最小的。更多的细节可以查看源码中 OneInputStreamOperator#processWatermark, TwoInputStreamOperator#processWatermark1 和 TwoInputStreamOperator#processWatermark2 方法.

### 内置的 WatermarkGenerators

#### 单调递增生成 watermark、
单调递增即数据的到来是有序并且是增序的，这意味着当前的时间戳就可以当做 watermark，因为不会存在迟到数据。
```java
WatermarkStrategy.forMonotonousTimestamps();
```

#### 固定延迟生成 watermark
使用这种方式有个前提，就是需要提前预估好数据流的延迟，否则数据容易产生误差。固定延迟意味着每次生成 watermark 都会往前倒退一段时间作为最终的 watermark 发往下游，超出固定延迟的数据将被丢弃，不会触发计算。
```java
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10));

```

### 思考
**1. 自己分别实现两种 watermark 的生成，观察一下 watermark 是如何触发计算的？**

可以参考[测试代码]()

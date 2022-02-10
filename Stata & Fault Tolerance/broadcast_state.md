### The Broadcast State Pattern
本节主要用来学习如何在实践中应用状态广播。需要预先了解下 [Stateful Stream Processing](https://timemachine.icu/flink_learning_notes/Overview/stateful_stream_processing.html) 相关的概念。

### Provided APIs  
为了讲解相关的 API，我们会从一个例子开始，逐步展示 Broadcast State 的相关作用和功能。假设存在一个 item 流，里面的数据具有不同的颜色和形状属性，我们希望找到符合某种特定模式的 item 对，比如一个矩形后跟一个三角形。而这种特定模式又会随着时间而演变。

在这个例子中，第一条流包含具有颜色和形状属性的 item，第二条流中包含特定的匹配模式，即规则。

从 Items 流开始，因为我们需要相同颜色的成对数据，故需要按照颜色 keyBy 分组，以保证相同颜色的数据流向同一个 task。

```java
// key the items by color
KeyedStream<Item, Color> colorPartitionedStream = itemStream
                        .keyBy(new KeySelector<Item, Color>(){...});
```
接下来看规则流（即 pattern 模式流），该流中包含的数据需要被广播到所有下游 task 上，并在本地存储它们（实际上是在内存）以保证规则可以应用到源源不断的 items 流上。大概的步骤是这样
- 广播规则，每个并行示例存储规则到状态中。
- 通过创建 MapStateDescriptor 来访问本地已经存储的规则状态。
```java
// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
			"RulesBroadcastState",
			BasicTypeInfo.STRING_TYPE_INFO,
			TypeInformation.of(new TypeHint<Rule>() {}));
		
// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);
```
最后，为了在 Items 流上应用广播的规则，我们需要
1. 连接两条流，即 connect stream
2. 定义我们自己的规则匹配逻辑

连接两条流的时候只能在非广播流上（在上面的例子中就是 Items 流）调用 connect 方法，这会返回一个 BroadcastConnectedStream 对象，我们可以通过该对象调用 process 函数，从而实现规则匹配逻辑，process 函数底层调用的类型取决于非广播流的的类型。
- KeyedStream -> KeyedBroadcastProcessFunction
- none-KeyedStream -> BroadcastProcessFunction

```java
DataStream<String> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // type arguments in our KeyedBroadcastProcessFunction represent: 
                     //   1. the key of the keyed stream
                     //   2. the type of elements in the non-broadcast side
                     //   3. the type of elements in the broadcast side
                     //   4. the type of the result, here a string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // my matching logic
                     }
                 );

```
### BroadcastProcessFunction and KeyedBroadcastProcessFunction 
和 CoProcessFunction 一样，广播状态相关的两个 processFunction 也实现了两个方法。processBroadcastElement() 用来处理规则流，processElement() 用来处理事件流，下面是这两个方法的签名：
```java
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}
```

```java
public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
```

两个 process 方法的不同之处在于 processElement 是 ReadOnlyContext，而 processBroadcastElement 是 Context。

### Important Considerations

### 思考

**1. Broadcast State 的应用场景？**
规则匹配，适用于低吞吐流和高吞吐流之间的 join 操作。

**2. processElement 只有状态读权限，为什么这样设计？什么场景下会存在状态的误操作？**
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

上下文的作用一般如下 （以下简称 ctx）
- 访问广播状态，即 broadcast state。ctx.getBroadcastState(MapStateDescriptor<K, V> stateDescriptor)
- 获取数据的事件时间：ctx.timestamp()
- 获取当前的 watermark：ctx.currentWatermark()
- 获取当前的处理时间： ctx.currentProcessingTime()
- 旁路输出数据：ctx.output(OutputTag<X> outputTag, X value)

getBroadcastState() 获取到的状态实际上就是广播状态，在上述例子中是 ruleStateDescriptor。

广播流具有对广播状态的读写权限，而非广播流只有读权限。这样做的原因是在应用广播状态时，Flink 没有跨任务通信。为了保证在各个并行实例间广播状态的一致性，我们只为广播端提供读写访问权限，它在所有任务中看到相同的元素，并且我们要求该端每个传入元素的计算在所有任务中都是相同的。如果忽略这条规则的话，就会破坏状态的一致性保证，进而造成难以调试复现的结果。

>> 在 processBroadcastElement() 中实现的逻辑必须在所有并行实例中具有相同的确定性行为，即不管并发多少，相同元素执行的逻辑结果是确定不变的。

由于 KeyedBroadcastProcessFunction 应用在 KeyedStream 上，相比 BroadcastProcessFunction 多了一些功能。
1. processElement 中的 ReadOnlyContext 允许访问 Flink 的底层计时器服务，用来注册基于事件时间和处理时间的计时器。当计时器被触发后，会调用 onTimer 方法。OnTimerContext 和 ReadOnlyContext plus 功能是一样的。
 - 查询被触发的计时器是基于事件时间还是处理时间。
 - 查询与计时器关联的 key。

2. processBroadcastElement() 方法中的 Context 包含方法 applyToKeyedState(StateDescriptor<S, VS> stateDescriptor, KeyedStateFunction<KS, S> function)。


最后回到我们最初的例子，KeyedBroadcastProcessFunction 大概就像下面这样 
```java
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
        new MapStateDescriptor<>(
            "items",
            BasicTypeInfo.STRING_TYPE_INFO,
            new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
            "RulesBroadcastState",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Rule>() {}));

    @Override
    public void processBroadcastElement(Rule value,
                                        Context ctx,
                                        Collector<String> out) throws Exception {
        ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
    }

    @Override
    public void processElement(Item value,
                               ReadOnlyContext ctx,
                               Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry :
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
    }
}
```
### Important Considerations

在描述了所提供的 API 之后，本节将重点介绍使用广播状态时要记住的重要事项。

- **没有跨任务通信**：正如前面所描述的一样，只有 BroadcastProcessFunction 可以修改更新广播状态。此外，用户必须确保所有任务对每个传入元素都以相同的方式修改广播状态的内容。否则，不同的任务可能会有不同的内容，导致结果不一致。
- **并行 task 之间广播流的数据顺序可能不一样**
- **每个 task 都对自己所拥有的广播状态进行 checkpoint：**虽然每个 task 实例都拥有相同的广播状态，各个 task 依然各自对自己所拥有的广播状态进行 checkpoint，这样做的主要原因是为了防止 flink 程序失败从 checkpoint 恢复时全部读取相同状态文件造成的访问热点问题。这样做的问题同样显而易见，会造成状态大小的增加（依赖于任务的并行度）。Flink 保证在恢复/重新缩放时不会有重复和丢失的数据。在以相同或更小的并行度进行恢复的情况下，每个任务都会读取其检查点状态。扩大规模后，每个任务读取自己的状态，其余任务（p_new-p_old）以循环方式读取先前任务的检查点。
- **不支持 RocksDB 状态后端：**广播状态在运行时保存在内存中，并且应该预先进行内存预估。这同样适用于所有的 operator state。

### 思考

**1. Broadcast State 的应用场景？**
规则匹配，适用于低吞吐流和高吞吐流之间的 join 操作。

**2. 为什么只有 BroadcastProcessFunction 可以修改更新广播状态？一开始读了几遍，都没太看懂**
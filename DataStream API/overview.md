### DataStream API 
DataStream API 用来对数据流进行摄入，过滤，更新，聚合等一系列操作。数据来源可以是多种多样的（比如静态文件，消息队列，socket 流等）。数据流处理的结果流向 sink，比如写数据到文件或者输出到标准输出。Flink 程序跑在多种上下文环境中，standalone 或者嵌入在其它程序中。程序可以执行在一个本地 JVM 进程，也可以运行在拥有多台机器的集群上。

### 什么是 DataStream
DataStream API 得名于 Flink 内部实现的数据集合类-DataStream。你可以认为 DataStream 是一种可以包含重复数据的数据集合，这些数据可以是有界和无界的，其处理 api 是相同的。emmn，这里也说明 Flink 统一了流批 api，都可以使用 DataStream 来对数据进行操作。

DataStream 类似于 Java Collection，但是在一些核心方法上有所不同。DataStream 是不可变的，这意味着它一旦创建便不能更新，并且不能直接检查元素，只能依靠 DataStream API 提供的 operator 进行操作，这些 operator 也叫作 transformations。


### Flink 程序剖析

Flink 程序通常由以下几部分组成：
1. 创建执行环境，及 Execution Environment。
2. 从外部数据源加载或者直接创建初始数据。
3. 指定初始数据流后续的转换操作。
4. 指定 sink，即计算结果该输出到什么地方。
5. 触发程序的执行。

下面会详细的介绍这几步过程。DataStream 的核心类都可以在 ` org.apache.flink.streaming.api` 中找到。

*StreamExecutionEnvironment* 是 Flink 程序的入口，可以通过以下方法创建
```java
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(String host, int port, String... jarFiles)
```

通常，你只需要使用 `getExecutionEnvironment()`，Flink 会根据上下文创建合适的执行环境：如果你在 IDE 中运行，那么就会创建 local environment 用以在本机执行程序。如果你将程序打成 jar 包并且通过命令行提交，那么 Flink 集群管理器会执行 main 方法并调用 `getExecutionEnvironment()`，返回一个集群下的执行环境。

env 中的一些方法可以用来指定 source。如果要读取文件的话可以采用以下代码：
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.readTextFile("file:///path/to/file");
```
这会创建 DataStream，接下来就可以在上面应用一些转换操作，得到新的 DataStream。比如下面对每行数据转 int。
```java
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
```
一旦 DataStream 完成最终的操作，就可以 sink 到外部系统中。
```java
writeAsText(String path)

print()
```
在完成以上程序后，需要通过 `env.execuet()` 方法触发程序的执行。execute 是同步的，需要等待程序执行完成返回最终结果。如果你不想等待 job 的执行就退出，那么可以使用 `executeAysnc()` 方法，它会返回一个 jobClient，用以与程序通信。比如，下面的代码使用 executeAysnc 来实现 execute 一样的功能。
```java
final JobClient jobClient = env.executeAsync();

final JobExecutionResult jobExecutionResult = jobClient.getJobExecutionResult().get();
```
Flink 程序都是惰性执行的：当程序的 main 方法被执行时，数据的加载和转换并不是立刻发生的，而是先生成 job graph。当执行调用 execute 时，这些操作才被真正的执行。惰性执行可以使我们构建复杂程序作为一个完整单元执行。
### 示例
下面是一个 wordcount 程序，首先命令行启动 `nc -lk 9999`，在启动 Flink 程序，命令行输入单词，就会看到程序的 标准输出。 

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(value -> value.f0)
                .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
```  

### Data Sources
Source 是指程序能够从哪里读取数据，你可以使用 `StreamExecutionEnvironment.addSource(sourceFunction)` 来添加自定义数据源。Flink 内部已经定义了一些简单数据源，但是还是经常需要通过 SourceFunction 来自定义实现非并行数据源，通过 ParallelSourceFunction 和 RichParallelSourceFunction 来定义并行数据源。

从 StreamExecutionEnvironment 可以访多个预定义的数据源。
基于文件的：
- readTextFile(path)  
- readFile(fileInputFormat, path) 
- readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)

### Data Sinks

### 执行参数

#### 容错管理

#### 延迟控制

### 调试

#### Local Execution Environment  

#### Collection Data Sources

#### Iterator Data Sink

### 思考
1. 创建执行环境时的上下文指的是啥？
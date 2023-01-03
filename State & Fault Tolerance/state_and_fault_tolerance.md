### Overview
Apache Flink 以其特有的方式处理数据类型和序列化，包含自己的类型描述符、泛型类型提取和类型序列化框架。本文档描述了这些概念及其背后的基本原理。

### 支持的数据类型
Flink 对 DataStream 中的元素类型做了一些规则限制，这样做的目的是方便框架推断类型，以确定较优的执行策略。

下面是其支持的七类数据类型

**1. Java Tuples and Scala Case Classes**
**2. Java POJOs**
**3. Primitive Types**
**4. Regular Classes**
**5. Values**
**6. Hadoop Writables**
**7. Special Types**

#### Tuples and Case Classes 
Tuples 属于复合类型，由固定数量的不同或者相同类型的字段组成。Java 本身提供了 Tuple1 到 Tuple25 的实现类。Tuple 的元素组成可以是 Flink 的任意类型，也可以嵌套 Tuple。可以直接通过 tuple.f4 或者 tuple.getField(int position) 的方式来访问 Tuple 的元素。Tuple 索引下标是从 0 开始的，这点与 Scala 不同， Scala 是通过 ._1 进行访问。
```java
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(value -> value.f0);
```

#### POJOS
在满足以下要求的时候，Java 或者 Scala 类会被 Flink 识别为 POJO 类型。
- 该类必须是 public 的。
- 该类必须含有 public（公共的）的无参构造器。
- 该类的属性字段必须是 public 的，或者可以通过 getter 、setter 方法访问。举个例子，对于字段 foo 来说，其对应的 getter 和 setter 方法必须被命名为 getFoo() 和 setfoo()。
- 该类所拥有的属性字段类型必须有对应的序列化器。

POJO 通常用 PojoTypeInfo 表示，并用 PojoSerializer 序列化。例外情况是 POJO 实际上是 Avro 类型（Avro 特定记录）或作为 “Avro 反射类型” 生成的。在这种情况下，POJO 由 AvroTypeInfo 表示并使用 AvroSerializer 序列化。你也可以在需要的时候注册自定义的序列化器，更多信息可以查看 [Serialization](https://nightlies.apache.org/flink/flink-docs-stable/dev/types_serialization.html#serialization-of-pojo-types)。

Flink 会自动分析 POJO 的字段结构，在日常工作中，POJO 会比其他类型使用的频率更高且处理速度可能会更快。

你可以通过以 flink-test-utils 中的 org.apache.flink.types.PojoTestUtils#assertSerializedAsPojo() 方法来测试某个类是否属于 POJO。如果你还想确保不会使用 Kryo 序列化 POJO 的任何字段，请改用 assertSerializedAsPojoWithoutKryo()。

下面的代码展示了一个具有两个公共字段的 POJO 类

```java
public class WordWithCount {

    public String word;
    public int count;

    public WordWithCount() {}

    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy(value -> value.word);
```

#### 原语类型
Flink 支持 Java 和 Scala 的所有原语类型，比如 Integer，String，Double。

#### General Class Types
这里的 General Class Types 指的是泛型？

Flink 支持绝大多数的 Java 和 Scala 类（不管是 API 提供还是自定义），除了一些包含不能序列化的字段的类，比如文件指针，io 流以及一些其它计算机本地资源。遵循 Java Beans 的类型通常会运行良好。

Flink 将所有未标识为 POJO 类型（参见上面的 POJO 要求）的类作为通用类类型进行处理。Flink 将这些数据类型视为黑盒，并且不访问它们的内容（例如，为了高效排序）。General Class Types 一般使用 kyro 进行序列化/反序列化。

#### Values 

Value 类型需要手动定义描述序列化和反序列化信息。Value 没有通过通用序列化框架，而是通过使用 read 和 write 方法实现 org.apache.flink.types.Value 接口，为序列化操作提供自定义代码。使用 Value Type 的原因往往是通用序列化带来的性能低下的问题。比如一个稀疏元素的数组，我们在知道这个数组大多数都是空值的情况下，可以对空值元素使用特殊的编码方式，而通用的序列化框架往往直接把整个数组的元素进行写入与读取。

org.apache.flink.types.CopyableValue 接口以类似的方式支持内部复制逻辑。

Flink 的基本类型自带预定义的 Value 类型（ByteValue, ShortValue, IntValue, LongValue, FloatValue, DoubleValue, StringValue, CharValue, BooleanValue）。这些 Value 类型充当基本数据类型的可变变体，它们的值可以改变、也允许复用已减少 GC 压力。

#### Hadoop Writables
你可以使用实现 org.apache.hadoop.Writable 接口的类型。 write() 和 readFields() 方法中定义的序列化逻辑将用于序列化。

#### 特殊类型
你可以使用特殊类型，比如 Scala 中的 Either、Option、Try 类型。Java 本身有自己的 Either 实现，和 Scala 中的 Either 类似，其提供了两种可能的数据返回，left 或者 right。Either 常用于错误处理以及需要输出不同类型记录的场景。

#### 类型擦除 & 类型推断

> 需要注意该小节仅针对于 Java。

Java 编译器在编译后丢弃了泛型类型信息，这在 Java 里被叫做类型擦除。这意味着在运行时，一个对象实例可能并不知道自己的类型信息。比如，DataStream<String> 和 DataStream<Long> 在 JVM 看来是一样的。

Flink 程序在执行时需要类型信息，Flink Java API 试图通过各种方式重构被丢弃的类型信息，并将其显式存储在数据集和对应的算子中。 您可以通过 DataStream.getType() 检索类型，该方法返回一个 TypeInformation 实例，这是 Flink 内部表示类型的方式。

类型推断具有局限性，在某些场景下需要程序编写者的配合。例如，从集合中创建数据集的方法，StreamExecutionEnvironment.fromCollection()，你可以在其中传递描述类型的参数。但像 MapFunction<I, O> 这样的通用函数也可能需要额外的类型信息。

[ResultTypeQueryable](https://github.com/apache/flink/blob/release-1.16//flink-core/src/main/java/org/apache/flink/api/java/typeutils/ResultTypeQueryable.java) 接口可以通过输入格式和函数来实现，以明确地告诉 API 它们的返回类型。调用函数的输入类型通常可以通过先前操作的结果类型来推断。

### Flink 类型管理

### 常见问题 

### Flink TypeInformation 类

#### POJO 类型的规则

#### 创建一个 TypeInformation 类及其对应的序列化器

### Scala API 中的类型信息

#### No Implicit Value for Evidence Parameter Error

#### 通用方法

### Java API 中的类型信息

#### Type Hints in the Java API 

#### Type extraction for Java 8 lambdas

#### Serialization of POJO types

### 禁用 kyro 回调

### 使用工厂类定义 TypeInformation

### 思考

**1. POJO 的序列化，PojoTypeInfo 是啥，和 kyro、avro 又是啥关系？**

**2. Java 或者 Scala 的原语类型是指啥？**

**3. Java 的类型擦除？**




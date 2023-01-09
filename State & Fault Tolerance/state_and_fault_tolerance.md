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
Flink 会尝试获取更多的类型信息以优化分布式计算中交换、存储数据的性能。可以把其想象成一个可以推断表结构的数据库（emmn，这个比喻没太 get 到）。通过类型信息，Flink 可以做很多优化上的事情：

- Flink 获取到的类型信息越多，其序列化和数据分布的规划就会做的更好。这对于 Flink 本身的内存使用非常重要（在堆内/堆外内存中处理序列化数据，更好的序列化器选择也会使序列化过程更叫高效和廉价）。

- 在很多情况下，用户不必关心序列化框架在做什么，也不用额外注册类型信息。

通常在程序真正执行前会尝试获取类型信息，比如对  DataStream 进行调用，或者在 execute()、print()、count() 或 collect() 等方法调用之前。

### 常见问题 
以下是用户在处理 Flink 类型中最常见的几个问题：

- **Registering subtypes：**注册子类型，如果一个函数签名使用了父类型，但是在真正执行的时候使用的传参是该父类型的子类，让 Flink 获取到我们使用了这些子类型会极大的提升程序性能。为此，可以在 StreamExecutionEnvironment 上为每个子类型调用 .registerType(clazz) 方法。

- **Registering custom serializers：**注册自定义的序列化器，Flink 在遇到无法处理的类型时会调用 kyro 序列化器。但是 kyro 并非能处理所有类型。例如，许多 Google Guava 集合类型在默认情况下无法正常工作，解决办法就是为这些类型注册额外的序列化器。可以在 StreamExecutionEnvironment 上调用 .getConfig().addDefaultKryoSerializer(clazz, serializer) 方法。许多库中都提供了额外的 Kryo 序列化器。有关使用外部序列化器的更多详细信息，请参考[3rd party serializer](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/datastream/fault-tolerance/serialization/third_party_serializers/)。

- **Adding Type Hints：**添加类型提示信息，有时，当 Flink 使用了所有技巧仍无法推断泛型类型时，用户必须传递类型提示信息。
这通常只在 Java API 中是必需的。关于类型提示的更多信息，可以参阅[Type Hints Section ](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/datastream/fault-tolerance/serialization/types_serialization/#type-hints-in-the-java-api)

- **Manually creating a TypeInformation：**手动创建 TypeInformation，由于 Java 的泛型类型擦除，Flink 无法推断数据类型的某些 API 调用可能需要这样做。可以参阅 [Creating a TypeInformation or TypeSerializer](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/datastream/fault-tolerance/serialization/types_serialization/#creating-a-typeinformation-or-typeserializer) 以了解更多。

### Flink TypeInformation 类
TypeInformation 类是所有类型描述的基类。它定义了类型的一些基本属性，并可以生成对应的序列化器，并且还可以生成相应类型的比较器。（*需要注意，Flink 中的比较器不仅仅是定义一个顺序，它们常用来处理 key*）。这个位置没太明白，Flink 的 comparators 还可以做什么呢？

在内部，Flink 对类型进行了以下归类划分：

- 基本类型：所有 Java 的原语类型及其封装，加上 void、StringDate、BigDecimal、BigInteger。
- 原始数组和对象数组。
- 复合类型
    - Java Tuples，最多 25 个元素，不支持 null 值。
    - Scala 样例类（包括 Tuples），不支持 null 值。
    - Row：可以理解为带有可选 Schema 信息的 Tuple，没有数量限制，允许 null 值存在。
    - POJOs：遵循 Java Bean 规则的类。
- 辅助类型：（Option、Either、Lists、Maps、...）
- 泛型：Flink 本身不处理其序列化，而是交给 kyro 来做。

POJOs 使用非常广泛，因为其支持组合任意复杂类型。且 Flink 对 POJOs 的处理也相对高效。

#### POJO 类型的规则
如果满足以下条件，Flink 会将数据类型识别为 POJO 类型（并允许按字段名称进行引用）：

- class 是 public 的并且是单例（不包括静态内部类）。
- 该类有一个公共的无参构造器。
- 该类的所有非静态字段，包括其子类，必须是 public 的。或者有相对应的公共 getter 和 setter 方法。

需要注意的是当用户自定义的数据类型不能被识别为 POJOs 时，会被当做泛型，使用 kyro 序列化处理。

#### 创建一个 TypeInformation 类及其对应的序列化器
因为 Java 的泛型擦除原因，所以需要将类型传递给 TypeInformation 构造器。对于非泛型，可以使用如下方法。
```java
TypeInformation<String> info = TypeInformation.of(String.class);
```
对于泛型，必须使用 TypeHint 标识泛型类型信息
```java
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
```
可以在 TypeInformation 实例上调用 typeInfo.createSerializer(config) 方法创建对应类型的序列化器。其中的 config 参数是 ExecutionConfig 类型，包含有关程序注册的自定义序列化程序的信息，可以在 DataStream 上调用 getExecutionConfig() 方法。在函数内部（如 MapFunction），您可以通过将函数设为 [Rich Function](https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/dev/datastream/fault-tolerance/serialization/types_serialization/) 并调用 getRuntimeContext().getExecutionConfig() 来获取它。

### Scala API 中的类型信息
Scala 通过*类型 manifests 和类标签*对运行时的类型信息有非常详尽的了解。通常，类型和方法可以访问它们的泛型参数的类型，因此Scala 程序不会像 Java 程序那样遭受类型擦除的困扰。

此外，Scala 允许通过 Scala 宏在 Scala 编译器中运行自定义代码。that means that some Flink code gets executed whenever you compile a Scala program written against Flink’s Scala API。此处有点不太理解，对于程序的底层编译运行还是一知半解。

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




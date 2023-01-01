### Overview
Apache Flink 以其特有的方式处理数据类型和序列化，包含自己的类型描述符、泛型类型提取和类型序列化框架。本文档描述了这些概念及其背后的基本原理。

### 支持的数据类型

#### Tuples and Case Classes 

#### POJOS

#### 原语类型

#### General Class Types

#### Values 

#### Hadoop Writables

#### 特殊类型

#### 类型擦除 & 类型推断


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


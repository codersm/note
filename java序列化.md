---
title: java序列化.md
date: 2016-07-07 14:25:54
tags: Java
---
# Java序列化

对象序列化保存的是对象的“状态”，即它的成员变量。由此可知，对象序列化不会关注类中的静态变量。Java序列化API为处理对象序列化提供了一个标准机制，该API简单易用。

## 代码实现

```java
protected ByteArrayOutputStream serializeJobData(JobDataMap data)
      throws IOException {
      if (canUseProperties()) {
          return serializeProperties(data);
      }
      try {
          return serializeObject(data);
      } catch (NotSerializableException e) {
          throw new NotSerializableException(
              "Unable to serialize JobDataMap for insertion into " +
              "database because the value of property '" +
              getKeyOfNonSerializableValue(data) +
              "' is not serializable: " + e.getMessage());
      }
  }
```

## 占位符

MessageFormat.format
在远程调用中，需要把参数和返回值通过网络传输，这个使用就要用到序列化将对象转变成字节流，从一端到另一端之后再反序列化回来变成对象。

## Java对象序列化

对象序列化保存的是对象的“状态”，即它的成员变量。由此可知，对象序列化不会关注类中的静态变量。Java序列化API为处理对象序列化提供了一个标准机制，该API简单易用。

### 代码实现

```java
protected ByteArrayOutputStream serializeJobData(JobDataMap data)
      throws IOException {
      if (canUseProperties()) {
          return serializeProperties(data);
      }
      try {
          return serializeObject(data);
      } catch (NotSerializableException e) {
          throw new NotSerializableException(
              "Unable to serialize JobDataMap for insertion into " +
              "database because the value of property '" +
              getKeyOfNonSerializableValue(data) +
              "' is not serializable: " + e.getMessage());
      }
  }
```

在远程调用中，需要把参数和返回值通过网络传输，这个使用就要用到序列化将对象转变成字节流，从一端到另一端之后再反序列化回来变成对象。

既然前面有一篇提到了hessian，这里就简单讲讲Java序列化和hessian序列化的区别。

Kryo

首先，hessian序列化比Java序列化高效很多，而且生成的字节流也要短很多。但相对来说没有Java序列化可靠，而且也不如Java序列化支持的全面。而之所以会出现这样的区别，则要从它们的实现方式来看。

先说Java序列化，具体工作原理就不说了，Java序列化会把要序列化的对象类的元数据和业务数据全部序列化从字节流，而且是把整个继承关系上的东西全部序列化了。它序列化出来的字节流是对那个对象结构到内容的完全描述，包含所有的信息，因此效率较低而且字节流比较大。但是由于确实是序列化了所有内容，所以可以说什么都可以传输，因此也更可用和可靠。
而hessian序列化，它的实现机制是着重于数据，附带简单的类型信息的方法。就像Integer a = 1，hessian会序列化成I 1这样的流，I表示int or Integer，1就是数据内容。而对于复杂对象，通过Java的反射机制，hessian把对象所有的属性当成一个Map来序列化，产生类似M className propertyName1 I 1 propertyName S stringValue（大概如此，确切的忘了）这样的流，包含了基本的类型描述和数据内容。而在序列化过程中，如果一个对象之前出现过，hessian会直接插入一个R index这样的块来表示一个引用位置，从而省去再次序列化和反序列化的时间。这样做的代价就是hessian需要对不同的类型进行不同的处理（因此hessian直接偷懒不支持short），而且遇到某些特殊对象还要做特殊的处理（比如StackTraceElement）。而且同时因为并没有深入到实现内部去进行序列化，所以在某些场合会发生一定的不一致，比如通过Collections.synchronizedMap得到的map。

补充：
1.原生的Serializable对象只序列化属性，不序列化方法，且不序列化静态变量
2.尽量自定义UID
3.处理对象为ObjectInputStream/ObjectOutputStream，尽量自定义序列方式，自定义writeObject和readObject方法

## Java序列化的缺点

### 无法跨语言

无法跨语言，是Java序列化最致命的问题。对于跨进程的服务调用，服务提供者可能会使用C++或者其他语言开发，当我们需要和异构语言进程交互时，Java序列化就难以胜任。

由于Java序列化技术是Java语言内部的私有协议，其他语言并不支持，对于用户来说它完全是黑盒。对于Java序列化后的字节数组，别的语言无法进行反序列化，这就严重阻碍了它的应用。

## 序列化后的码流太大

评判一个编解码框架的优劣时，往往会考虑以下几个因素：

- 是否支持跨语言，支持的语言种类是否非富；
- 编码后的码流大小；
- 编解码的性能；
- 类库是否小巧，API使用是否方便；
- 使用者需要手工开发的工作量和难度；

在同等情况下，编码后的字节数组越大，存储的时候就越占空间,存储的硬件就越高,并且
在网络传输时更占宽带，导致系统的吞吐量降低。

### 序列化性能太低

## 业界主流的编解码框架

### google的Protobuf

Protobuf全称Google Protocol Buffers，它由谷歌开源而来，在谷歌内部久经考验。它将数据结构以.proto文件进行描述，通过代码生成工具可以生成对应数据结构的POJO对象和Protobuf相关的方法和属性。

它的特点如下：

- 结构化数据存储格式（XML，JSON等）；
- 搞笑的编解码性能；
- 语言无关、平台无关、扩展性好；
- 官方支持Java、C++和Python三种语言；

> 尽管XML的可读性和可扩展性非常好，也非常适合描述数据结构，但是XML解析的时间开销和XML为了可读性而牺牲的空间开销都非常大，因此不适合做高性能的通信协议。

Protobuf优点：

- 文本化的数据结构描述语言，可以实现语言和平台无关，特别适合异构系统间的集成；
- 通过标识字段的顺序，可以实现协议的前后兼容；
- 自动代码生成，不需要手工编写同样数据结构的C++和Java版本；
- 方便后续的管理和维护。相比于代码，结构化的文档更容易管理和维护。

### Facebook的Thrift

Thrift是为了解决Facebook各系统间大数据量的传输通信以及系统之间语言环境不同需要跨平台的特性，因此Thrift可以支持多种程序语言。


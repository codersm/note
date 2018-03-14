---
title: kafka
date: 2016-07-09 13:57:33
tags: 消息中间件
---

- Kafka

<!--more-->

# 1、Kafka定义
kafka是一个分布式消息系统，由LinkedIn使用scala语言编写，用作LinkedIn的活动流(Activity Stream)和运营数据处理管道(Pipeline)的基础，具有高水平扩展和高吞吐量。

目前越来越多的开源分布式处理系统如Apache flume、Apache Storm、Spark、elasticsearch都支持与Kafka集成。

messaging terminology(消息术语)
- Topic：是特定类型的消息流。Topic是消息的分类名，
每一种消息都是一个Topic，多种消息就创建多个Topic。
- Producer(生产者)：发布消息到kafka。
- Consumer(消息者)：订阅主题和处理发布的消息。
- broker：kafka集群中的一台服务器。


客户端和服务端通信通过简单、高效、和语言无关的TCP协议。

## 搭建Kafka运行环境
## API  

# 2、Topic

分布式
  日志分区分布在kafka集群的服务器，这些服务器处理数据和请求一个共享
分区。容错是通过服务器配置分区的备份。
  每个分区都由一个服务器作为“leader”，零或若干服务器作为“followers”。
leader处理分区所有读和写请求，followers复制leader。一旦leader down了，
followers其中的一个会自动成为新的leader。每个服务器都会同时扮演两个角色：作为所持有的一部分分区的leader，同时作为其他分区的followers，这样集群就会据有较好的负载均衡。  

Producers
  Producers将消息发布到它指定的topic中，并负责决定发布到那个分区。以循环方式简单地负载均衡或者根据分区函数进行分区。通常使用第二种方式。

Consumers
    发布消息通常有两种模式：队列和发布-订阅模式。在队列模式下，消费者从一台
服务读取消息，并且每条消息只能发送消费者池中的一个消费者；在发布/订阅模式下
，消息广播到所有的消费者。    

kafka对这个两种进行抽象概括——comsumer group
>Consumers label themselves with a consumer group name, and each message published to a topic is delivered to one consumer instance within each subscribing consumer group. Consumer instances can be in separate processes or on separate machines.


Kafka的保证(Guarantees)
 - 生产者发送到一个特定的Topic的分区上的消息将会按照它们发送的顺序依次加入
 消费者收到的消息也是此顺序
 - 如果一个Topic配置了复制因子( replication facto)为N， 那么可以允许N-1服务器当掉而不丢失任何已经增加的消息

使用场景
  - Messaging(消息系统)
  - Website Activity Tracking(站点的用户活动追踪)：用来记录用户的页面浏览，搜索，点击等。
  - Metrics(操作审计)： 用户/管理员的网站操作的监控。
  - Log Aggregation(日志聚合)
  - Stream Processing
  - Event Sourcing

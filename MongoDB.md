---
title: MongoDB
date: 2016-07-05 18:25:18
tags: MongoDB
---

# MongoDB学习笔记

MongoDB特点

* 索引
* 复制
* 分片
* 丰富的查询语法

1、易于使用

2、易于扩展

3、丰富的功能

4、卓越的性能

MongoDB术语

文档

集合

实例

MongoDB分页

索引

复制

分片

原子操作

ObjectId

Map ~~Reduce（计算模型）~~


MongoDB
  MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。在高负载的情况下，添加更多的节点，可以保证服务器性能。
  MongoDB 旨在为WEB应用提供可扩展的高性能数据存储解决方案。
  MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。


MongoDB与关系数据库比较
  sql：数据库(db)--->表(table)--->记录(record)--->字段(column)
  mongodb：数据库--->collection---->document--->字段

特性：
    Auto-Sharding:sacle sql时的分表
    没有事务

在使用mongodb时，需要思考的两点就是：表join到底要不要，事务支持到底要不要。
mongodb中，collection是schema-less的。在sql中，我们需要用建表语句来表明数据应该具有的形式，而mongodb中，可以在同一张里存各种各样不同的形式的数据。同一个collection中，可以有些document具有100个字段，而另一些，则只具有5个字段。如果你分不清这个特性的使用场景，那么请像使用sql一样的，尽可能保证一个collection中数据格式是统一的。这个 schema-less 的特性，有个比较典型的场景是用来存储日志类型的数据，可以搜搜看这方面的典型场景。


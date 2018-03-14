---
title: javassist学习记录
date: 2016-12-06 22:08:06
tags: 字节码
---

Javassist是一个用于处理Java字节码的类库。Java字节码存储在称为类文件的二进制文件中，每个类文件包含一个Java类或接口。

- CtClass
    Javassist.CtClass类是类文件的抽象表示,CtClass（编译时类）对象是一个处理类文件的句柄。
- ClassPool
    控制使用Javassist的字节码修改,ClassPool对象是表示类文件的CtClass对象的容器。
```java
    ClassPool pool = ClassPool.getDefault();
    CtClass cc = pool.get("test.Rectangle");
```

# 读写字节码
## 定义类
## 冻结类
## 类路径查找

# ClassPool

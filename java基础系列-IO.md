---
title: Java IO
date: 2016-10-18 21:22:02
tags: Java
---
# Java IO

- 定义  
  向某种特定介质写入数据。
- 类结构体系  
- 方法
  ```java
  public abstract void write(int b) throws IOException;

  public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
  }

  public void flush() throws IOException {}


  public void close() throws IOException {}
  ```
write(int b)方法接受一个0到255之间的整数作为参数，将对应的字节
写入到输出流中。  

实际上会写入一个无符号字节。Java没有无符号字节数据类型，所以这里要使用int来代替。当使用write(int b)将int写入一个网络连接时，线缆上只会放8个二进制位。如果将一个超出0~255的int传入write(int b),将写入这个数的最低字节，其他3字节将被忽略。

一次写入1字节通常效率不高。使用write(byte[] data)或write(byte[] data,int offset,int length)通常比一次写入data数组中的1字节要快得多。

在写入数据完成后，刷新(flush)输出流非常重要。

当结束一个流的操作时，要通过调用它的close()方法将其关闭。这会释放与这个流关联的所有资源，如文件句柄或端口。在Java 6和更早版本中，明智的做法是在finally块中关闭流。
Java 7引入了“带资源的try”构造，可以更间接地完成这个清理。
```java
try(OutputStream out =  new FileOutputStream("/tmp/data.txt")) {
            // TODO dosomething ....

 } catch ( IOException e) {
     e.printStackTrace();
 }
```
现在不再需要Finally子句。Java会对try块参数表中声明的所有AutoCloseable对象自动调用close()

> 只要对象实现了Closeable接口，都可以使用“带资源的try”构造，这包括几乎所有需要释放的对象。到目前为止，JavaMail Transport对象是我见过的唯一的例外。这些对象还需要
显式地释放。


### 输入流
- 定义  
  从某种特定介质中读取数据。
- 常用类
- 方法
  read()方法从输入流的源中读取1字节数据，作为一个0到255的int返回。流的结束通过返回-1来表示。read()方法会等待并阻塞其后任何代码的执行，直到有1字节的数据可供读取。输入和输出可能很慢，所以如果程序在做其他重要的工作，要尽量将I/O放在单独的线程中。

#### Java IO包装流如何关闭

  1、JAVA的IO流使用了装饰模式,关闭最外面的流的时候会自动调用被包装的流的close()方法?
  2、如果按顺序关闭流，是从内层流到外层流关闭还是从外层到内存关闭？

**BufferedInputStream.close()方法**如下：

```java
public void close() throws IOException {
      byte[] buffer;
      while ( (buffer = buf) != null) {
          if (bufUpdater.compareAndSet(this, buffer, null)) {
              InputStream input = in;
              in = null;
              if (input != null)
                  input.close();
              return;
          }
          // Else retry in case a new buf was CASed in fill()
      }
  }
```

可以上述代码可以看出，BufferedInputStream.close()方法调用了input.close()方法。
>JAVA的IO流使用了装饰模式,关闭最外面的流的时候会自动调用被包装的流的close()方法。

一般情况下是：先打开的后关闭，后打开的先关闭。
另一种情况：看依赖关系，如果流a依赖流b，应该先关闭流a，再关闭流b
当然完全可以只关闭处理流，不用关闭节点流。处理流关闭的时候，会调用其处理的节点流的关闭方法
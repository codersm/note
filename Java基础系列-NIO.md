---
title: Java NIO
date: 2017-08-15 22:30:37
tags: Java
---
# Java NIO

## NIO Buffers

缓冲区在java.nio包中定义。它定义了所有缓冲区通用的核心功能：限制，容量和当前位置。

Java NIO缓冲区用于与NIO通道进行交互。它是我们可以写入数据的内存块，我们以后可以再次读取。内存块用NIO缓冲区对象包装，它提供了更简单的方法来处理内存块。

### Buffer Basics

```java
public abstract class Buffer {

    /**
     * The characteristics of Spliterators that traverse and split elements
     * maintained in Buffers.
     */
    static final int SPLITERATOR_CHARACTERISTICS =
        Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    // Used only by direct buffers
    // NOTE: hoisted here for speed in JNI GetDirectBufferAddress
    long address;

    // Creates a new buffer with the given mark, position, limit, and capacity,
    // after checking invariants.
    //
    Buffer(int mark, int pos, int lim, int cap) {       // package-private
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos)
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }

    /**
     * Returns this buffer's capacity.
     *
     * @return  The capacity of this buffer
     */
    public final int capacity() {
        return capacity;
    }

    /**
     * Returns this buffer's position.
     *
     * @return  The position of this buffer
     */
    public final int position() {
        return position;
    }

    /**
     * Sets this buffer's position.  If the mark is defined and larger than the
     * new position then it is discarded.
     *
     * @param  newPosition
     *         The new position value; must be non-negative
     *         and no larger than the current limit
     *
     * @return  This buffer
     *
     * @throws  IllegalArgumentException
     *          If the preconditions on <tt>newPosition</tt> do not hold
     */
    public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;
        return this;
    }

    /**
     * Returns this buffer's limit.
     *
     * @return  The limit of this buffer
     */
    public final int limit() {
        return limit;
    }

    /**
     * Sets this buffer's limit.  If the position is larger than the new limit
     * then it is set to the new limit.  If the mark is defined and larger than
     * the new limit then it is discarded.
     *
     * @param  newLimit
     *         The new limit value; must be non-negative
     *         and no larger than this buffer's capacity
     *
     * @return  This buffer
     *
     * @throws  IllegalArgumentException
     *          If the preconditions on <tt>newLimit</tt> do not hold
     */
    public final Buffer limit(int newLimit) {
        if ((newLimit > capacity) || (newLimit < 0))
            throw new IllegalArgumentException();
        limit = newLimit;
        if (position > limit) position = limit;
        if (mark > limit) mark = -1;
        return this;
    }

    /**
     * Sets this buffer's mark at its position.
     *
     * @return  This buffer
     */
    public final Buffer mark() {
        mark = position;
        return this;
    }

    /**
     * Resets this buffer's position to the previously-marked position.
     *
     * <p> Invoking this method neither changes nor discards the mark's
     * value. </p>
     *
     * @return  This buffer
     *
     * @throws  InvalidMarkException
     *          If the mark has not been set
     */
    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }

    /**
     * Clears this buffer.  The position is set to zero, the limit is set to
     * the capacity, and the mark is discarded.
     *
     * <p> Invoke this method before using a sequence of channel-read or
     * <i>put</i> operations to fill this buffer.  For example:
     *
     * <blockquote><pre>
     * buf.clear();     // Prepare buffer for reading
     * in.read(buf);    // Read data</pre></blockquote>
     *
     * <p> This method does not actually erase the data in the buffer, but it
     * is named as if it did because it will most often be used in situations
     * in which that might as well be the case. </p>
     *
     * @return  This buffer
     */
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }

    /**
     * Flips this buffer.  The limit is set to the current position and then
     * the position is set to zero.  If the mark is defined then it is
     * discarded.
     *
     * <p> After a sequence of channel-read or <i>put</i> operations, invoke
     * this method to prepare for a sequence of channel-write or relative
     * <i>get</i> operations.  For example:
     *
     * <blockquote><pre>
     * buf.put(magic);    // Prepend header
     * in.read(buf);      // Read data into rest of buffer
     * buf.flip();        // Flip buffer
     * out.write(buf);    // Write header + data to channel</pre></blockquote>
     *
     * <p> This method is often used in conjunction with the {@link
     * java.nio.ByteBuffer#compact compact} method when transferring data from
     * one place to another.  </p>
     *
     * @return  This buffer
     */
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }

    /**
     * Rewinds this buffer.  The position is set to zero and the mark is
     * discarded.
     *
     * <p> Invoke this method before a sequence of channel-write or <i>get</i>
     * operations, assuming that the limit has already been set
     * appropriately.  For example:
     *
     * <blockquote><pre>
     * out.write(buf);    // Write remaining data
     * buf.rewind();      // Rewind buffer
     * buf.get(array);    // Copy data into array</pre></blockquote>
     *
     * @return  This buffer
     */
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }

    /**
     * Returns the number of elements between the current position and the
     * limit.
     *
     * @return  The number of elements remaining in this buffer
     */
    public final int remaining() {
        return limit - position;
    }

    /**
     * Tells whether there are any elements between the current position and
     * the limit.
     *
     * @return  <tt>true</tt> if, and only if, there is at least one element
     *          remaining in this buffer
     */
    public final boolean hasRemaining() {
        return position < limit;
    }

    /**
     * Tells whether or not this buffer is read-only.
     *
     * @return  <tt>true</tt> if, and only if, this buffer is read-only
     */
    public abstract boolean isReadOnly();

    /**
     * Tells whether or not this buffer is backed by an accessible
     * array.
     *
     * <p> If this method returns <tt>true</tt> then the {@link #array() array}
     * and {@link #arrayOffset() arrayOffset} methods may safely be invoked.
     * </p>
     *
     * @return  <tt>true</tt> if, and only if, this buffer
     *          is backed by an array and is not read-only
     *
     * @since 1.6
     */
    public abstract boolean hasArray();

    /**
     * Returns the array that backs this
     * buffer&nbsp;&nbsp;<i>(optional operation)</i>.
     *
     * <p> This method is intended to allow array-backed buffers to be
     * passed to native code more efficiently. Concrete subclasses
     * provide more strongly-typed return values for this method.
     *
     * <p> Modifications to this buffer's content will cause the returned
     * array's content to be modified, and vice versa.
     *
     * <p> Invoke the {@link #hasArray hasArray} method before invoking this
     * method in order to ensure that this buffer has an accessible backing
     * array.  </p>
     *
     * @return  The array that backs this buffer
     *
     * @throws  ReadOnlyBufferException
     *          If this buffer is backed by an array but is read-only
     *
     * @throws  UnsupportedOperationException
     *          If this buffer is not backed by an accessible array
     *
     * @since 1.6
     */
    public abstract Object array();

    /**
     * Returns the offset within this buffer's backing array of the first
     * element of the buffer&nbsp;&nbsp;<i>(optional operation)</i>.
     *
     * <p> If this buffer is backed by an array then buffer position <i>p</i>
     * corresponds to array index <i>p</i>&nbsp;+&nbsp;<tt>arrayOffset()</tt>.
     *
     * <p> Invoke the {@link #hasArray hasArray} method before invoking this
     * method in order to ensure that this buffer has an accessible backing
     * array.  </p>
     *
     * @return  The offset within this buffer's array
     *          of the first element of the buffer
     *
     * @throws  ReadOnlyBufferException
     *          If this buffer is backed by an array but is read-only
     *
     * @throws  UnsupportedOperationException
     *          If this buffer is not backed by an accessible array
     *
     * @since 1.6
     */
    public abstract int arrayOffset();

    /**
     * Tells whether or not this buffer is
     * <a href="ByteBuffer.html#direct"><i>direct</i></a>.
     *
     * @return  <tt>true</tt> if, and only if, this buffer is direct
     *
     * @since 1.6
     */
    public abstract boolean isDirect();


    // -- Package-private methods for bounds checking, etc. --

    /**
     * Checks the current position against the limit, throwing a {@link
     * BufferUnderflowException} if it is not smaller than the limit, and then
     * increments the position.
     *
     * @return  The current position value, before it is incremented
     */
    final int nextGetIndex() {                          // package-private
        if (position >= limit)
            throw new BufferUnderflowException();
        return position++;
    }

    final int nextGetIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferUnderflowException();
        int p = position;
        position += nb;
        return p;
    }

    /**
     * Checks the current position against the limit, throwing a {@link
     * BufferOverflowException} if it is not smaller than the limit, and then
     * increments the position.
     *
     * @return  The current position value, before it is incremented
     */
    final int nextPutIndex() {                          // package-private
        if (position >= limit)
            throw new BufferOverflowException();
        return position++;
    }

    final int nextPutIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferOverflowException();
        int p = position;
        position += nb;
        return p;
    }

    /**
     * Checks the given index against the limit, throwing an {@link
     * IndexOutOfBoundsException} if it is not smaller than the limit
     * or is smaller than zero.
     */
    final int checkIndex(int i) {                       // package-private
        if ((i < 0) || (i >= limit))
            throw new IndexOutOfBoundsException();
        return i;
    }

    final int checkIndex(int i, int nb) {               // package-private
        if ((i < 0) || (nb > limit - i))
            throw new IndexOutOfBoundsException();
        return i;
    }

    final int markValue() {                             // package-private
        return mark;
    }

    final void truncate() {                             // package-private
        mark = -1;
        position = 0;
        limit = 0;
        capacity = 0;
    }

    final void discardMark() {                          // package-private
        mark = -1;
    }

    static void checkBounds(int off, int len, int size) { // package-private
        if ((off | len | (off + len) | (size - (off + len))) < 0)
            throw new IndexOutOfBoundsException();
    }

}
```

## NIO Channels

在Java NIO中，通道是用于在实体和字节缓冲区之间有效传输数据的介质。它从一个实体读取数据，并将其放在缓冲区块中以供消费。

通道作为Java NIO提供的网关来访问I/O机制。通常，通道与操作系统文件描述符具有一对一关系，用于提供平台独立操作功能。

### Channel Basics

通道实现使用本地代码执行实际工作。通道接口允许我们以便携和受控的方式访问低级I/O服务。

```java
public interface Channel extends Closeable {

    /**
     * Tells whether or not this channel is open.
     *
     * @return <tt>true</tt> if, and only if, this channel is open
     */
    public boolean isOpen();

    /**
     * Closes this channel.
     *
     * <p> After a channel is closed, any further attempt to invoke I/O
     * operations upon it will cause a {@link ClosedChannelException} to be
     * thrown.
     *
     * <p> If this channel is already closed then invoking this method has no
     * effect.
     *
     * <p> This method may be invoked at any time.  If some other thread has
     * already invoked it, however, then another invocation will block until
     * the first invocation is complete, after which it will return without
     * effect. </p>
     *
     * @throws  IOException  If an I/O error occurs
     */
    public void close() throws IOException;

}
```

### Channel Implementations

### Basic Channel Example

```java
public static void main(String[] args) throws IOException {
        FileInputStream input = new FileInputStream ("D:\\1.txt");
        ReadableByteChannel source = input.getChannel();
        FileOutputStream output = new FileOutputStream ("D:\\2.txt");
        WritableByteChannel destination = output.getChannel();
        copyData(source, destination);
        source.close();
        destination.close();
    }

private static void copyData(ReadableByteChannel src,
                             WritableByteChannel dest) throws IOException{

    ByteBuffer buffer = ByteBuffer.allocateDirect(20 * 1024);
    while (src.read(buffer) != -1)
    {
        // The buffer is used to drained
        buffer.flip();
        // keep sure that buffer was fully drained
        while (buffer.hasRemaining())
        {
            dest.write(buffer);
        }
        buffer.clear(); // Now the buffer is empty, ready for the filling
    }
}
```

## 分散/收集或向量I/O

## Data Transfer between Channels(通道之间的数据传输)

在Java NIO中，我们可以非常频繁地将数据从一个通道传输到另一个通道。批量传输文件数据是非常普遍的，因为几个优化方法已经添加到FileChannel类中，使其更有效率。

FileChannel类提供两个用于数据传输的方法：

* FileChannel.transferTo()

    ```java
    public abstract long transferTo(long position, long count,
                                WritableByteChannel target) throws IOException;
    ```

    transferTo()方法允许从FileChannel到其他通道的数据传输。

* FileChannel.transferFrom()

    ```java
    public abstract long transferFrom(ReadableByteChannel src,
                                 long position, long count) throws IOException;
    ```

## NIO Selector(选择器)

在Java NIO中，选择器是可选通道的多路复用器，可用作可以进入非阻塞模式的特殊类型的通道。它可以检查一个或多个NIO通道，并确定哪个通道准备好进行通信，即读取或写入。

选择器用于使用单个线程处理多个通道。因此，它需要较少的线程来处理这些通道。在线程之间切换对于操作系统来说是昂贵的。因此，使用它可以提高系统效率。

**使用选择器来处理3通道的线程的图示**:

![使用选择器来处理3通道的线程的图示](http://i2.bvimg.com/598959/54b971ce559abec4.png)

### Creating a Selector

我们可以通过调用Selector.open()方法来创建一个选择器，如下所示：

```java
Selector selector = Selector.open();
```

### Opening a server socket channel

```java
ServerSocketChannel serverSocket = ServerSocketChannel.open();  
InetSocketAddress hostAddress = new InetSocketAddress("localhost", 8080);  
serverSocket.bind(hostAddress);
```

### Selection of Channels using a Selector

在使用选择器注册一个或多个通道时，可以调用select()方法之一。该方法返回一个准备好用于我们要执行的事件的通道，即连接，读取，写入或接受。

可用于选择通道的各种select()方法是:

* int select(): The integer value return by the select() method inform about how many channels are ready for the communication.

* int select(long TS): This method is same as select() except it blocks the output for a maximum of TS(in millisecond) time period.

* int selectNow(): It doesn't block the output and returns immediately with any channel which are ready.

**selectedKeys()**

## NIO Pipe

Java NIO管道用于在两个线程之间建立单向数据连接。It has a sink channel and source channel. The data is writing to the sink channel and this data can then be read from a source channel.

在Java NIO中，java.nio.channel.pipe包用于按顺序读取和写入数据。管道用于确保数据必须以写入管道的相同顺序读取。

```java
public class PipeExample {
  public static void main(String[] args) throws IOException {
      //The Pipe is created
      Pipe pipe = Pipe.open();
      //For accessing the pipe sink channel
      Pipe.SinkChannel skChannel = pipe.sink();
      String td = "Data is successfully sent for checking the java NIO Channel Pipe.";
ByteBuffer bb = ByteBuffer.allocate(512);
      bb.clear();
      bb.put(td.getBytes());  
      bb.flip();  
       //write the data into a sink channel.  
      while(bb.hasRemaining()) {  
          skChannel.write(bb);  
      }  
      //For accessing the pipe source channel  
      Pipe.SourceChannel sourceChannel = pipe.source();  
      bb = ByteBuffer.allocate(512);  
     //The data is write to the console     
      while(sourceChannel.read(bb) > 0){  
          bb.flip();  
            
          while(bb.hasRemaining()){  
              char TestData = (char) bb.get();  
              System.out.print(TestData);  
          }  
          bb.clear();  
      }  
  }  
}  
```


---
title: Java NIO
date: 2017-08-14 16:37:20
tags: Java
---
# Java NIO

Java NIO(New IO)是一个可以替代标准Java IO API的IO API（JDK1.4开始），Java NIO提供了
与标准IO不同的IO工作方式。

* Java NIO: Channels and Buffers（通道和缓冲区）

    标准的IO基于字节流和字符流进行操作的，而NIO是基于通道（Channel）和缓冲区（Buffer）进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。

* Java NIO: Non-blocking IO（非阻塞IO）

    Java NIO可以让你非阻塞的使用IO，例如：当线程从通道读取数据到缓冲区时，线程还是可以进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。

* Java NIO: Selectors（选择器）

    Java NIO引入了选择器的概念，选择器用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个的线程可以监听多个数据通道。

## Channel和Buffer

Channel是对原I/O包中的流的模拟。到任何目的地(或来自任何地方)的所有数据都必须通过一个Channel对象。一个Buffer实质上是一个容器对象。发送给一个通道的所有对象都必须首先放到缓冲区中；同样地，从通道中读取的任何数据都要读到缓冲区中。

Buffer 是一个对象， 它包含一些要写入或者刚读出的数据。 在 NIO 中加入 Buffer 对象，体现了新库与原 I/O 的一个重要区别。在面向流的 I/O 中，您将数据直接写入或者将数据直接读到 Stream 对象中。
在 NIO 库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的。在写入数据时，它是写入到缓冲区中的。任何时候访问 NIO 中的数据，您都是将它放到缓冲区中。
缓冲区实质上是一个数组。通常它是一个字节数组，但是也可以使用其他种类的数组。但是一个缓冲区不 仅仅 是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

Channel是一个对象，可以通过它读取和写入数据。拿 NIO 与原来的 I/O 做个比较，通道就像是流。

通道与流的不同之处在于通道是双向的。而流只是在一个方向上移动(一个流必须是InputStream或者OutputStream的子类)， 而通道可以用于读、写或者同时用于读写。因为它们是双向的，所以通道可以比流更好地反映底层操作系统的真实情况。特别是在 UNIX 模型中，底层操作系统通道是双向的。

## 缓冲区内部细节

**状态变量**

可以用三个值指定缓冲区在任意时刻的状态：
* position
* limit
* capacity

这三个变量一起可以跟踪缓冲区的状态和它所包含的数据。

**Position**

position 变量跟踪已经写了多少数据。更准确地说，它指定了下一个字节将放到数组的哪一个元素中。因此，如果您从通道中读三个字节到缓冲区中，那么缓冲区的 position 将会设置为3，指向数组中第四个元素。

同样，在写入通道时，您是从缓冲区中获取数据。 position 值跟踪从缓冲区中获取了多少数据。更准确地说，它指定下一个字节来自数组的哪一个元素。因此如果从缓冲区写了5个字节到通道中，那么缓冲区的 position 将被设置为5，指向数组的第六个元素。

**Limit**

limit变量表明还有多少数据需要取出(在从缓冲区写入通道时)，或者还有多少空间可以放入数据(在从通道读入缓冲区时)。position总是小于或者等于limit。

**capacity**

缓冲区的 capacity 表明可以储存在缓冲区中的最大数据容量。实际上，它指定了底层数组的大小 ― 或者至少是指定了准许我们使用的底层数组的容量。
limit 决不能大于capacity。

**访问方法**

ByteBuffer 类的 get() 和 put() 方法直接访问缓冲区中的数据

**缓冲区分配和包装**

在能够读和写之前，必须有一个缓冲区。要创建缓冲区，您必须 分配 它。我们使用静态方法 allocate() 来分配缓冲区：

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
```

allocate() 方法分配一个具有指定大小的底层数组，并将它包装到一个缓冲区对象中。您还可以将一个现有的数组转换为缓冲区，如下所示：

```java
byte array[] = new byte[1024];
ByteBuffer buffer = ByteBuffer.wrap(array);
```
wrap() 方法将一个数组包装为缓冲区。必须非常小心地进行这类操作。一旦完成包装，底层数据就可以通过缓冲区或者直接访问。

**缓冲区分片**

slice() 方法根据现有的缓冲区创建一种 子缓冲区 。也就是说，它创建一个新的缓冲区，新缓冲区与原来的缓冲区的一部分共享数据。

**缓冲区份片和数据共享**

**只读缓冲区**

只读缓冲区非常简单 ― 您可以读取它们，但是不能向它们写入。可以通过调用缓冲区的 asReadOnlyBuffer() 方法，将任何常规缓冲区转换为只读缓冲区，这个方法返回一个与原缓冲区完全相同的缓冲区(并与其共享数据)，只不过它是只读的。

只读缓冲区对于保护数据很有用。在将缓冲区传递给某个对象的方法时，您无法知道这个方法是否会修改缓冲区中的数据。创建一个只读的缓冲区可以 保证 该缓冲区不会被修改。

不能将只读的缓冲区转换为可写的缓冲区。

**直接缓冲区**

**内存映射文件 I/O**

内存映射文件 I/O 是一种读和写文件数据的方法，它可以比常规的基于流或者基于通道的 I/O 快得多。

内存映射文件 I/O 是通过使文件中的数据神奇般地出现为内存数组的内容来完成的。这其初听起来似乎不过就是将整个文件读到内存中，但是事实上并不是这样。一般来说，只有文件中实际读取或者写入的部分才会送入（或者 映射 ）到内存中。

现代操作系统一般根据需要将文件的部分映射为内存的部分，从而实现文件系统。Java 内存映射机制不过是在底层操作系统中可以采用这种机制时，提供了对该机制的访问。
尽管创建内存映射文件相当简单，但是向它写入可能是危险的。仅只是改变数组的单个元素这样的简单操作，就可能会直接修改磁盘上的文件。修改数据与将数据保存到磁盘是没有分开的。

**分散和聚集**

分散/聚集 I/O 是使用多个而不是单个缓冲区来保存数据的读写方法。

一个分散的读取就像一个常规通道读取，只不过它是将数据读到一个缓冲区数组中而不是读到单个缓冲区中。同样地，一个聚集写入是向缓冲区数组而不是向单个缓冲区写入数据。

分散/聚集 I/O 对于将数据流划分为单独的部分很有用，这有助于实现复杂的数据格式。


分散/聚集 I/O 对于将数据划分为几个部分很有用。例如，您可能在编写一个使用消息对象的网络应用程序，每一个消息被划分为固定长度的头部和固定长度的正文。您可以创建一个刚好可以容纳头部的缓冲区和另一个刚好可以容难正文的缓冲区。当您将它们放入一个数组中并使用分散读取来向它们读入消息时，头部和正文将整齐地划分到这两个缓冲区中。
我们从缓冲区所得到的方便性对于缓冲区数组同样有效。因为每一个缓冲区都跟踪自己还可以接受多少数据，所以分散读取会自动找到有空间接受数据的第一个缓冲区。在这个缓冲区填满后，它就会移动到下一个缓冲区。

**聚集写入**

**文件锁定**

**网络和异常**

异步 I/O 调用不会阻塞。相反，您将注册对特定 I/O 事件的兴趣 ― 可读的数据的到达、新的套接字连接，等等，而在发生这样的事件时，系统将会告诉您。

异步 I/O 的一个优势在于，它允许您同时根据大量的输入和输出执行 I/O。同步程序常常要求助于轮询，或者创建许许多多的线程以处理大量的连接。使用异步 I/O，您可以监听任何数量的通道上的事件，不用轮询，也不用额外的线程。

异步 I/O 中的核心对象名为 Selector。Selector 就是您注册对各种 I/O 事件的兴趣的地方，而且当那些事件发生时，就是这个对象告诉您所发生的事件。

**字符集**

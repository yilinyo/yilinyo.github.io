
---
title: 非阻塞IO NIO 入门
tag: NIO
date: 2024-1-27 15:34:00
---

# NIO

非阻塞IO

> 本笔记资料 来自[黑马程序 Netty 教程 ](https://www.bilibili.com/video/BV1py4y1E7oA/?p=6&share_source=copy_web&vd_source=5c53fad723f9304699742f8633214dd3)及 自己的一些总结
## ByteBuffer 


**在内存开辟一个缓冲区，大小不宜过大**


### ByteBuffer 的分配和状态

```java
    FileChannel channel = file.getChannel();
     ByteBuffer buffer = ByteBuffer.allocate(10);
```
##### 一开始 的 状态是**写模式** 也就是分配完空间后

![](https://s2.loli.net/2024/01/26/8WIijoOlae1JFGk.png)

写模式下，position 是写入位置，limit 等于容量，下图表示写入了 4 个字节后的状态

![](https://s2.loli.net/2024/01/26/DzacF3P7rvwTKEy.png)

##### flip 动作发生后，position 切换为读取位置，limit 切换为**读取限制**

![](https://s2.loli.net/2024/01/26/ZEVTJ2I58Hekstn.png)

读取 4 个字节后，状态

![](https://s2.loli.net/2024/01/26/tF2nQoiPXZ6I8VY.png)

##### clear 动作发生后，状态又变回**写**，注意这里面内容也清空，所以一般读取完才调用

![](https://s2.loli.net/2024/01/26/8WIijoOlae1JFGk.png)

compact 方法，是把**未读完的部分向前压缩**，然后切换至**写模式**

![](https://s2.loli.net/2024/01/26/4LqTp86f1bEiGnz.png)


不同类型的空间分配
```java
    ByteBuffer buf1 = ByteBuffer.allocate(16);  //分配堆内存 会GC调整 读写慢
    ByteBuffer buf2 = ByteBuffer.allocateDirect(16); //分配直接内存 调用操作系统 分配慢
```


### ByteBuffer 常见方法

#### 写数据

```java
int readBytes = channel.read(buf); //channel写
buf.put((byte)127); //put写
buf.put(byte[]); //写字符数组
```

#### 读数据

```java
int writeBytes = channel.write(buf); //channel读
byte b = buf.get(); //get读 且会让position后移
buf.get(int i) //方法获取索引 i 的内容，它不会移动读指针
```

#### 调指针

```java
buf.rewind(); //rewind 方法将 position 重新置为 0
// rewind 增强
buf.mark();//为当前postion 做一个标记
buf.reset(); //重置postion 为mark位置
buf.limit(); //获取limit
buf.limit(16); //设置limit16
```

#### 字节数组到ByteBuffer转换
```java
// string 到 buffer 完成后自动会变成读模式
ByteBuffer buffer1 = StandardCharsets.UTF_8.encode("你好");
ByteBuffer buffer2 = Charset.forName("utf-8").encode("你好");
ByteBuffer buffer = ByteBuffer.wrap("hello".getBytes());

//转换 且需要在写模式
CharBuffer buffer3 = StandardCharsets.UTF_8.decode(buffer1);

System.out.println(buffer3.toString()); //你好
```

## ⚠️ FileChannel 工作模式

> FileChannel 和传统的文件 I/O（例如 FileInputStream、FileOutputStream）之间有几个重要的区别：
> 
> 非阻塞 I/O：
> 
> FileChannel 支持非阻塞 I/O 操作，这意味着你可以使用 FileChannel 的某些方法进行异步 I/O 操作，而不必等待每个操作完成。
> 传统的文件 I/O 是阻塞的，即在进行读或写操作时，程序会一直等待直到操作完成。
> ByteBuffer 使用：
> 
> FileChannel 与 ByteBuffer 配合使用，通过将数据存储在 ByteBuffer 中来进行读写操作。
> 传统的文件 I/O 使用 InputStream 和 OutputStream，并且通常需要在读取或写入数据之前创建一个字节数组。
> 文件锁定：
> 
> FileChannel 具有支持文件锁定的能力，可以通过 FileLock 对象实现对文件的独占或共享锁定。
> 传统文件 I/O 通常不提供直接的文件锁定机制。
> 内存映射：
> 
> FileChannel 允许将文件的一部分或整个文件映射到内存中，以便直接在内存中进行读写操作，提高性能。
> 传统文件 I/O 没有内存映射的直接支持。
> 性能优势：
> 
> 由于 FileChannel 允许进行一些底层的优化，因此在某些情况下，它可以提供更好的性能，特别是对于大量数据的读写操作。
> 传统文件 I/O 可能会在处理大量数据时变得相对较慢。

> FileChannel 只能工作在阻塞模式下



#### 获取

不能直接打开 FileChannel，必须通过 FileInputStream、FileOutputStream 或者 RandomAccessFile 来获取 FileChannel，它们都有 getChannel 方法

* 通过 FileInputStream 获取的 channel 只能读
* 通过 FileOutputStream 获取的 channel 只能写
* 通过 RandomAccessFile 是否能读写根据构造 RandomAccessFile 时的读写模式决定，指定rw



#### 读取

会从 channel 读取数据填充 ByteBuffer，返回值表示读到了多少字节，-1 表示到达了文件的末尾

```java
int readBytes = channel.read(buffer);
```



#### 写入

写入的正确姿势如下， SocketChannel

```java
ByteBuffer buffer = ...;
buffer.put(...); // 存入数据
buffer.flip();   // 切换读模式

while(buffer.hasRemaining()) {
    channel.write(buffer);
}
```

在 while 中调用 channel.write 是因为 write 方法并不能保证一次将 buffer 中的内容全部写入 channel



#### 关闭

channel 必须关闭，不过调用了 FileInputStream、FileOutputStream 或者 RandomAccessFile 的 close 方法会间接地调用 channel 的 close 方法

#### 位置

获取当前位置

```java
long pos = channel.position();
```

设置当前位置

```java
long newPos = ...;
channel.position(newPos);
```

设置当前位置时，如果设置为文件的末尾

* 这时读取会返回 -1 
* 这时写入，会追加内容，但要注意如果 position 超过了文件末尾，再写入时在新内容和原末尾之间会有空洞（00）



#### 大小

使用 size 方法获取文件的大小



#### 强制写入 ✨✨

***操作系统出于性能的考虑，会将数据缓存，不是立刻写入磁盘。可以调用 force(true)  方法将文件内容和元数据（文件的权限等信息）立刻写入磁盘***

#### 两个channel传递数据

```java
String FROM = "helloword/data.txt";
String TO = "helloword/to.txt";
long start = System.nanoTime();
try (FileChannel from = new FileInputStream(FROM).getChannel();
     FileChannel to = new FileOutputStream(TO).getChannel();
    ) {
    from.transferTo(0, from.size(), to);
} catch (IOException e) {
    e.printStackTrace();
}
long end = System.nanoTime();
System.out.println("transferTo 用时：" + (end - start) / 1000_000.0);
```
### Selector 管理Channel
![image](https://s2.loli.net/2024/01/26/6jpHGOtM92TJQNS.png)


```java
   //建立Selector
        Selector selector = Selector.open();

        ByteBuffer buff = ByteBuffer.allocate(16);

        //打开一个ServerSocketChannel
        ServerSocketChannel ssc = ServerSocketChannel.open();

        //设置阻塞模式为非阻塞
        ssc.configureBlocking(false);

        // 建立Selector 和 Channel的联系
        // （管理员）事件发生后，得到事件和哪个channel发生
        SelectionKey sscKey = ssc.register(selector, 0, null);

        sscKey.interestOps(SelectionKey.OP_ACCEPT);
        //bind 一个端口
        ssc.bind(new InetSocketAddress(8888));

        while (true) {

            //没有事件就阻塞，有就继续
            //事件未处理就不会阻塞
            selector.select();

            //拿到所有事件集合
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();


            while(iterator.hasNext()){

                SelectionKey key = iterator.next();

                //事件用掉要从集合删除
                iterator.remove();

                log.debug("key {}",key);

                if(key.isAcceptable()) {

                    ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                    
                    SocketChannel sc = channel.accept();
                    //设置非阻塞
                    sc.configureBlocking(false);
                    //注册到selecor
                    SelectionKey sk = sc.register(selector, 0, null);

                    sk.interestOps(SelectionKey.OP_READ);
                } else if (key.isReadable()) {

                    try {
                        SocketChannel channel = (SocketChannel) key.channel();
                        int read = channel.read(buff);
                        if(read==-1)   key.cancel(); //正常断开
                        else {
                            buff.flip();             ByteBufferUtil.debugRead(buff);
                            buff.clear();
                        }
                    } catch (IOException e) {
                        //反注册 移除selector的 key ，因为断开会发生一个读事件
                        key.cancel();
                        throw new RuntimeException(e);

                    }

                }

            }
```

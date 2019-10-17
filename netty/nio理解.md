# nio理解
## nio三大核心
- 缓冲区Buffer:在nio中，所有数据都是用缓冲区处理的。实质上是一个数组。
- 通道Channel:Channel是一个通道，与流的不同处在于通道是双向的，读、写可以同时进行。
- 多用复用器Selector:提供选择已经就绪的任务的能力。Selector会不断地轮询注册在其上的Channel,如果某个Channel上面发生读或写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。
## nio实例TimeServer
### nio服务端主要创建过程
- 初始化ServerSocketChannel,所有监听客户端连接的父通道
```java
serverSocketChannl = ServerSocketChannel.open();json
serverSocketChannl.configureBlocking(false);
serverSocketChannl.socket().bind(new InetSocketAddress(port), 1024);
```
- 创建Rector线程，创建多路复用器，并将ServerSocketChannel注册到Reactor线程的多路复用器Selector上，监听连接操作
```java
selector = Selector.open();
serverSocketChannl.register(selector, SelectionKey.OP_ACCEPT);
```
- 轮询准备就绪的Key
```java
selector.select(1000);
Set<SelectionKey> selectionKeys = selector.selectedKeys();
Iterator<SelectionKey> it = selectionKeys.iterator();
SelectionKey key = null;
while (it.hasNext()){
    key = it.next();
    //handle(key);
}
```java
- 监听到有新的客户端接入时，处理新的接入请求，创建子通道并注册到Reactor线程的多路复用器Selector上，监听读操作。
```java
ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
SocketChannel sc = ssc.accept();
sc.configureBlocking(false);
sc.register(selector, SelectionKey.OP_READ);
```
- 读写操作都先经过缓冲区
```java
//读
ByteBuffer readBuffer = ByteBuffer.allocate(1024);
int readBytes = sc.read(readBuffer);
//写
ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
writeBuffer.put(bytes);
writeBuffer.flip();
channel.write(writeBuffer);
```
### 时序图
![Alt text](/pic/2.png)
## nio实例TimeClient
### nio客户端主要创建过程
- 初始化SocketChannel,并连接服务端
```java

```
- 创建Rector线程，创建多路复用器，判断是否连接成功。如果连接成功，则直接注册读状态到Reactor线程的多路复用器Selector上，如果没有连接成功，则注册OP_CONNECT状态到Reactor线程的多路复用器Selector上，监听TCP ACK应答
```java
if (connected) {
    socketChannel.register(selector, SelectionKey.OP_READ);               
} else {
    socketChannel.register(selector, SelectionKey.OP_CONNECT);
}
```
- 轮询准备就绪的Key
```java
selector.select(1000);
Set<SelectionKey> selectionKeys = selector.selectedKeys();
Iterator<SelectionKey> it = selectionKeys.iterator();
SelectionKey key = null;
while (it.hasNext()){
    key = it.next();
    //handle(key);
}
```
- 读写操作都先经过缓冲区
```java
//读
ByteBuffer readBuffer = ByteBuffer.allocate(1024);
int readBytes = sc.read(readBuffer);
//写
ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
writeBuffer.put(bytes);
writeBuffer.flip();
channel.write(writeBuffer);
```
### 时序图
![Alt text](/pic/3.png)
## java nio的演进
在早期的JDK1.4和1.5版本之前，JDK的Selector基于select/poll模型实现，它是基于IO复用技术的非阻塞IO,不是异步IO.在JDK1.5 update10和Linux core2.6以上版本，Sun优化了Selector的实现，它在底层使用epoll替换了select/poll，上层的API并没有变化，可以认为JDK NIO的一次性能优化，但是没改变IO模型。JDK1.7提供的NIO2.0新增了异步的套接字通道，它是真正的异步IO,在异步IO操作的时候可以传递信号变量，当操作完成之后会回调相关的方法，异步IO也被称为AIO。
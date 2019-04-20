# nio理解
## nio三大核心
- 缓冲区Buffer:在nio中，所有数据都是用缓冲区处理的。实质上是一个数组。
- 通道Channel:Channel是一个通道，与流的不同处在于通道是双向的，读、写可以同时进行。
- 多用复用器Selector:提供选择已经就绪的任务的能力。Selector会不断地轮询注册在其上的Channel,如果某个Channel上面发生读或写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。
## nio实例TimeServer
### nio服务端主要创建过程


## nio实例TimeClient
## java nio的演进
在早期的JDK1.4和1.5版本之前，JDK的Selector基于select/poll模型实现，它是基于IO复用技术的非阻塞IO,不是异步IO.在JDK1.5 update10和Linux core2.6以上版本，Sun优化了Selector的实现，它在底层使用epoll替换了select/poll，上层的API并没有变化，可以认为JDK NIO的一次性能优化，但是没改变IO模型。JDK1.7提供的NIO2.0新增了异步的套接字通道，它是真正的异步IO,在异步IO操作的时候可以传递信号变量，当操作完成之后会回调相关的方法，异步IO也被称为AIO。
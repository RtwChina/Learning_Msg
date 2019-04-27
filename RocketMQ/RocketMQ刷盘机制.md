**RocketMQ刷盘机制**

标签：【RocketMQ】


[TOC]


# 刷盘

1. https://www.jianshu.com/p/d06e9bc6c463



## RocketMQ的刷盘

![RocketMQ的两种刷盘](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-09-050547.png)

1. 可以看到同步和异步刷盘的区别在于将Send指令发送至虚拟机内存的时候，是直接返回还是等待硬盘刷新成功后再返回。





# 类结构







# 同步刷盘

![RocketMQ存储系统之同步刷盘](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-09-052439.png)

1. 左边的Producer, broker, CommitLog为工作线程。主要是将消息内容放到MappedFile内存中去。
   - 但是如果每次来一条消息都进行刷盘，那不是完犊子了。因此我们工作线程会把刷盘请求封装成GroupCommitRequest.
   - GroupCommitRequest会进行阻塞等待，刷盘成功后才会返回给工作线程。



## 代码

1. CommitLog##puMessage.
2. CommitLog##handleDiskFlush(result, putMessageResult, msg);
3. GroupCommitService##putRequest(): 将刷盘请求塞入到service中，等待GroupCommitService的服务线程同一刷盘，这时候工作线程会在这儿阻塞，知道相对应的刷盘请求成功。
4. GroupCommitService服务线程的刷盘：doCommit—》MappedFile##flush,
   -  this.fileChannel.force(false) , 直接缓存的刷盘，通知操作系统你好去刷盘了。
   - this.mappedByteBuffer.force();mapped缓存的刷盘，通知操作系统你好去刷盘了。



# 异步刷盘

![RocketMQ的异步刷盘](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-09-072019.png)



1. 异步刷盘有两种方式，结合我们是否开启堆外线程池的两种方式。当使用Mappedbuffer的时候可以直接进行PageCache的写入磁盘。

   - 当使用堆外内存池的时候就需要先将其buffer刷入到FileBuffer中(即PageCache中)，然后再将其刷入到磁盘。

2. 结合Java源码如图：


![RocketMQ存储系统之异步刷盘_详细类](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-10-135955.png)

- 橙色，即使用MappedByteBuffer可以直接在FlushCommitLogService中调用FileChannel#force将数据刷入到磁盘。
- 紫色，使用堆外内存，需要先通过CommitRealTimeService先将堆外内存导入到虚拟内存中，然后再通过FileChannel#force将数据刷入到磁盘。







# 总结

1. **虽然Broker配置成同步刷盘，但客户端不关心是否已经刷盘成功，同步刷盘策略退化成了异步刷盘策略**。
2. Consumer有可能在消息没有刷到磁盘的时候，就可以把消息拿到了，毕竟是有MappedFile缓存不是吗！

## 同步刷盘和异步刷盘的区别

1. 同步刷盘

   - 数据可靠性高

   - 同步刷盘性能比异步刷盘性能要低

   - 适用于金融等对数据可靠性要求高的场景

2. 异步刷盘

   - Broker的性能和吞吐量高

   - 客户端延时低

   - Broker异常关闭时，有少量消息丢失

## TransientStorePool

1. TransientStorePool即堆外内存，只有MessageStoreConfig##transientStorePoofEnable 为true，刷盘策略为异步刷盘，并且Broker为主节点时才启用TransientStorePool。
2. 一般有两种，有两种方式进行读写
3. 第一种：Mmap+PageCache的方式，==读写消息都走的是pageCache==，这样子读写都在pageCache里面不可避免会有锁的问题，在并发的读写操作情况下，会出现缺页中断降低，内存加锁，污染页的回写。
4. 第二种：DirectByteBuff(堆外内存)+PageCache的两层架构方式，这样子可以实现读写消息分离，==写入消息时候写到的是DirectByteBuff(堆外内存)==，==读消息走的是PageCache==（对于，DirectByteBuffer是两步刷盘，一步是刷到PageCache，还有一步是刷到磁盘文件中），带来的好处就是，避免了内存操作的很多容易堵的地方，降低了时延，比如说缺页中断降低，内存加锁，污染页的回写。
5. 因此我们在内存足够，也是使用异步刷盘的时候可以打开堆外内存。
























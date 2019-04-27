**RocketMQ消息映射**

标签：【RocketMQ】



# 1. 啥是消息映射

1. RocketMQ是把消息存储在本地文件系统中，但是肯定需要将消息的内容映射在内存中不然全是IO操作不是完犊子了。
2. 因此咱么的MappedFile就应允而生了，每一个CoomitLog、ConsumerQueue、IndexFile在本地文件中都是分多个文件存储的，那么相对于咱么的内存每个文件对应一个MappedFile实例对象。

![RocketMQ内存映射文件结构](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-07-014256.png)

3. 



# 2. 主要类对象

## AllocateRequest

1. AllocateRequest是内部类，代表了==分配请求==。实现了Comparable接口的，用于自定义分配请求在优先级队列的优先级。
2. 为什么还有优先级？

   - 一次AllocateRequest会创建两个MappedFile文件，其中第一个文件创建好之后咱么就能使用了。优先级就是干这个的。

3. 成员变量

   |      字段      | 类型                | 说明                                                         |
   | :------------: | ------------------- | ------------------------------------------------------------ |
   |    Filepath    | String              | 文件路径                                                     |
   |    fileSize    | Int                 | 文件大小                                                     |
   | CountDownLatch | CountDownLatch      | 用于实现分配映射文件的等待通知线程模型。初始值为1，0代表完成映射文件的创建 |
   |   mappedFile   | volatile MappedFile | 根据路径已经文件大小创建的映射文件                           |

   

## AllocateMappedFileService

1. AllocateMappedFileService是一个服务线程，外部服务会通过putRequestAndReturnMappedFile来创建新建文件映射请求。

   - 主要流程是创建一个AllocateRequest实例，然后塞到其中的requestTable和requestQueue中

2. 两个重要的属性：

   - ConcurrentMap<String, AllocateRequest> requestTable :
   - 用于保存当前所有待处理的分配请求，其中key是fiePath，value 是分配请求。如果分配请求被成功处理，即获取到映射文件，则requestTable中移除。


   - PriorityBlockingQueue<_AllocateRequest> requestQueue :
   - 分配请求队列，注意是优先级队列。从该队列中获取请求，进而根据请求创建映射文件。

3. 看一看分配文件映射的具体流程：

   ![AllocateMappedFileService分配文件映射的流程 ](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-08-013953.png)

   - 首先是CommitLog##putMessage会调用AllocateMappedFileService的putRequestAndReturnMappedFile来创建AllocateRequest实例。

     - 默认AllocateRequest的CountDownLatch的值为1。
     - putRequestAndReturnMappedFile会调用CountDownLatch的getCountDownLatch().await(waitTimeOut, TimeUnit.MILLISECONDS)。一直等到CountDownLatch的值为0。

   - 那么AllocateRequest实例是放入requestTable和requestQueue中的，其中requestTable的key是起始物理值，也就是文件的名称。

   - AllocateMappedFileService会有有一个服务线程从requestQueue中取相对应的AllocateRequest实例，当真正把文件创建好之后，才会执行CountDownLatch##countDown。那么就会唤醒应用线程的putRequestAndReturnMappedFile方法。

4. 总结：
	- 也就是说CountDownLatch用来做同步的作用。相当于FutureTask。
	
   - ConcurrentMap和PriorityBlockingQueue一次放一个实例，我的理解是map可实现去重，PriorityBlockingQueue可实现排序。相当于线程安全的TreeMap。





## ByteBuff

![MappedBuffer](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-08-083153.png)

- RocketMQ中有两种ByteBuff,一种是mappedByteBuff和WriteBuffer。
  - RocketMQ有两种刷盘方式分别对应两种ByteBuff。在图上就对应绿色路线和橙色路线。
- mappedByteBuffer就是对应mmap, 文件直接映射到内存中，我们修改内存就是修改文件。

1. 当使用this.fileChannel.map()进行对commitLog内存映射的时候获得mappedByteBuffer的时候，我们内存中只是存在虚拟内存中，但是物理内存很有可能并没有分配，
   - 所以说当我们读写mappedByteBuffer的时候很有可能会遇到==缺页的情况（还没有分配物理内存）==。因此预热就诞生了。
   - MappedFile的warmMappedFile==就是一个预热过程，防止出现缺页情况==。

2. warmMappedFile：具体的，先对当前映射文件的每个内存页写入一个字节0，当刷盘策略为同步刷盘时，执行强制刷盘，并且是每修改pages个分页刷一次盘
      * 然后将当前映射文件全部的地址空间锁定在物理存储中，防止其被交换到swap空间
      * 再调用madivse, 传入WILL_NEED策略，将刚刚锁住的内存预热，其实就是告诉内核，我马上就要用WILL_NEED这块内存
      * 但是当Linux发现内存现在有点吃紧，然后又发现有一块内存很空闲，那么就会把该内存放到SWAP区，这就很尴尬了，因此mlock就诞生了。

3. mLock: 将当前映射文件全部的地址空间锁定在物理存储中，==防止其他被交换到SWAP空间==。

4. 什么时候使用mmap或者堆外内存值：

   - 从transientStorePool中获取，消息先写入该buffer，然后再写入到fileChannel。

     * 只有MessageStoreConfig##transientStorePoofEnable 为true，刷盘策略为异步刷盘，并且Broker为主节点时才启用。也就是writeBuffer初始化的时候才能获取到值。

     * 什么时候属于transientStorePoofEnabl开启呢，看如下代码

       ```java
       public boolean isTransientStorePoolEnable() {
           return transientStorePoolEnable && FlushDiskType.ASYNC_FLUSH == getFlushDiskType()
               && BrokerRole.SLAVE != getBrokerRole();
       }
       ```

     * 否则使用的是mmap，详见appendMessagesInner 

   - 总结下：开启堆外内存开关，异步刷盘，Broker为主节点的时候才会使用堆外内存，否则使用mmap。



## mappedFIle

```java
// JVM中映射（mmap)的虚拟内存总大小，初始值为0
    private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);

    // 映射文件的个数，初始值为0
    private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);

    // 当前写入的位置，当值等于fileSize 时代表文件写满了。注意，这里记录的不是真正刷入
    // 磁盘的位置，而是写入到buffer的位置，初始值为0
    protected final AtomicInteger wrotePosition = new AtomicInteger(0);

    //ADD BY ChenYang 当前提交的位置所谓提交就是将writeBuffer的数据写到fileChannel的位置，初始值为 0
    protected final AtomicInteger committedPosition = new AtomicInteger(0);

    // 当前刷盘的位置，初始值为0
    private final AtomicInteger flushedPosition = new AtomicInteger(0);

    // 映射文件的大小，参照
    protected int fileSize;

    // 文件通道，以支持文件的随机读写。通过fileChannel将此通道的文件区域直接映射到内
    // 存中，对应的内存映射为可以直接通过 mmappedByteBuffer读写commitLog文件
    protected FileChannel fileChannel;

   /**
     * Message will put to here first, and then reput to FileChannel if writeBuffer is not null.
     *
     * 从transientStorePool中获取，消息先写入该buffer，然后再写入到fileChannel。
     * 只有MessageStoreConfig.transientStorePoofEnable 为true
     ，刷盘策略为异步刷盘，并且Broker为主节点时才启用。也就是writeBuffer初始化的时候才能获取到值。
     * 否则使用的是mmap，详见appendMessagesInner
     * 堆外内存
     */
    protected ByteBuffer writeBuffer = null;
```







# 源码分析

## 获取映射文件

1. 获取映射文件的代码调用流如下所示：
   - DefaultMessageStore#putMessage(MessageExtBrokerInner msg)
   - CommitLog#putMessage(final MessageExtBrokerInner msg)
   - MappedFileQueue#getLastMappedFile()



## 创建映射文件代码流

1. 创建映射文件代码流如下
   - CommitLog#putMessage(final   MessageExtBrokerInner(msg)
   - MappedFileQueue#getLastMappedFile(final long   startOffset)
   - MappedFileQueue#getLastMappedFile(final long   startOffset ,boolean  needCreate)
   - AllocateMappedFileService#putRequestAndReturnMappedFile(String   nextFilePath, String   nextNextFilePath, int  fileSize) 单独的服务线程

2. 和获取映射文件代码不同的是在getLastMappedFile重载方法中（带有参数），**还有一个关键点是在putMessageLock锁中运行的。也就是只有单线程操作**。



## 消息写入流

1. CommitLog#putMessage(final MessageExtBrokerInner msg)
2. MappedFile#appendMessage(final MessageExtBrokerInner msg, final AppendMessageCallback cb)
3. DefaultAppendMessageCallback#doAppend()
      - 写入消息会有两种方式，即两种ByteBuff，mmap和直接内存














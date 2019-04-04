**RocketMQ消息存储**

标签：【RocketMQ】



# 1. 概述消息存储

1. 消息存储大致分为三类：
   - 分布式KV存储(LevelDB、RocksDB、Redis)
   - NewSQL存储(TiDB)
   - 文件系统(RocketMQ、Kafka、RabbitMQ)

2. RocketMQ与Kafka的存放模式：![RocketMQ与Kafka的存放模式](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-04-131928.png)
   - Kafka针对每个TOPIC都存储一个文件。
   - RocketMQ则做了很多突破，把所有的Message都存储到一个文件中。

3. RocketMQ存储架构：

   ![RocketMQ存储架构图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-04-132502.png)

   - RocketMQ所有的Topic的消息内容都存储在CommitLog中。

   - 会有多个CosunmerQueue,其中每一项内容含有CommitLogOffSet(消息的物理地址)

     MessageSize(消息大小)、TagHashCode（Tag的hash值）

   - Consumer是从ConsumerQueue的minOffset到maxOffset消费过去的，consumerOffset是当前消费进度。

   - doDispatch是一个异步线程，会定时。。。













# 2. 名称介绍

1. RocketMQ中， 一种类型的消息会放到一个Topic 里，为了能够并行， 一般一个 Topic 会有多个 Message Queue （也可以 设置成一个）。
2.  Offset 是指某个Topic 下的一条消息在某个 Message Queue 里的 位置， 通过 Offset 的值可以定位到这条消息， 或者指示 Consumer 从这 条消息 开始向后继续处理。



## 2.1 DefaultMessageStore

1. 主要的Broker的存储 信息类。
   - CommitLog：CommitLog文件在Broker上的缓存消息，里面还有MappedFileQueue，这就是CommitLog各个文件的对象了。
   - consumeQueueTable： 所有的Topic中的所有ConsumerQueue.



# 3. 存储结构类

![ROcketMQ存储类图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-04-133441.png)

- MappedFile（映射文件）：是重要的CommitLog、ConsumerQueue、IndexFile的组成部分。一个MappedFile类就相当于一个本地文件。可以这么理解

## Consumer

![ConsumerQueue存储结构](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-04-142523.png)



## Index
1. Index文件存放结构如下：
![Index结构](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-05-013529.png)

2. Index文件的具体内容如下

![Index文件格式](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-05-023454.png)

- 索引槽。即slot，默认每个文件配置500万个，每个slot是4位的int类型数据。计算slotPos = Math.abs(keyHash)%hashSlotNum。
- 消息在IndexFile中的偏移量absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos *HASH_SLOT_SIZE
- 也就是说我们可以通过keyHash值快速找到具体的索引内容，这其实是hashMap的文件形式实现，key就是消息的key（一般都是在tool中使用）。
  - 如果hash冲突的话，看索引内容的下一个索引槽地址就知道这就是一个LinkedList。

- 注意：IndexFile中会根据==uniqKey==（客户端生成的MessageID，也来源于Message的Properties）、==keys==来进行构建IndexFile





# 3. 消息存储结构

1. RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似于索引文件，存储的是指向物理存储的地址。

![RocketMQ消息的结构图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-21-030848.png)

2. 设计成这样子有以下几个好处：
   - CommitLog 顺序写，可以大大提高写入效率。
   - 虽然是随机读，但可以利用操作系统的PageCache机制，可以批量地从磁盘读取。话说缓存IO也是用了PageCache机制。
   - 为了保证完全的顺序写，需要ConsumeQueue这个中间件结构。，因为ConsumeQueue里只存偏移量信息锁，所以可以全部读入内存。



# 入口



## Broker收到请求入口

1. BrokerController##registerProcessor中有很多Request的请求Code，可以根据这些Code来找到到底是哪一个类进行处理的。

2. 对于普通消息发送，Broker的处理逻辑大致如下：

   - SendMessageProcessor.processRequest

   - DefaultMessageStore.putMessage(MessageExtBrokerInner msg)

   - CommitLog.putMessage(MessageExtBrokerInner msg)。里面会有对延迟消息的处理
     - 存储到MapedFileQueue的MapedFile
     - 同步刷盘：GroupCommitService(独立的线程)
     - 异步刷盘：CommitRealTimeService/FlushCommitLogService(独立的线程)



## ConsumerQueue入口

1. DefaultMessageStore#doReput(独立线程)，会一直循环取commitLog中的数据信息，然后构造成 ConsumerQueue和IndexFile。
   - doReput中的SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset); 会从CommitLog中的MapperFiler中==取出还没有构建成consumerQueue的消息==。
   - 后续会将SelectMappedBufferResult转化为DispatchRequest，里面带有消息的所有信息。
2. 具体的dipatch是在DefaultMessageStore##doDispatch中，会循环CommitLogDispatcher的dispatch,因此就分为ConsumerQueue和IndexFile的dispatch。
3. 下面来分析下CommitLogDispatcherBuildConsumeQueue.dispatch(DispatchRequest request)：
   - ConsumeQueue.putMessagePositionInfo：构建供消费端使用的逻辑队列数据，即ConsumerQueue。
   - FlushConsumeQueueService.doFlush()：这也是一个独立的线程，会将ConsumerQueue中的信息异步刷盘。

4. ==最后内存中ConsumerQueue都存在于DefaultMessageStore的consumeQueueTable中。==



![消息写入流程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-07-014717.png)










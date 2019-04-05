**RocketMQ消息消费_消费**

标签：【RocketMQ】



[TOC]

# 1. 消息消费

1. 回顾下消息拉取，PullMessageService负责对消息队列进行消息拉取，从远端服务器拉取后存入ProcessQueue消息队列处理队列中，然后调用ConsumerMessageService来进行消息消费，使用线程池来进行消息消费
   - 将消息拉取和消息消费解耦开来。

2. 该篇文章主要讲的是Consumer对收到消息响应后的消息消费部分



# 2.组件

## 2.1 ConsumerMessageService

```java
 // 并发消息业务事件类
    private final MessageListenerConcurrently messageListener;
    private final BlockingQueue<Runnable> consumeRequestQueue;

    // consumeExecutor:消费端消费线程池
    private final ThreadPoolExecutor consumeExecutor;
    private final String consumerGroup;

    // 添加消费任务到consumeExecutor延迟调度器
    private final ScheduledExecutorService scheduledExecutorService;

    // 定时删除过期消息线程池
    private final ScheduledExecutorService cleanExpireMsgExecutors;
```

1. 主要含有三个线程池，用来处理消息消费。
2. 每一个Consumer实例都会创建一个新的ConsumerMessageService，总结下：
   - 一个JVM实例中，只会有一个MQClientInstance实例，那么就只会有一个线程去不停地从ProcessQueue中拿到消息消费请求对象，然后提交给Broker。
   - 一个JVM实例中，如果创建多个不同ConsumerGroupName的Consumer实例，则他们的消息消费线程池是不同的。



## 2.2 延迟级别

1. RocketMQ 支持定时消息，但是不支持任意时间精度，仅支持特定的 level，例如定时 5s， 10s， 1m 等。其中，level=0 级表示不延时，level=1 表示 1 级延时，level=2 表示 2 级延时。
2. messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h







# 3. 流程

## 3.1 submitConsumeRequest

1. 该方法的作用主要是向ConsumerMessageService提交返回信息，其中会有两步骤，我们来看看：
2. ConsumeMessageBatchMaxSize：并发消费时，一次消费消息的数量，默认为1，假如修改为50，此时若有100条消息，那么会创建两个线程，每个线程分配50条消息。
3. 一般提交的msg数量(即Consuemer拉取条数)是32条，因此默认会走到第二部分

1. 第一部分：当提交的msg数量小于ConsumeMessageBatchMaxSize时，构造一个ConsumeRequest然后提交到consumeExecutor中去。
2. 第二部分：当提交的msg数量大于ConsumeMessageBatchMaxSize时，则对Msg进行分页，每页ConsumeMessageBatchMaxSize条消息，创建多个ConsumeRequest任务并提交到消费线程池。
3. 提交到线程池后，就运行ConsumeRequest中的RUN方法。



## 3.2 ConsumeRequest

1. 该ConsumeRequest指的是ConsumeMessageConcurrentlyService中的内部类;
2. Step1:判断processQueue的isDropped，防止消息重新加载时，如果该消息队列被分配给消费组内其他消费者，阻止消费者继续消费。
3. Step2: 如果消费者注册了消息消费者hook(钩子函数，在消息消费之前，消费之后执行相关方法)
4. Step3: 设置消息的重试主题，并开始消费消息，并返回该批次消息消费结果。
5. Step4: 执行Consumer的监听器，也就是用户在业务上注册的MessageListener。
   - 当用户实现consumeMessage的时候==return null的话，则Consumer会默认重试==，所以说在正常情况下return CONSUME_SUCCESS；
6. Step5: 根据是否出现异常等，判断处理结果，设置returnType
7. Step6: 如果消费者注册了消息消费者hook(钩子函数，在消息消费之前，消费之后执行相关方法)。
8. Step7: 判断processQueue的isDropped，进行消息消费成功的一些处理方法。
   - 也就是说当消息消费进入到Step4之后，若出现宕机了，然后重新负载那么有可能消息会被重复消费。
   - 后续会去执行ConsumeMessageConcurrentlyService##processConsumeResult

###processConsumeResult 

1. 根据消息监听器返回的结果，计算ackIndex，如果返回CONSUME_SUCCESS，设置AckIndex=msgs.size()-1。
   - 如果是RECONSUME_LATER；ACK = -1。
2. 如果是广播模式监听器返回RECONSUME_LATER，则消息不会被消费，只是输出下日志。
3. 如果是集群模式，只有在业务返回RECONSUME_LATER的时候，该消息需要再发送ACK消息。
   - 若ACK消息发送失败，则直接将发送ACK失败的消息再次封装为ConsumerRequest。延迟5S后再次消费。
   - 若ACK消息发送成功，则该消息会延迟消费。
4. 从ProcessQueue中移除这些消息，返回的偏移量是移除该批消息后最小的偏移量，然后该偏移量更新消息消费进度，以便消费者重启后能从上一次消费进度消费，避免消息重复。
   - 注意：当消息为RECONSUME_LATER，消息消费也会向前推进，因为在RECONSUME_LATER返回给Broker时，RocketMQ会出现生成一条内容一样的消息，但是拥有新的msgId，也就是拥有一个全新的队列偏移量。



## 3.3 ACK消息

1. ACK消息是在Consumer集群模式下,==消息监听器的消费结果为RECONSUME_LATER，则需要将这些消息发送给Broker延迟消息==。消息监听器的结果是由应用开发指定的，当我们返回RECONSUME_LATER的时候，Consuemr任然会删除本地的消费进度，因为Broker会再次通过重试主题把消息发给我们。
2. 最终是先封装成ConsumerSendMsgBackRequestHeader然后通过remotingClient.invokeSync的==同步发送==出去的。
   - 需要注意的是ConsumerSendMsgBackRequestHeader中的delayLevel，延迟级别。默认是0，但是我们可以手动定制。

3. 当Broker收到ACK消息，则会根据原先的消息创建一个新的消息对象，重试消息会拥有自己的唯一消息Id(msgId)并存入到commitLog文件中。并不会去更新原先的消息。还有一个要点就是会把主题修改为==重试主题( %RETRY% + 消费组名称 )。==
4. 如果该消息的延迟级别大于0，则将其主题再次修改为定时任务主题("SCHEDULE_TOPIC_XXXX")。



# 4. 消费进度储存

1. 广播模式：消息进度存储在本地，也就是和消费者绑定。会5S一次持久化。实现在LocalFileOffsetStore。

2. 集群模式：很明显需要存储在Broker端，因此是通过通过网络连接来更新消息消费进度。实现在RemoteBrokerOffsetStore的offsetTable中。

   - 如果需要从磁盘读取消息消费进度，则会通过网络请求发送QUERY_CONSUMER_OFFSET。

3. 消费者线程池每处理完一个消息消费任务(ConsumerRequest)时会从ProcessQueue中移除本批消费的消息，==并返回ProcessQueue中最小的偏移量。==用该偏移量更新消息队列消费进度，也就是说更新消费进度与消费任务中的消息没什么关系。

   - 因此这也是会造成消息重复

4. 回顾一下：msgTreeMap的类型，TreeMap,按消息的offset升序排序，返回的result,如果treemap中不存在任何消息，那就返回该处理队列最大的偏移量+1，如果移除自己本批消息后，处理队列中，还存在消息，则返回该处理队列中最小的偏移量，==也就是此时返回的偏移量有可能不是消息本身的偏移量，而是处理队列中最小的偏移量==。

   - 优点：防止消息丢失（也就是没有消费到）。
   - 缺点：会造成消息重复消费。

   



# 5. 定时消息机制

1. RocketMQ 并不支持任意时间精度的定时调度，因为如果要实现的话则需要在Broker层做消息排序**(参考JDK的ScheduledExcutorService的实现原理)**，加上持久化的话会造成比较大的性能消耗
2. 定时消息机制实现主要是在ScheduleMessageService中，调用顺序为构造方法-》load()->start()
   - load主要是将延时级别的数据从文件转到内存，ScheduleMessageService#delayLevelTable。
   - Start: 主要是开启延迟队列的定时任务。



## 5.1 定时调度逻辑

1. 在ScheduleMessageService的start中，会对每一个延时级别创建一个DeliverDelayedMessageTimerTask内部类并执行定时任务。
2. 那我们的关注点又到了DeliverDelayedMessageTimerTask##RUN方法，其中只调用了executeOnTimeup方法。下面着重分析下executeOnTimeup方法：
3. STEP1：主要是去CommitLog中获取对应延迟级别，offset的消息队列。
4. STEP2：==循环消息队列消息，清除延迟级别属性并回复原先的主题、队列，再次创建一条新的消息存入到commitLog中并转发到消息消费队列供消息消费者消费==。



## 5.2 总结

1. 当Broker收到任何带有延时级别的消息时，会将该消息属性替换成延时级别的属性，然后按照延时级别添加到延时级别队列中。
   - RocketMQ使用TImer类来存储定时任务执行的TimeTask, 每一个延时级别都有一个TimeTask实例（DeliverDelayedMessageTimerTask），那么也就是说Timer实例中的==堆队列具有18个延时任务==，在达到相应时间点的时候就会自动触发延时任务。
2. 每个延时级别队列都会有延时定时任务来定时处理延时级别队列。当获取到某一个延迟任务已经可以发送给Consumer的时候，会将其从延迟任务队列中取出来，处理下被替换掉的属性放到再次放到commitLog中。
3. 问题比较抽象的是为什么ScheduleMessageService中使用Timer来做延迟任务？





# 6. 消息过滤机制

1. RocketMQ支持表达式过滤与类过滤两种模式。
   - 其中表达式又分为TAG和SQL92。
   - 类过滤模式允许提交一个过滤类到FilterServe。

2. RocketMQ消息过滤方式不同于其他消息中间件，是在订阅时做过滤。
3. 下图是ConsumerQueue的存储格式：
   - 之所以不存储tag字符串，是因为将ConsumerQueue设计为定长结构，加快消息消费的加载性能。
   - 在Broker端中拉取消息时，遍历ConsumerQueue只对比消息tag的hashcode，如果匹配则返回。
   - Consumer收到消息后，同样需要先对消息进行过滤，只是此时比较的消息tag的值而不再是hashcode。

![ConsumerQueue的存储格式](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-03-061618.png)



## 6.1 入口

1. 订阅是通过consumer的实例调用subscribe()接口，该方法中会将过滤信息封装成SubscriptionData，然后放到RebalanceImpl对象中。

2. DefaultMQPushConsumerImpl##PullMessage会通过：

   ```java
           final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
   
   ```

取出相对应的过滤信息对象，然后将过滤信息封装在ProcessRequest中传递到Broekr。

3. 在Broker中的消息过滤代码在：DefaultMessageStore##getMessage()中，具体代码如下：

   ```java
   // 消息过滤 主要是为TAG服务
   if (messageFilter != null
       && !messageFilter.isMatchedByConsumeQueue(isTagsCodeLegal ? tagsCode : null, extRet ? cqExtUnit : null)) {
       if (getResult.getBufferTotalSize() == 0) {
           status = GetMessageStatus.NO_MATCHED_MESSAGE;
       }
       continue;
   }
   
   // 从CommitLog中把数据取出来
   SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
   if (null == selectResult) {
       if (getResult.getBufferTotalSize() == 0) {
           status = GetMessageStatus.MESSAGE_WAS_REMOVING;
       }
   
       nextPhyFileStartOffset = this.commitLog.rollNextFile(offsetPy);
       continue;
   }
   
   // 消息过滤 主要是为了SQL92服务
   if (messageFilter != null
       && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null)) {
       if (getResult.getBufferTotalSize() == 0) {
           status = GetMessageStatus.NO_MATCHED_MESSAGE;
       }
       // release...
       selectResult.release();
       continue;
   }
   ```

   - 也就是说TAG是在ConsumerQueue中就能过滤的，SQL需要从commitLog中取出相对应的消息才能进行过滤。因此推荐用TAG啊。







# 7. 顺序消费机制

>1. https://blog.csdn.net/meilong_whpu/article/details/77076591#t2
>2. http://www.iocoder.cn/RocketMQ/message-send-and-consume-orderly/

1. RocketMQ支持局部消息顺序消费，可以确保同一个消息消费队列中的消息被顺序消费。
   - 顺序消息消费与并发消息消费的第一个关键区别：顺序消息在创建消息队列拉取任务时需要在Broker服务器锁定该消息队列。

2. Consumer严格顺序消费，需要通过三把锁保证严格顺序消费：
   - Broker消息队列锁(分布式锁)：
     - 集群模式下：consumer从Broker获得锁后，才能进行消息拉取、消费。
     - 广播模式下：Consumer无需该锁。
   - Consumer消息队列锁（本地锁）：Consumer获得该锁才能操作消息队列。也就是processQueue.setLocked(true)
   - Consumer消息处理队列消费锁（本地锁）：Consumer获得该锁才能消费消息队列。（这个锁有点水，主要是在移除不需要的队列相关的信息操作和消费消息时候的加锁）



## 7.1 锁定消息队列

1. 在RebalanceImpl中的rebalanceByTopic会调用到其lock函数。当然是只有顺序消费才有可能去锁定消息队列的哦。
2. Lock函数主要是连接至Broker获得指定消息队列的分布式锁，若获得成功则设置processQueue.setLocked(true)。



## 7.2 移除消息队列

1. 集群模式下，Consumer移除自己的消息队列时，会向Broker解锁该消息队列(广播模式下不需要)。详见RebalancePushImpl##removeUnnecessaryMessageQueue。
2. Consumer在移除消息队列时也会检查一下pq.getLockConsume()是否加锁，即获取**消息队列消费锁**，主要是为了避免和消息队列消费冲突。
   - 如果获取锁失败，则移除消息队列失败，等待下次重新分配消费队列时，再进行移除。
   - 如果未获得锁而进行移除，则可能出现另外的 Consumer和当前 Consumer 同时消费该消息队列，导致消息无法严格顺序消费。为什么呢?因为未获得锁则返回flase, 然后RebalanImpl就不会将processQueueTable中移除相对应的MessageQueue。然后就不会将负载均衡的结果发送给对应Broker。



## 7.3 消费消息队列

![消息消费流程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-04-014601.png)



### ConsumeRequest

1. STEP1：获得Consumer 消息队列锁。

2. STEP2：从消息处理队列顺序获得消息。**和并发消费获得消息不同。并发消费请求在请求创建时，已经设置好消费哪些消息。**

3. STEP3：获得Consumer消息处理队列消费锁。相比【Consumer消息队列锁】，其粒度较小。这就是上文提到的❓**为什么有Consumer消息队列锁还需要有 Consumer 消息队列消费锁呢**的原因。
4. STEP4：**执行消费**。
5. STEP5：处理消费结果。

### 处理消息结果

1. 顺序消费消息结果 (ConsumeOrderlyStatus) 有四种情况：

- SUCCESS ：消费成功**但不提交**。
- ROLLBACK ：消费失败，消费回滚。
- COMMIT ：消费成功提交并且提交。
- SUSPEND_CURRENT_QUEUE_A_MOMENT ：消费失败，挂起消费队列一会会，稍后继续消费。

考虑到 ROLLBACK 、COMMIT 暂时只使用在 MySQL binlog场景，官方将这两状态标记为@Deprecated。当然，相应的实现逻辑依然保留。

2. 在**并发消费**场景时，==如果消费失败，Consumer 会将消费失败消息发回到 Broker重试队列，跳过当前消息，等待下次拉取该消息再进行消费==。

   - ==但是在**完全严格顺序消费**消费时，这样做显然不行。也因此，消费失败的消息，会挂起队列一会会，稍后继续消费。==

   - 不过消费失败的消息一直失败，也不可能一直消费。当超过消费重试上限时，Consumer 会将消费失败超过上限的消息发回到 Broker 死信队列。



# 8. 消息重试

![消息重试过程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-16-011304.png)

- Consumer会通过心跳的形式把该Topic所订阅的信息发送至Broker, 因此Broker含有
- 用户只要是return RECONSUMER_LATER就可以了。
1. 完整的一套流程有三个消息：
   - 首先Broker先把消息发送给Consumer进行消费。
   - 然后Consuemer收到消息后，返回RECONSUMER_LATER给Broker，Broker收到后将该消息修改成延迟消息类型，修改其Topic。
   - 当延迟时间到了之后就会再次将该Topic拿出来，恢复原来的Topic属性，再次进行发送。
















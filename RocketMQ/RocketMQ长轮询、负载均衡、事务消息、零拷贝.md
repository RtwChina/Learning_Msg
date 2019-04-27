**RocketMQ长轮询、负载均衡、事务消息、零拷贝**

标签：【RocketMQ】



[TOC]

# 1. 消息拉取长轮询简介

1. 长轮询本身不是一个真正的推送，长轮询是一个传统轮询技术的变种，它允许模拟推送技术。
2. RocketMQ并没有实现真正推模式，而是通过消费者主动向消息服务器拉取消息。
3. 那么就诞生了大名鼎鼎的消息长轮询机制，当消息向RocketMQ发送消息拉取时，消息并未到达消费队列，会有两种处理方式：
   - 未开启长轮询：会在服务端等到shortPollingTimeMills时间后(挂起)再去判断消息是否已经到达消息队列，若未到达则提示消息拉取客户端PULL_NOT_FOUND。
   - 开启长轮询：==RocketMQ一方面会每5S轮询检查一次消息是否可达，同时一有新消息到达后立马通知挂起线程再次验证新消息是否是自己感兴趣的消息==。如果是则从CommitLog文件提取消息返回给拉取客户端。否则直到挂起超时(超时时间通过DefaultMQPullConsumer#setBrokerSuspendMaxTImeMills设置)。
4. 通过在Broker端配置longPollingEnable为true来开启长轮询。



## 1.1 简介

![RocketMQ的长轮询](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-18-141046.png)

- 当Broker端收到Consumer的PullRequest请求时，发现现在没有任何可以返回的数据，那么可以看到左侧的PR，将该PullRequest来Hold住，放到PullRequestTable的队列中。
- 当Producer发送Topic至Broker端后，sendMessageProcesser将该数据存放在CommitLog(PageCache 或 堆外内存池)中，ReputMessageService会调用arriving来通知相应的PullRequestTable有消息来啦，起床干活了。
- brokerAllowSuspend: Broker端 表示是否允许Broker可以把这个请求Hold住。在PullRequest第一次发送至Broker端才会设置为true,虽然当ReputMessageServcice再次塞进去后就会设置为flase。
- 当然Broker当没有数据的时候也不会一直hold住PullRequest, 当hold超时之后也会放生PullRequest。
- pullKernelImpl：sysFlag参数可以表示是否消息拉取时支持挂起



## 1.2 入口

1. 在PullMessageProcessor##processRequest()。参数brokerAllowSuspend表示Broker是否支持未找到消息时挂起。
2. RocketMQ轮询机制由两个线程共同来完成：
   - 1) PullRequestHoldService: 每隔5s重试一次。当开启长轮询情况下会是隔5S尝试一次，判断新消息是否到达。如果未开启长轮询则默认等待1s再次尝试。
   - 2) DefaultMessageStore##ReputMessageService, 该线程主要是负责在CommitLog将消息转发给ConsumeQueue、IndexFile。如果Broker端开启了长轮询并且角色主节点，则最终将调用PullRequestHoldService线程的NotifyMessageArriving方法唤醒其线程，保证长轮询模式使得消费拉取能够实现。



## 1.3 总结

1. 长轮询主要是防止Broker没有相应Topic消息的时候，Consumer还一直发送PullRequest的消息。原因如下：

   - 明确Consumer发送消息消费请求是需要PullRequest的，但是一旦发送Pull请求给Broker之后，就从阻塞队列中把PullRequest拿出来了，知道Broker异步回调才会通过PullResponse组装成下一次的PullRequest。
   - ==Broker可以把PullRequest拿到，当没有任何消息的时候可以不立即返回PullRequest，hold在PullRequestHoldService的pullRequestTable中，等待线程的自我轮序来取消息。==

   



# 2. 消息队列负载

1. 一直PullMessageService在启动时由于LinkedBlockingQueue<_PullRequest> pullRequestQueue中没有_PullRequest的话会一直阻塞。
2. 问题1：PullRequest什么时候创建并加入到pullRequestQueue中以唤醒PullMessageService线程。
3. 问题2：集群内多个消费者是如何负载主题下的多个消费对立，并且如果有新的消费者加入时，消息队列如何重新分布。
4. RocketMQ的消息队列重新分布是由RebalanceService线程来实现的，一个MQClientInstance持有一个RebalanceService实现，并随着MQClientInstance的启动而启动。
5. 明确一点每个MQPushConsumer有一个的RebalanceImpl实例对象。



## 2.1 入口

1. MQClientInstance#doRebalance() —>RebalanceImpl# doRebalance—> RebalanceImpl#rebalanceByTopic()
   - 其中MQClientInstance的consumerTable指的是当前JVM实例，所启动的Consuemr对象，按照consumerGroup来作为Key值。
2. RebalanceService中的RUN函数会每隔20S运行一次MQClientInstance# doRebalance方法。
3. MQClientInstance# doRebalance遍历当前JVM实例的所有Consuemr对象，然后逐个执行MQConsumerInner##doRebalance()。
   - MQConsumerInner也就是DefaultMQPullConsumerImpl，或者DefaultMQPushConsumerImpl。
4. DefaultMQPushConsumerImpl##doRebalance()在当前Consumer不暂停的情况下运行RebalanceImpl##doRebalance()。
5. RebalanceImpl##doRebalance()遍历当前所订阅的所有Topic，执行RebalanceImpl##rebalanceByTopic()。



## 2.2 RebalanceImpl##rebalanceByTopic()

1. 消费者模式的集群和广播分为两种。因此负载均衡在该方法中也会分成两个方式来达成。先来分析下集群模式情况下的负载均衡。
2. STEP1：从主体订阅信息缓存表中获取主题队列信息。
3. STEP2：随机选择一个Broker，发送请求从Broker中该消费组内当前所有的消费者客户端ID。
   - 没什么Broker中具有所有的消费者客户端信息呢！因为每一个Consumer都会发送心跳信息给每一个Broker。所以说Broker中具有所有的消费者信息。
4. STEP3：对该Topic相关的所有MessageQueue和ConsumerId进行排序，比较重要，
   需要保证同一个ConsuemrGroup中的RebalanceImpl的列表是相同。
   - 当Broker检测到有Consumer实例变化的时候，会通知所有的Consumer再次进行负载均衡，保证大家的MessageQueue和ConsumerId是相同的。
5. STEP4：负载均衡算法：AllocateMessageQueueStrategy
   - 算法1：AllocateMessageQueueAveragely，平均分配！也就是如果有q1,q2,q3,q4,q5,q6,q7; 三个Consumer: q1,q2,q3；q4,q5,q6；q7
   - 算法2：AllocateMessageQueueAveragelyByCircle，平均轮询分配！也就是如果有q1,q2,q3,q4,q5,q6,q7; 三个Consumer: q1,q4,q7；q2,q5,q8；q3
   - 其他
   - 主要算法的原则是，==Conusmer可以订阅多个MessageQueue,但是一个MessageQueue只能被一个Consuemr订阅。==

6. STEP5：调整拉取队列，如果当前新增了MessageQueue,则会新建PullReques来放到PullMessageService中。



## 2.3 总结

1. 问题1：PullRequest啥时候加载到PullMessageService中，以唤醒PullMessageService线程。
   - 负载均衡的时候，如果发现有新分配的消息队列会创建对应的PullRequest对象放到PullMessage中去。
2. 问题2：集群内多个消费者是如何负载主题下的多个消费队列。
   - 会从Broker中查询出当前Topic对于的所有消费者，进行排序后进行负载均衡。其中当Broker收到Consumer下线的时候，也会触发负载均衡。
3. ClientRemotingProcessor##processRequest中当收到某一个Consuemr调整的时候会调用resetOffset->resume，其中也会调用负载均衡算法。
   - ==所以我认为Consumer进行负载均衡很有可能多个之间是不同步的，因此非常有可能一个ConsumerMessage被多个Consumer消费，虽然只能在短时间内。不过这只会导致重复消费。==
4. 广播消费：每个消费者消费Topic下的所有队列。
5. Producer端，每个实例在发消息的时候，默认会轮询所有的message queue发送，以达到让消息平均落在不同的queue上。而由于queue可以散落在不同的broker，所以消息就发送到不同的broker下，如下图：

![PullMessageService线程与RebalanceService线程交互图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-25-091723.jpg)







# 3. 零拷贝







## 入口

PullMessageProcessor##processRequest：384行





# 4. 事务消息

![RocketMQ的事务消息](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-25-013218.png)

- 下面的方式当然不可取啦！
  - 原因1：网络随时会发生消息，有可能消息在Broker上落盘成功，但是收到的Producer的Response是失败的，那么其实是把消息发过去了，但是Producer以为没成功。
  - 原因2：网商延迟，很有可能发消息的操作要很久，导致很大一个事务都完成不了，产生超时。







## 4.1 交互设计

![RocketMQ分布式事务交互设计和类设计](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-27-013204.png)

1. Producer发送Half(prepare)消息到broker；
2. half消息发送成功之后执行本地事务；
3. （由用户实现）本地事务执行如果成功则返回commit，如果执行失败则返回roll_back。
4. Producer发送确认消息到broker（也就是将步骤3执行的结果发送给broker），这里可能broker未收到确认消息，下面分两种情况分析：

- **如果broker收到了确认消息：**

> 如果收到的结果是commit，则broker视为整个事务过程执行成功，将消息下发给Conusmer端消费；

> 如果收到的结果是rollback，则broker视为本地事务执行失败，broker删除Half消息，不下发给consumer。

- **如果broker未收到了确认消息：**

> broker定时回查本地事务的执行结果；

> （由用户实现）如果本地事务已经执行则返回commit；如果未执行，则返回rollback；

> Producer端回查的结果发送给broker；

> broker接收到的如果是commit，则broker视为整个事务过程执行成功，将消息下发给Conusmer端消费；如果是rollback，则broker视为本地事务执行失败，broker删除Half消息，不下发给consumer。如果broker未接收到回查的结果（或者查到的是unknow），则broker会定时进行重复回查，以确保查到最终的事务结果。

**补充**：对于过程3，如果执行本地事务突然宕机了（相当本地事务执行结果返回unknow），则和broker未收到确认消息的情况一样处理。

## 4.2 实现

1. 实现TransactionListener接口，自定义方法。
   - TransactionCheckListener接口定义了检查本地事务状态的行为。在Producer发送消息成功之后，会调用executeLocalTransaction来检查本地事务是否成功提交
   - 判断成功的方式是按照executeLocalTransaction的返回值。
2. 创建TransactionMQProducer对象，

```java
TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();
TransactionMQProducer producer = new TransactionMQProducer(producerGroup);
producer. setTransactionCheckListener(transactionCheckListener);
producer.sendMessageInTransaction(msg, null);

```

4. 详见MyRocketMQ的org.apache.rocketmq.example.transaction.TransactionProducer

### 具体编写

1. 一般我们实现TransactionListener的两个方法：
   - executeLocalTransaction：检查本地事务是否提交成功，
   - checkLocalTransaction：回查，检查本地事务是否提交成功，
2. 两个方法都有一个入参是 Message msg，也就是发送的消息。

2. 一般是特别搞一张表，如果本地事务提交成功，则插入一条数据并塞入msg的properties中，那么我们检查本地事务是否提交成功，就可以检查事务表是否有该条数据。







## 4.3 源码分析

1. Broker中会收到事务消息，具体是在SendMessageProcessor##processRequest-》》sendMessage。

   ```java
   if (traFlag != null && Boolean.parseBoolean(traFlag)) {
   .....
   // 处理事务消息的Prepare消息。
   putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);
   } else {
   // 其他一般的消息
   putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
   }
   ```

2. 对消息落盘前的处理：

   ```java
   // 落盘的一些准备工作
       private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {
           // 先将真实的Topic，queueId退避到消息的properties里面，然后设置TOPIC为RMQ_SYS_TRANS_HALF_TOPIC,QueueId为0
           // 达到两个目的：1. 消息存储无差别。2.对原TOPIC的消费者不可见
           MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
           MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,
               String.valueOf(msgInner.getQueueId()));
   
           // 将消息的sysFlag重置为0，这个操作也非常重要。因为CommitLogDispatcherBuildConsumerQUeue在更新ConsumerQueue时会跳过TRANSACTION_PREPARED_TYPE
           // 和TRANSACTION_ROLLBACK_TYPE的消息，这里重置为0后，就不会影响对RMQ_SYS_TRANS_HALF_TPOIC的ConsuemrQueue的更新，也就是让更新ConsumerQueue时不会跳过它。
           msgInner.setSysFlag(
               MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
           msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
           msgInner.setQueueId(0);
           msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
           return msgInner;
       }
   ```

3. 当Broker收到producer端发来的消息事务的提交，回滚状态时，EndTransactionProcessor##processRequest()

   ```java
   // 消息commit
           if (MessageSysFlag.TRANSACTION_COMMIT_TYPE == requestHeader.getCommitOrRollback()) {
               result = this.brokerController.getTransactionalMessageService().commitMessage(requestHeader);
               if (result.getResponseCode() == ResponseCode.SUCCESS) {
                   RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
                   if (res.getCode() == ResponseCode.SUCCESS) {
                       // 还原Half,message真实的Topic、QueueId
                       MessageExtBrokerInner msgInner = endMessageTransaction(result.getPrepareMessage());
                       msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), requestHeader.getCommitOrRollback()));
                       msgInner.setQueueOffset(requestHeader.getTranStateTableOffset()); msgInner.setPreparedTransactionOffset(requestHeader.getCommitLogOffset());     msgInner.setStoreTimestamp(result.getPrepareMessage().getStoreTimestamp());
   
                       // 再次把消息存入commitlog来进行处理。
                       RemotingCommand sendResult = sendFinalMessage(msgInner);
                       if (sendResult.getCode() == ResponseCode.SUCCESS) {
                           // 删除half消息，其实不是正在的删除，只不过把该消息放到RMQ_SYS_TRANS_OP_HALF_TOPIC队列中，用于到时候在回查逻辑，表示该消息的状态已经被确认了
                           this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
                       }
                       return sendResult;
                   }
                   return res;
               }
           // 消息rollBack
           } else if (MessageSysFlag.TRANSACTION_ROLLBACK_TYPE == requestHeader.getCommitOrRollback()) {
               result = this.brokerController.getTransactionalMessageService().rollbackMessage(requestHeader);
               if (result.getResponseCode() == ResponseCode.SUCCESS) {
                   RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
                   if (res.getCode() == ResponseCode.SUCCESS) {
                       // 删除half消息，其实不是正在的删除，只不过把该消息放到RMQ_SYS_TRANS_OP_HALF_TOPIC队列中，用于到时候在回查逻辑，表示该消息的状态已经被确认了
                       this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
                   }
                   return res;
               }
           }
   ```

   - 简而言之就是：收到Commit指令时，将消息恢复到原来的状态(Topic，QueueId),设置sysFlag为Commit状态，==然后再塞入到CommitLog中。==再将该消息存入到OP_HALF_TOPIC队列中，主要是为回查排除(说明该消息我已经收到确认消息了)
   - 收到RollBack后，将该消息存入到OP_HALF_TOPIC队列中，主要是为回查排除(说明该消息我已经收到确认消息了)。


































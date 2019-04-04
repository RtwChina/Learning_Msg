**RocketMQ总结**

标签：【RocketMQ】



[TOC]

# 大致结构图

![RocketMQ整体架构部署图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-18-083153.png)

- 每个NameServer之间是不通讯的。
- Producer、Consumer 同一时间，与NameServer集群中其中一台建立长连接。
- Producer与每一个Broker的Master保持长连接。
- Broker中的Master与Slave与每一台NameServer保持长连接。
- Consumer与所有的Broker建立长连接。

### MQ客户端的通讯(MQClientInstance)

MQClientInstance##startScheduledTask() 中会有很多定时任务，进行通讯的检查。

1. Consumer和Producer会每隔两分钟与用户所注册的NameSrv保持通讯。主要是心跳。
2. 每隔30S，更新当前所有与当前有关的Topic的路由元信息（包括Consumer和Producer）,路由元信息包括：
   - 每一个Topic对应的，所有与之相关的TopicQueue和BrokerQueue。
3. 每隔30S，进行Broker心跳检测，因为我们在第二点的时候已经从namesrv中获得所有与客户端有关系的broker信息，因此我们可以在第三点进行心跳检测。



### Broker端的通讯

BrokerController##start() 中会启动定时任务，==将Broker的Topice配置信息上传至所有的NameSrv.==



# 1. 网络

1. NameSrv存储了Topic路由信息（Broket注册的IP地址）。
2. Producer与NameServer集群中的其中一个节点（**随机选择**）建立长连接，定期从NameServer取到Topic路由信息（Broket注册的IP地址），并向提供Topic服务的Master建立长连接，并且定时（默认30秒）向Master 发送心跳。
3. Producer与NameServer集群中的其中一个节点（**随机选择**）建立长连接，定期从NameServer取到Topic路由信息（Broket注册的IP地址），并向==每一个==提供Topic服务的Broker Master建立长连接，并且定时（默认30秒）向Master 发送心跳。
4. 那么也就是说每一个与Topic相关的Broker中都具有所有订阅该Topic的Consumer,那么我们其中一个Consumer就能去随机的一个Broker中拿到所有订阅该Topic相关的Consuemr对象了，以实现负载均衡。



# 2. 消息消费要点

1. Pull模式和Push模式：我们在RocketMQ中一般使用的是Push模式即 用户层面感知到的是MQ的服务端主动推送消息到我们客户端。
   - 其实内部RocketMQ使用的是Pull模式来模拟出Push模式，这时候就出现了一个较大的缺点，当服务单一直没有数据时，客户端疯狂发送拉取请求，这不是小牛拉大车吗？
2. 长轮询：长轮询就是处理以上出现的情况，可以有效地控制当Broeker中的没有相对应数据时，减缓拉取请求。
3. 负载均衡：RocketMQ中保证一个ConsumerQueue只被一个Consumer进行消费，那么由谁来控制呢，不同于Kafuka,RocketMQ是通过每个Consumer自己执行负载均衡算法进行分配，达到只被一个Consumer消费的目的。
   - 当然这个肯定是有风险的，有可能会有一小段时间多个Consumer消费同一个ConsumerQueue

4. 顺序消息：RocketMQ可实现顺序消息，不过牺牲了很大的吞吐量。需要Producer和Consumer共同配合同一个Topic才能实现顺序消息。
5. 延迟消息：RocketMQ可实现固定延迟时间的延迟消息(延迟级别)。



# 3. Pull Push

1. 顾名思义，Pull为拉，Push为推，一个被动一个主动，其实很好理解。
   - Push方式：由消息中间件(MQ消息服务器代理)主动地将消息推送给消费者。但是，在消费者的处理消息的能力较弱的时候(概括起来地说就是**“慢消费问题”**)，而MQ不断地向消费者Push消息，消费者端的缓冲区可能会溢出，导致异常；
   - Pull方式：由消费者客户端主动向消息中间件(MQ消息服务器代理)拉取消息。
     - 采用Pull方式，如何设置Pull消息的频率需要重点去考虑，举个例子来说，可能1分钟内连续来了1000条消息，然后2小时内没有新消息产生（概括起来说就是**“消息延迟与忙等待”**）。
     - 如果每次Pull的时间间隔比较久，会增加消息的延迟，即消息到达消费者的时间加长，MQ中消息的堆积量变大；若每次Pull的时间间隔较短，但是在一段时间内MQ中并没有任何消息可以消费，那么会产生很多无效的Pull请求的RPC开销，影响MQ整体的网络性能；

2. RocketMQ可选Pull、Push模式，只不过Push模式是使用Pull来假装的，即consumer轮询从broker拉取消息。那么问题来了，假装的Push模式坑定还是有"慢消费问题"。
   - 长轮询登场：RocketMQ将Consumer对Broker的拉取信息封装为一个请求对象实例，那么我们通过这个性质就很容易的做到：流量控制，拉取时机。
     - 一个拉取请求：Consumer发送PullRequest至Broker，Broker从ConsumerQueue中取出消息，将PullRequest封装成PullResponse返回给Consumer, Consumer收到PullResponse后修改对象为PullRequest，进行消息处理。
   - 长轮询其实很简单，就是Broker收到Consumer的拉取请求后，如果发现Broker端没有新消息，那么就可以将该请求对象拦截下来，等待一段时间再返回给Consumer。
   - 流量控制是在Consumer端做的，当发现Consumer收到拉取对象返回的时候，完全消耗不过来，那么也会停止将拉取对象发送至Broker，实现流控。





## 消息消费大致流程(Push模式)

![消息拉取整体流程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-18-084101.png)

1. RebalanceService是负载均衡算法：主要作用是调整每一个Consuemer所消费的ConsumerQueue。毕竟需要保证一个ConsumerQueue只能被一个Consuemer消费。
   - RebalanceService可以创建PullRequest（单个ConsuemerQueue对应一个PullRequest）对象, PullRequest最后放到PullMessageService的pullRequestQueue阻塞队列中。
2. PullMessageService主要是业务发送消息请求：保存了当前Consuemer的所有PullRequest阻塞队列，单线程循环去阻塞队列中取出PullRequest，然后通过NettyRemotingClient发送消息。
   - 当Netty获得了返回对象PullResult后，放到CallbackExecutor线程池中进行执行其onSuccess()
3. ConsumeMessageConcurrentlyService主要是当Client收到拉取请求的响应后，对PullResponse进行处理，这里主要是业务处理了.
4. ConsumerRequest: 主要是执行任务的封装类，主要是调用应用所指定的操作，然后根据不同的返回值进行不同的收尾处理。
5. RemoteBrokerOffsetStore: 主要是辅助线程，用于同步消息消费的进度。

## Pull模式

1. RocketMQ是通过Pull模式来模拟出的Pull，因此可以看到Push模式中使用的PullMessageService, 很简单，Pull模式也就直接使用PullMessageService了。



# 4. 长轮询

![RocketMQ的长轮询](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-18-141046.png)

- 当Broker端收到Consumer的PullRequest请求时，发现现在没有任何可以返回的数据，那么可以看到左侧的PR，将该PullRequest来Hold住，放到PullRequestTable的队列中。
- 当Producer发送Topic至Broker端后，sendMessageProcesser将该数据存放在CommitLog(PageCache 或 堆外内存池)中，ReputMessageService会调用arriving来通知相应的PullRequestTable有消息来啦，起床干活了。
- brokerAllowSuspend: Broker端 表示是否允许Broker可以把这个请求Hold住。在PullRequest第一次发送至Broker端才会设置为true,虽然当ReputMessageServcice再次塞进去后就会设置为flase。
- 当然Broker当没有数据的时候也不会一直hold住PullRequest, 当hold超时之后也会放生PullRequest。



## 源码目录

1. 入口：PullMessageProcessor##processRequest，412行

   - case   ResponseCode.PULL_NOT_FOUND；

2. 入口代码：

   ```java
   this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
   
   ```

   - 将 topic@queueId 设为Key, 放到PullRequestHoldService##pullRequestTable中，是一个ConcurrentHashMap, MQ会一边Put，一边轮询该map进行处理。因此选择了ConcurrentHashMap。
   - pullRequestTable的Value是ArrayList<_PullRequest>, 里面存了很多Hold住的PullRequeust，我们就是根据这些PullRequest来实现拉取功能。

3. 关键代码：

   ```
   @Override
       public void run() {
           while (!this.isStopped()) {
               try {
                   // ***长轮询的一些wairForRuning**
                   // 遍历每一个pullRequestTable的Key,然后对其中每一个PullRequest来判断是否有消息到来了。
                   this.checkHoldRequest();
                   long costTime = this.systemClock.now() - beginLockTimestamp;
                   if (costTime > 5 * 1000) {
                       log.info("[NOTIFYME] check hold request cost {} ms.", costTime);
                   }
               } catch (Throwable e) {
                   log.warn(this.getServiceName() + " service has exception. ", e);
               }
           }
   
           log.info("{} service end", this.getServiceName());
       }
   ```

   

# 5. 负载均衡

1. 首先我们已经了解一个Consumer对一个Topic的ConsumerQueue的拉取请求被封装为PullRequest，那么问题就来了：
   - PullRequest什么时候创建并加入到pullRequestQueue中以唤醒PullMessageService线程。
   - 集群内多个消费者是如何负载主题下的多个消费对立，并且如果有新的消费者加入时，消息队列如何重新分布。

2. 咱么先简单地了解下：
   - 我们每一个Consumer中都可以去NameSrv获取咱么感兴趣的Broker。已经同样其他关注该Topic的Consuemr实例。
   - 获取到某一个Topic的所有Broker和Consuemer，然后进行Consuemer列表排序。
   - 排序过后就可以按照算法指定Consumer消费哪几个ConsumerQueue.



## 5.1 源码分析

1. 负载均衡算法的启动

```java
public void doRebalance() {
    // 遍历已经注册的消费者，对消费者执行doRebalance
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            try {
                impl.doRebalance();
            } catch (Throwable e) {
                log.error("doRebalance exception", e);
            }
        }
    }
}
```
- 这个doRebalance是放在RebalanceService中的，

2. 下面是对于Pull模式的rebalanceByTopic中的某一段，从方法名就可以知道Client会对当前Consumer所订阅的所有Topic轮询执行一次rebalanceByTopic。

```java
case CLUSTERING: 
{
  // STEP1：从主体订阅信息缓存表中获取主题队列信息
  Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);

  // STEP2：随机选择一个Broker，发送请求从Broker中该消费组内当前所有的消费者客户端ID。
  List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
  if (null == mqSet) {
    if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
      log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
    }
  }

  if (null == cidAll) {
    log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
  }

  if (mqSet != null && cidAll != null) {
    List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
    mqAll.addAll(mqSet);

    // STEP3：对该Topic相关的所有MessageQueue和ConsumerId进行排序，比较重要，
    // 需要保证同一个ConsuemrGroup中的RebalanceImpl的列表是相同，但是我觉得每个RebalanceImpl中的列表相同是不可能的啊？？？？
    Collections.sort(mqAll);
    Collections.sort(cidAll);

    // STEP4：负载均衡算法
    AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

    List<MessageQueue> allocateResult = null;
    try {
      allocateResult = strategy.allocate(
        this.consumerGroup,
        this.mQClientFactory.getClientId(),
        mqAll,
        cidAll);
    } catch (Throwable e) {
      log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                e);
      return;
    }

    Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
    if (allocateResult != null) {
      allocateResultSet.addAll(allocateResult);
    }
    // STEP5：调整拉取队列，如果当前新增了MessageQueue,则需要新建PullReques来放到PullMessageService中
    boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
    if (changed) {
      log.info(
        "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
        strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
        allocateResultSet.size(), allocateResultSet);
      this.messageQueueChanged(topic, mqSet, allocateResultSet);
    }
  }
  break;
}
```

- 主要做的内容如下：
  - 从主体订阅信息缓存表中获取主题队列信息。
  - 随机选择一个Broker，发送请求从Broker中该消费组内当前所有的消费者客户端ID。
  - 对该Topic相关的所有MessageQueue和ConsumerId进行排序，比较重要，
  - 调用负载均衡算法
  - 调整拉取队列，如果当前新增了MessageQueue,则需要新建PullReques来放到PullMessageService中

3. 待排序的MessageQueue是从：updateTopicRouteInfoFromNameServer方法进行更新的，从方法名字就知道是从NameServer获取的.



# 6. 顺序消息

1. RocketMQ支持局部消息顺序消费，**可以确保同一个消息消费队列中的消息被顺序消费**。
   - 顺序消息消费与并发消息消费的第一个关键区别：顺序消息在创建消息队列拉取任务时需要在Broker服务器锁定该消息队列。

2. Consumer严格顺序消费，需要通过三把锁保证严格顺序消费：
   - Broker消息队列锁(分布式锁)：
     - 集群模式下：consumer从Broker获得锁后，才能进行消息拉取、消费。
     - 广播模式下：Consumer无需该锁。
   - Consumer消息队列锁（本地锁）：Consumer获得该锁才能操作消息队列。也就是processQueue.setLocked(true)
   - Consumer消息处理队列消费锁（本地锁）：Consumer获得该锁才能消费消息队列。（这个锁有点水，主要是在移除不需要的队列相关的信息操作和消费消息时候的加锁）

![消息消费流程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-04-014601.png)







# 7. 定时(延迟)消息

1. RocketMQ 并不支持任意时间精度的定时调度，因为如果要实现的话则需要在Broker层做消息排序**(参考JDK的ScheduledExcutorService的实现原理)**，加上持久化的话会造成比较大的性能消耗
2. 设计关键点：定时消息单独一个主题：SCHEDULE_TOPIC_XXXX，也就是说比如SCHEDULE_TOPIC_01对应的延迟任务是第2级别。依次类推。
   - 在消息发送时，如果消息的延迟级别delayLevel大于0，将消息的原主题名称、队列ID存入消息的属性中，然后改变消息的主题、队列等，消息将最终转发到延迟队列的消费队列中。



## 源码

1. CommitLog##putMessage() 中有一段是处理当Message具有延迟级别的情况。
   - 当发现消息具有延迟级别的话，会直接修改消息的Topic和QueueID，然后再将消息刷盘！














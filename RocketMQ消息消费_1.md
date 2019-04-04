**RocketMQ消息消费_1**

标签：【RocketMQ】

[TOC]

# 1. 概述消息消费

1. 消息消费以组的模式展开，一个消费组内可以包含多个消费者，每一个消费者可订阅多个主题。
2. 消费组之间有集群模式与广播模式两种消费模式。
   - 集群模式  Clustering：主题下的同一条消息**只允许被其中一个消费者消费**。
   - 广播模式 Broadcasting：主题下的同一条消息将被集群内的**所有消费者消费一次**。

3. 集群模式下 一个消息队列同一时间只允许被一个消费者消费，一个消费者可以消费多个消息队列。
4. ==RocketMQ支持局部顺序消息消费==，也就是保证同一个消息队列上的消息顺序消费，不支持消息全局顺序消费。若要实现某一个主题的全局顺序消息消费，可以将该主题的队列数设置为1。

5. 可通过如下方式设置消费组是什么消费模式：

   > ```java
   > consumer.setMessageModel(MessageModel.CLUSTERING);
   > ```

6. consumer group 的概念：即消费同一类消息的多个 consumer 实例组成一个消费者组，也可以称为一个 consumer 集群，这些 consumer 实例使用同一个 group name。需要注意一点，==除了使用同一个 group name，订阅的 tag 也必须是一样的，==只有符合这两个条件的 consumer 实例才能组成 consumer 集群。

## 1.1 消息消费方式

1. 对于任何一款消息中间件而言，消费者客户端一般有两种方式从消息中间件获取消息并消费：push(推)、pull(拉)模式。
2. Push方式：由消息中间件(MQ消息服务器代理)主动地将消息推送给消费者。
   - 但是，在消费者的处理消息的能力较弱的时候(概括起来地说就是**“慢消费问题”**)，而MQ不断地向消费者Push消息，消费者端的缓冲区可能会溢出，导致异常；

3. Pull方式：由消费者客户端主动向消息中间件(MQ消息服务器代理)拉取消息。
   - 采用Pull方式，如何设置Pull消息的频率需要重点去考虑，举个例子来说，可能1分钟内连续来了1000条消息，然后2小时内没有新消息产生（概括起来说就是**“消息延迟与忙等待”**）。
   - 如果每次Pull的时间间隔比较久，会增加消息的延迟，即消息到达消费者的时间加长，MQ中消息的堆积量变大；若每次Pull的时间间隔较短，但是在一段时间内MQ中并没有任何消息可以消费，那么会产生很多无效的Pull请求的RPC开销，影响MQ整体的网络性能；
   - **在RocketMQ里，有一种优化的做法——长轮询 Pull ，来平衡推拉模型各自的缺点**

4. 对于RocketMQ来说，**Push和Pull模式都是采用消费端主动拉取的方式**，即consumer轮询从broker拉取消息。区别：
   - Push方式里：consumer把轮询过程封装了，**并注册MessageListerner监听器**，取到消息后，唤醒MessageListener的consumeMessage()来消费，对用户而言，感觉消息是被推送过来的。
   - Pull方式里，取消息的过程需要用户自己写，首先通过打算消费的Topic拿到MessageQueue的集合，遍历MessageQueue集合，然后针对每个MessageQueue批量取消息，一次取完后，记录该队列下一次要取的开始offset，直到取完了，再换另一个MessageQueue。



## 1.2 提高Consumer消费能力

​	当Consumer处理速度跟不上消息的产生速度，就会造成消息挤压，解决方法如下：

1. 提高消费并行度：
   - 在同一个 ConsumerGroup下（ Clustering 方式），可以通过==增加Consumer实例==的数量来提高并行度。
   - **注意总的 Consumer 数量不要超过 Topic 下Read Queue数量，超过的 Consumer 实例接收不到消息**。
   - 此外，**通过提高单个 Consumer 实例中的并行处理的线程数** 可以在同一个 Consumer 内增加并行度 来提高吞吐量（设置方法是修改 consumeThreadMin 和 consumeThreadMax ）
2. 以批量方式消费：
   - 通过批量方式消费来提高消费的吞吐量。实现方式是设置 Consumer 的 consumeMessageBatchMaxSize 这个参数 ，默认是1，如果设置为N，在消息多的时候每次收到的是个长度为 N 的消息链表 。
3. 检测延时情况，丢弃非重要消息。



## 1.3 负载均衡

1. 要做负载均衡，必须知道一些全局信息，也就是一个ConsumerGroup里到底有多少个 Consumer，知道了全局信息，才可以根据某种算法来分配。 
2. ==在 RocketMQ 中，负载均衡或者消息分配是在 Consumer 端代码中完成的==，Consumer从 NameSrv(updateTopicRouteInfoFromNameServer)处获得全局信息，然后自己做负载均衡，只处理分给自己的那部分消息 。

详见《RocketMQ实战与原理解析》 P81



## 1.4 整体流程

![消息拉取整体流程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-14-140032.png)

- Client端将拉取请求封装成PullRequest存放在PullRequestQueue中，然后PullMessageService启一个单线程取PullRequest，通过网络向Broker发送请求。
- Netty工作线程收到Broker的回复后，因为是异步的，所以会将PullResponse再次封装成PullRequest放回PullRequestQueue中，然后也会把收到的信息放在ConusmerMessageConcurrentLyService中。
- 可以在均衡的时候看到Client会对**每一个TopicQueue创建一个PullRequest**。每一个PullRequest中的消息都是放在==一颗红黑树(ProcessQueue)==中，key值是offest。





# 2. 常用组件介绍

1. MQConsumer --> MQPushConsumer -->DefaultMQPushsumer。推模式的主要API
2. MQConsumer --> MQPullConsumer -->DefaultMQPullsumer。拉模式的主要API
3. 以上可以看出推拉模式各一套。

## 2.1 ProcessQueue

1. RocketMQ 定义了 一 个快照类 ProcessQueue 来解决这些问题，在 Push Consumer 运行的时候， ==每个 Message Queue 都会有个对应的 Process Queue 对象，保存了这个 Message Queue 消息处理状态的快照== 。
2. PullMessageService从消息服务器默认每次拉取32条消息，==按消息的队列偏移量顺序存放在ProcessQueue中==。PullMessageService然后将消息提交到消费者消费线程池，消息成功消费后从ProcessQueue中移除。
3. ProcessQueue##putMessage：
   - 添加消息，PullMessageService拉取消息后，先调用该方法将消息添加到ProcessQueue。
   - a. 在PushConsumerImpl中的PullCallback##onSuccess中调用到。
   - b. PullConsumerImpl中也会调用到。
4. ProcessQueue##rollback：
   - 将consumingMsgOrderlyTreeMap移动到msgTreeMap。
   - 清空consumingMsgOrderlyTreeMap。



## 2.2 Offsetstore

1. 在使用 DefaultMQPushConsumer 的时候，我们不用关心OffsetStore的事 , 但是如果 PullConsumer ，我们就要自己处理 OffsetStore 了



## 2.3 pullRequestQueue

1. pullRequestQueue作为PullMessageService中的一个阻塞队列（LinkedBlockingQueue）。存放着PullRequest。
2. PullMessageService会有一个线程不停地轮询去pullRequestQueue中take出PullRequest。
   - 后面会在PullCallback##OnSuccess中收到Broker发送的消息后封装下再put到pullRequestQueue中去的。



# 3. 消费者启动

1. 以推模式的启动流程为例。
2. 用户API入口为 DefaultMQPushConsumer#start()。
   - 主要操作都在DefaultMQPushConsumerImpl#start()中。
3. DefaultMQPushConsumerImpl#start()的步骤如下：
4. Step1:构建主题订阅消息SubscriptionData并加入到RebalanceImpl的订阅消息中。
5. Step2:初始化MQClientInstance 、 RebalanceImpl 消息重新负载实现类.
6. Step3:初始化消息进度，==若消息消费是集群模式，那么消息进度保存在Broker。否则保存在消费端。==
7. Step4:根据是否是顺序消息，创建消费端消费线程服务。ConsumeMessageService主要负责消息消费，内部维护一个线程池。
8. Step5:向MQClientInstance注册消费者，并启动MQClientInstance。注意在生产者，消费者同一个JVM实例中，只会创建同一个MQClientInstance。



##DefaultMQPushConsumerImpl.copySubscription()

1. 该方法主要是将 **%RETRY%+消费组名** 的Topic放在RebalanceImpl##subscriptionInner()中，即订阅该主题。下面分析下为什么这么做！

2. Consumer的订阅消息存放在RebalanceImpl##subscriptionInner()中。这里的主要来源有两个：
   - 第一：DefaultMQPushConsumer##subscribe(), 我们用户手动订阅。
   - 第二：订阅重试主题消息，从这里可以看出RocketMQ 消息重试是以消费组为单位，而不是主题，因为Topic名为**%RETRY%+消费组名** 。消费者启动时会自动订阅该主题，参与该主题的消息队列负载。0

##MQClientInstance.start()

```java
case CREATE_JUST:
    this.serviceState = ServiceState.START_FAILED;
    // If not specified,looking address from name server
    if (null == this.clientConfig.getNamesrvAddr()) {
        this.mQClientAPIImpl.fetchNameServerAddr();
    }
    // Start request-response channel
    this.mQClientAPIImpl.start(); //netty相关通信的初始化，然后开启一个定时任务扫描responseTable
    // Start various schedule tasks
    // 一些定时消息，更新Broker,Producer,Consumer相应信息。
    this.startScheduledTask();
    // Start pull service
    this.pullMessageService.start(); //开启线程从processQueueTable获取PullRequest，并请求broker
    // Start rebalance service
    this.rebalanceService.start();
    // Start push service
    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
    log.info("the client factory [{}] start OK", this.clientId);
    this.serviceState = ServiceState.RUNNING;
    break;
```

1. startScheduledTask: 主要是更新Broker,Producer,Consumer相应信息。
   - 据我了解都是根据当前Consumer、Producer所订阅的Topic, 然后拿到与topic相关的一些Broker信息



# 4. 消息拉取

1. 基于PUSH模式来详细分析消息拉取机制。分为广播和集群。

2. 在MQClientInstance的启动流程( start() )可以得知，RocketMQ使用一个单独的线程PullMessageService来负责消息的拉取。

   - 可以发现在MQClientInstance中只有PullMessageService，体现RocketMQ内部实现都是通过Pull来实现的PUSH的，且PullMessageService线程的pullMessage，**只为DefaultMQPushConsumerImpl所服务，也就是为PUSH服务**。
   - Pull模式的拉取消息是通过应用程序手动调用API。

3. 在MQClientInstance#start()中的 > this.pullMessageService#start() 会开启PullMessageService的run方法。

4. 消息拉取分为3个主要步骤：

   - 第一步：拉取客户端消息拉取的封装
   - DefaultMQPushConsumerImpl##pullMessage，主要是组装一下PullRequest。
   - 第二步：消息消息查找并返回消息
   - 调用PullAPIWrapper##pullKernelImpl()来去Broker中拉取消息，若是异步的话则在DefaultMQPushConsumerImpl##pullMessage中创建的PullCallback的OnSuccess方法中将返回的消息存放在PullRequest中。
   - 第三步：消息拉取客户端处理返回的消息

    



## 4.1 PullMessageService的实现机制

1. 下面是PullMessageService中RUN方法的代码：

```java
@Override
    public void run() {
        log.info(this.getServiceName() + " service started");

        // 1. 可以通过stopped 来作为是否关闭当前事件运行的标志位。
        while (!this.isStopped()) {
            try {
                // 2. 从阻塞队列中获取PullRequest，
                PullRequest pullRequest = this.pullRequestQueue.take();
                // 3. 然后最终调用的是DefaultMQPushConsumerImpl的pullMessage方法
                this.pullMessage(pullRequest);
            } catch (InterruptedException ignored) {
            } catch (Exception e) {
                log.error("Pull Message Service Run Method exception", e);
            }
        }

        log.info(this.getServiceName() + " service end");
    }
```

2. 可以看到我们主要是拿到pullRequestQueue中的PullRequest，然后再进行处理。
3. 那么后面介绍下是啥时候将PullRequest放到PullRequestQueue中的



## 4.2 PullRequest的创建及添加

1. 唯一关联的入口为PullMessageService#executePullRequestImmediately(),下面是调用该接口的接口：
   - DefaultMQPushConsumerImpl#executePullRequestImmediately
   - PullMessageService#executePullRequestLater

2. 总结一下,==主要会在两个地方塞入==：
   - 第一个：在RicketMQ根据PullRequest拉取任务执行完一次消息拉去任务后，又将PullRequest放入到pullRequestQueue中。（PullCallBack的onSuccess中），==这里主要是已经从Broker中拿到该Request的数据信息了，然后将数据信息放到PullRequest的ProcessQueue的msgTreeMap中==，否则一个PullRequest肯定是没有数据的啊！！
   - 第二个：在RebalancceImpl中创建，消息队列负载机制。



## 4.3 pullMessage处理PullRequest

1. PullMessage是只有拿到PullRequest后才会进行相应处理，否则会一直在BlokingQueue中阻塞。
2. 在DefaultMQPushConsumerImpl##pullMessage(final PullRequest pullRequest)；以下是主要内容：
   - Part1 : 拿到PullRequest中的ProcessQueue。且当ProcessQueue废弃、消费者处于暂停状态、消费者挂起时会将PullRequest放到线程池中稍后执行。
   - Part2：进行消息流量控制，两个维度，消息数量达到阔值（默认1000个），或者消息体总大小(默认100m)。当超量时就会将PullRequest放到线程池中稍后执行。
   - Part3：获取主题订阅信息。
   - Part4：Pull模式拉取回调，其中会执行executePullRequestImmediately来把pullRequest塞回去。
   - Part5：构建拉取消息系统Flag: 是否支持comitOffset,suspend,subExpression,classFilter。
   - Part6：重点关注，通过MQClientInsatance拉取消息。pullAPIWrapper##pullKernelImpl()

### 流量控制

1. 我们是把PullRequest的processQueue.getMsgCount()的消息数与定值比较来做流量控制，
2. 当超过规定数值时，会延迟一定时间再将该ProcessQueue塞到PullRequest中去。
3. 个人认为：首先这是在单线程的RUN中进行流量控制的，消费信息是在线程池中进行的(Onsuccess的话是依靠回调，与原线程不相同)，==因此可以这么理解流量控制的线程和处理消息的线程完全分开==，但是通过一个Atomic数值、BlockingQueue连接在一起。做到了流量控制。



## Onsuccess

1. OnSuccess是在NettyRomotingClient中的CallBackExecutor的线程池中收到返回TCP信息后运行的。

2. 其中OnSuccess会调用：来交给我们用户注册的ConsumeMessageService的线程池去进行处理。（主要分为并发和顺序）

   ```java
   DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                       pullResult.getMsgFoundList(),
                                       processQueue,
                                       pullRequest.getMessageQueue(),
                                       dispatchToConsume);
   ```



## pullKernelImpl

1. Step1 : 根据BrokerName、BorkerId从MQClientInstance中获取Broker地址，在整个RocketMQ Broker的部署结构中，相同名称的Broker构成主从结构，其BrokerID会不一样，**在每次拉取消息后，会给出一个建议，下次拉取从主节点还是从节点拉取**。
2. Step2 : 若消息为类过滤模式，则需要获取Broker上的FilterServer地址，从FilterServer上拉取消息，否则从Broker上拉取消息。
3. Step3 : 拉取请求都组装完毕后向MQClientInsatance拉取消息。
4. 后续就是通过pullMessageAsync或者pullMessageSync去发起拉取消息请求了。





## 4.4 消息服务端Broker组装消息

1. 见PullMessageProcessor##processRequest

P143





## 4.5 消息拉取客户端处理消息

1. 从PullMessageService##PullMessage() -> DefaultMQPushConsumerImpl##pullMessage ->PullAPIWrapper##pullKernelImpl() -> MQClientAPIImpl##pullMessage -> pullMessageAsync ()。
2. pullMessageAsync主要在Netty中添加了一个回调函数，该回调函数中会执行转换消息格式、执行OnSucess等。
3. pullMessageAsync () 中可以拿到Broker的返回消息对象，然后转化一下格式交给pullCallback.onSuccess来做。其中调用链如下：
   - ~~DefaultMQPullConsumerImpl##pullAsyncImpl中new PullCallback()匿名内部类OnSuccess():拉取到消息的后续处理,将消息字节数组解码成消息列表填充msgFountList,并对消息进行消息过滤（TAG）模式。~~
   - 直接调用到：DefaultMQPushConsumerImpl##pullMessage中创建的PullCallback的OnSuccess方法

4. 接下来看一看具体的PullCallback##OnSuccess



## PullCallback##OnSuccess

1. 该OnSuccess主要是在DefaultMQPushConsumerImpl中的内部类的实现方法。

2. Step1：拉取到消息的后续处理：首先对PullResult进行处理，主要完成如下3件事：1）对消息体解码成一条条消息 2）执行消息过滤 3）执行回调钩子。

3. Step2：更新PullRequest的下一次拉取偏移量。

4. Step3：当MsgFoundList为空时， 将pullRequest 塞入到PullRequestQueue中，以便DefaultMQPushConsumerImpl能够拿到pullRequest后可以再次拉取消息。

   - 为什么pullResult.getMsgFoundList()有可能为空：是因为在RocketMQ根据TAG消息过滤，在服务端只验证了TAG的hashCode，在客户端再次对消息进行过滤，估可能会出现MsgFoundList()为空的情况。

5. Step4 : 将消息放入消费队列中：就是将拉取的消息，放入到ProcessQueue的msgTreeMap容器中。

6. Step5 : 消费消息服务提交，这里十分有必要对顺序消息与非顺序消息的消费方式分别了解一下

   - > ```java
     > DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest()// 交由consumeMessageService进行处理(分为顺序和并发两种)
     > ```

   - 提交给consumeMessageService处理后，OnSuccesss就不关心了。

7. Step7 :然后根据消息拉取时间间隔 再次进行拉取。

![RocketMQ拉取流程](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-23-061350.jpg)
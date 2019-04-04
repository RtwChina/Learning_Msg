**RocketMQ消息发送**

标签：【RocketMQ】



# 1. 漫谈RocketMQ消息发送

1. RocketMQ支持3种消息发送方式：同步(sync)、异步(async)、单向(oneway)。
2. 同步：发送者向MQ执行发送消息API时，同步等待，直到消息服务器返回发送结果。
3. 异步：发送者向MQ执行发送消息API时，指定消息发送成功后的回调函数，然后调用消息发送API后，立即返回，线程不阻塞，知道运行结束，消息发送成功或失败的回调在一个新的线程中执行。
4. 单向：消息发送者向MQ执行发送消息API，直接返回，不等待消息服务器的结果，也不注册回调函数，也就是说只管发不靠谱消息是否成功存储在消息服务器上。



# 2. RocketMQ消息

1. 消息结构体：org.apache.rocketmq.common.message.Message。
2. 其中properties属于扩展属性：
   - tag：消息TAG，用于消息过滤。
   - keys：Message索引键，多个用空格隔开，RocketMQ可以根据这些KEY来快速检索到消息。
   - waitStoreMsgOK：消息发送时是否等消息存储完成后再返回。
   - delayTimeLevel：消息延迟级别。



# 3. RocketMQ生产者启动

1. 向Broker发送消息实现基本都在DefaultMQProducer中。
2. 启动Producer:  DefaultMQProducer.start();
3. ==一个客户端只能产生一个MQClientInstance实例对象==，产生方式使用了工厂模式与单例模式。MQClientInstance.start()方法启动一些服务.
4. **MQClientInstance**：主要封装了RocketMQ网络处理API，是消息生产者(Producer), 消息消费者(Consumer) 与NameServer、Broker打交道的网络通道。
   - 其中的clientID = 客户端IP + instance + (unitname可选)。若同一台物理服务器部署两个应用程序，instance若为默认值则会设置为进程ID。
   - 一个客户端只能产生一个MQClientInstance实例对象

5. 后续会把当前的将当前的生产者 注册到 MQClientInstance中。



# 4. RocketMQ生产者应用

1. 生产者 向消息队列里写人消息，不 同的业务场景需要生产者采用不同的写人 策略 。 比如同步发送、异步发送、 延迟发送、 发送事务消息等，

## 4.1 同步、异步、延迟等

1. 同步：

   >```java
   >SendResult sendResult = producer.send(msg);
   >```

2. 异步：

   > ```
   > producer.send(msg, new SendCallback() {
   >     @Override
   >     public void onSuccess(SendResult sendResult) {
   >         System.out.println("大家好");
   >     }
   > 
   >     @Override
   >     public void onException(Throwable e) {
   >         System.out.println("异常咯");
   > 
   >     }
   > });
   > ```

3. 延迟:

   > ```java
   > msg.setDelayTimeLevel(3); // 设置延迟级别对于延迟时间
   > ```

4. 指定相应Queue发送：

   >```java
   >
   >SendResult send(final Message msg, final MessageQueueSelector selector, final Object arg)// 创建MessageQueueSelector，可指定Queue
   >    
   >SendResult send(final Message msg, final MessageQueue mq) // 直接指定MessageQueue
   >```



# 5. RocketMQ消息发送

1. 消息发送有三大流程：验证消息、查找路由、消息发送。

2. 使用如下发送消息时：一般send函数中没有添加SendCallback都是同步发送，添加的是异步

3. > ```
   > SendResult sendResult = producer.send(msg);
   > ```

3. SendResult的结果含义：

   ```java
   public enum SendStatus {
       // 发送成功
       SEND_OK,
       // 没有在规定时间内完成刷盘(需要Broker的刷盘策略被设置成SYNC_FLUSH才会报错)
       FLUSH_DISK_TIMEOUT,
       // 主备模式下，并且Broker被设置为SYNC_MASTER方式，没有在设定时间内完成主从同步
       FLUSH_SLAVE_TIMEOUT,
       // 主备模式下，并且Broker被设置成SYNC_MASTER，但没有找到配置成Slave的Broker
       SLAVE_NOT_AVAILABLE,
   }
   ```

## 5.1 消息长度验证

1. 消息长度不能等于0且默认不能超过允许发送消息的最大长度 4M。



## 5.2 查找主题路由信息

1. 消息发送之前，首先需要获取主题的路由信息，只有获取了信息才知道消息要发送到具体的Broker节点。
2. 入口是DefaultMQProducer的send()  --->>  DefaultMQProducerImpl的sendDefaultImpl()。
3. 其中tryToFindTopicPublishInfo是通过Topic获取路由信息，有可能从缓存中取，有可能从NameSrv中取。



## 5.3 选择消息队列

1. 我们知道在同一个Broker中的Topic中可能有多个队列（Queue），我们就需要选择Broker集群中的某一个消息队列。
2. 根据路由信息选择消息队列，返回的消息队列按照broker、序号排序。
3. 选择消息队列主要有两种方式：
   - **默认，不启用Broker故障延迟机制。然后就是轮询MssageQueue。**
   - 启用Broker故障延迟机制。

### isAvailable

1. 该方法表示当前时间该Broker是否可用，多久后可用详见updateFaultItem。
   - 一般多久后可用：即上一次发送失败的延时时间，当没有延迟事件比如报错的时默认30s
   - 一般updateFaultItem是在DefaultMQProducerImpl##sendDefaultImpl中会调用，由此来记录Broker的状态。



### 5.3.1 默认，不启用Broker故障延迟机制

1. 入口在TopicPublishInfo的selectOneMessageQueue中。
2. 会剔除掉上次发送失败的队列，然后在余下的中按照顺序选择一个。
   - 缺点：如果Broker宕机，由于路由算法中的消息队列是按Broker排序，下一次的选择可能会在一次失败，带来不必要损耗。这就是Broker故障延迟机制粉墨登场了。





### 5.3.2 启用Broker故障延迟机制

1. 入口在MQFaultStrategy的selectOneMessageQueue中。其中this.sendLatencyFaultEnable默认为false,启动时为true.
2. 优先获取可用队列，依次选择一个broker获取队列（也就是轮询），最差返回任意broker的一个队列。





## 5.4 消息发送

1. 消息发送API核心入口： DefaultMQProducerImpl#sendKernelImpl。





# 6. 思考题

1. 我们Producer本地有Topic的路由缓存，虽然当本地没有路由缓存时会去NameSrv中手动获取，但是这也不是很靠谱的。那么当一个Broker宕掉后，我们Producer是如何处理的。
2. 消息重试能hold住所有异常吗？
   - 消息重试次数有限，不能覆盖所有Broker，部分MQBrokerException异常不支持重试。
3. 发送超时时间是仅一次发送还是包含重试？
   - 旧版本超时时间为单次，但在新版本中包含多次重试。
4. Math.abs()一定返回非负数吗？
   - Integer.MIN_VALUE返回本身，各位取反+1.


















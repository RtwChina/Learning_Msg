**RocketMQ整体分析**

标签：【RocketMQ】



# 1. RocketMQ整体

![RocketMQ图示](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-13-021212.png)

​	rocketmq的主要部分是由4种集群构成的：namesrv集群、broker集群、producer集群和consumer集群。

1. namesrv集群：也就是==注册中心==，rocketmq在注册中心这块没有使用第三方的中间件，而是自己写的代码来实现的，代码行数才1000行，producer、broker和consumer在启动时都需要向namesrv进行注册，==namesrv服务之间不通讯==。
   - NameServer相当于配置中心，维护Broker集群、Broker信息、Broker存活信息、主题与队列信息等。
2. broker集群：==broker提供关于消息的管理、存储、分发等功能，是消息队列的核心组件==。rocket关于broker的集群提供了主要两种方案，一种是主从同步方案，消息同时写到master和slave服务器视为消息发送成功；另一种是异步方案，slave的异步服务负责读取master的数据，本人在选择时更倾向于异步方案。
3. producer集群：消息的生产者，每个producer都需要属于一个group，producer的group概念除了在事务消息时起到一些作用，但是其它时候，更多的还只是一个虚拟的概念。
4. consumer集群：消息的消费者，有两个主要的consumer:DefaultMQPullConsumer和DefaultMQPushConsumer，深入代码后可以发现，rocket的consumer都是采用的pull模式来处理消息的。在集群消息的配置下，集群内各个服务平均分配消息，当其中一台consumer宕机，分配给它的消息会继续分配给其它的consumer。
5. 要点：
   - Consumer与每一个相关Topic的Broker的互相连接，Consumer是从NameSrv中获取所有与某一个Topic的相关Broker地址。



# 2. 概念介绍

## 2.1 Topic

1. RocketMQ的topic和 JMS的topic不一样，JMS中topic表示发布订阅貌似。在RocketMQ中Topic是生产者在发送消息和消费者在拉取消息的==类别==。Topic与生产者和消费者的关系非常松散：
   - 具体来说，**一个Topic可能有0个，一个或多个生产者向它发送消息**；相反，**一个生产者可以发送不同类型Topic的消息**。类似的，消费者组可以订阅一个或多个主题，只要该组的实例保持其订阅一致即可。
   - 也可以理解为第一级消息类型，类比于书的标题。

```java
// 在Producer中使用Topic：
Message msg = new Message("TopicTest" /* Topic */,
                    "TagA",("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
```

2. 各个概念之间的联系
![RocketMQ的水平扩展](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-14-024238.png)

- 上图中Broker Cluster和Broker Group之间的关系定义成1：N的关系，是为了表明两者之间存在包含关系。在RocketMQ的代码中并没有限制一个Broker Group必须从属于一个Broker Cluster。
- 将一个Consumer Group对应业务系统中的一个独立的业务模块，是一个比较值得推荐的ConsuerGroup划分方法。

3. 在正式使用RocketMQ进行消息发送和接收前，要先创建Topic。建议一个应用只用一个Topic，不同消息子类型用Tag来标识，==服务器端基于Tag进行过滤==。

### 2.1.1 Topic，Topic分片

![Topic分片](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-14-024832.png)

1. 为了简化分析过程，在这张图中没有包含Slave Broker。Broker1,Broker2和Broker3都是Master Broker。如果各Master Broker有Slave Broker，Slave Broker中的结构和其对应的Master Broker完全相同。

2. 从本质上来说，**RocketMQ中的Queue是数据分片的产物**。为了更好地理解Queue的定义，我们还需要引入一个新的概念：Topic分片。在分布式数据库和分布式缓存领域，分片概念已经有了清晰的定义。
   - ==对于RocketMQ，一个Topic可以分布在各个Broker上，我们可以把一个Topic分布在一个Broker上的子集定义为一个Topic分片==。
   - 对应上图，TopicA有3个Topic分片，分布在Broker1,Broker2和Broker3上，TopicB有2个Topic分片，分布在Broker1和Broker2上，TopicC有2个Topic分片，分布在Broker2和Broker3上。

3. **将Topic分片再切分为若干等分，其中的一份就是一个Queue。**每个Topic分片等分的Queue的数量可以不同，由用户在创建Topic时指定。



### 2.1.2 Queue（Message Queue）

![TopicA的Queue](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-14-025121.png)

1. 如图所示，TOPIC_A在一个Broker上的Topic分片有5个Queue，一个Consumer Group内有2个Consumer按照集群消费的方式消费消息，按照平均分配策略进行负载均衡得到的结果是：第一个 Consumer 消费3个Queue，第二个Consumer 消费2个Queue。如果增加Consumer，每个Consumer分配到的Queue会相应减少。Rocket MQ的负载均衡策略规定：Consumer数量应该小于等于Queue数量，如果Consumer超过Queue数量，那么多余的Consumer 将不能消费消息。

2.  在一个Consumer Group内，==Queue和Consumer之间的对应关系是一对多的关系==：一个Queue最多只能分配给一个Consumer，一个Cosumer可以分配得到多个Queue。这样的分配规则，每个Queue只有一个消费者，可以避免消费过程中的多线程处理和资源锁定，有效提高各Consumer消费的并行度和处理效率。

3. 由此，我们可以给出Queue的定义：

>  Queue是Topic在一个Broker上的分片等分为指定份数后的其中一份，是负载均衡过程中资源分配的基本单元。



## 2.2 Tag

1. Tag，标签，子主题，为用户提供了额外的灵活性。有了标签来自同一业务模块的具有不同目的的消息可以具有相同的主题和不同的标记。
   - 第二级消息类型，类比于书的目录，方便检索使用消息。

```java
//在Producer中使用Tag：
Message msg = new Message("TopicTest",
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
```



## 2.3 GroupName

1. 代表具有相同角色的生产者组合或消费者组合，称之为生产者组或消费者组。

2. 有了GroupName，在集群下，动态扩展容量很方便。只需要在新加的机器中，==配置相同的GroupName。启动后，就立即能加入到所在的群组中，参与消息生产或消费==。

   ```java
   // 在Producer中使用GroupName：
   DefaultMQProducer producer = new DefaultMQProducer("group_name_1");// 使用GroupName来初始化Producer，如果不指定，就会使用默认的名字：DEFAULT_PRODUCER
   
   ```




## 2.4 ProducerGroup

1. 用来表示一个发送消息应用，一个 Producer Group 下包含多个 Producer 实例。

   - 可以是多台机器，

   - 可以是一台机器的多个进程

   - ~~**可以是一个进程的多个 Producer 对象**。~~, ==同一个JVM实例中只能存到一个ProducerGroupName的一个实例，即不可创建多个相同名称的Roducer实例。==在如下代码中会报错的。

     > ```java
     > boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
     > ```

2. 一个 Producer Group 可以发送多个 Topic 消息。

3. Producer Group 作用如下:
   - 标识一类 Producer
   - 可以通过运维工具查询返个収送消息应用下有多个 Producer 实例
   - 发送分布式事务消息时，如果 Producer 中途意外宕机，Broker会主动回调Producer Group内的任意一台机器来确认事务状态。



## 2.5 ConsumerGroup

1. 



# 3. Broker组件

1. RocketMQ 分布式集群是通过 Master 和 Slave 的配合达到高可用性的，首先说一下 Master 和 Slave 的区别：在 Broker 的配置文件中，参数 brokerId 使用的值为 0 表明这个 Broker 是 Master ，大于 0 表明这个Broker是Slave，同 时 brokerRole 参数也会说明这个 Broker 是 Master 还是 Slave 。
2. ==Master角色的Broker支持读和写，Slave角色的Broker仅支持读==， 也就是Producer只能和Master角色的Broker连接写人消息； Consumer可以连接 Master 角色的Broker ，也可以连接Slave角色的Broker来读取消息 。
3. 在Consumer中，不需要设置从Master读还是从Slave读，因为Consumer会被自动些换到从Slave读。
4. Broker是支持集群扩容的，Topic继续在原先的Broker上存储，然后可以为新的Broker创建Topic新的读写队列。



## 3.1 Broker主从模式

1. 由于消息分布在各个broker上，一旦某个broker宕机，则该broker上的消息读写都会受到影响。所以rocketmq提供了master/slave的结构，salve定时从master同步数据，如果master宕机，==则slave提供消费服务，但是不能写入消息==，此过程对应用透明，由rocketmq内部解决。

2. 这里有两个关键点：

- 一旦某个broker master宕机，生产者和消费者多久才能发现？受限于rocketmq的网络连接机制，默认情况下，最多需要30秒，但这个时间可由应用设定参数来缩短时间。这个时间段内，发往该broker的消息都是失败的，而且该broker的消息无法消费，因为此时消费者不知道该broker已经挂掉。

- ==消费者得到master宕机通知后，转向slave消费==，但是slave不能保证master的消息100%都同步过来了，因此会有少量的消息丢失。但是消息最终不会丢的，一旦master恢复，未同步过去的消息会被消费掉。

3. 也就是说建议将Topic放在多台Broker主机器上，防止Down掉后消息生产者无法发送相应Topic的消息。





## 3.2 刷盘

1. 消息通过Producer写入RocketMQ的时候，有两种写磁盘方式：

	- 异步刷盘方式：在返回写成功状态时，消息可能只是被写人了内存的PAGECACHE ，写操作的返回快，吞吐量大 ；当内存里的消息量积累到一定程度时 ，统一触发写磁盘动作，快速写人 。

	- 同步刷盘方式：在返回写成功状态时， 消息已经被写人磁盘。 具体流程是，消息写入内存的 PAGECACHE 后，立刻通知刷盘线程刷盘， 然后等待刷盘完成， 刷盘线程执行完成后唤醒等待的线程， 返回消息写成功 的状态 。





# 4. 消息消费

## 4.1 顺序消息

1. 顺序消息分为全局顺序和布局顺序。

   - 全局：即整个Topic的消息都是有序的，RocketMQ中需要把Topic的读写队列数设置为一，Producer和Consumer的并发设置也要是一。
   - 局部：即某一个MessageQueue是有序的，需要消费端和发送端相互配合。

2. 实现如下：

   ```java
   // 生产者，需要指定对应MessageQueue
   public class OrderedProducer {
       public static void main(String[] args) throws Exception {
           //Instantiate with a producer group name.
           MQProducer producer = new DefaultMQProducer("example_group_name");
           //Launch the instance.
           producer.start();
           String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
           for (int i = 0; i < 100; i++) {
               int orderId = i % 10;
               //Create a message instance, specifying topic, tag and message body.
               Message msg = new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
                       ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
               SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
               @Override
               public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                   Integer id = (Integer) arg;
                   int index = id % mqs.size();
                   return mqs.get(index);
               }
               }, orderId);
   
               System.out.printf("%s%n", sendResult);
           }
           //server shutdown
           producer.shutdown();
       }
   }
   
   // 消费者，需要注册MessageListenerOrderly
   public class OrderedConsumer {
       public static void main(String[] args) throws Exception {
           DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");
   
           consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
   
           consumer.subscribe("TopicTest", "TagA || TagC || TagD");
   
           consumer.registerMessageListener(new MessageListenerOrderly() {
   
               AtomicLong consumeTimes = new AtomicLong(0);
               @Override
               public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                          ConsumeOrderlyContext context) {
                   context.setAutoCommit(false);
                   System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                   this.consumeTimes.incrementAndGet();
                   if ((this.consumeTimes.get() % 2) == 0) {
                       return ConsumeOrderlyStatus.SUCCESS;
                   } else if ((this.consumeTimes.get() % 3) == 0) {
                       return ConsumeOrderlyStatus.ROLLBACK;
                   } else if ((this.consumeTimes.get() % 4) == 0) {
                       return ConsumeOrderlyStatus.COMMIT;
                   } else if ((this.consumeTimes.get() % 5) == 0) {
                       context.setSuspendCurrentQueueTimeMillis(3000);
                       return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                   }
                   return ConsumeOrderlyStatus.SUCCESS;
   
               }
           });
   
           consumer.start();
   
           System.out.printf("Consumer Started.%n");
       }
   }
   ```

3. 总结,  rocketmq的顺序消息需要满足2点：

   1.Producer端保证发送消息有序，且发送到同一个队列。
   2.consumer端保证消费同一个队列。

4. 说明 MessageListenerOrderly 并不是简单地禁 并发处理 。 在 MessageListenerOrderly 的实现中，为每个 ConsumerQueue 加个锁，消费每个消息前，需要先获得这个消息对应的 Consumer Queue 锁，==这样保证了同一时间， 同一个Consumer Queue 的消息不被并发消费，但不同 Consumer Queue 的消息可以并发处理==。






## 4.2 消息重复

1. 同时做到确保一定投递和不重复是很难的，RocketMQ选择了确保一定投递，保证消息不丢失，但有可能消息重复。
2. Producer有个函数设置在同步方式下自动重试的测试，在网络波动情况下，Broker端收到了消息但是没有正确返回发送成功的状态，就产生了消息重复。
3. 需要用户程序来实现消息的幂等能防止消息重复。



## 4.3 消息过滤

1. 消息过滤的方式：Tag、SQL表达式、Filter Server方式。
   - Tag: 在Consumer订阅Topic的时候设置，是Broker进行对Tag的hash值来进行过滤的。
   - SQL表达式：支持逻辑复杂但实现费时。



## 4.3.1 Filter Server方式过滤

1. 要使用Filter Server，首先要在启动 Broker 前在配置文件里加上 filterServerNums = 3 这样的配 置 ， Broker 在 启动的时 候 ， ==就会在本机启动3个Filter Server 进程==。 
2. Filter Server 类似一个RocketMQ的Consumer进程， **它从本机Broker获取消息， 然后根据用户上传过来的 Java 函数进行过滤， 过滤后的消息再传给远端的 Consumer**。







# 5. 消息查询

1. RocketMQ的消息查询有两种：broker生成的消息ID、客户端SDK生成的UniqueKey。

2. 按照MessageID查询(依赖CommitLog)，==因为Broker生成MessageId的时候带了物理绝对地址。==

   - MQAdminImpl.viewMessage(String MessageId)

   - MQAdminImpl.viewMessage(**final** String addr, **final long** phyoffset, **final long** timeoutMillis)

   - QueryMessageProcessor.viewMessageById(ChannelHandlerContext ctx, RemotingCommand request)

3. 按照uniqueKey/Keys查询(依赖Index和CommitLog)

   - MQAdminImpl##queryMessage(String topic, String key, **int** maxNum, **long** begin, **long** end,**boolean** isUniqKey)
   - MQAdminImpl##queryMessage(final String addr,final QueryMessageRequestHeader requestHeader,final long timeoutMillis,final InvokeCallback invokeCallback,final Boolean isUnqiueKey)
   - QueryMessageProcessor##queryMessage(ChannelHandlerContext ctx, RemotingCommand request)












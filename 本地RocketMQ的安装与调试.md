**本地RocketMQ的安装与调试**

标签：【RocketMQ】

# 1. 启动

1. 进入RocketMQ-ALL的源码项目。

2. 执行maven打包：

   > mvn -Prelease-all -DskipTests clean install -U

3. ~~进入打包好的文件~~

   >cd /Users/rtw/IdeaProjects/RocketMQ/distribution/target/apache-rocketmq/bin

4. 进入distribution/conf. 将其中的broker.conf 、 logback_broker.xml、logback_namesrv.xml复制到rocketmq_all/conf下。

5. 修改broker.conf 的配置:

   ```config
   brokerClusterName = DefaultCluster
   brokerName = broker-a
   brokerId = 0
   deleteWhen = 04
   fileReservedTime = 48
   brokerRole = ASYNC_MASTER
   flushDiskType = ASYNC_FLUSH
   namesrvAddr=127.0.0.1:9876
   #存储路径
   storePathRootDir=/Users/rtw/IdeaProjects/RocketMQ/store
   #commitLog 存储路径
   storePathCommitLog=/Users/rtw/IdeaProjects/RocketMQ/store/commitlog
   #消费队列存储路径存储路径
   storePathConsumeQueue=/Users/rtw/IdeaProjects/RocketMQ/store/consumequeue
   #消息索引存储路径
   storePathIndex=/Users/rtw/IdeaProjects/RocketMQ/store/index
   #checkpoint 文件存储路径
   storeCheckpoint=/Users/rtw/IdeaProjects/RocketMQ/store/checkpoint
   #Broker 的角色
   #- ASYNC_MASTER 异步复制Master
   #- SYNC_MASTER 同步双写Master
   #- SLAVE brokerRole=ASYNC_MASTER
   #刷盘方式
   #- ASYNC_FLUSH 异步刷盘
   #- SYNC_FLUSH 同步刷盘 flushDiskType=ASYNC_FLUSH
   #checkTransactionMessageEnable=false
   #abort 文件存储路径
   abortFile=/Users/rtw/IdeaProjects/RocketMQ/store/abort
   ```

6. 运行org.apache.rocketmq.namesrv.NamesrvStartup，需要进行配置：

   - 添加Environment variables:  ROCKETMQ_HOME=/Users/rtw/IdeaProjects/RocketMQ

7. 运行org.apache.rocketmq.broker.BrokerStartup，需要进行配置：

   - 添加Environment variables:  ROCKETMQ_HOME=/Users/rtw/IdeaProjects/RocketMQ
   - 配置Program arguments,也就是项目的broker.conf的位置: 
   > -c /Users/rtw/IdeaProjects/RocketMQ/conf/broker.conf

8. 日志位置见RocketMQ/conf/logback_broker.xml.

9. 运行org.apache.rocketmq.example.quickstart.Producer创建消息，注意需要添加producer.setNamesrvAddr("127.0.0.1:9876");

10. 运行org.apache.rocketmq.example.quickstart.Consumer消费消息，注意添加consumer.setNamesrvAddr("127.0.0.1:9876");



# 2. RocketMQ

1. RocketMQ是一款高性能消息中间件，其核心的优势：
   - 可靠的消息存储。
   - 消息发送的高性能与低延迟。
   - 强大的消息堆积能力与消息处理能力。
   - 严格的顺序消息存储。
   - 懂得取舍，消息中间件的理想状态是一条消息能且只被消费一次，但要做到这一点必然需要牺牲性能。**RocketMQ保证消息至少被消费一次，但不承诺消息不会被消费者多次消费。其消息的幂等由消费者自己实现**。



## 2.1 设计理念

1. 使用NameServer，摒弃了业内常用的Zookeeper充当信息管理的“注册中心”。
   - 因为Topic路由信息无须在集群之间保持强一致性，追求最终一致性，并且能容忍分钟级的不一致。
2. 高效的IO存储机制。
3. 容忍设计缺陷，RocketMQ保证消息至少被消费一次，但不承诺消息不会被消费者多次消费。其消息的幂等由消费者自己实现。



## 2.2 设计目标

RocketMQ作为一款消息中间件，需要解决如下问题：

1. 架构模式：

   - RocketMQ与大部分消息中间件一样，采用发布订阅模式，基本的参与组件主要包括：消息发送者、消息服务器（消息存储）、消息消费、路由发现。

2. 顺序消息：

   - 消息消费者按照消息达到消息存储服务器的顺序消费。

3. 消息过滤：

   - RocketMQ消息过滤支持在 ==服务端== 与 ==消费端==的消息过滤机制。
   - 消息在Broker端过滤。Broker可以只将消息消费者感兴趣的消息发送给消息过滤机制。
   - 消息在消费端过滤，消息过滤方式完全由消息消费者自定义，但**缺点是有很多无效消息会从Broker传输到消费者。**

4. 消息存储：

   - 消息的堆积能力 和 消息存储性能。RocketMQ追求消息存储的高性能，引入内存映射机制，所有有主题的消息顺存储在同一个文件中。

5. 消息高可用性：

6. 消息到达（消费）低延迟：

7. 确保消息必须被消费一次：

   - RocketMQ通过消息消费确认机制（ACK）来确保消息至少被消费一次，但由于ACK消息有可能丢失等其他原因，==RocketMQ无法做到消息只被消费一次，有重复消费的可能。==

     



# 3. 启动参数

1. brokerRole=？；SYNC_MASTER、ASYNC_MASTER、SLAVE。SYNC和ASYNC表示Master和Slave同步消息设置，SYNC的意思是当Slave和Master消息同步完成后，再返回发送成功的状态。




























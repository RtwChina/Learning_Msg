**RocketMQ路由中心**

标签：【RocketMQ】



# 1. 概述NameSrv

1. NameServer 是整个消息队列中 的状态服务器， 集群的各个组件通过它来了 解全局的信息 。 同时午， 各个角色的机器都要定期 向 NameServer 上报自己的状 态，超时不上报的话， NameServer 会认为某个机器出故障不可用了， ==其他的组件会把这个机器从可用列表里移除== 。
2. NameServer 本身是无状态的， 也就是说NameServer中的Broker 、 Topic等状态信息不会持久存储， ==都是由各个角色定时上报并存储到内存中的==（NameServer **支持配置参数的持久化**， 一般用不到） 。



##RocketMQ整体架构部署图

![RocketMQ整体架构部署图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-11-012219.png)

- 每个NameServer之间是不通讯的。
- Producer、Consumer 同一时间，与NameServer集群中其中一台建立长连接。
- Producer与每一个Broker的Master保持长连接。
- Broker中的Master与Slave与每一台NameServer保持长连接。
- Consumer与所有的Broker建立长连接。





# 2. NameSrv相关组件介绍

## 2.1 RouteInfoManager

1. 存储了topic对于的MessageQueue，所有Broker及其相关信息等。
2. 详见代码注释

## 2.2 MQClientInstanc

1. RocketMQ中消息发送者、消息消费者都属于”客户端“，每一个客户端就是一个MQClientInstance，每一个ClientConfig对应一个示例，故不同的生产者、消费端，如果引用同一个客户端配置(ClientConfig)，则它们共享一个MQClientInstance实例。

2. 客户端（MQClientInstanc）中连接的建立时机为按需创建，也就是在需要对端进行数据交互时才建立的。

# Broker启动

![Broker启动流程简要梳理](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-11-013432.png)

- BrokerOuterAPI#registerBrokerAll中就是把心跳信息发送至NameServer。







# 3. NameSrv启动

1. NameServer的地址和端口号可以填写多个，用分号隔开。
2. 使用org.apache.rocketmq.namesrv.NamesrvStartup.main0( )启动。
3. 在NamesrvStartup.createNamesrvController方法，该方法主要是根据配置信息初始化NameSrv。
   - namesrvConfig，nettyServerConfig，分别为业务参数、网络参数。
4. 看回来start(controller);该方法正在开启了NameSrv:
   - 创建线程池。（initialize方法中）
   - 为线程池添加任务：定时任务1：NameServer每隔10S扫描一次Broker，移除处于不激活状态的Broker（initialize方法中）
   - 为线程池添加任务：定时任务2：NameServer每隔10分钟打印一次Broker配置（initialize方法中）
   - 添加线程池的注册钩子



# 4. 路由元信息

1. 元信息都放在RouteInfoManager类中，详见源代码，即阮天天的注释。

2. 需要注意的是brokerAddrTable存放的是Broker(可以是主从模式)， clusterAddrTable存放的是集群Broker

3. ```java
   1. clusterName 集群名称
   2. brokerName Broker名称，其中主从模式的brokerName是同一个，只不过地址不同，brokerId=0表示Master, 大于0表示从Slave
   ```



# 5. 路由注册

## 5.1 Broker处理心跳包

1. RocketMQ的路由注册是通过Broker与NameServer的心跳功能实现的。

2. Boker的心跳包是 BrokerController的start中发送的：
   -  BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
   - doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
3. RocketMQ 的网络传输基于Netty，每一个请求RocketMQ都会定义一个RequestCode.



## 5.2 NameSrv处理心跳包

1. 入口在DefaultRequestProcessor的processRequest中：
   - 当收到request的Code为REGISTER_BROKER的话，会走到registerBrokerWithFilterServer中

2. 主要操作在 RouteInfoManager的registerBroker中，会维护集群，BrokerData。



### 5.2.1 设计亮点

1. NameServe与Broker保持长连接，Broker状态存储在brokerLiveTable中，NameServe每收到一个心跳包将更新brokerLiveTable中关于Broker的状态信息。
2. 在同一时刻NameServe只处理一个Broker心跳包，多个心跳包请求串行执行。这也是读写锁经典实用场景。（RouteInfoManager的registerBroker方法）



# 6. 路由删除



# 7. 路由发现

1. 路由发现是非实时的，当Topic路由出现变化后，NameSrv不主动推送给客户端，而是由客户端定时拉取主题最新的路由。
2. requestCode= GET_ROUTEINTO_BY_TOPIC; 具体实现在DefaultRequestProcessor的getRouteInfoByTopic中。

# 8. 路由总结

![路由维护机制](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-02-15-014622.jpg)



1. 这就有一个问题，Broker的宕掉会在120S后才会被NameSrv发现，那么在这期间Producer获得宕掉Broker怎么办呀？请看下一节。



![RocketMQ Topic路由注册、剔除机制 ](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-13-141752.png)

1、Broker向NameServer以每30s的频率向NameServer发送心跳包。

2、NameServer收到Broker的心跳包时会记录收到心跳包的时间。

3、NameServer以每10s的频率扫描Broker注册表，移除在连续120s内未收到的Broker。

注：NameServer的实现基于内存，NameServer并不会持久化路由信息，持久

化的重任是交给Broker来完成。
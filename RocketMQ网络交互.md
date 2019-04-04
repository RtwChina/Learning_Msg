**RocketMQ网络交互**

标签：【RocketMQ】



[TOC]

# 1. 大致结构

![RocketMQ整体架构部署图](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-03-17-020220.png)

- 每个NameServer之间是不通讯的。
- Producer、Consumer 同一时间，与NameServer集群中其中一台建立长连接。
- Producer与每一个Broker的Master保持长连接。
- Broker中的Master与Slave与每一台NameServer保持长连接。
- Consumer与所有的Broker建立长连接。

### MQ客户端的通讯

1. Consumer和Producer会每隔两分钟与用户所注册的NameSrv保持通讯。主要是心跳。
2. 每隔30S，更新当前所有与当前有关的Topic的路由元信息（包括Consumer和Producer）,路由元信息包括：
   - 每一个Topic对应的，所有与之相关的TopicQueue和BrokerQueue。
3. 每隔30S，进行Broker心跳检测，因为我们在第二点的时候已经从namesrv中获得所有与客户端有关系的broker信息，因此我们可以在第三点进行心跳检测。



### 服务端网络组件启动过程

1. BrokerController负载管理各组件的生命周期，initialize、start、shutdown分别负责初始化、启动、关闭各组件，在初始化阶段针对不同请求注册processor
2. processor分为七大类：send,pull,query，clientManage(心跳)，consumerManage(offset管理)，endTransaction(提交或回滚事务)，deafult(控制台命令)，不同请求由不同线程池处理。
3. NettyServerConfig中listenPort默认是8888，但BrokerStartUp启动时改成了10911.
4. 分别在10909和10911启动监听，10909为fastRmotingServer，**也被称为VIP通道，所有发送消息请求和控制命令走10909.**



### 服务端Handler

1. 入站Handler

   - HandshakeHandler，第一个入站handler，当客户端配置userTLS=true时生效，为服务端动态创建SSLHandler,并动态删除自己。

   - NettyDecode,解码器，根据长度字段解析,解决拆包粘包问题,格
     式协议见 rocketing_ design4,先去掉 length(数据包总长度)字段,再解析
     header length、 header、body

   - NettyserverHandler,处理具体业务逻辑

2. 出站 Handler:
     - Netty encoder,绷码器,按协议拼装报文
     - FileRegion Encoder,.只在客广端配置ses时生效,将到 eRegion转化为
       Byte Buf, SSLHandleri要 ByteBu(类型

3. 出入站 Handler: 

  - dlestatehandle空闲连接检查,依赖 defaultEvent ExecutorGroup线程执行调度任务,发现空闲连接触发DLE事件,由 NettyConnectManageHandler处理IDLE事件
	- Netty ConnectManageHandler,连接管理,主要用来关闭连接
  - SSLHandler,第一个入站 handler,最后一个出站 handler

# 2. 类结构

![rocketmqRemoting类图](../../../../rocketmqRemoting类图.png)

- 服务端 NettyRemotingServer
- 客户端 NettyRemoingClient
- 类图 doc/cn/image/rocketMQ_design_3



# 源码分析



## 入口

1. 在NettyRemotingServer的构造函数中，是MQ的Netty服务端启动初始化的代码：
   - 会根据是否开启useEpoll来开启Linux使用Epoll来作为Netty的EventLoopGroup
   - EventLoopGroup最主要的作用是在其EventLoop的Run方法中。

2. NettyRemotingServer##satart主要是对Server参数的配置，可以看到是一些Netty的BoosStrap的配置，包括很多Handler。

   - HandShakeHandler是比较特殊的，主要是动态添加SSLHandler并删除自己。

3. 源码

   ```java
   ServerBootstrap childHandler =
   this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
       .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
       .option(ChannelOption.SO_BACKLOG, 1024)
       .option(ChannelOption.SO_REUSEADDR, true)
       .option(ChannelOption.SO_KEEPALIVE, false)
       .childOption(ChannelOption.TCP_NODELAY, true)
       .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
       .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
       .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
       .childHandler(new ChannelInitializer<SocketChannel>() {
           @Override
           public void initChannel(SocketChannel ch) throws Exception {
               ch.pipeline()
                   .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME,   // 这里的handler是单独配了一个线程池，因此不是在worker中进行handler的运行
                       new HandshakeHandler(TlsSystemConfig.tlsMode))
                   .addLast(defaultEventExecutorGroup,
                       new NettyEncoder(),
                       new NettyDecoder(),
                       new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                       new NettyConnectManageHandler(),
                       new NettyServerHandler()
                   );
           }
       });
   ```

   

### HandlShakeHandler

1. FileRegionEncoder是一种零拷贝的实现





### **NettyDecoder**

![RocketMQ网络协议](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-04-03-012134.png)

1. 解码：主要是将ByteBuff解码出来，封装成一个RemotingCommand对象。



### RemotingCommand协议

1. 该协议是RocketMQ进行网络通讯的基础协议。其中NettyDecode, NettyEncode都是对该协议的处理。

![RocketMQ网络协议](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-04-03-012134.png)

- Message Length: 长度为4个字节，表示type&header length + header + body的长度，不包含本身。
- type + header length: 长度为4个字节，最高位字节表示序列化类型，其他三个字节表示header字段的长度。
- header, 定义多种类型的header, POJO
- Body，消息体。



# RocketMQ线程模型

![MQ的线程模型](http://rtt-picture.oss-cn-hangzhou.aliyuncs.com/2019-04-04-014300.png)

- Boos线程主要用来处理新连接。
- Worker线程只是用于将请求参数分发到DefaultEventExecutorGroup线程池中。
- DefaultEventExecutorGroup：将请求在HandlerPirpline中跑一遍。(在childHandler的时候配置的)
- 业务逻辑线程：具体的业务逻辑会在自己配置的线程池中运行。BrokerController的registerProcessor中配置



# 消息发送Netty流程

1. 消息发送分为同步，异步，一次性，入口是在NettyRemotingClient##invoke**



### 异步发送

1. 会调用到NettyRemotingAbstract##invokeAsyncImpl：

   ```java
   public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
           final InvokeCallback invokeCallback)
           throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
           long beginStartTime = System.currentTimeMillis();
           final int opaque = request.getOpaque();
           // 信号量来控制非同步发送流程
           boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
           if (acquired) {
               final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
               long costTime = System.currentTimeMillis() - beginStartTime;
               if (timeoutMillis < costTime) {
                   once.release();
                   throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
               }
   
               final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
               // 缓存请求结果，在请求返回回来后才能继续执行后续回调操作
               // opaque是请求和结果标识的唯一id
               this.responseTable.put(opaque, responseFuture);
               try {
                   channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                       @Override
                       public void operationComplete(ChannelFuture f) throws Exception {
                           if (f.isSuccess()) {
                               // 发送成功需要标识以下
                               responseFuture.setSendRequestOK(true);
                               return;
                           }
                           // 处理发送失败的情况
                           requestFail(opaque);
                           log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                       }
                   });
               } catch (Exception e) {
                   // 异常处理
               }
           } else {
               // 控流出现
           }
       }
   ```

   - 信号量来控制非同步发送流程。
   - 缓存请求结果，在请求返回回来后才能继续执行后续回调操作opaque是请求和结果标识的唯一id，存放在NettyRemotingAbstract#responseTable中，
   - 添加发送是否成功的后续处理



### 异步消息发送响应处理

1. NettyClientHandler ————》 NettyRemotingAbstract.processMessageReceived

2. processResponseCommand方法是处理请求响应的。






























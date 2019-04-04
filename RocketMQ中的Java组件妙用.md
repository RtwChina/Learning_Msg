**RocketMQ中的Java组件妙用**

标签：【RocketMQ】【Java】

​    

# Atomic

1. 在ProcessQueue中用于对ProcessQueue的消息计数。



# TreeMap

1. TreeMap是一个==有序==的Key-Value集合，是通过==红黑树==实现的。因此其基本操作containsKey、get、put和remove的时间复杂度是log(n)。
2. 要注意的是TreeMap不是线程安全的，因此在RocketMQ中使用了ReadWriteLock来进行锁定。
3. 因为在RocketMQ中TreeMap<Long, MessageExt>，Key使用里以MessageQueue的Offset作为Key，Value是其具体的消息内容。

### MQ中的应用

1. 作为PrecessQueue中的一个属性，主要用作消息存储容器：键为消息在ConsumeQueue中的偏移量，MessageExt为消息实体。



### put特性

1. 和常用的HashMap一样，当讲数据put到map的时候，如果原先Key中已经有值了，会替代点且返回原先的值。
2. 那么可根据其返回的值是否为空来判断这次添加数据到treeMap中有没有增加其容量。
3. 为什么有可能多次将相同的key塞到TreeMap?
   - 应该是一定程度上解决消息重复消费。

### 有序性

1. 可直接通过LastKey和firstKey获得两者的key值，拿到消息差。
2. 可按照MessageQueue的offset去ProcessQueue中取最小的offset的message：
   - takeMessags—》ConsumeMessageOrderlyService中的Run进行消费，也就是传说中顺序消费。



# Timer

1. 先明确一下Timer:
   - 内部实现很简单，维护一个数组，该数组中存放的是TimerTask(即runnable)，然后该数组使用了堆排序！！。也就是说第一个数组一定是最小的（可根据TimerTask下一次运行的时间进行排序）,
   - 起一个单线程，疯狂循环，然后取出数组中“最小的”TimerTask进行执行。
2. 缺点：
   - 单线程，因此速度会很慢哦！
   - 当一个任务爆掉（抛出异常）的时候，其他所有的任务都会同归于尽。

## 应用

1. RocketMQ中的ScheduleMessageService使用TImer类来存储定时任务执行的TimeTask, 每一个延时级别都有一个TimeTask实例（DeliverDelayedMessageTimerTask），那么也就是说Timer实例中的==堆队列具有18个延时任务==，在达到相应时间点的时候就会自动触发延时任务。

## 原因

1. Time的优点主要是简单易用，维持一个线程就可以了。
2. 因为延迟任务中主要执行的是去DefaultMessageStore#consumeQueueTable中取出延时队列Topic的ConsumerQueue的消息对象。将消息的Topic等换成原生Topic后再异步刷到ComiitLog中，因此耗时不长。使用了Timer！
3. 其他为什么使用Timer，不使用SchedulePoolExecutor的原因我没找到。



# CopyOnWriterArrayList



## 应用

1. 在MappedFileQueue中，具有当前CoomitLog、ConsumerQueue、IndexFile的每个文件的映射文件即MappedFile.
2.  CopyOnWriteArrayList<_MappedFile> mappedFiles 来存储MappedFile。
3. mappedFiles的初始化是在MappedFileQueue##load中，是作为一个独立线程运行的。
4. 下面是一个获取最后一个MappedFile的代码：

```java
// 获取最后一个MappedFile
    public MappedFile getLastMappedFile() {
        MappedFile mappedFileLast = null;

        while (!this.mappedFiles.isEmpty()) {
            try {
                mappedFileLast = this.mappedFiles.get(this.mappedFiles.size() - 1);
                break;
            } catch (IndexOutOfBoundsException e) {
                //continue;
            } catch (Exception e) {
                log.error("getLastMappedFile has exception.", e);
                break;
            }
        }

        return mappedFileLast;
    }
```





## ConcurrentHashMap  PriorityBlockingQueue

1. 在AllocateMappedFileService中 还有CountDownWatch





# ServiceThread



# **Semaphore**

1. 信号量
2. 在NettyRemotingAbstract中会用到Semaphore




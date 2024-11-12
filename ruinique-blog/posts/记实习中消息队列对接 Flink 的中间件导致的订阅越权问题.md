---
title: 记实习中消息队列对接 Flink 的中间件导致的订阅越权问题
cover: /kuaishou_bug.jpg
date: 2024-03-05 10:00:00
categories: 技术
author: ruinique
---
# 背景

笔者在实习的时候遇到了一个很奇怪的问题，在使用 Flink 组件的时候，我们利用 RocketMQ 作为我们的消息源，使用了公司的中间件。在单消息源聚合的时候，我们没有发现什么问题，但是在多个消息源聚合的时候，我们发现我们的消费者组订阅了不属于自己的 topic 的消息（也没有消费，因为我们的消息消费会先序列化成对象，如果消费了，就会消费失败，但是我也没看到消费失败的消息），遂开始排查。

# 排查过程

## 定位问题出处

既然是我们的消息订阅出了问题，我们先去检查了是否我们显式的填错了，幸运而又遗憾的是，我们在这方面没有犯错。那么接下来我们开始从涉及到 `RocketMQ DataSource` 的地方排查起，因为我们这个函数前面不涉及 `RocketMQ`，这个函数调用的后面又直接是反序列化操作了，所以问题一定处在这个函数这里。
```java
public RocketmqSource<XxxMsg> getRocketmqSource() {  
    return getRocketmqSourceBase(XxxMsg.class,"topic","group");
```
我们接着看 `getRocketmqSourceBase` 的源码：
```java
protected <T extends GeneratedMessageV3> RocketmqSource<T> getRocketmqSourceBase(Class<T> tClass  
        , String topic, String group) {  
    return new RocketmqSource<>(  
            getMsgProtoBufRocketMqDeSchema(tClass),  
            getConsumerProperties(topic, group));  
}
```
可以发现我们这个函数只是简单的封装了一下 `topic` ， `group` 和 `deSchema` 的配置， 这个时候已经开始不对劲了，因为这个 `RocketmqSource` 是我们引入的实习我司的第三方中间件，所以接下来的代码都是反编译的 QAQ。
先看一下我们 `RocketmqSource` 构造函数的实现：
```java
public RocketmqSource(DeserializationSchema<OUT> schema, Properties properties) {  
    String topic = properties.getProperty("connector.topic");  
    String group = properties.getProperty("connector.group");  
    String appKey;  
    if (!RocketmqUtils.judgeConfig(topic, group, false)) {  
        appKey = "topic 或 group配置错误， topic: " + topic + ", group: " + group;  
        throw new MQConnectorException(appKey);  
    } else {  
        this.properties = properties;  
        this.deserializer = new InnerDeserializer(schema);  
        appKey = properties.getProperty("connector.app-key", "general");  
        if (!FrameworkFacade.verifyAppKey(appKey, group, topic, false)) {  
            throw new MQConnectorException("appKey verify error, key is: " + appKey + ", topic is: " + topic + ", group is: " + group);  
        }  
    }
```
可以看出这里面也只是拿到我们的 `topic` 和 `group` 并且校验了他们（PS：这里的 `judgeConfig()` 只校验了字符串合法性）。
这时候看似遇到了僵局，但是实际上我们不能光看构造函数，因为像这种中间件里的这种 `Source` 往往实现了某个生命周期，我们看到他继承了了 `RichParallelSourceFunction` 和三个接口：
1. `CheckpointedFunction`：这个接口用于定义在 `Flink` 任务进行 `checkpoint` 时需要执行的操作，如保存和恢复状态。  
2. `CheckpointListener`：这个接口用于定义在 `Flink` 任务的 checkpoint 完成或失败时需要执行的操作。  
3. `ResultTypeQueryable<OUT>`：这个接口用于定义如何获取 `RocketmqSource` 输出的类型信息。
当然我们这里设计的这个生命周期和这三个接口暂时不涉及这三个接口，真正有关的是 `RichParallelSourceFunction`。
这个东西是 `Flink` 中 `Rich Function` 的一种，`Rich Function` 是 `Flink` 中的一个概念，它是所有函数的基类，提供了一些生命周期方法和运行时上下文的访问。富函数有以下特性：  
1. 生命周期方法：富函数有 `open` 和 `close` 方法，这两个方法在 `Flink` 任务的生命周期中会被调用。`open` 方法在任务初始化时被调用，可以用于执行一些初始化操作；`close` 方法在任务结束时被调用，可以用于执行一些清理操作。  
2. 运行时上下文访问：富函数可以通过 `getRuntimeContext` 方法访问运行时的上下文，这个上下文提供了一些有用的信息，如当前子任务的索引、当前任务的并行度等。  
3. 配置参数访问：富函数可以通过 `getIterationRuntimeContext` 方法访问迭代运行时的上下文，这个上下文提供了一些有用的信息，如当前迭代的超步数等。
这个 `RichParallelSourceFunction` 则是一个并行的作为消息源的抽象类，作为消息源的话就涉及到 `Flink` 有界流，无界流，checkpoint，watermark 那些概念了，并行则可以起多线程去做任务，这里我们暂时都不 care，我们看到这个生命周期以后，我们就知道可能在 `open()` 里面被订阅了，下面是 `open()` 在 `RocketmqSource` 中的实现：
```java
public void open(Configuration parameters) throws Exception {  
    Map<String, String> confMap = parameters.toMap();  
    this.properties.putAll(confMap);  
    this.sourceConfig = RocketmqUtils.genSourceConfig(this.properties);  
    RuntimeContext ctx = this.getRuntimeContext();  
    int indexOfTask = ctx.getIndexOfThisSubtask();  
    int taskNumber = ctx.getNumberOfParallelSubtasks();  
    boolean enableCheckpoint = ((StreamingRuntimeContext)this.getRuntimeContext()).isCheckpointingEnabled();  
    if (this.offsetTable == null) {  
        this.offsetTable = new ConcurrentHashMap();  
    }  
  
    if (this.restoredOffsets == null) {  
        this.restoredOffsets = new ConcurrentHashMap();  
    }  
  
    this.initOffsetTableFromRestoredOffsets();  
    if (this.pendingOffsetsToCommit == null) {  
        this.pendingOffsetsToCommit = new LinkedMap();  
    }  
  
    if (this.waterMarkForAll == null) {  
        this.waterMarkForAll = new WaterMarkForAll(this.sourceConfig.getWaterMarkWindowMill());  
    }  
  
    if (this.timer == null) {  
        this.timer = Executors.newSingleThreadScheduledExecutor();  
    }  
  
    this.runningChecker = new AtomicBoolean(true);  
    Counter outputCounter = this.getRuntimeContext().getMetricGroup().counter("tps_counter", new SimpleCounter());  
    Meter tpsMetric = this.getRuntimeContext().getMetricGroup().meter("tps", new MeterView(outputCounter, 60));  
    this.source0 = new Source0(this.sourceConfig, indexOfTask, taskNumber, this.runningChecker, this.offsetTable, this.deserializer, enableCheckpoint, tpsMetric, this.waterMarkForAll, this.properties);  
    logger.info("source open success,the config is: {}", this.sourceConfig.toString());  
}
```
1. 配置参数的处理：首先，它将传入的 `Configuration` 对象转换为 `Map`，然后将这些配置参数添加到 `properties` 属性中。接着，它使用 `RocketmqUtils.genSourceConfig` 方法生成 `sourceConfig` 对象。  
2. 运行时上下文的获取：然后，它获取了运行时上下文 `RuntimeContext`，并从中获取了当前子任务的索引 `indexOfTask`，任务的并行度 `taskNumber`，以及是否启用了 `checkpoint enableCheckpoint`。  
3. 初始化各种状态和配置：接着，它初始化了一些状态和配置，包括 `offsetTable`、`restoredOffsets`、`pendingOffsetsToCommit`、`waterMarkForAll`、`timer` 和 `runningChecker`。  
4. 创建度量指标：然后，它创建了两个度量指标 `outputCounter` 和 `tpsMetric`，用于监控任务的运行情况。  
5. **创建 `Source0` 对象：最后，它创建了一个 `Source0` 对象，并将之前获取和初始化的各种状态和配置传入。`Source0` 是 `RocketmqSource` 的内部类，用于实现具体的消息消费逻辑。**
我们可以看到具体的逻辑还得到 `Source0` (显然是反编译的锅才叫这个名字）的构造函数里面去看：
```java
class Source0<T> extends Thread implements ConfigChangeListener {
	...
	public Source0(SourceConfig sourceConfig, int taskIndex, int taskTotal, AtomicBoolean runningChecker, Map<MessageQueue, Long> offsetTable, InnerDeserializer<T> deserializer, boolean enableCheckpoint, Meter tpsMetric, WaterMarkForAll waterMarkForAll, Properties userConfig) throws Exception {  
	    this.sourceConfig = sourceConfig;  
	    this.runningChecker = runningChecker;  
	    this.taskIndex = taskIndex;  
	    this.taskTotal = taskTotal;  
	    this.offsetTable = offsetTable;  
	    this.deserializer = deserializer;  
	    this.enableCheckpoint = enableCheckpoint;  
	    this.waterMarkForAll = waterMarkForAll;  
	    this.userConfig = userConfig;  
	    this.tpsMetric = tpsMetric;  
	    this.consumerWrapper = new ConsumerWrapper(sourceConfig, taskIndex, taskTotal, userConfig);  
	    this.setName("consumer-manager-thread");  
	    this.setDaemon(false);  
	    this.buildConfigWatcher();  
	}
	...
}
```
可以发现构造函数还是做了一系列初始化，这里 `ConsumerWrapper` 里面做了 `group` 和 `topic` 之间订阅关系的检查，这也是为啥中间件团队告诉我们不会出现订阅越权的问题。但是 `Source0` 实际上继承了 `Thread` 类，也是说，他在被构造出来的时候相当于起了一个线程，会执行对应的 `run()` 方法，问题就出在 `run()` 方法里：
```java
@Override  
public void run() {  
    boolean consumeStart = false;  
    Map<MessageQueue, Tuple2<Thread, AtomicBoolean>> workers = new ConcurrentHashMap<>();  
    final String threadPrefix = "rmq-pull-thread-";  
    while (runningChecker.get()) {  
        List<MessageQueue> mqs = consumerWrapper.fetchSubjectMessageQueue();  
        if (!consumeStart || !mqs.equals(this.messageQueues)) {  
            long start = System.currentTimeMillis();  
            consumeStart = true;  
            logger.info(  
                    "the route of is updated. {}--{}", this.messageQueues.size(), mqs.size());  
            // 启动新增加的queue  
            for (MessageQueue mq : mqs) {  
                if (!workers.containsKey(mq)) {  
                    AtomicBoolean flag = new AtomicBoolean(true);  
                    Thread t = new Thread(() -> doPull(mq, context, flag));  
                    t.setName(threadPrefix + mq.getBrokerName() + "-" + mq.getQueueId());  
                    t.start();  
                    Tuple2<Thread, AtomicBoolean> tuple2 = new Tuple2<>(t, flag);  
                    workers.put(mq, tuple2);  
                }  
            }  
            // 关闭过期的queue  
            for (Map.Entry<MessageQueue, Tuple2<Thread, AtomicBoolean>> entry :  
                    workers.entrySet()) {  
                if (!mqs.contains(entry.getKey())) {  
                    Tuple2<Thread, AtomicBoolean> value = entry.getValue();  
                    value.f1.set(false);  
                    Thread t = value.f0;  
                    t.interrupt();  
                    try {  
                        t.join(1000);  
                    } catch (InterruptedException e) {  
                        // ignore  
                    }  
                    workers.remove(entry.getKey());  
                }  
            }  
            // 更新messageQueues  
            this.messageQueues = mqs;  
            // 更新offsetTable和关闭consumer顺序不能变  
            checkOffsetTable(mqs, context);  
            // 在所有的线程变更结束后，才最queue做替换。  
            consumerWrapper.finishRouteChangeCheck();  
            logger.info(  
                    "update route finished cost {} Mill", System.currentTimeMillis() - start);  
        }  
        RocketmqUtils.waitForMs(routeUpdateMill);  
    }  
    logger.warn("this task is stopped the main thread will exit");  
    workers.forEach(  
            (mq, tuple2) -> {  
                tuple2.f1.set(false);  
                Thread t = tuple2.f0;  
                t.interrupt();  
                try {  
                    t.join(1000);  
                } catch (InterruptedException e) {  
                    // ignore  
                }  
            });  
    workers.clear();  
    checkOffsetTable(new ArrayList<>(), context);  
    messageQueues.clear();  
    this.consumerWrapper.shutdown();  
    for (ConfigWatcher configWatcher : configWatchers) {  
        configWatcher.stop();  
    }  
}
```
我们可以发现，这个方法里获取 `MessageQueue` 的代码在这里
`List<MessageQueue> mqs = consumerWrapper.fetchSubjectMessageQueue();`
那么问题就出在 `fetchSubjectMessageQueue()` 里面了，我们接下来看 `fetchSubjectMessageQueue()` 的实现：
```java
public List<MessageQueue> fetchSubjectMessageQueue() {  
    List<MessageQueue> queues = new ArrayList();  
    Map<MessageQueue, ConsumerHolder> map = new HashMap();  
    Iterator var3 = this.consumerHolderList.iterator();  
  
    while(var3.hasNext()) {  
        ConsumerHolder consumerHolder = (ConsumerHolder)var3.next();  
        DefaultMQPullConsumer consumer = consumerHolder.getConsumer();  
  
        try {  
            Set<MessageQueue> messageQueues = consumer.fetchSubscribeMessageQueues(this.topic);  
            if (messageQueues != null && !messageQueues.isEmpty()) {  
                List<MessageQueue> allocate = RocketmqUtils.allocate(messageQueues, this.taskTotalNum, this.indexOfThisTask);  
                queues.addAll(allocate);  
                Iterator var8 = allocate.iterator();  
  
                while(var8.hasNext()) {  
                    MessageQueue queue = (MessageQueue)var8.next();  
                    map.put(queue, consumerHolder);  
                }  
            }  
        } catch (Exception var10) {  
            logger.error("fetchSubjectMessageQueue error， it will retry in next circle, {}", var10.getMessage());  
        }  
    }  
  
    this.fetchedMqMap = map;  
    this.queueMap.putAll(map);  
    return queues;  
}
```
我们可以发现他是直接去遍历我们的 `this.consumerHolderList` ，那么这玩意又是怎么初始化的呢？
他在 `ConsumerWrapper` 的 `init()` 方法里被初始化：
```java
private void init() throws Exception {  
    List<ClusterInfo> clusterInfos = this.sourceConfig.getClusterInfos();  
    if (clusterInfos.isEmpty()) {  
        PerfUtil.logPerf(System.nanoTime(), "mq.consumer.start", "ConfigEmpty", this.topic, this.group);  
        String errMsg = "请检查topic和group是否填错，请检查申请单是否正确上线";  
        logger.error(errMsg);  
        throw new MQConnectorException(errMsg);  
    } else {  
        Iterator var2 = clusterInfos.iterator();  
  
        while(var2.hasNext()) {  
            ClusterInfo info = (ClusterInfo)var2.next();  
            String key = this.buildKey(info, this.indexOfThisTask);  
            synchronized(CONSUMER_MAP) {  
                ConsumerHolder consumerHolder = (ConsumerHolder)CONSUMER_MAP.get(key);  
                if (consumerHolder == null) {  
                    consumerHolder = new ConsumerHolder(this.sourceConfig, info, this.indexOfThisTask, this.taskTotalNum, this.userConfig, key);  
                    CONSUMER_MAP.put(key, consumerHolder);  
                } else {  
                    consumerHolder.addRefCount();  
                }  
  
                this.consumerHolderList.add(consumerHolder);  
            }  
        }  
  
        this.fetchSubjectMessageQueue();  
    }  
}
```
他主要干了这几件事：
1. 获取集群信息：首先，它从 `sourceConfig` 中获取集群信息 `clusterInfos`。  
2. 检查集群信息：如果 `clusterInfos` 为空，那么它会记录一条性能日志，并抛出一个 `MQConnectorException` 异常。  
3. 创建消费者：**如果 `clusterInfos` 不为空，那么它会遍历 `clusterInfos`，对于每一个 `ClusterInfo`，它会生成一个键 `key`，然后在 `CONSUMER_MAP` 中查找是否已经存在一个 `ConsumerHolder`。如果不存在，那么它会创建一个新的 `ConsumerHolder`，并将其添加到 `CONSUMER_MAP` 和 `consumerHolderList` 中。如果已经存在，那么它会增加 `ConsumerHolder` 的引用计数，并将其添加到 `consumerHolderList` 中**。
4. 获取消息队列：最后，它调用 `fetchSubjectMessageQueue()` 方法获取消息队列。
接下来我们要看看这个 `clusterInfos` 是从哪弄的：
追溯到上面的复制，我们可以发现 `open()` 这里面有一个 `this.sourceConfig = RocketmqUtils.genSourceConfig(properties);`
它是根据我们的配置信息拿的集群信息。
那么问题出现在哪呢？这里的 `CONSUMER_MAP` 他设置成静态的了，也就是所有实例共享的，代码如下：
`private static final Map<String, ConsumerHolder> CONSUMER_MAP = new ConcurrentHashMap();`
自然也就在我们进行消费的时候，最后也会放到我们的 `consumerHolderList` 里面，最后也就不管三七二十一全部给他订阅上了，这就引发了交叉订阅。`MAP` 里面的 `key` 是集群+ `taskIndex`维度，如果同一个`taskIndex`有多个消费组，那么这些消费组用是其中某个消费组的底层`RockemqConsumer`实例，用这个实例进行`fetch`和消费时，自然会记录为这个实际消费组，而非业务`ConsumerWrapper`中的消费组。

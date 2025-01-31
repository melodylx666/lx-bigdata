# 动态CEP解析

## 参考资源

[架构文章](https://juejin.cn/post/7218824938093920312)。
[shirukai/flink-dynamic-cep: FlinkCEP动态多规则实现及自定义规则处理引擎 (github.com)](https://github.com/shirukai/flink-dynamic-cep)


## 案例与实现

主要是对于引入开源的项目的探究和总结，并对多个不同开源实现的的差异进行对比。

### 实现的需求

开源版本的CEP的pattern到NFA这一步是在编译时候就完成的，运行时不能修改。
那么对于单规则修改，或者是多规则的执行，就无法实现，或者是代价很大。所以很多公司都根据自己的需求选择对开源版本代码进行修改，从而实现动态规则修改。

### 实现思路

#### 背景

以经典的机房温度检测场景为例子，假设现在进入的一个$record$的格式是`|rackID|温度|功率|其他|,`而我们的需求是需要针对不同机房温度的变化趋势进行检测，从而可以进行Warning或者是Alert。

### 方法

面对这个问题，使用CEP的通用方法是对以$rackID$为键，做一个键控流，然后对keyedStream上做SimpleCondition，或者是IterativeCondtion，从而实现基于当前数据,[ 历史数据 ] 的检测，然后最后在对PatternStream上执行ProcessFunc逻辑。

这个场景中，pattern初始化的nfa模型是固定的，也就是CepOperator类中的

```
private final NFACompiler.NFAFactory<IN> nfaFactory;

private transient NFA<IN> nfa;
```

成员变量，它并不能支持运行时的动态修改。

为了能够修改，一种可能的实现如下(类似PPT链接中的实现，是一个三角形)(暂时讨论keyedStream)：

1. 把规则存储DB中，并在JobMaster端启动异步线程(通过Coodrinator)，对于规则进行定时扫描
2. 实现两对Operator-Coordiantor。
   1. 其中的RuleDistributorCoordinator，通过外部实现的discover，对于DB进行定时扫描，并将数据读取，其一部分生产者的作用。
   2. 而RuleProcessCoordinator，通过Coordinator store同RuleDistributorCoordinator获取数据，同样可以拿到变更的数据(不再次扫描，应该是为了对齐和同步)，并且起到一部分消费者线程的作用。
3. 两个Coordinator都基于事件驱动，与对应的Operator(subtask)进行通信
4. 当数据流(键控流)进入之后，RuleDistributorOperator对于数据流进行修改，将elem和rule进行拼接，形如$elem -> (elem,rule1),(elem,rule2)$,默认情况下是如此。也通过参数控制。，然后对于新的keyedStream,以$rackID,ruleID$ 拼接，变为key，再进行一次keyBy。
5. 对于新的keyBy流，RuleProcessOperator对于通过与Coordinator的最新通信，得到最新的规则信息，从而动态加载pattern与function，然后动态生成nfa,进行匹配。

## 具体实现思路

Flip200中开始提到了我们需要添加的三类接口/类，分别是

1. PatternProcessor接口: 具有keyby+process的组合功能
2. PatternProcessorManger接口: 用于处理模式更新，并提供当前processor的信息
3. PatternProcessorDiscover接口：发现模式更改，并通知processor进行更新
4. PatternProcessorDiscoverFactory接口: 引入 PatternProcessorDiscovererFactory 是为了使用 PatternProcessorManager 创建 PatternProcessorDiscoverer
5. CEP.patternProcess():将数据流转换为匹配之后的流，并返回结果

并建议添加两个算子

1. key-gen-operator:对于每个进来的数据，对每种规则进行拼接，用dataid 和 ruleid 进行拼接，作为数据的key
2. 紧接着，做一次keyby，使用生成的key
3. cep-operator: 真正做模式匹配的operator。详细责任见原文如下：
   > The first duty is almost the same as that of CepOperator, except that it needs to handle multiple and dynamic patterns now. The second duty has been achieved by PatternStream.process() method, but now it needs to be achieved by inner implementations.
   >

建议添加 OperatorCoordinators

1. 这个算子的提出始于Flip-27，并且机制已经被实现的很好了，直接继承使用即可。(查看具体提出，其实是在自定义数据源用的较多)

定时调度部分

1. 通过getTimeStamp()来进行时间的调度
2. 当新规则到达，首先会被缓存到operator中。当watermark推进到了，应该进行调度的时间与watermark时间对比，并相应更新模式

容错：在检查点，则将模式和部分匹配结果存储在状态后端。

不足：该设计只保证最终一致性

**使用协调器而不是广播流的原因**

> Unlike input data that is flowing in as a stream, updates are actually control messages that, instead of passing in as datastream, should be sent out by a control center like JobManager. OperatorCoordinators meet this semantics and thus is a more proper choice.

下面详细说明每一个部分

### 协调器部分

如果要创建一个协调器，首先考察Coordiantor接口的定义。

> A coordinator for runtime operators. The OperatorCoordinator runs on the master, associated with the job vertex of the operator. It communicates with operators via sending operator events.

意味着一个协调器通常运行在JobManager，用于和算子之间通过event通信

> All coordinator methods are called by the Job Manager's main thread (mailbox thread). That means that these methods must not, under any circumstances, perform blocking operations (like I/ O or waiting on locks or futures). That would run a high risk of bringing down the entire JobManager.
> Coordinators that involve more complex operations should hence spawn threads to handle the I/ O work. The methods on the OperatorCoordinator. Context are safe to be called from another thread than the thread that calls the Coordinator's methods.

这段话意味着，我们如果要在coordinator上实现I/O,则有必要单独开线程，防止阻塞Job manager。具体来说，我们自己实现的协调器Context实例，内部包含了两个线程池。如果调用 context.sendToOperator，则会将事件委托给协调器线程执行。**用于确保事件是在协调器线程执行，而不是其他线程**

```java
/**
     * 协调器线程中发送事件给指定的subTask
     * @param subtaskId subtask的ID,比如1
     * @param event 比如RuleBindingEvent(bindings=[RuleBinding(id=1),RuleBinding(id=2)])
     */
    public void sendEventToOperator(int subtaskId, OperatorEvent event) {
        callInCoordinatorThread(
                () -> {
                    final OperatorCoordinator.SubtaskGateway gateway =
                            subtaskGateways.get(subtaskId);
                    if (gateway == null) {
                        LOG.warn(
                                String.format(
                                        "Subtask %d is not ready yet to receive events.",
                                        subtaskId));
                    } else {
                        //将RuleBindingEvent(bindings=[RuleBinding(id=1),RuleBinding(id=2)])发送给指定的subTask
                        gateway.sendEvent(event);
                    }
                    return null;
                },
                String.format("Failed to send event %s to subtask %d", event, subtaskId));
    }
```

协调器接口还有3个内部接口，分别是Context, SubtaskGateway，以及Provider。

1. 首先是Context

   > The context gives the OperatorCoordinator access to contextual information and provides a gateway to interact with other components, such as sending operator events.
   >
2. 然后是SubtaskGateway

   > The SubtaskGateway is the way to interact with a specific parallel instance of the Operator (an Operator subtask), like sending events to the operator.
   >
3. 最后是Provider

   > 这个provider可以根据所传递的Context，来创建一个Coordiantor。例如可以被Job manager调用，当调度器以及执行图被创建的时候，然后就可以在这个时候初始化Coordinator。这里我发现coordiantor内部内置了一个
   >
   > ```
   > SourceCoordinatorProvider
   > ```
   >
   > 则**可以参考该类来进行实现**。同样可以参考阿里的实现。
   >

**协调器有一个会基于检查点进行恢复的子类**

#### Provider

首先考虑实现

```
RuleDistributorCoordinatorProvider
```

这个类是用于提供协调器的类，他继承的就是上面的子类型，也就是可以基于检查点进行恢复。对于该类，进行初始化之后，可以向它来传递context，来获得对应的协调器实例。

这个类实现的方法在idea中找不到跳转，但是打断点可以发现，实际上在JobGraph的创建，也就是从StreamGraph转换为JobGraph的时候，会被强转从而被使用。在JobGraph创建的时候，会维护一个provider链表。

#### Coordinator

然后考虑

```java
public RuleDistributorCoordinator(String operatorName,
String ruleUpdatedQueueId,
RuleDiscovererFactory discovererFactory,
CoordinatorContext coordinatorContext
) {
this.operatorName = operatorName;
this.discovererFactory = discovererFactory;
this.context = coordinatorContext;
this.ruleUpdatedQueueId = ruleUpdatedQueueId;
updatedEventQueue = getRuleUpdatedEventQueue();
}
```

这个类是个核心类，它实现了

```
OperatorCoordinator, RuleManager
```

两个接口。

其中RuleManager接口只有一个方法，就是当规则改变的时候需要做出的动作。这也就是 FLIP-200中提到的 PatternProcessorManger接口。它和另一个RuleDiscover接口是相互配合的。RuleDiscover接口也就只有一个方法，就是发现规则的变动。

其中的方法如下：

1. start方法：开启一个协调器，在实现中，这里会给RuleDiscover类型的成员变量进行初始化与赋值，也就是RuleDiscover接口的实现类。然后将轮询DB的任务提交到Coodriantor的线程池中，它的两个线程池，大小都是1，是为了保护线程执行的顺序是串行执行。
2. handEventFromOper:这个方法不需要实现，因为这里的事件传递是单向的，从JobManager -> subTask
3. checkpointCoordiantor: 这个方法的入参是一个检查点id和字节数组类型的future，需要做的就是想线程池提交执行任务，如果输了，就用完成的结果填充future，否则必须用异常进行填充，这是所暴露出来的接口所规定的。
4. 对于接口所定义的，从checkpointListener所提供的两个方法，并不需要具体的实现，**有时间再具体研究**
5. 然后就是为了一致性的要求，需要保证3个方法的严格按照顺序执行(其中1，4是一种方法，代表新周期的开始)

   1. > executionAttemptReady(int, int, OperatorCoordinator. SubtaskGateway): Called once you can send events to the subtask execution attempt. The provided gateway is bound to that specific execution attempt. This is the start of interaction with the operator subtask attempt.
      >

      第一个方法是开始和subtask进行交互的
   2. > executionAttemptFailed(int, int, Throwable): Called for each subtask execution attempt as soon as the attempt failed or was cancelled. At this point, interaction with the subtask attempt should stop.
      >

      第二个方法是当给subtask的交互在subtask端失败的时候，需要coordinator立即停止进行交互
   3. > subtaskReset(int, long) or resetToCheckpoint(long, byte[]): Once the scheduler determined which checkpoint to restore, these methods notify the coordinator of that. The former method is called in case of a regional failure/ recovery (affecting possible a subset of subtasks), the later method in case of a global failure/ recovery. This method should be used to determine which actions to recover, because it tells you which checkpoint to fall back to. The coordinator implementation needs to recover the interactions with the relevant tasks since the checkpoint that is restored. It will be called only after
      >

      第三个方法用于重置到指定检查点的位置，实现思路如下：对于每个checkpoint所保存的数据，都用checkpointId + byte[] data来唯一标识。所以当进行重置的时候，就可以通过readObject重新读取恢复数据。这里检查点只维护一个状态，就是 RuleBindingEvent。至于discover，可以直接重新创建。要注意方法中所提醒的对于空状态的特判。
   4. > executionAttemptReady(int, int, OperatorCoordinator. SubtaskGateway): Called again, once the recovered tasks (new attempts) are ready to go. This is later than subtaskReset(int, long), because between those methods, the new attempts are scheduled and deployed.
      >

      一旦从检查点恢复，则就可以再次尝试与subtask进行交互。
6. 以及一个辅助方法 runInEventLoop

   ```
   context.runInCoordinatorThread
   ```

   把任务委托给协调器线程池进行执行
7. 以及需要实现ruleManager接口的onRuleUpdated方法
   首先，不同的Coordinator之间，可以通过 CoordinatorStore 进行资源的共享，他就是一个并发的hashMap，从而支持生产者消费者模式。然后，还可以将事件发送给对应的那个算子
   这两者的完成，都通过context来完成

#### 两种Coordiantor职责与交互

首先是`RuleDistributorCoordinator`

它通过discover，查看DB规则是否改变。如果改变，则对于每个规则(新规则，也可以是规则最新版)，将规则相关参数封装到event
此处使用的event一共有两种

1. RuleBindingEvent
   1. 定义：规则绑定事件
   2. 内容：包含一个RuleBinding列表
   3. RuleBinding：规则绑定实体。包含规则id,version，以及绑定的key。
2. RuleUpdatedEvent
   1. 定义：规则更新事件
   2. 内容：包含一个RuleUpdated列表
   3. RuleUpdated: 规则更新实体。包含规则定义时json字符串除了binding部分的所有字段。
3. 通过一个共享map进行消息通信，这是`Coordinator`的常用手段。

### 算子开发部分

算子同样按照FLIP-200中思路实现，先实现一个key-Gen oper，再实现一个cep oper。

#### key-Gen-Oper

自定义一个StreamOperator，原理就是可以继承其抽象类AbstractStreamOperator,并覆盖其中的方法。

如果需要接受上游数据，就必须实现OneInputStreamOperator或者TwoInputStreamOperator接口，主要通过输入的数量来判断。

具体步骤如下：

* 继承AbstractStreamOperator抽象类，实现OneInputStreamOperator接口
* 重写open方法，进行资源初始化，调用flink提供的定时接口，并注册定时器
* 重写initializeState/snapshotState方法,需要做缓存，则需要使用到状态
* 重写processElement方法，将数据缓存到state，到达一定量就输出(collect)
* 实现定时相关的接口，并实现其中的回调方法

方法

1. open: 首先进行各变量的初始化，初始化collector以及是否绑定参数的相关变量。然后对distributor接口进行初始化。它是一个RuleDistributor接口，只有一个方法，则可以直接使用lambda创建实例。里面同时封装了数据的发送，也就是collector逻辑。
2. processElement: 直接调用封装好的函数进行调用。这里将func视作first-class进行传递

内部类和内部接口

1. 内部接口：

   ```
   RuleDistributor
   ```

   主要用于实现first-class
2. 内部类：

   ```
   KeyBindingMap
   ```

   是对于key绑定规则的map的封装，并且支持正则功能

#### cep-Process-Oper

简单来说，是基于开源实现的`Flink`基础之上，提供了关于规则动态加载的功能。关于规则加载与序列化和反序列，阿里并没有公开完全。但是本文章所给连接中的开源项目完整的进行了实现。并且提供了基于`CustomArgsCondition`与`AviatorCondition`的两种规则，并且可以外加`jar`包，被自定义的`ClassLoader`进行加载并应用。同时一些需要写到算子`open`函数中的全局配置也可以动态加载进来。

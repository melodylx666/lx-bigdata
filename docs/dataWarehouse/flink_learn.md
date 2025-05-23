# Flink总结

> [Apache Flink Documentation | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/)
>
> [Flink流式计算在节省资源方面的简单分析 ](https://mp.weixin.qq.com/s/mN4eQklYJAy4qXK3vhWK3Q)
>
> 必须声明一点，Flink是移动数据而不是移动计算，Flink和Spark模型有本质的不同。Spark之所以说是移动计算，是因为在批处理任务执行的时候将task发到尽可能同数据近的节点，这是批处理的操作，一个stage结束才开始下一个stage，所以有机会根据输出/存储的数据位置控制task的发送位置。但是Flink的流处理是直接开启整个计算图，然后等待数据流入，所以Flink本质是移动数据。当然，这里特指其流处理模式。其实Flink也可以跑批，这时候它就会控制计算任务是一个个启动而不是全部启动，类似Spark的方式。

## Flink Core

### Flink架构

> [Flink 架构 | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/docs/concepts/flink-architecture/)

Flink架构一共分为三层，分别是`Deploy`,`Runtime`,以及`API`层，如下图所示:

![image.png](assets/FlinkStructure.png)

#### Flink Runtime

**核心组件**

Flink是一个标准的主从分布式引擎，其`Runtime`架构，核心组件包括: Client,JobManager,TaskManager.

首先是`Client`,其只是负责准备数据流，然后将其发送给`JobManager`。

然后是`JobManager`，其包括三大核心组件:

* `ResourceManager`：负责集群中资源管理和分配，比如`task-slots`。Flink针对不同的资源提供者，比如`YARN,K8s`都特化了不同的`ResuorceManager`。
* `Dispatcher`，可以为提交上来的每一个`Job`启动一个`JobMaster`,同时运行着`Flink web-ui`。
* `JobMaster`：负责单个`JobGraph`的执行。

最后是`TaskManager`，其也称为`worker`，用于执行作业流的`task`，并缓存和交换数据流。其上资源调度的最小单位是`task-slot`，一个`slot`可以执行一个`task`，当然，这个`task`可能有多个算子(算子链的情况)。

针对`TaskManager`这点，这里着重说明一下。

一个`TasnManger`是一个JVM进程，而一个`TaskSlot`是一个执行线程。这里和普通线程池不同就是其会将该进程的内存资源对每个`slot`进行平均，但是`CPU`资源并不会平均分配，依旧是一个进程内CPU共享。

一个进程内的`tasks`好处显而易见，它们共享`TCP连接，文件缓存`等资源，减少了开销。

![image.png](assets/taskManager.png)

`Flink`中的算子链，其实就是`Spark`中对应的`一个stage`中的`task`，利用并行计算将效率最大化。具体来说，一个`chain task`可以被投放到一个`backend`的线程池的一个线程执行-即`task-slot`，因为操作系统的最小执行单位是线程，所以该集群的最小执行单位、资源分配单位就是`slot`。

`Flink`允许一个`job`内的所有`tasks`共享`slot`。

> 集群所需的task slot == Job中的最大并行度

**作业提交流程**

其作业的提交流程包括11步，如下:

![image.png](assets/jobSubmit.png)

精确到每一步:

1. 编写Flink代码，将作业进行submit,首先到 Client。
2. Client会将`Job`进行切分，转换为`JobGraph`（app模式在JobManager执行这步），将其提交到集群中的`Dispatcher`。
3. 集群中的`Dispathcer`启动一个`JobMaster`，后者将任务逻辑进行解析，解析为`ExecutionGraph`，其是在物理层面具有并行度的执行图，也就是说其可以被分发到多个节点进行并行计算。同时会运行`Flink-webui`。
4. `JobMaster`向`ResuorceManager`申请所需要的计算节点资源，`ResourceManager`随后向资源提供框架，比如`YARN`申请对应资源。
5. 随后在集群中开启`TaskManager`进程，然后将资源划分，向`RM`注册`task-slot`。
6. 随后告诉`JobMaster`其可用的`slot`资源，随后可以在其上执行作业。（通过rpc）发到对应节点即可。

#### 2种部署模式

> Pre-job模式已经废弃，所以不总结

**Session模式**

提前准备一个`Flink`集群，并全局复用。用户的`Client`只管提交作业即可，其余的`JobManager,taskManager`均复用一个集群的。

> 注意这里的Job解析为`JobGraph`同样是在Client端进行

特点如下:

* 集群和作业声明周期不同
* 不用作业资源不隔离
* 作业部署速度快，这是主要目的，比如即席查询场景

**Application模式**

这里的`Job解析为JobGraph`就在`JobManager`端执行，通过它来解析并下载依赖，使得`Client`十分轻量级。

![image.png](assets/application.png)

特点如下:

* 集群和作业生命周期相同。
* 不同Job之间资源完全隔离，每个都是起一个小集群，(这是主要特点，完全隔离)
* 部署速度慢
* 资源抢占

### DataStream API

`Flink`一共有四种API

* code
  * 有状态流
  * DataStream
* 关系型
  * Table API
  * SQL API

#### 从官网例子迭代

官网的例子是逐步推进的欺诈检测例子，相比常见的`wordcount`，它更有代表性。

整体的思路，就是基于先验知识，进行异常点检测，发现异动点。

**主程序**

首先来看主程:

```java
public class FraudDetectionJob {
	public static void main(String[] args) throws Exception {
		//
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		DataStream<Transaction> transactions = env
			.addSource(new TransactionSource())
			.name("transactions");
		DataStream<Alert> alerts = transactions
				.keyBy(x -> x.getAccountId())
				.process(new FraudDetector())
				.name("fraud-detector");
		alerts
			.addSink(new AlertSink())
			.name("send-alerts");

		env.execute("Fraud Detection"); //不能忘了
	}
}
```

首先就是创建一个`env`，创建`Flink`运行环境。如果要有`web-ui`，则需要配置`rest-port`。

然后是添加数据流，这里使用`Flink`内置的`mock`数据源，其有三个字段:

* accountId
* timestamp
* amount

然后对数据流进行转换操作，首先进行`keyby`，其创建一个`keyedStream`，后续的函数会分别作用于每个流之上。其实就是要进行一次`exchange`，保证后续的每个并行`task`流入的数据是同一个`key`。同时，`keyBy`操作可以保证用户追踪流的状态上下文等细粒度控制。

然后就是对每个流(key相同)进行业务逻辑的处理，就是这里传入的自定义处理逻辑，随后再看。

最后将每个流输出的结果添加到自定义`sink`中。

计算图从一开始就全部启动，等待数据流入，这点和`Spark批处理`很不相同。

**第一版业务逻辑**

```java
public class FraudDetector extends KeyedProcessFunction<Long,Transaction,Alert>{
	private static final long serialVersionUID = 1L;
	private static final double SMALL_AMOUNT = 1.00;
	private static final double LARGE_AMOUNT = 500;
	private transient ValueState<Boolean> flagState; //维护一个流的state，其穿透context
	//每个流开一次，操作其全局变量与状态
	@Override
	public void open(Configuration parameters) throws Exception {
		//一个valueState描述符，作用类比 fd，或者handle。
		//给本流注册一个state
		ValueStateDescriptor<Boolean> flag = new ValueStateDescriptor<>("flag", Types.BOOLEAN);
		flagState = this.getRuntimeContext().getState(flag);
	}

	@Override
	public void processElement(Transaction value, Context ctx, Collector<Alert> out) throws Exception {
		Boolean lastTransWasSmall = flagState.value();
		if(lastTransWasSmall != null){
			if(value.getAmount() > LARGE_AMOUNT){
				Alert alert = new Alert();
				alert.setId(value.getAccountId());
				out.collect(alert);
			}
			flagState.clear();
		}
		if(value.getAmount() < SMALL_AMOUNT){
			flagState.update(true);
		}
	}
}
```

这是一个作用于`keyedStream`上的函数，所以可以维护`state`。这里使用一个`ValueState`来记录上一个的金额是否满足了条件。

所以它不断匹配第一个满足条件的小金额，然后在之后将状态机转换，开始匹配下一个大金额。

> 看出来其用CEP写应该会很简单，后续会专门使用CEP写一次。

上述逻辑已经比较完备，唯一缺点就是没有时效性保证。也就是说，一旦匹配到第一个小金额，状态机就不会回退，会一直等到下一个大金额，哪怕时间间隔是一年。所以会有下述的第二版。

**第二版-引入时间**

依旧是因为`keyedStream`，所以可以有定义逻辑的操作，也就是在特殊时间触发回调函数。显然，该业务逻辑的定时回调函数应该是清空之前匹配的小数据，从头开始，因为过期了。

```java
public class FraudDetector extends KeyedProcessFunction<Long,Transaction,Alert>{
	private static final long serialVersionUID = 1L;
	private static final double SMALL_AMOUNT = 1.00;
	private static final double LARGE_AMOUNT = 500;
	private transient ValueState<Boolean> flagState; //维护一个流的state，其穿透context
	private transient ValueState<Long> timerState;
	//每个流开一次，操作其全局变量与状态
	@Override
	public void open(Configuration parameters) throws Exception {
		//一个valueState描述符，作用类比 fd
		//给本流注册一个state
		ValueStateDescriptor<Boolean> flagD = new ValueStateDescriptor<>("flag", Types.BOOLEAN);
		flagState = this.getRuntimeContext().getState(flagD);

		//同样操作，注册一个timer并拿到其fd
		ValueStateDescriptor<Long> timerD = new ValueStateDescriptor<>("timer-state", Types.LONG);
		timerState = this.getRuntimeContext().getState(timerD);
	}

	@Override
	public void processElement(Transaction value, Context ctx, Collector<Alert> out) throws Exception {
		Boolean lastTransWasSmall = flagState.value();
		if(lastTransWasSmall != null){
			if(value.getAmount() > LARGE_AMOUNT){
				Alert alert = new Alert();
				alert.setId(value.getAccountId());
				out.collect(alert);
			}
			flagState.clear();
		}
		if(value.getAmount() < SMALL_AMOUNT){
			flagState.update(true);

			//注册一个timer，到时间就触发onTimer回调。
			//这里之所以额外维护一个state作为timer，是因为定时任务是事件匹配所驱动的。
			long timer = ctx.timerService().currentProcessingTime() + 1000 * 60;
			ctx.timerService().registerProcessingTimeTimer(timer);
		}
	}

	@Override
	public void onTimer(long timestamp, KeyedProcessFunction<Long, Transaction, Alert>.OnTimerContext ctx, Collector<Alert> out) throws Exception {
		timerState.clear();
		flagState.clear();
	}

}
```

#### Source

**简单数据源**

对于已有类型的简单数据源，使用`fromData`即可.

```java
		ArrayList<Integer> integers = new ArrayList<>(10);
		for (int i = 0; i < 10; i++) {
			integers.add(i);
		}
		DataStreamSource<Integer> integerDataStreamSource = env.fromData(integers);
```

**自定义数据源**

```java
    public static class Order {
        public int id;
        public int num;
        public String name;
    }

    //自定义数据源
    private static class UserDefinedSource implements ParallelSourceFunction<Tuple3<Integer, Long, Order>> {
        private volatile boolean running = true;

        @Override
        public void run(SourceContext<Tuple3<Integer, Long, Order>> ctx) throws Exception {
            while (running) {
                int id = RandomUtils.nextInt(0, 100);
                int num = RandomUtils.nextInt(0, 999);
                String name = RandomStringUtils.randomAlphabetic(5);
                Order order = new Order();
                order.id = id;
                order.num = num;
                order.name = name;
                ctx.collect(Tuple3.of(id, System.currentTimeMillis(), order));
                Thread.sleep(3000);
            }
        }

        @Override
        public void cancel() {
            running = false;
        }
    }
```

只需要实现各种`SourceFunction`接口的两个方法，分别是输出数据和流的关闭逻辑也就是`run和cancel函数`。

**Kafka数据源**

这里需要额外添加依赖，可以指定主题进行消费。

```java
public class FlinkKafkaUtils {

    public static Properties getProducerProperties(String brokers) {
        Properties properties = getCommonProperties();
        properties.setProperty("bootstrap.servers", brokers);
        properties.setProperty("metadata.broker.list", brokers);
        properties.setProperty("zookeeper.connect", Constants.HOST_NAME+":"+Constants.ZOOKEEPER_PORT);
        return properties;
    }

    public static Properties getCommonProperties() {
        Properties properties = new Properties();
        properties.setProperty("linger.ms", "100");
        properties.setProperty("retries", "100");
        properties.setProperty("retry.backoff.ms", "200");
        properties.setProperty("buffer.memory", "524288");
        properties.setProperty("batch.size", "100");
        properties.setProperty("max.request.size", "524288");
        properties.setProperty("compression.type", "snappy");
        properties.setProperty("request.timeout.ms", "180000");
        properties.setProperty("max.block.ms", "180000");
        return properties;
    }

    public static FlinkKafkaConsumer<String> getKafkaEventSource(String topic){
        Properties props = getProducerProperties(Constants.BROKERS);
        props.setProperty("auto.offset.reset", "latest");
        //指定Topic
        FlinkKafkaConsumer<String> source = new FlinkKafkaConsumer<>(topic, new SimpleStringSchema(), props);
        return source;
    }

    public static FlinkKafkaConsumer<String> getKafkaEventSource(){
        Properties props = getProducerProperties(Constants.BROKERS);
        props.setProperty("auto.offset.reset", "latest");
        //指定Topic
//        FlinkKafkaConsumer<String> source = new FlinkKafkaConsumer<>(Constants.CLIENT_LOG, new SimpleStringSchema(), props);
        FlinkKafkaConsumer<String> source = new FlinkKafkaConsumer<>(Constants.STOCK_LOG, new SimpleStringSchema(), props);
        return source;
    }
}
```

#### Operator-概述

> [概览 | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/docs/dev/datastream/operators/overview/)

**数据流转换**

* map
* flatmap
* filter
* keyBy，`数组，以及没有重写hashCode方法的POJO类`不能作为key
* reduce，将局部流数据进行滚动聚合
* window：对键控流对每个流上进行开窗，开窗就是根据某些特征对流数据进行分组(每组数据是一个`iterator`)，流转换为`WindowedStream`
* windowAll：对普通流的所有数据进行开窗,流转换为`AllWindowedStream`
* window apply：对流的每个开窗使用`function`，此处需要是`[All]WindowFunction`
* window reduce：对流的每个开窗做`reduce`并返回结果
* union：同类型多流合并，只保证单流数据相对位置不变
* connect: 不同类型多流合并
* window join：根据指定的`key`和`窗口`对数据进行`join`。是`type JOIN = (DataStream,DataStream) => DataStream `
* interval join : 并不和开窗耦合，只是将两个键控流中`某个key`相等并且在指定范围内的数据进行`Join`
* ......

**物理分区分布**

* partitionCustom：自定义分区，可传入分区器
* shuffle：随机分区
* broadcast：将该流元素广播到每个分区

**资源隔离**

多个算子在特定情况可以进行chain,然后可以在同一个线程中执行（slot）。默认开启算子链模式。

那么如何禁用呢?

只需要后跟一个`disableChaining`即可，即可将算子链打断。

如何控制算子执行的`slot`呢?

可以通过`slotSharingGroup("name")`来将某些算子放入同一个`slot`中去执行。

#### 从《流式系统》看算子

> 结合《流式系统》书籍进行总结

数据的处理，通常可以通过`what,where,when,how`模式进行总结。

* what：计算输出什么结果
* where：以事件时间计，结果在何处被计算
* when：以处理时间计，结果在何时被物化
* how：结果的改进/修正如何相互关联。是丢弃，累积，还是替换?

**what**

这个问题通过具体使用的开窗函数进行回答，也就是对每个窗口的处理逻辑。

开窗函数有如下几种，控制程度依次增加:

* reduce：输入输出类型一直，从左到右卷。
* aggregate:输入输出类型可以不同，最终基于累加器计算出输出结果，粒度更细。
* process:前两者都是边来边更新，这个函数是到触发时间时拿到数据的`iterator`以及上下文，可以进行任意逻辑的计算。

可以看到，其实`process`是在表达能力和性能之间做了权衡，暴露给用户更强的操控力。

> `ProcessWindowFunction`还可以让用户拿到当前流甚至所有流的当前窗口的state，进行更强的操作。

**where**

开窗，回答的就是where。

开窗是处理无界流的关键，因为它可以将满足一定特征的数据装入大小有限的桶中，再对每个桶进行处理。这就是上面的概述所说的开窗就是依据某个特征对数据流进行分组。

开窗可以作用于普通流，以及键控流，通常都是使用键控流再开窗，这样的流具有并行处理窗口函数的能力。

窗口的生命周期: 在第一个属于该窗口的元素到达的时候创建，然后在时间超过窗口的截止时间戳 + 用户定义的`allowed-lateness`之后被完全删除。当然，这里清空(`purge`)的的是窗口的数据元素，并不是窗口元数据，也就说，窗口也可能会进入新的数据。

```java
stream
       .keyBy(...)               <-  仅 keyed 窗口需要
       .window(...)              <-  必填项："assigner"
      [.trigger(...)]            <-  可选项："trigger" (省略则使用默认 trigger)
      [.evictor(...)]            <-  可选项："evictor" (省略则不使用 evictor)
      [.allowedLateness(...)]    <-  可选项："lateness" (省略则为 0)
      [.sideOutputLateData(...)] <-  可选项："output tag" (省略则不对迟到数据使用 side output)
       .reduce/aggregate/apply()      <-  必填项："function"
      [.getSideOutput(...)]      <-  可选项："output tag"
```

如何按照时间边界来划分多条数据流呢?

通常有三种方式:

* 滚动窗口/固定窗口(Tumbing Windows)
* 滑动窗口(Sliding Windows)
* 会话窗口(Session Windows)

![image.png](assets/windowType.png)

在Flink中，可以通过`Window Assigners`进行指定，也就是指定`key`被划分到什么位置.

比如还是官网的例子，使用`Transcation数据源`，数据形式如下:

```plaintext
7> Transaction{accountId=1, timestamp=1546272000000, amount=188.23}
8> Transaction{accountId=2, timestamp=1546272360000, amount=374.79}
9> Transaction{accountId=3, timestamp=1546272720000, amount=112.15}
```

则如果要对其进行固定开窗(滚动开窗),则可以如下方式:

```java
        stream
                .keyBy(Transaction::getAccountId)
                .window(TumblingEventTimeWindows.of(Time.seconds(5)))
```

如果是滑动窗口，则是`SlidingEventTimeWindows.ofxxx`。

会话窗口的话，可以有不同策略，分别是固定间隔，动态间隔等。

**when**

触发器(trigger)就是用来实现每个窗口的结果，以处理时间计(或者其他标准)，该何时物化输出的。

主要的触发器有两类：

* 完整性触发器:到达某个阈值一次性输出最终结果。语义类似于批处理。
* 重复更新触发器:对每个数据进来，所计算的中间结果，都输出下来，这个最常见。

前者到某个阈值之后直接输出最终结果，而后者会输出每个数据到来之后的结果。

如果你熟悉`Scala`，则完整性触发器效果就是`reduce`，而重复更新触发器效果就是`scanLeft`。

而Flink中，分配窗口的时候会分配默认触发器，这个默认触发器是`完整性触发器`。分别有`ProcessingTime`语义和`eventTime`语义。

如何指定`重复更新触发器`呢?

Flink中有一个`CountTrigger`，可以基于数量来触发窗口计算。

> Flink中只有使用`event-time`的时候才有watermark(其实是启发式水位)，其余部分都是固定的，也就是窗口在结束的时间线统一触发输出。

#### watermark

`Flink`不同于`Spark`，它的计算图一旦构成，就会全部启动，所以会有人说`Flink`是移动数据而不是移动计算，因为它整体来看是数据从固定流图中进行流动。而不是和Spark一样整体上来看是串行启动。

而数据的流动，靠的就是`watermark`。

**作用**

关于流式数据处理的`when`部分，上面已经介绍了是通过`trigger`来进行触发。

那么`trigger`到底根据什么可以判定数据流的某个窗口不会再有数据进来了(从而开始触发计算)呢？

答案就是`watermark`，它给`trigger`一个断言(assert），让其可以放心进行输出最终结果/关闭窗口。它也是推动无界流处理进度的核心概念。

具体而言，水位告诉`trigger`，当前系统的数据不会再有比时间`T`更晚的数据进来。

**性质**

通常，我们都是利用水位所提供的断言性质，来实现两方面的需求:

* 进行窗口的最终计算/触发计算，以及关闭窗口
* 对数据进行排查，看哪部分数据推进的慢/有问题

**分类**

根据`watermark`所提供的断言到底准不准，可以分为`完美水位和启发式水位`。这是一个权衡的艺术。

而水位到底准不准，主要和一个因素有关，就是`timestamp`的语义，到底是`event-time`,`injection-time`,还是`process-time`。

> `injection-time`本身又是一个权衡的产物。

因为真实的数据的产生来自外部系统，`Flink`不可能掌握外部系统的所有信息(由于外部系统本身，网络拥堵等原因)。所以我们得出一个结论，就是一般情况下，使用`event-time`语义不可能达成完美水位。

如果要完美水位，则有两种方法:

* 使用`injection-time/process-time`语义，让`Flink`掌控全局数据推进。
* 使用静态的，按`event-time`排序的数据集。

启发式水位的创建(针对`event-time语义`)，其效果的差距会很大，因为终究会有迟到的数据。要建立一个好的启发式水位，则有如下方式:

* 使用动态的，按照`event-time`排序的动态数据集(方便对数据整体进行估算，来优化启发式水位设置)，说的就是`Kafka`。

> 再次强调，Flink的watermark特指启发式水位，针对`event-time`语义。

Flink中的内置的基于事件时间的水位生成器策略有两种。

* forMonoTimeStamps：适用于数据源数据按照事件时间有序存储,比如Kafka
* forBoundOutofOrderness：适用于考虑延迟(或者数据源乱序存储)的情况，将数据的水位推进延迟一段时间。

**多stage传递**

对于不同stage(经过exchange)而言(Stage内部是并行逻辑，可以统一视作一个大算子链)

* 其输入的水位是`上游各数据源Stage的event-time`最小值。
* 其输出的水位是以下三者的最小值
  * 输入水位
  * 状态组件水位
  * 输出缓冲区水位

```java
public class watermarkDemo {
    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        conf.setString("rest.port", "9091");
        conf.setBoolean("web.ui.enable", true);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(conf);
        //启发式水位的标准实现之一:假定数据源数据按时间戳递增存储(Kafka就完美符合)
        WatermarkStrategy<Transaction> strategy01 = WatermarkStrategy.<Transaction>forMonotonousTimestamps()
                .withTimestampAssigner((element, Timestamp) -> element.getTimestamp()); //如何从元数据中获取时间戳
        //如果数据源存储还时间乱序的话,或者网络发送乱序，则需要使用forBoundedOutOfOrderness实现
        WatermarkStrategy<Transaction> strategy02 = WatermarkStrategy.<Transaction>forBoundedOutOfOrderness(Duration.ofSeconds(1))
                .withTimestampAssigner((element, Timestamp) -> element.getTimestamp());
        DataStream<Transaction> stream = env.addSource(new TransactionSource())
                .assignTimestampsAndWatermarks(strategy01); //通常水位直接在源端设置
        stream
                .keyBy(Transaction::getAccountId)
                .window(TumblingEventTimeWindows.of(Duration.ofSeconds(10)))
                .aggregate(new AggregateFunction<Transaction, Tuple2<Double, Integer>, Double>() {
                    @Override
                    public Tuple2<Double, Integer> createAccumulator() {
                        return new Tuple2<>(0.0, 0);
                    }

                    @Override
                    public Tuple2<Double, Integer> add(Transaction value, Tuple2<Double, Integer> accumulator) {
                        return new Tuple2<>(accumulator.f0 + value.getAmount(), accumulator.f1 + 1);
                    }

                    @Override
                    public Double getResult(Tuple2<Double, Integer> accumulator) {
                        var total = accumulator.f0;
                        var count = accumulator.f1;
                        if (count == 0) return 0.0;
                        return total / count;
                    }

                    @Override
                    public Tuple2<Double, Integer> merge(Tuple2<Double, Integer> a, Tuple2<Double, Integer> b) {
                        return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
                    }
                })
                .filter(x -> x > 1000)
                .print();

        env.execute();
    }
}
```

#### Sink

也就是处理之后的数据都发送/存储到哪里。

* 打印到控制台，则strean.print()
* 发送到`Kafka`，则先创建一个SinkFunction，它是一个`Kafka生产者`，然后stream.addSink即可。
* 自定义Sink,则需要实现`SinkFunction`接口的`invoke`方法。

```java
    public static class MySink implements SinkFunction<Integer>{
        @Override
        public void invoke(Integer value, Context context) throws Exception {
            System.out.println(value);
        }
    }
```

#### 算子数据传输策略

一共有8种数据传输策略。

* Forward: 上游一个subtask只流向下游一个subtask，Flink在能`Forward`的情况下都会使用它
* Rebalance：上游的一个subtask数据均匀发送到下游每个subtask，一般用于数据倾斜处理(打散)。也就是说，他一般也是用于多线并行的情况，只不过该变了数据分布而已，计算逻辑照常。它可以处理上下游并行度相同的问题，也可以处理上下游并行度不同的情况(默认就是rebalance)。
* Shuffle：只是和`rebalance`的分发逻辑不同，一个是`round-robin`，一个是`random`.其余完全一致。
* KeyGroup：使用`keyby`之后，上下游传输逻辑就是`keyGroup`，一般通过`Hash`来决定发送到下游哪个`subtask`，一定会保证同一个key的全部在一个`subtask`中。要注意它和`rebalance以及shuffle`完全不同，这里要和`spark`区分开。
* Rescale：将上游`subtask`的数据均匀发送给下游的`subtask`，不过并不是全部的，而是下游局部的`subtask`。
* Broadcast：上游一个`subtask`数据会全部发送给所有的下游`subtask`。
* Global: 将上游所有`subtask`数据汇聚到下游一个`subtask`中。

#### RichFunction

![image.png](assets/RichFunction.png)

文档中的定义如下:

> An base interface for all rich user-defined functions. This class defines methods for the life cycle of the functions, as well as methods to access the context in which the functions are executed.

也就是说，通过`RichFunction`，我们可以控制这个函数的整个生命周期的行为。

其中的`open/close`，是每个`subtask`初始化、结束时候会执行一次。

而`getRuntieContext`，就是本`subtask`的执行上下文。

> 注意这里为什么是subtask，因为`Flink`是流处理，会直接开多并行度的全局计算图。而同一个stage(比如Map[x])中会有多个并行的subtask，这里每个`subtask`都要初始化一次。

#### 双流Join

Flink进行双流Join的算子有两个:

* window join：根据指定的 key 和窗口 join 两个数据流。
* interval join：根据 key 相等并且满足指定的时间范围内（`e1.timestamp + lowerBound <= e2.timestamp <= e1.timestamp + upperBound`）的条件将分别属于两个 keyed stream 的元素 e1 和 e2 Join 在一起。

其中,

* `type window-join = （DataStream,DataStream） => DataStream`
* `type interval-join = （KeyedStream,KeyedStream） => DataStream`

**window join**

如下例子:

对两个流进行Join，开窗为处理时间的固定窗口，长度为5。它会将两个流中相同窗口内的指定键一直的数据做笛卡尔积(一一处理)，然后输出。

```java
public class DoubleJoinDemo {
    public static void main(String[] args) {
        Configuration conf = new Configuration();
        conf.setString("rest.port", "9091");
        conf.setBoolean("web.ui.enable", true);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(conf);
        env.setParallelism(2);
        DataStreamSource<Transaction> stream01 = env.addSource(new TransactionSource());
        DataStreamSource<Transaction> stream02 = env.addSource(new TransactionSource());
        stream01
                .join(stream02)
                .where(Transaction::getAccountId).equalTo(Transaction::getAccountId)
                .window(TumblingProcessingTimeWindows.of(Duration.ofSeconds(5)))
                .apply(new JoinFunction<Transaction, Transaction, String>(){

                    @Override
                    public String join(Transaction first, Transaction second) throws Exception {
                        return first.getAmount() + "" + second.getAmount();
                    }
                })
                .filter(x -> x.startsWith("1"))
                .print();
        try {
            env.execute();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

![image.png](assets/window-join.png)

**interval join**

这种join和窗口没有关系，或者说它是两边窗口无限大的情况。因为通常某些操作必然有先后顺序，不会划分到同一窗口。

interval join只支持`event-time`。ok，那么必然要指定`watermarkStateagy`。并且可以指定窗口左右区间包含不包含。

```java
public class IntervalJoinDemo {
    public static void main(String[] args) {
        Configuration conf = new Configuration();
        conf.setString("rest.port", "9091");
        conf.setBoolean("web.ui.enable", true);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(conf);
        env.setParallelism(2);

        WatermarkStrategy<Transaction> strategy = WatermarkStrategy.<Transaction>forMonotonousTimestamps()
                .withTimestampAssigner((element, timestamp) -> element.getTimestamp());

       DataStream<Transaction> stream01 = env.addSource(new TransactionSource()).assignTimestampsAndWatermarks(strategy);
        DataStream<Transaction> stream02 = env.addSource(new TransactionSource()).assignTimestampsAndWatermarks(strategy);
        KeyedStream<Transaction, Long> keyedStream01 = stream01.keyBy(Transaction::getAccountId);
        KeyedStream<Transaction, Long> keyedStream02 = stream02.keyBy(Transaction::getAccountId);

        keyedStream01
                .intervalJoin(keyedStream02)
                .between(Duration.ofMillis(-500), Duration.ofMillis(500))
                .lowerBoundExclusive()
                .upperBoundExclusive()
                .process(new ProcessJoinFunction<Transaction, Transaction, Double>() {
                    @Override
                    public void processElement(Transaction left, Transaction right, Context ctx, Collector<Double> out) throws Exception {
                        out.collect(left.getAmount() + right.getAmount());
                    }
                })
                .filter(x -> x > 1000)
                .print();
        try {
            env.execute();
        } catch (Exception e){
            e.printStackTrace();
        }

    }
}
```

![image.png](assets/interval-join.png)

#### 数据乱序

> 数据乱序是导致完美水位无法产生的根本原因。

**产生原因**

* 数据源本身乱序存储
* 数据源的产生就是乱序，比如用户突然开启飞行模式
* 共享资源限制，比如网络拥塞

**可能的解决方案**

* 让水位延迟推进: forboundedOutOfOrderness
* 水位正常推进，但是窗口延迟关闭，仍然可以重新触发计算: allowLateness
* 旁路输出

注意几者可以结合提高效率。

> 注意allLateness在Flink中的实现是重新输出新结果，并不会覆盖旧结果。它会将原有数据存储在subtask本地状态中，等待结合输出新的结果。显然，会导致状态存储过大的问题。
>
> 也就是说，同一个窗口可能会送给下游多个结果，需要下游进行取舍/更新结果。

**排查思路**

1. 数据处理前，对数据乱序程度进行排查，看看乱序的限度到底有多大(不断调整延迟让数据丢失最少，就可以看出最大延迟)
2. 数据处理时，及时监控乱序程度以及水位推进情况。使用`Flink-metric`体系进行监控
3. 数据处理后，根据结果及时调整前面设置的参数，保障下一次产出。

### 状态

> 这里当前基于的版本是`Flink1.20`，而`Flink2.0`的版本引入了存算分离和异步状态API，在后面再分析。

#### 实现思路

主要有三点:

* 状态本地持久化
* 精确一次的一致性快照保证异常容错
* 统一API接口

**状态本地持久化**

`Flink`的状态数据，存储在`subtask`的本地机器的内存与磁盘上。

这样可以在对状态进行更新、访问的时候使得延迟达到最低。

> 但是状态本地化的同时带来问题，如果计算节点宕机，则状态存储会一并丢失-容错很难做。
>
> 状态集中管理的关系型数据库的容错管理相比之下就方便一些。

那怎么解决呢？

Flink采用了快照机制，定时将状态快照，统一存储到远端，比如HDFS上。这样节点宕机之后的状态恢复压力减小很多。

**精确一次的一致性快照**

Flink实现了一个名为`checkpoint`的分布式轻量级异步快照，保证了精确一次的数据处理，以及一致性状态。其实主要存储的就是`state`。

通过它实现了处理数据逻辑，和异常容错机制的解耦，用户只需要在使用状态接口的时候开启`checkpoint`即可。

当有状态算子并行度发生变化怎么办？如何将磁盘中存储的数据重新分配？

Flink将状态进行了分类:

* 算子状态:可以平均分割
* 键值状态：自动根据键值和最大并行度进行分配

当任务中增加、减少了算子，该怎么办？

Flink提出`savepoint`机制，也就是用户手动触发的`checkpoint`。

#### 两种状态

Flink状态可以分为两类，分别是算子状态，和键值状态。以及一种特化的算子状态-广播状态

**算子状态**

> 最常见的场景就是数据源连接

因为状态本地化的原因，Flink中每个subtask只能访问本地的状态数据。所以一个穿透当前`subtask`的状态就是算子状态。

Flink内置的KafkaConsumer就使用了算子状态，通常情况下，其对每个`partition`开一个`Source[x]`，也就是一个subtask,然后在该subtask内维护消费的`offset`。

> 所以Flink和Kafka的结合属于外部维护offset，在我的kafka文档中分析过这种情况

具体API:

* ListState
* UnionListState

**键值状态**

键值状态针对键控流。

因为虽然是进行了`keyBy`，但是一个subtask并不是只会流一种key(其实`Spark`也一样，总不能并行度上亿吧)。那么如何给上层用户提供单个key处理的假象呢？

考虑Java并发编程中的`theadLocal`，它就是一个全局的存储在堆上的Map，维护每个线程对应的副本值。

到了这里类似，在`subtask`中维护一个Map，给每个key对应的state都存储上。

也就说，这个状态值粒度被缩小到了`一个subtask的一种key`，所以称为键值状态。

键值状态更常见:

* ValueState
* MapState
* ListState
* ReducingState
* AggregatingState

**广播状态**

广播状态作用粒度同样是一个`subtask`.

其通常用在规则日志流中。也就是说，获取最新的一条规则日志流，然后将当前正在处理的流和其进行关联(connect)

然后就可以将当前规则广播到所有处理(process)算子当中。

> 需要开自定义算子

#### 精确一次性与快照

> 首先是处理系统内部精确一次性，然后是端到端精确一致性。

**单机系统**

首先看单机系统。

一个单机系统的数据处理，可以分为数据读取，数据处理，状态存储，数据输出。

因为数据读取到内存-数据处理-状态存储 这个过程并不是原子性的，所以必然会有问题。（Kafka的消费问题也是这个原因）

那发生单机故障在恢复的时候，会有什么后果呢？

* 至多一次
* 至少一次
* 精确一次

首先是至多一次. 它只依靠状态持久化。会丢数据，也就是数据读取进来但是没有处理，但是数据源没有任何标记，认为其已经处理完毕。

然后是至少一次，它依靠状态持久化 + 数据源`offset`。这样可以做到数据重放，起码不会丢数据。但是不能判定数据到底有没有处理过，所以可能会重复，也就是至少一次。

最终是精确一次。它依靠状态-对应`offset`持久化 。也就是说，把`offset`的管理和状态管理统一起来。这样每次恢复，都知道当前已经有的状态，以及该从哪里重放。需要注意，这里的持久化需要和数据处理操作，也就是状态修改是互斥的，不能并行执行。

则单机系统的精确一次性实现流程如下:

* 数据源端
  * 有重放的数据源，比如Kafka
* 数据处理端
  * 有远端持久化存储:如比HDFS。
  * 一致性快照由单机应用自身定期触发，同时持久化状态和对应的数据源偏移量，并和状态修改保持互斥。
  * 宕机恢复的时候先读取持久化状态，再对数据源从对应偏移量开始进行重放。

如此，可以完美实现单机的精确一次性。

**分布式系统**

分布式系统如何实现内部的精确一次性处理呢？

对于分布式系统，比如Flink来说，只需要看逻辑数据流图即可。因为执行层面的每个算子内的`subtask`只有数据的区别，其他完全一致。

所以整体的分布式保证可以从三部分看：

* 数据源(Source)
* 转换过程
* 数据汇

数据源(Source)，可以直接照搬单机的精确一次性模式。其实数据源放在外部数据系统，比如`Kafka`来说，就是消费者。当然可以采用上面的例子。

那么中间的Transform算子可不可以照搬呢？

当然不可以。因为网络信道本质是不可回溯的。

假设现在Transform算子也可以记录上游的`offset`，但是还是会出问题。因为系统内部，上游无法确定发送给下游的数据是否为收到，也就是处于`unknow`状态。那么上游只能认为已经发送，而下游并没有收到。这样下游记录的上游的`offset`和上游记录自身的`offset`会存在不一致，导致数据丢失。

并且无法重发-因为上游无法从下游/很难从下游获取到底哪个数据丢失了。

一个可行的解决办法就是，下游算子在停止处理数据并将状态持久化的时候，将网络信道的数据全部接受，一并持久化，防止它们丢失在信道中。

> barrier的作用和watermark类似

那么下游算子到底等待多长时间呢？当`Source`算子开始进行状态持久化的时候，也就是其认为数据被处理完毕之后，向下游发送一条`barrier`的特殊数据作为信号，则下游可以认为信道清空了。

上述操作，就可以将上下游的`offset`进行对齐。仔细思考，下游其实就可以不用存储`offset`了，因为状态连带下次恢复要计算的数据全部已经持久化了。

最后是Sink。Sink算子上游是算子，等同于Transform算子。

**优化**

上面是将状态和后续到达数据全部持久化，可不可以直接接着计算呢？

当然可以，只要将计算延长至`barrier`数据到达之前即可。

**最终的Flink checkpoint机制**

* 触发：由`JobMaster`内部的协调器统一发送命令，给最上游的`Source`算子的所有`subtask`开始进行一致性快照。
* 状态存储: `Source`算子存储`state + 对应offset`，而其余算子存储`state`即可
* 流程:
  * `Source`算子在收到协调器发送的`event`之后，停止发送数据，同时将`barrier`发送/广播给下游的subtask，开始进行持久化。
  * 下游算子正常处理数据，直到`barrier`，停止处理并继续向下游发送`barrier`，同时进行持久化。
  * 最终协调器收到所有subtask发送的`event`，则认为checkpoint结束。

**端到端精确一次性**

现在系统内部可以实现精确一次性，如何端到端精确一次性呢？

* 数据源:2PC，比如Flink内置的kafkaConnector
* 数据汇：幂等写入,比如Kafka，Redis

#### 状态后端

除了远端存储，`subtask`本地的状态如何存储呢？

`Flink`内置两种状态后端:

* HashMap - 内存
* RockDB - 磁盘，类似leveldb,都是本地LSM KV存储引擎

> 从2.0开始还有新的状态后端

```java
        Configuration conf = new Configuration();
        conf.set(StateBackendOptions.STATE_BACKEND,"hashmap");
//        conf.set(StateBackendOptions.STATE_BACKEND,"rocksdb");
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(conf);
```

配置如上。默认是`hashmap`。

#### 有状态流与事件驱动

> [事件驱动应用 | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/docs/learn-flink/event_driven/)

其实最开始的官网欺诈检测列子，使用的就是有状态流API。

它不同于窗口函数，可以提供任意粒度的时间控制，比如自定时过期时间来重新改变状态机进行时间匹配。

```java
public class FraudDetector extends KeyedProcessFunction<Long,Transaction,Alert>{
	private static final long serialVersionUID = 1L;
	private static final double SMALL_AMOUNT = 1.00;
	private static final double LARGE_AMOUNT = 500;
	private transient ValueState<Boolean> flagState; //维护一个流的state，其穿透context
	private transient ValueState<Long> timerState;
	//每个流开一次，操作其全局变量与状态
	@Override
	public void open(Configuration parameters) throws Exception {
		//一个valueState描述符，作用类比 fd
		//给本流注册一个state
		ValueStateDescriptor<Boolean> flagD = new ValueStateDescriptor<>("flag", Types.BOOLEAN);
		flagState = this.getRuntimeContext().getState(flagD);

		//同样操作，注册一个timer并拿到其fd
		ValueStateDescriptor<Long> timerD = new ValueStateDescriptor<>("timer-state", Types.LONG);
		timerState = this.getRuntimeContext().getState(timerD);
	}

	@Override
	public void processElement(Transaction value, Context ctx, Collector<Alert> out) throws Exception {
		Boolean lastTransWasSmall = flagState.value();
		if(lastTransWasSmall != null){
			if(value.getAmount() > LARGE_AMOUNT){
				Alert alert = new Alert();
				alert.setId(value.getAccountId());
				out.collect(alert);
			}
			flagState.clear();
		}
		if(value.getAmount() < SMALL_AMOUNT){
			flagState.update(true);

			//注册一个timer，到时间就触发onTimer回调。
			//这里之所以额外维护一个state作为timer，是因为定时任务是事件匹配所驱动的。
			long timer = ctx.timerService().currentProcessingTime() + 1000 * 60;
			ctx.timerService().registerProcessingTimeTimer(timer);
		}
	}

	@Override
	public void onTimer(long timestamp, KeyedProcessFunction<Long, Transaction, Alert>.OnTimerContext ctx, Collector<Alert> out) throws Exception {
		timerState.clear();
		flagState.clear();
	}

}
```

具体有5种有状态流处理函数:

* KeyedProcessFunction,专门作用于键控流
* CoProcessFunction，作用于进行conncect之后的流
* ProcessWindowFunction：作用于窗口数据，可拿到状态
* ProcessJoinFunction：作用于`interval-join`之后的数据。这种Join也只能通过这种方式进行处理。
* BroadcastProcessFunction：作用于广播状态场景。

```java
        keyedStream01
                .intervalJoin(keyedStream02)
                .between(Duration.ofMillis(-500), Duration.ofMillis(500))
                .lowerBoundExclusive()
                .upperBoundExclusive()
                .process(new ProcessJoinFunction<Transaction, Transaction, Double>() {
                    @Override
                    public void processElement(Transaction left, Transaction right, Context ctx, Collector<Double> out) throws Exception {
                        out.collect(left.getAmount() + right.getAmount());
                    }
                })
                .filter(x -> x > 1000)
                .print();
        try {
            env.execute();
        } catch (Exception e){
            e.printStackTrace();
        }
```


#### 广播状态

> [Broadcast State 模式 | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/docs/dev/datastream/fault-tolerance/broadcast_state/)

广播状态，通常用于一个简单的规则流发布的场景，可以做简单的匹配，或单纯的过滤。

```java
public class BroadCastDemo {
    public static void main(String[] args) {
        Configuration conf = new Configuration();
        conf.setString("rest.port", "9091");
        conf.setBoolean("web.ui.enable", true);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(conf);
        DataStreamSource<Transaction> datastream = env.addSource(new TransactionSource());
        DataStreamSource<Rule> rulestream = env.addSource(new RuleStreamSource());
        KeyedStream<Transaction, Long> userPartitionedStream = datastream.keyBy(Transaction::getAccountId);
        MapStateDescriptor<String, Rule> descriptor = new MapStateDescriptor<>("ruleBroadcastState", String.class, Rule.class);
        BroadcastStream<Rule> ruleBroadcastStream = rulestream.broadcast(descriptor);

        SingleOutputStreamOperator<String> processed = userPartitionedStream
                .connect(ruleBroadcastStream)
                .process(new KeyedBroadcastProcessFunction<Long, Transaction, Rule, String>() {
                    private MapStateDescriptor<String, Rule> descriptor = new MapStateDescriptor<>("ruleBroadcastState", String.class, Rule.class);
                    //处理主数据流元素
                    @Override
                    public void processElement(Transaction value, ReadOnlyContext ctx, Collector<String> out) throws Exception {
                        Rule rule = ctx.getBroadcastState(descriptor).get("rule");
                        if (rule != null) {
                            if (rule.isValid(value)) {
                                out.collect(value.toString());
                            }
                        }
                    }
                    //处理广播规则流元素
                    @Override
                    public void processBroadcastElement(Rule value, Context ctx, Collector<String> out) throws Exception {
                        ctx.getBroadcastState(descriptor).put("rule", value);
                    }
                });
        processed.print();
        try {
            env.execute();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    private static class RuleStreamSource implements SourceFunction<Rule> {
        private boolean running = true;
        @Override
        public void run(SourceContext<Rule> ctx) throws Exception {
            while(running){
                Thread.sleep(500);
                ctx.collect(new Rule(RandomUtils.nextDouble()));
            }
        }
        @Override
        public void cancel() {
            running = false;
        }
    }
    public static class Rule{
        private Double amountLimit;
        public Rule(Double amountLimit){
            this.amountLimit = amountLimit;
        }
        public boolean isValid(Transaction transaction) {
            return transaction.getAmount() < amountLimit;
        }
    }
}
```

这个实例对`Flink`官网例子进行修改，通过内置数据源去做广播流。使用方法:

* 为广播流分配一个状态fd
* 然后和主数据流(通常为键控流)进行connect
* 对流进行process，此时一个`subtask`可以同时拿到最新的rule，以及最新的数据。但是两者在两个函数内，怎么交互呢？
  就是通过一个`subtask`内的全部变量-state来交互。这里的状态fd和第一步定义的需要完全一致。然后就可以在主数据流处理逻辑中使用了。

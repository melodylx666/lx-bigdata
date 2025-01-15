# asyncTool框架梳理

> 原项目地址：[asyncTool: 解决任意的多线程并行、串行、阻塞、依赖、回调的并行框架，可以任意组合各线程的执行顺序，带全链路执行结果回调。多线程编排一站式解决方案。来自于京东主App后台。 (gitee.com)](https://gitee.com/jd-platform-opensource/asyncTool)

## 梳理依据

* 框架作者专栏
* QuickStart
* GITEE 提交记录
* 单元测试

## 结构分析

### 基本结构

#### worker

一个worker，是对一段耗时逻辑的封装，是一个最小执行单元。其中的泛型`T,V`分别代表入参和返回值类型。

```java
@FunctionalInterface
public interface IWorker<T, V> {
    /**
     * 在这里做耗时操作，如rpc请求、IO等
     *
     * @param object      object
     * @param allWrappers 任务包装
     */
    V action(T object, Map<String, WorkerWrapper> allWrappers);

    /**
     * 超时、异常时，返回的默认值
     *
     * @return 默认值
     */
    default V defaultValue() {
        return null;
    }
}
```

同时，它还有一个子接口，是添加限时逻辑的worker。

```java
public interface ITimeoutWorker<T, V> extends IWorker<T, V> {
    /**
     * 每个worker都可以设置超时时间
     * @return 毫秒超时时间
     */
    long timeOut();

    /**
     * 是否开启单个执行单元的超时功能（有时是一个group设置个超时，而不具备关心单个worker的超时）
     * <p>注意，如果开启了单个执行单元的超时检测，将使线程池数量多出一倍</p>
     * @return 是否开启
     */
    boolean enableTimeOut();
}
```

#### callback

回调接口

```java
/**
 * 每个执行单元执行完毕后，会回调该接口</p>
 * 需要监听执行结果的，实现该接口即可
 *
 * @author wuweifeng wrote on 2019-11-19.
 */
@FunctionalInterface
public interface ICallback<T, V> {

    /**
     * 任务开始的监听
     */
    default void begin() {

    }

    /**
     * 耗时操作执行完毕后，就给value注入值
     * 参数分别是:是否执行成功，原始入参，执行结果
     */
    void result(boolean success, T param, WorkResult<V> workResult);
}
```

#### wrapper

对每个`worker`以及在其上安装的`callback`进行包装，也就是对于某个任务来说，其`wrapper = worker + callback`

然后将任务流的`wrapper`组合到一起，形成任务执行依赖图。这里和传统的DAG有所不同(比如Spark中的)，它可以指定某个点的依赖项到底用不用全部完成。通过`must`变量控制。

对于单个`wrapper`，其有入边集合以及出边集合，分别用`dependWrappers`和`nextWrappers`控制。其中由于上游依赖可以不必全部执行，则需要对上游的每个`wrapper`再次包装，只多了一个字段`must`.

则最终`dependWrappers`是一个`List<DependWrapper>`类型的变量

#### 组合

如果使用最原始的`Future`,那么结果的获取逻辑还是需要使用`get`进行阻塞主线程，而`get`的时机很难把握。

在JDK5中没有异步回调，想要后续操作结果就只能主线程同步阻塞。但是Netty，以及JDK8的Future都已经实现了异步回调的功能，可以在异步线程中执行所安装的回调，而不是主线程。 如果主线程想拿到结果再执行，必然是需要阻塞的。但是阻塞的时机如果过早或者过晚，就使得异步执行效果减弱甚至有bug了。所以如果回调逻辑和位置无关的话，放到异步线程执行效果更好。

如果要自己实现一个异步任务，模拟JDK8和Netty的Future，也就是可以安装回调函数，那么基本的模式就是`wrapper=worker+callback`

```java
public class BootStrap {
    public static void main(String[] args) {
        BootStrap bootStrap = new BootStrap();
        Worker worker = bootStrap.newWorker();
        Wrapper wrapper = new Wrapper();
        wrapper.setWorker(worker);
        wrapper.setParam("hello");

        bootStrap
                .doWork(wrapper)
                // 添加回调任务，异步线程执行完毕会在其线程调用
                .addListener(new Listener() {
                    @Override
                    public void result(Object result) {
                        System.out.println(Thread.currentThread().getName());
                        System.out.println(result);
                    }
                });
        System.out.println(Thread.currentThread().getName());
    }
    //开启一个线程执行异步任务，然后直接返回
    private Wrapper doWork(Wrapper wrapper){
        new Thread(() -> {
            Worker worker = wrapper.getWorker();
            // 执行任务并得到结果
            String action = worker.action(wrapper.getParam());
            // 使用监听器处理结果
            wrapper.getListener().result(action);
        }).start();
        return wrapper;
    }
    private Worker newWorker(){
        return new Worker() {
            @Override
            public String action(Object object) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return object + "world";
            }
        };
    }

}
```

输出结果为：并没有阻塞主线程，并且回调被执行。

`main`

`Thread-0`

`helloworld`

如果使用JDK8的Future，则可以如下实现同等效果:

```java
public class FutureTest {
    public static void main(String[] args) {
        CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "hello";
        }).thenAcceptAsync(res -> {
            System.out.println(Thread.currentThread().getName());
            System.out.println(res + " world");
        });
        System.out.println(Thread.currentThread().getName());
        future.join();
    }
}
```

### 使用流程

#### Wrapper创建

代码模式如下：

```java
        WorkerWrapper<String, String> workerWrapper1 =  new WorkerWrapper.Builder<String, String>()
                .worker(w1)
                .callback(w1)
                .param("1")
                .next(workerWrapper2)
                .build();
```

其中build方法逻辑如下所示：

```java
        public WorkerWrapper<W, C> build() {
            WorkerWrapper<W, C> wrapper = new WorkerWrapper<>(id, worker, param, callback);
            wrapper.setNeedCheckNextWrapperResult(needCheckNextWrapperResult);
            //创建中通过调用next等方法，将需要依赖的wrapper传入，通过nextWrappers等变量维护。
            // 在这里就可以依据这些变量，对依赖关系进行添加。
            if (dependWrappers != null) {
                for (DependWrapper workerWrapper : dependWrappers) {
                    workerWrapper.getDependWrapper().addNext(wrapper);
                    wrapper.addDepend(workerWrapper);
                }
            }
            if (nextWrappers != null) {
                for (WorkerWrapper<?, ?> workerWrapper : nextWrappers) {
                    boolean must = false;
                    //selfIsMustSet：存储强依赖于自己的wrapper集合
                    if (selfIsMustSet != null && selfIsMustSet.contains(workerWrapper)) {
                        must = true;
                    }
                    workerWrapper.addDepend(wrapper, must); //添加依赖,并说明当前项是该项的强依赖
                    wrapper.addNext(workerWrapper);
                }
            }

            return wrapper;
        }
```

#### 执行流程

异步任务通过Async的静态方法,以下面的同步阻塞版本为例。

线程池可以是用户自己创建的，或者默认的COMMON_POOL。

```java
    /**
     * 如果想自定义线程池，请传pool。不自定义的话，就走默认的COMMON_POOL
     */
    public static boolean beginWork(long timeout, ExecutorService executorService, WorkerWrapper... workerWrapper) throws ExecutionException, InterruptedException {
        if(workerWrapper == null || workerWrapper.length == 0) {
            return false;
        }
        //这里是list,因为可能依赖关系如下：
        //A ---->
        //       ---->C
        //B ---->
        List<WorkerWrapper> workerWrappers =  Arrays.stream(workerWrapper).collect(Collectors.toList());
        return beginWork(timeout, executorService, workerWrappers);
    }
```

接下来开始创建异步任务执行并同步阻塞：

```java
    public static boolean beginWork(long timeout, ExecutorService executorService, List<WorkerWrapper> workerWrappers) throws ExecutionException, InterruptedException {
        if(workerWrappers == null || workerWrappers.size() == 0) {
            return false;
        }
        //保存线程池变量，可能是自定义的线程池，也可能是默认的COMMON_POOL
        Async.executorService = executorService;
        //定义一个map，存放所有的wrapper，key为wrapper的唯一id，value是该wrapper，可以从value中获取wrapper的result
        Map<String, WorkerWrapper> forParamUseWrappers = new ConcurrentHashMap<>();
        //对第1层的workerWrapper进行遍历，启动异步任务，生成CompletableFuture数组
        CompletableFuture[] futures = new CompletableFuture[workerWrappers.size()];
        for (int i = 0; i < workerWrappers.size(); i++) {
            WorkerWrapper wrapper = workerWrappers.get(i);
            futures[i] = CompletableFuture.runAsync(() -> wrapper.work(executorService, timeout, forParamUseWrappers), executorService);
        }
        try {
            //等待所有任务完成，这里调用get同步阻塞
            CompletableFuture
                    .allOf(futures)
                    .get(timeout, TimeUnit.MILLISECONDS);
            return true;
        } catch (TimeoutException e) {
            Set<WorkerWrapper> set = new HashSet<>();
            totalWorkers(workerWrappers, set);
            for (WorkerWrapper wrapper : set) {
                wrapper.stopNow();
            }
            return false;
        }
    }
```

其中的核心在于`wrapper.work`方法中

在该方法中，首先进行剪枝判断，若不需要执行，则直接结束当前任务，开始下一个任务。

当确定要执行的时候，分两种情况。首先是只有一个依赖的情况。如果该依赖已经完毕，则直接`fire`，即创建异步任务并执行，主线程同步阻塞等待结果。这里要阻塞的原因当然是因为同一条路径上的每个任务是串行关系，需要同步等待。然后执行`beginNext`方法，该方法的核心逻辑与`Async.beginWork`方法全完一致，当然这里的入参中`fromWrapper`就是当前的`wrapper`而不是`null`了。

而当有多个依赖的时候，分为三种情况：

```
     *  1.如果所有依赖对于当前任务都不是必须的，则直接执行当前任务
     *  2.如果依赖中有必须的，但是当前任务是非必须的那个依赖所触发的，则不执行，并直接返回
     *  3.如果依赖中有必须的，并且当前任务是必须的依赖所触发的，则需要判断是否所有的必须依赖都执行完毕了。如果执行完毕了，则执行当前任务，否则不执行
```

则分别处理即可。

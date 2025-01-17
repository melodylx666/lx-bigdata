# 规则动态加载

> 以Flink程序所使用的JDK11为例子

## 类加载概述

类加载器用于将`class`文件加载到内存中。默认有三种类加载器：

* `Bootstrap classLoader` 启动/引导类加载器
* `Platform classLoader` 平台类加载器
* `App classLoader` 系统类加载器

```java
    @Test
    public void testALlClassLoader() throws ClassNotFoundException {
        assert ClassLoaderTest.class.getClassLoader().getName().equals("app");
        assert ClassLoaderTest.class.getClassLoader().getParent().getName().equals("platform");
        assert ClassLoaderTest.class.getClassLoader().getParent().getParent() == null;
    }
```

### Bootstrap classLoader

主要加载JDK核心类库，由C++实现，如果想要获取该类加载器，会返回null,因为对c++类的引用Java层面并不可见。

```java
    @Test
    public void testBootstrapClassLoader() throws ClassNotFoundException {
        assert String.class.getClassLoader() == null;
        assert ClassLoaderTest.class.getClassLoader().getParent().getParent() == null;
    }
```

### Platform classLoader

加载JDK的Module部分。

```java
    @Test
    public void testPlatformClassLoader() throws ClassNotFoundException {
        assert ClassLoaderTest.class.getClassLoader().getParent().getName().equals("platform");
        var parent = ClassLoaderTest.class.getClassLoader().getParent();
        var definedPackages = parent.getDefinedPackages(); //返回此加载器定义的所有软件包
        Arrays.stream(definedPackages).forEach(p -> {
            System.out.println(p.getName());
        });
    }
```

### Application classLoader

加载应用程序的类

```java
    @Test
    public void testAppClassLoader() throws ClassNotFoundException {
        ClassLoader appClassLoader = ClassLoaderTest.class.getClassLoader();
        assert appClassLoader.getName().equals("app");
        var aClass = appClassLoader.loadClass("com.lx.cep.dynamicDemo.hotDetect.MyCustomConditionV1");
        assert aClass.getName().equals("com.lx.cep.dynamicDemo.hotDetect.MyCustomConditionV1");
    }
```

## 类加载顺序

### parent first

`JDk`默认是`parent first`的加载模式，也就是双亲委派模式。如果一个类加载器要加载某个类，首先他不会自己去加载，而是递归的让父加载器去加载。如果`bootstrap classLoader`不能加载的话，则子加载器才尝试加载。

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

**作用**：

* 首先可以避免类的重复加载
* 可以保证核心API不被覆盖掉

### childFirst

Flink中，有两种类加载器，分别是`childFirst`以及`parentFirst`，两者都是`URLClassLoader`的子类。Flink等大数据集群默认都是`child-first`加载。

其中`parentFirst`类只是简单的包装了一下`URLClassLoader`，遵循双亲委派模式。

而`childFirst`类加载器，打破了双亲委派。

常见的Tomcat，由于需要部署多个Web应用，需要相互隔离，则`childFirst`是必须的。而常见的大数据集群，比如`Spark`以及`Flink`，同样同时运行着很多的`application`.那么如果用户的`application`和集群的依赖相互冲突，保证用户的jar包优先级也是必然要求。

Flink的实现中，还将部分类强制设置为parent模式。考虑正常在idea中运行Flink集群，都需要勾选`provided`选项，这就是由于Flink会将应用最小集暴露给用户，在轻度使用下，用户可以完全享受到Flink官方更新提供好的依赖环境。

## URLClassLoader

可以通过URL来加载指定的jar包和其中的类.

假设在某个特定的jar包中有如下的类：

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
    public static String HelloStr(String str) {
        return "hello" +" " + str;
    }
}
```

则可以实现通过URL动态加载该类

```java
    @Test
    public void testURLClassLoader() throws Exception {
        File file = new File("F:\\workspace_scala\\stream_rule_flink\\rule_minning\\target\\rule_minning-1.0-SNAPSHOT.jar");
        URL[] urls = {file.toURI().toURL()}; //将路径转换为URL数组
        URLClassLoader urlClassLoader = new URLClassLoader(urls); //加载jar文件
        var aClass = urlClassLoader.loadClass("com.example.Hello"); //加载指定类
        Object obj = aClass.newInstance();
        Method helloStr = aClass.getMethod("HelloStr",String.class);
        var str = (String)helloStr.invoke(obj, "world");
        assert str.equals("hello world");
    }
```

## 项目实现

在动态规则实现中，如果要实现动态规则的实时更新，或许可以按照如下方式进行：

首先，在本地部署和集群同样环境的Flink，防止类加载完不能使用。

然后，当需要规则上线的时候，在外部创建规则类，并暴露出全类名以及URL。

该规则jar包可以部署到远程存储中。

### 后端部分

在后端部分的Service层：`ServiceImpl`可以根据唯一标识找到jar包，并加载其中的类，即可动态创建规则，并存储到DB中。

为了简化以及解耦，可以将`where`部分的类抽离出来，并将常用的模式枚举出来，可做到前端平台以方便选择。这样就可以达到

> [探索Flink动态CEP：杭州银行的实战案例 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/14658914164)

里面所给出的平台的效果。

```java
        Object b =  new String[]{"70","80"}; //这里转换为Object，防止下面的构造器解包将数组每一项当做一个参数
        //测试URL方式加载自定义类
        File file = new File("F:\\workspace_scala\\stream_rule_flink\\engine_module\\target\\classes");
        URL url = file.toURI().toURL();
        System.out.println(url);
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{url});
        var aClass = urlClassLoader.loadClass("org.apache.flink.cep.a_demo.MyCustomConditionV3");
        var constructor = aClass.getConstructor(Object[].class);
        Object obj = constructor.newInstance(b);

        Pattern<TemperatureEvent, TemperatureEvent> customPattern2 = Pattern.<TemperatureEvent>begin("start")
                .where((IterativeCondition<TemperatureEvent>) obj)
                .next("middle")
                .where((IterativeCondition<TemperatureEvent>) obj)
                .within(Time.seconds(10));
        System.out.println(CepJsonUtils.convertPatternToJSONString(customPattern2));
```

### Flink引擎部分

在Flink引擎部分：为了使得Flink作业不停止，可以将jar包通过`flink-client`的方式使用`add jar`命令进行热加载。然后规则就可以定时从外部DB加载到应用中。

> [JAR 语句 | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/dev/table/sql/jar/#add-jar)

或者，为了统一，可以使用Flink RestFul API来上传Jar包或者删除Jar包。

> [REST API | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.20/zh/docs/ops/rest_api/)

例子如下：

```
### curl -X POST -H "Expect:" -F "jarfile=@path/to/flink-job.jar" http://hostname:port/jars/upload
POST http://localhost:9091/jars/upload
Content-Type: multipart/form-data; boundary=WebAppBoundary

--WebAppBoundary
Content-Disposition: form-data; name="jarfile"; filename="F:\workspace_scala\stream_rule_flink\rule_minning\target\rule_minning-1.0-SNAPSHOT.jar"

--WebAppBoundary--

###



### Get config
GET http://localhost:9091/config
Accept: application/json

### GET datasets
GET http://localhost:9091/datasets
Accept: application/json

### 查看上传的jar包
GET http://localhost:9091/jars
```

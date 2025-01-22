# HDFS  与 YARN

`HDFS`以及`Yarn`目前已经分别是大数据集群的存储以及调度的事实标准了，而`MapRecude`可能会在依旧使用`Hive on MapReduce`的时候才使用了，并且`API`很少写，完全可以被`Spark`淘汰，所以暂时不总结。

## HDFS

HDFS诞生于GFS论文，其是一个工程特性很强的分布式文件系统，它做了很多工程技巧的组合。

### 单Master

没有引入`zookeeper`下的HDFS是一个单Master的主从架构，并不是一个真正的高可用系统。但是，HDFS通过很多设计来尽量保证了HDFS的可靠性。

从不同的角度看HDFS，这个Master分别起着`目录服务`，`主从同步复制的主节点`，`主从异步复制的主节点`的作用。

#### 目录服务

HDFS，首先必然是一个`File System`。那么就和其他的任何文件系统一样，通过目录可以定位资源。

单机下的文件系统的目录服务加载在主存，而实际的数据在磁盘，并且可以通过内存中的`inode number`，对应到磁盘中`super block`的`inode`，然后通过`index`从`block`中查找数据。

而HDFS做了扩展，目录服务在`NodeManager`节点，实际的数据存储在上千个结点的磁盘中，由其上的`DataNode`维护。同时，每个`chunk`做了冗余存储：防止丢数据，同时将读写压力分摊到每个节点。

而`NodeManagr`，也就是`master`，则会`内存中`维护整个文件系统的三级元数据信息，如下：

* `Namespace`：有哪些文件
* `File to chunk Handle`：单个`File`有哪几个`chunk`，也就是`split`
* `chunk Handle to chunkServer`：单个`chunk`在哪个`chunkServer(DataNode)`上

**读数据请求**

而当`client`发出读写请求(一般是`offset + length`)之后，`master`通过三级元数据信息，找到`client`要操作的`chunk handle`以及`server 位置`，包括了所有副本，所以是多份。

然后`client`向其中最优的`server`进行通信，向其请求具体的`chunk的某个范围`，然后`server`返回数据。

**写数据请求**

`client`向`master`交互，之后得到`所有副本的server`位置。

而`client`需要把数据写到拿到的所有`server`中。这里`client`与`server`采用的是`pipeline`方式将所有数据送到每个`server`的`buffer`，然后由主副本节点将写请求按照统一顺序分发，然后所有`server`将数据落盘。

#### 容灾备份与快速启动

`master`所有的数据存储于`内存`，这样的考虑是基于可用性来说的，但是单单存储在内存，对于可靠性没有任何保证。

**checkpoint and edit log**

所以引入了`checkpoint`，也就是`检查点or快照`机制。在`HDFS中称为 FSImage`。顾名思义，就是给`master`所维护的三级元数据信息定期完全备份一份，所以可以看到实际上`FSImage`就是一个`目录结构`的文件。

并且，引入`edit log`，将每次检查点之后的操作日志记录下来，这样，任何时刻只要拿到`FSImage`，并`redo  edit log`，则可以得到最新状态的`master`。


**backup node**

`HDFS`同样支持`backup node`的功能:[Apache Hadoop 3.4.1 – HDFS 用户指南](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Backup_Node)

这里`HDFS`就是做了主从同步复制，`buckup node`需要和主节点完全一致，并且同步写完才能返回。


**shadow node**

又称辅助`namenode`，`2NN`。

这里`HDFS`做了主从异步复制，`shadow node`每次拿到的都是`master`中的`FSImage`以及`edit`，然后在自己这里进行合并，那么主节点可以在需要的时候读取该影子节点合并好的结果，而不是自己进行合并。异步任务的优势在这体现的很充分。


### 网络IO

#### pipeline

相比于硬盘的读写速度，真正的瓶颈在网络IO。

相比于全部依靠`client`的中心化模式，去中心化也就是`p2p`的流水线模式可以将总体吞吐量增加`n`倍，如果一共有`n`个节点的话。


### 一致性

这里指的就是每个`chunk`的多个副本之间的一致性。

#### 写入失败

写入失败，由于不是事务，则HDFS中数据必然会有不一致。

#### 顺序写入成功

顺序写入，则文件中数据是一致，并且确定的。

#### 并发写入成功

由于没有原子性以及事务的保证，同样会出现和预期不一致，以及覆盖的情况。所以需要在应用层保证数据写入是一致的。

#### 追加写入

也就是这里并不指定要在`chunk`的哪个位置写入数据，而是交给`主副本所在的server`来协调，则这里因为有了主副本节点的同一控制，则避免了并发写入的覆盖问题，单个`chunk`的写入最终都是协调顺序之后的结果。

而如果这个`chunk`有副本写入失败，则主副本节点会告诉`client`重新写入，则再次在该`chunk的所有副本追加写入`。

所以`HDFS`保证的是**至少一次**，也就是对于要写入HDFS集群的这个`chunk`中的数据而言，这个数据在每个副本中至少保存了一次，而追加顺序可以不一致，副本与副本可以是不一致。

针对重复的数据，应用层可以做额外的处理，比如加入ID以及时间戳，最后对数据去重即可。



## YARN

> [Apache Hadoop 3.3.6 – Apache Hadoop YARN - Hadoop 框架](https://hadoop.apache.ac.cn/docs/r3.3.6/hadoop-yarn/hadoop-yarn-site/YARN.html)

### 架构

#### 基本思想

YARN 的基本思想是将资源管理和作业调度/监控的功能拆分为单独的守护程序。这个想法是拥有一个全局 ResourceManager (*RM*) 和每个应用程序的 ApplicationMaster (*AM*)。应用程序可以是单个作业或作业的 DAG。

ResourceManager 和 NodeManager 构成了数据计算框架。

ResourceManager 是最终权威，它在系统中的所有应用程序之间仲裁资源。

NodeManager 是每个机器的框架代理，负责容器(Container)，监视它们的资源使用情况（CPU、内存、磁盘、网络），并将这些信息报告给 ResourceManager/Scheduler。

每个应用程序的 ApplicationMaster 实际上是一个特定于框架(比如Spark,Flink)的库，负责从 ResourceManager 协商资源，并与 NodeManager 合作执行和监视任务。


#### 组件

`ResourceManager`主要有两个组件，`Scheduler`以及`ApplicationsManager`

`Scheduler `负责根据容量、队列等常见约束将资源分配给各种正在运行的应用程序。其调度策略是可插拔的，最终有公平调度器，容量调度器等。

`ApplicationsMaster`负责接收作业提交，协商用于执行特定于应用程序的 `ApplicationMaster `的第一个容器，并提供在发生故障时重新启动 `ApplicationMaster` 容器的服务。

`ApplicationMaster`是一个与具体框架耦合的进程，比如一个集群可以运行`Spark & Flink`任务，但是单个`Job`的`AM`就是提交`Job`之后根据具体框架与任务才创建的。

#### 调度

调度策略有公平调度器，以及容量调度器，可以在`yarn-default.xml`文件中配置默认调度。

具体调度在`yarn-site.xml`中配置。

**CapacityScheduler**

核心特点是多租户，也就是不同的组织与部门，运行不同的任务，可以弹性共用一套`YARN`集群。

核心结构是一个资源的多级队列，一个组织的队列之间可以互相共享资源。

任务的优先级只支持FIFO

**FairScheduler**

所哟的应用程序可以公平的共享集群中的资源。默认情况下只根据`内存`进行判断，但是也可以修改为基于`内存 + cpu`进行判断。

默认情况下。所有任务都位于一个`default`队列中，当然也可以持多级层次队列，以`root`为顶层队列。

每个队列中，默认是通过基于内存的公平调度原则来调度任务，但是也可以配置FIFO。

默认情况下，允许所有任务运行，但是也可以限制特定用户以及每个队列的任务数量上限，防止某个用户独占集群。

### 通用app创建

一般概念是*应用程序提交客户端*向 YARN *ResourceManager* (RM) 提交*应用程序*。这可以通过设置`YarnClient`对象来完成。在启动`YarnClient`后，客户端可以设置应用程序上下文，准备包含*ApplicationMaster* (AM) 的应用程序的第一个容器，然后提交应用程序。您需要提供诸如应用程序运行所需本地文件/jar 的详细信息、需要执行的实际命令（带有必要的命令行参数）、任何操作系统环境设置（可选）等信息。

`实际上，您需要描述需要为您的 ApplicationMaster 启动的 Unix 进程`。

然后，YARN ResourceManager 将在已分配的容器上启动 ApplicationMaster（如指定）。ApplicationMaster 与 YARN 集群通信，并处理应用程序执行。它以异步方式执行操作。

在应用程序启动期间，ApplicationMaster 的主要任务是：

1.与 ResourceManager 通信以协商和分配未来容器的资源

2.在容器分配后，与 YARN *NodeManager*（NM）通信以在它们上启动应用程序容器。

任务 1 可以通过`AMRMClientAsync`对象异步执行，事件处理方法指定在`AMRMClientAsync.CallbackHandler`类型的事件处理程序中。需要将事件处理程序明确地设置为客户端。

任务 2 可以通过启动一个可运行对象来执行，该对象在分配容器时启动容器。作为启动此容器的一部分，AM 必须指定具有启动信息（例如命令行规范、环境等）的`ContainerLaunchContext`。

在应用程序执行期间，ApplicationMaster 通过`NMClientAsync`对象与 NodeManager 通信。所有容器事件都由与`NMClientAsync`关联的`NMClientAsync.CallbackHandler`处理。典型的回调处理程序处理客户端启动、停止、状态更新和错误。ApplicationMaster 还通过处理`AMRMClientAsync.CallbackHandler`的`getProgress()`方法向 ResourceManager 报告执行进度。

## 1.1 Spark简介
### 关于Spark
1. Spark 几个主要的特点
	1. 运行速度快
		1. Spark 使用DAG的执行引擎，以支持循环数据流与内存计算
		2.  基于内存的执行速度比Hadoop MapReduce快上百倍，基于磁盘的执行速度也要快上十倍
	2. 容易使用：Spark支持使用Scala、Java、Python和R语言进行编程，简洁的API设计有助于支持用户轻松构建并行程序，并且通过Spark Shell 进行交互编程
	3. 通用性：Spark提供了完整而强大的技术栈，包括SQL查询、流式计算、机器学习和图算法组件，这些组件可以无缝整合在同一个应用中，足以应对复杂的计算；
	4. 运行模式多样：Spark可运行于独立的集群模式中，或者运行于Hadoop中，也可运行于Amazon EC2等云环境中，并且可以访问HDFS、Cassandra、HBase、Hive等多种数据源。
## 1.2 Spark 运行架构
### 基本概念
1. RDD 弹性分布式数据集的简称，是一个分布式内存的一个抽象的概念，提供了一个高度受限的共享内存模型。
2. DAG：是Directed Acyclic Graph（有向无环图）的简称，反映RDD之间的依赖关系；
3. Executor：是运行在工作节点（Worker Node）上的一个进程，负责运行任务，并为应用程序存储数据；
4. 应用：用户编写的Spark应用程序；
5. 任务：运行在Executor上的工作单元；
6. 作业：一个作业包含多个RDD及作用于相应RDD上的各种操作；
7. 阶段：是作业的基本调度单位，一个作业会分为多组任务，每组任务被称为“阶段”，或者也被称为“任务集”。

### 架构设计
1. Spark 运行架构
![Spark 运行架构](https://lh3.googleusercontent.com/PTCSNVJUsLlQrBUl5Ule2I0tdKHPuPPRz9rrWBpuEHh_G1gMBS1mYJvwIOWm9JWrUmhDKFhGXJM5 "Spark 运行架构")
2. 如图所示，Spark运行架构包括集群资源管理器（Cluster Manager）、运行作业任务的工作节点（Worker Node）、每个应用的任务控制节点（Driver）和每个工作节点上负责具体任务的执行进程（Executor）。其中，集群资源管理器可以是Spark自带的资源管理器，也可以是YARN或Mesos等资源管理框架。
3. Spark中各种概念之间的相互关系
![enter image description here](https://lh3.googleusercontent.com/HHI2F5_t9OmMYUqULU4aRHfAEEZQKLEr05TpVgWEJiXr05D5GXWc-vT1_euLIo3911SLgTQQHEQ_ "Spark中各种概念之间的相互关系")
4. 在Spark中，一个应用（Application）由一个任务控制节点（Driver）和若干个作业（Job）构成，一个作业由多个阶段（Stage）构成，一个阶段由多个任务（Task）组成。当执行一个应用时，任务控制节点会向集群管理器（Cluster Manager）申请资源，启动Executor，并向Executor发送应用程序代码和文件，然后在Executor上执行任务，运行结束后，执行结果会返回给任务控制节点，或者写到HDFS或者其他数据库中。
### Spark 运行的基本过程
1. 当一个Spark应用被提交时，首先需要为这个应用构建起基本的运行环境，即由任务控制节点（Driver）创建一个SparkContext，由SparkContext负责和资源管理器（Cluster Manager）的通信以及进行资源的申请、任务的分配和监控等。SparkContext会向资源管理器注册并申请运行Executor的资源；  
2. 资源管理器为Executor分配资源，并启动Executor进程，Executor运行情况将随着“心跳”发送到资源管理器上；  
3. SparkContext根据RDD的依赖关系构建DAG图，DAG图提交给DAG调度器（DAGScheduler）进行解析，将DAG图分解成多个“阶段”（每个阶段都是一个任务集），并且计算出各个阶段之间的依赖关系，然后把一个个“任务集”提交给底层的任务调度器（TaskScheduler）进行处理；Executor向SparkContext申请任务，任务调度器将任务分发给Executor运行，同时，SparkContext将应用程序代码发放给Executor；  
4. 任务在Executor上运行，把执行结果反馈给任务调度器，然后反馈给DAG调度器，运行完毕后写入数据并释放所有资源
![enter image description here](https://lh3.googleusercontent.com/ey8OFWHtgMQG8XfH_DzLFRZejHm7BuE8BbknB0bThuNgc-KWONPAvlf8l95kD8xkw48v3Xe11J24 "Spark运行基本流程图")
5. Spark 运行架构的特点
	1. 每个应用都有自己专属的Executor进程，并且该进程在应用运行期间一直驻留。Executor进程以多线程的方式运行任务，减少了多进程任务频繁的启动开销，使得任务执行变得非常高效和可靠；  
	2. Spark运行过程与资源管理器无关，只要能够获取Executor进程并保持通信即可； 
	3.   Executor上有一个BlockManager存储模块，类似于键值存储系统（把内存和磁盘共同作为存储设备），在处理迭代计算任务时，不需要把中间结果写入到HDFS等文件系统，而是直接放在这个存储系统上，后续有需要时就可以直接读取；在交互式查询场景下，也可以把表提前缓存到这个存储系统上，提高读写IO性能；  
	4. 任务采用了数据本地性和推测执行等优化机制。数据本地性是尽量将计算移到数据所在的节点上进行，即“计算向数据靠拢”，因为移动计算比移动数据所占的网络资源要少得多。而且，Spark采用了延时调度机制，可以在更大的程度上实现执行过程优化。比如，拥有数据的节点当前正被其他的任务占用，那么，在这种情况下是否需要将数据移动到其他的空闲节点呢？答案是不一定。因为，如果经过预测发现当前节点结束当前任务的时间要比移动数据的时间还要少，那么，调度就会等待，直到当前节点可用。
## 1.3 RDD 的设计与运行原理
### RDD的设计背景
1. 理念来自于AMP实验室的一篇论文《Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing》
2. Spark的核心是建立在统一的抽象RDD之上，使得Spark的各个组件可以无缝进行集成，在同一个应用程序中完成大数据计算任务。
3. RDD就是为了满足这种需求而出现的，它提供了一个==通用==抽象的数据架构，不必担心底层数据的分布式特性，只需将具体的应用逻辑表达为一系列转换处理，不同RDD之间的转换操作形成依赖关系，可以实现管道化，从而避免了中间结果的存储，大大降低了==数据复制、磁盘IO和序列化开销==。
### RDD的概念
1. RDD是一个分布式对象集合，本质上是一个只读分区记录集合
2. 每个RDD可以分成多个分区，每个分区就是一个数据集片段，并且一个RDD的不同分区可以被保存到集群中不同的节点上，从而可以在集群中的不同节点上进行并行计算。
3. RDD提供了一种高度受限的共享内存模型，即RDD是只读的记录分区的集合，不能直接修改，只能基于稳定的物理存储中的数据集来创建RDD，或者通过在其他RDD上执行确定的转换操作（如map、join和groupBy）而创建得到新的RDD
4. RDD提供了一组丰富的操作以支持常见的数据运算，分为“行动”（Action）和“转换”（Transformation）两种类型，前者用于执行计算并指定输出的形式，后者指定RDD之间的相互依赖关系。两类操作的主要区别是，转换操作（比如map、filter、groupBy、join等）接受RDD并返回RDD，而行动操作（比如count、collect等）接受RDD但是返回非RDD（即输出一个值或结果）。
5. RDD提供的转换接口都非常简单，都是类似map、filter、groupBy、join等粗粒度的数据转换操作，而不是针对某个数据项的细粒度修改。因此，RDD比较适合对于数据集中元素执行相同操作的批处理式应用，而不适合用于需要异步、细粒度状态的应用，比如Web应用系统、增量式的网页爬虫等。
6. 正因为这样，这种粗粒度转换接口设计，会使人直觉上认为RDD的功能很受限、不够强大。实际上RDD已经被实践证明可以很好地应用于许多并行计算应用中，可以具备很多现有计算框架（比如MapReduce、SQL、Pregel等）的表达能力，并且可以应用于这些框架处理不了的交互式数据挖掘应用。
7. Spark用Scala语言实现了RDD的API，程序员可以通过调用API实现对RDD的各种操作。RDD典型的执行过程如下：
	1.  RDD读入外部数据源（或者内存中的集合）进行创建；
	2. RDD经过一系列的“转换”操作，每一次都会产生不同的RDD，供给下一个“转换”使用；
	3. 最后一个RDD经“行动”操作进行处理，并输出到外部数据源（或者变成Scala集合或标量）
	4. RDD采用了惰性调用，即在RDD的执行过程中（如图9-8所示），真正的计算发生在RDD的“行动”操作，对于“行动”之前的所有“转换”操作，Spark只是记录下“转换”操作应用的一些基础数据集以及RDD生成的轨迹，即相互之间的依赖关系，而不会触发真正的计算。
![enter image description here](https://lh3.googleusercontent.com/H6HUMAdTSbmtNfgi0SU7IMC9EfsXai4Y4jTFBkHCbtNGlguautGtnirc3HRkFvQwwVn9DK98u3RP "Spark的转换和行动操作")
8. RDD 执行过程的一个实例
![enter image description here](https://lh3.googleusercontent.com/XUeVUTODnkSeIB9Vl9CYt4muuEahQEj3Rt9ow_ywPPOP89m7g-7PczdPrBPl5D7JeHBu4OGvL87p "RDD执行过程的一个实例")
9. 上述这一系列处理称为一个“血缘关系（Lineage）”，即DAG拓扑排序的结果。采用惰性调用，通过血缘关系连接起来的一系列RDD操作就可以实现管道化（pipeline），避免了多次转换操作之间数据同步的等待，而且不用担心有过多的中间数据，因为这些具有血缘关系的操作都管道化了，一个操作得到的结果不需要保存为中间数据，而是直接管道式地流入到下一个操作进行处理。同时，这种通过血缘关系把一系列操作进行管道化连接的设计方式，也使得管道中每次操作的计算变得相对简单，保证了每个操作在处理逻辑上的单一性；相反，在MapReduce的设计中，为了尽可能地减少MapReduce过程，在单个MapReduce中会写入过多复杂的逻辑。
### RDD特性
1.  Spark采用RDD以后能够实现高效计算的主要原因如下：
	1.  高效的容错性
	2. 中间结果持久化到内存
	3. 存放的数据可以是Java对象，避免了不必要的对象序列化和反序列化开销。
### RDD 之间的依赖关系
1. 窄依赖
	1. 窄依赖表现为一个父RDD的分区对应于一个子RDD的分区，、
	2. 多个父RDD的分区对应于一个子RDD的分区；
2. 宽依赖
	1. 一个父RDD的一个分区对应一个子RDD的多个分区。
3. 总结
	1. 如果一个父RDD的一个分区只对应一个子RDD的一个分区所使用的就是窄依赖，否则就是宽依赖
	2. 窄依赖典型操作包括Map、filter、union等
	3. 宽依赖依赖的典型操作包括groupByKey、sortByKey等。
	4. 对于join操作，可以分为两种情况
		1. 对于输入的协同划分，属于窄依赖 。 所谓协同划分（co-partitioned）是指多个父RDD的某一分区的所有“键（key）”，落在子RDD的同一个分区内，不会产生同一个父RDD的某一分区，落在子RDD的两个分区的情况。
		2. 对输入做非协同划分，属于宽依赖，如图9-10(b)所示。  对于窄依赖的RDD，可以以流水线的方式计算所有父分区，不会造成网络之间的数据混合。对于宽依赖的RDD，则通常伴随着Shuffle操作，即首先需要计算好所有父分区数据，然后在节点之间进行Shuffle。
![enter image description here](https://lh3.googleusercontent.com/CsgbsUue5JlZL5StZdXMFGmoRxS76Z-Lwdrh0OYcOnHnNjAAMxYkeSnzQDiW8mo97ZCcm6P4UXOJ "宽依赖和窄依赖的区别")
4. 好处
	1.  天生的容错性，大大加速了Spark的执行速度。
		1. 因为RDD数据集通过“血缘关系”记住了它是如何从其他RDD中演进过来的，
		2. 它记录的只是==粗粒度==的转换操作的行为，当这个RDD的部分数据丢失时，
		3. 他可以通过血缘关系获得足够的信息重新运算和恢复丢失的数据，由此带来性能的提升。
5. 特点
	1. 窄依赖的失败恢复更为高效
		1.  它只需要根据父RDD分区重新计算丢失的分区即可（不需要重新计算所有分区），而且可以并行地在不同节点进行重新计算。
		2. 而对于宽依赖而言，单个节点失效通常意味着重新计算过程会涉及多个父RDD分区，开销较大。
	 2. Spark还提供了数据检查点和记录日志，用于持久化中间RDD。
	 3. 在进行数据恢复时，Spark会对数据检查点的开销和重新计算RDD的开销进行比较，从而自动选择最优的优化策略
### 阶段划分
1. 做法
	1. Spark通过分析RDD的依赖关系生成DAG
	2. 再通过分析各个RDD中的分区之间的依赖关系来决定如何划分阶段
		1.  在DAG中进行反向解析，遇到宽依赖就断开，遇到窄依赖就把当前的RDD加入到当前的阶段中；
		2. 将窄依赖尽量划分在同一个阶段中，可以实现流水线计算
		3. 具体的阶段划分算法请参见AMP实验室发表的论文《Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing》
2. 图示
![根据RDD分区的依赖关系划分阶段](https://lh3.googleusercontent.com/8zfSk7Ylo-pxjY1hWKUoSaJXoqcs4VKGMdzkCRHNEqmc6y0bUFnw69WPsKh4lgV89701xy921uXc "根据RDD分区的依赖关系划分阶段")
3. 目的
	1.  把一个DAG图划分成多个“阶段”以后，每个阶段都代表了一组关联的、相互之间没有Shuffle依赖关系的任务组成的任务集合。
	2. 每个任务集合会被提交给任务调度器（TaskScheduler）进行处理，由任务调度器将任务分发给Executor运行。
### RDD运行过程
1. 创建RDD
2. SparkContext负责计算RDD之间的依赖关系，构建DAG
3. DAGScheduler负责把DAG分解为多个阶段，每个阶段包含多个任务，每个任务会被任务调度器分发到多个工作节点（Worker Node）上Executor去执行
	![RDD在Spark中的运行过程](https://lh3.googleusercontent.com/zoiV7xSKomI2InrLrzP_r7iViNGTRqY1_vG2OTQd33eqZgW9hXvd1ELBMKIxmC-rLOGFVpB8DPlj "RDD在Spark中的运行过程")
## 1.4 Spark 部署模式
### Spark三种部署模式
Spark应用程序在集群上部署运行时,可以由不同的组件为其提供资源管理调度服务(资源包括CPU和内存等)。Spark支持三种典型集群部署方式:
1. standalone 模式
	1. Spark 该种模式在与MapReduce 1.0 完全一致
	2. 不同的是统一设计了一种统一的槽提供给各种任务
2. Spark on Mesos 模式
	1. Mesos是一种资源调度框架，可以为在它上面运行的Spark提供服务。
	2. Spark运行在Mesos上比运行在YARN上更灵活
	3. Spark官方推荐使用这种方式
3. Spark on YARN 模式
	1. Spark可以运行在YARN之上,与Hadoop 统一部署
	2. 资源管理调度依赖于YARN,分布式存储依赖于HDFS
![Spark on YARN架构](https://lh3.googleusercontent.com/OURDmCuokFWgGX9LqZBSrd5eenrqbVMlLN9AoZ2vqw1x5qy3GGV3GWz9sbeNgcaujkqYxGOaLBLJ "Spark on YARN架构")
### 从"Hadoop+Storm"架构转向Spark架构
1. 为了能同时进行批处理与流处理,企业通常采用"Hadoop+Storm"架构
2. 在这种部署中,Hadoop和Storm部署在资源管理架构YARN(或者Mesos)之上,接受统一的资源管理和调度,并共享底层数据存储(HDFS/HBase/Cassandra)![采用“Hadoop+Storm”部署方式的一个案例](https://lh3.googleusercontent.com/bksxNLUdHzv3_25Pwns787d9ctFeBx3zKvwiXHSJjnVx8tzbQoyovryAbro_3CNrvTJHUNVs_6Bk "采用“Hadoop+Storm”部署方式的一个案例")
3. Spark同时支持批处理和流处理
	1. 实现一键式安装和配置、线程级别任务监控和告警
	2. 降低硬件集群、软件维护、任务监控和应用开发难度
	3. 便于做成统一的硬件、计算平台资源池
4. Spark Streaming 无法支持毫秒级的响应。因此对于毫秒级实时相应的企业应用而言，仍需要采用流计算框架
![用Spark架构同时满足批处理和流处理需求](https://lh3.googleusercontent.com/-uzPvVwWA2QAyuQfgfZfsDLJ2zgSeOdTK6NsccWo5Fx6pa82mSZ9MJvJnguYe1zY8_7hYiPITyJN "用Spark架构同时满足批处理和流处理需求")
### Hadoop和Spark的统一部署
1. 原因
	1. Hadoop生态系统中一些组件实现的功能目前无法被Spark取代
	2. 企业中已有很多的现有应用，都是基于Hadoop组件开发的，完全转移到Spark需要一些成本
2. 在YARN上统一部署的好处
	1. 计算资源按需伸缩
	2. 不同负载应用混搭，集群利用率高
	3. 共享底层存储，避免数据跨集群迁移  
![Hadoop和Spark的统一部署](https://lh3.googleusercontent.com/okmNY8gRHgLTDNiFPSSoRK150uYCI47zGD9v8ReeI2KCi5E_BjbELuKukpz4WncAmKJyCOetVGQ2 "Hadoop和Spark的统一部署")

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwNTkwMDQ5MDQsLTEwMDA4NDI2MjAsLT
E5MjU0MTgwNTMsLTE0ODI2Nzg0MzIsMTY5NTkzNjc0OSw1MjI1
MDI4MTQsLTEzODI5OTI4MzksLTEyMDEwMDE1NTcsMTQwOTYwNT
U0MywtMTE0ODMwNDI4NywtODM1NDUxMDAwLC0xMDIwODMzMjUx
LC01MDE1NzQ4NDgsLTUwMTU3NDg0OCwxMjU4MDg3NTcxLC05MT
Q2NjEyNDgsLTE1NTc1MjAxMiwtMTYzNDc1Njc4MiwxODI4NjIw
MTY4LC0xOTc0Nzc3MjVdfQ==
-->
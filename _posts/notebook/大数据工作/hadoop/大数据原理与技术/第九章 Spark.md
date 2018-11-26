## 第9章 Spark
### 本章概览
1. 简要介绍了Spark与Scala编程语言
2. 分析Spark与Hadoop区别
3. Spark的生态系统和架构设计以及Spark的部署和应用方式
4. 介绍Spark的安装与基本编程实践
### 9.1 概述
1. Spark 的特点
	1. 运行速度快：DAG，基于内存
	2. 容易使用：编程简单，可以通过Spark Shell 进行交互式编程
	3.  通用性
	4. 运行模式多样 
2. Scala 简介
	1. Scala 的优点
		1. Scala具有强大的并发性。支持函数式编程，可以更好的支持分布式系统
		2. Scala语法简洁，提供了优雅的API
		3. Scala兼容Java ，运行速度快，能够融到Hadoop生态圈中
		4. Scala 在Spark中，相比其他编程语言的优势，Scala提供了REPL，因此可以在Spark Shell 中进行交互式编程
3. Spark和Hadoop的对比
	1. Hadoop的缺点：
		1. 表达能力有限
		2. 磁盘IO开销大
		3. 延迟高
		4. Spark的优点
		5. Spark的计算模式也属于MapReduce,但不局限于Map/Reduce
		6. Spark 基于内存计算
		7. Spark基于DAG 任务调度机制
### 9.2 Spark生态系统
1. 大数据处理主要包括三个类型。
	1. 复杂的批量数据处理
	2. 基于历史数据的交互式查询
	3. 实时数据流的数据查询
2. BADS架构
	1. Spark设计遵循“一个软件栈满足不同的应用场景”
	2. 图 9-4
![BADS架构](https://lh3.googleusercontent.com/w0QrpQQXBaHUo3onCabiiqOVbJdbTLb_GUb6ur1Ms3ssc8iY8alt6QYkWFHBLkkZHImfKaTytTgp "BADS架构")
3. Spark的核心组件
	1. Spark Core
		1. Spark Core 包含Spark的基本功能，如内存计算，任务调度，部署模式，故障恢复，存储管理
		2. 主要面向批处理
		3. Spark 建立在统一抽象的RDD之上，使其可以以基本一致的方式应对不同的大数据处理场景
	2. Spark SQL:
		1. SparkSQL允许开发人员直接处理RDD，同时也可以查询Hive，HBASE等外部数据
		2. SparkSQL一个重要的特点是其能够统一处理关系表和RDD，编程人员不需要自己编写Spark 应用程序，使用SQL命令进行查询，并进行更复杂的数据分析
	3. Spark Streaming
		1. Spark Streaming 支持高吞吐量、可容错处理的实时数据处理
		2. 其核心思想是将流数据分解成一系列短小的批处理，每个短小的批处理都可以使用Spark Core 进行快速批处理。
		3. Spark 支持多种数据输入源，如Kafka、Flume和TCP套接字等 
	4. MLlib
		1. 提供了常用的机器学习算法的实现 
	5. GraphX
		1.  是Spark中用于图计算的API，可认为是Pregel在Spark上的重写和优化
4. 无论是SparkSQL、Spark Streaming、MLlib还是GraphX，都是可以使用的Spark Core的API处理问题。他们之间的方法是通用的，处理的数据可以共享，不同之间的数据可以无缝集成
5. 表 9-1
![Spark的应用场景](https://lh3.googleusercontent.com/FSxMOInI51CSI6u97wtQpuS61DGckTCFAR6a7PLjDMPJ3muhifqGWl-ncA3O2cU2RkFrbNW2bjR2 "Spark的应用场景")
### 9.3 Spark 运行架构
#### 9.3.1 基本概念
1. RDD: 是弹性分布式数据集（Resilient Distributed Dataset）的英文缩写，是分布式内存的抽象概念
2. DAG：是Directed Acyclic Graph(有向无环图)的英文缩写，反映了RDD之间的依赖关系。
3. Executor：是运行在工作节点（Worker Node）上的一个进程，负责运行任务，并未运行程序存储数据
4. 应用：用户编写的Spark应用程序
5. 任务：运行在Executor上的工作单元
6. 作业：一个作业包含多个RDD即作用于相应RDD上的各种操作
7. 阶段：是作业的基本调度单位，一个作业会被分为多组任务，每组任务被称为“阶段”，或者被称为“任务集”
#### 9.3.2 架构设计
1. 图9-5
![Spark 运行架构](https://lh3.googleusercontent.com/91xCfZVRMeiv5Bhmr1jBLd2CcJPie1If_JDhICKMu2oVka-i8UKY7VTsRmw-Dh9uQKd-P7Bzvjml "Spark 运行架构")
2. 与MapReduce 计算框架相比，Spark所采用的Executor有两个优点：
	1. 利用多线程来执行具体任务，减少任务启动开销
	2. Executor中有一个BlockManager存储模块，会将内存和磁盘共同作为存储设备，当多次迭代计算时，可以将中间结果存储到这个存储模块里，下次需要直接从直接读该存储模块的数据，而不需要读写到HDFS等文件系统里，减少IO开销；或者在交互式查询场景下，预先将表缓存到该存储系统上，从而可以提高读写IO性能
3. 在Spark中，一个应用（Application）由一个任务控制节点（Driver）和若干个作业（Job）构成，一个作业由多个Stage构成，一个阶段有多个任务（Task）组成。
4. 图9-6
![Spark 中各种概念之间的相互关系](https://lh3.googleusercontent.com/AK8IxxENtEJ9PzP1VebEWdHcH4NGbCGgfegc2OZ3TL6RhpK3ZhgY8KOdjI-jV-Bi5nmMvS2JeGZr "Spark 中各种概念之间的相互关系")
5. 大致流程：
	1. 当执行一个应用时，任务控制节点会向集群管理器（Cluster Manager）申请资源，启动Executor，并向Executor发送应用程序代码和文件
	2. 然后在Executor上执行任务，运行结束后，执行结果会返回给任务控制节点，或者写到HDFS或者其他数据库中。 
### 9.3.3 Spark运行基本流程
1. 图9-7
![Spark运行基本流程图](https://lh3.googleusercontent.com/bqnSK_z07Diut-4sXwhIKROr_ApWBYTNUlh2sbEZJHC0oPe9yLKTAeTeSDRmWHHWVS4dslvDiS42 "Spark运行基本流程图")
#### 9.3.4 RDD设计与运行原理
1. 
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzU5MjE5OTkzLDMxMjQyMjQyNywtMTU0ND
gxNjQzMSwtMTcyMDczNDMyNywtMTgxNTM2MTUzNiwtMTk2MDk0
MjAyNyw1OTg4NjM4MjUsOTQ0NjY2NTEwLC0xMTIwMjA2NjQsMT
Y2MjI1MTIzLDE4MzIwNjg5MjgsLTE5NjIwMTMyNjAsMTQwMzk4
NzkzMCw2ODE5NTQwNjQsNTg5MDc4MDUyLDMxMTE1OTM1NCwxNj
A2NzcxNzUwXX0=
-->
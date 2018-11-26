## 第八章 Hadoop 再探讨
### 写在章节前面的话
1. Hadoop的优化和发展主要体现在两个方面：
	1. Hadoop两大核心组件MapReduce和HDFS的架构的改进与提升。
	2. hadoop系统的其他组件不断丰富，通过优化和提升，Hadoop可以支持更多的应用场景，提高更高的集群可用性，也带来更高的资源利用率
2. 本章内容概览
	1. Hadoop局限
	2. Hadoop在核心组件组件的新发展
	3. Hadoop生态系统的成员 
### 8.1 Hadoop的优化与发展
1. Hadoop1.0 核心组件(仅指MapReduce和HDFS)主要存在以下不足：
	1. 抽象层次低，编码复杂
	2. 表达能力有限
	3. 开发者需要自己管理作业依赖关系
	4. 难以看到程序整体逻辑。
	5. 执行迭代操作效率低。
	6. 资源浪费
	7. 实时性差
2. Hadoopdoop改进
	1. 表8.1
	![Hadoop框架自身的改进：从1.0到2.0](https://lh3.googleusercontent.com/8idxJJs20uBPUyCNFQUjoBCPlSPbwihASTMeQvik93ChaabO0eGDWhq7Rz81RdpK5ELURGaEvtfH "Hadoop框架自身的改进：从1.0到2.0")
	2. 表8.2
![不断完善的Hadoop生态系统](https://lh3.googleusercontent.com/meqovQ06f6fmcHf-bL3NQ5_RjoRvKJK9og7GdUUaGCy-u_u_48h8y01SIiKlYC3m5Vjje8YymWQW "不断完善的Hadoop生态不断完善的Hadoop生态系统系统")
### 8.2 HDFS2.0 新特性
#### 8.2.1 HDFS HA
1. 两个名称节点的状态同步，可以借助于一个共享存储系统来实现，比如 NFS、QJM、或者Zookeeper.
	1. 活跃名称节点的更新数据写入到共享存储系统
	2. 待命名称节点会一直监听该系统，一旦有新的写入，就立即从公共存储系统中读取这些数据并加载到自己的内存中，从而保证名称节点状态的完全同步。 
2. 要保证待命名称节点一直包含最新的集群各个块的位置信息
	1. 需要给数据节点配备两个名称节点的地址，并把块的位置信息和心跳信息同时发送到两个名称节点。
	2. Zookeeper可以确保任意时刻只有一个名称节点对外服务。如果HDFS集群中有“两个管家”，会导致数据丢失或者其他异常。
#### 8.2.2 HDFS 联邦
1. HDFS1.0 中存在的问题
	1. 单点故障
	2. 可扩展性
		1. 不可以横向扩展
		2. 不可以纵向扩展
			1. 会带来过长的系统启动时间
			2. 当在内存空间清理时，发送错误，就会导整个HDFS集群宕机。
	3.  性能
		1. 整个文件系统的性能会受限于单个名称节点的吞吐量
	4.  隔离性
		1. 单个名称节点无法难以提供不同程序之间的隔离性，一个程序可能影响其他运行的程序
2.  HDFS联邦的设计
	1. 
3. HDFS联邦的访问方式
4. HDFS联邦相对于HDFS1.0的优势
	1. 可扩展性
	2. 性能更好
	3. 良好的隔离性
	4. HDFS联邦并不能解决单点故障的问题，需要为每个名称节点部署一个后备名称节点，以应对名称节点宕机后对业务产生的影响
### 8.3 新一代资源管理调度框架YARN
#### 8.3.1 MapReduce1.0 的缺陷
1.  采取了Master/Slave 架构设计，包括一个JobTrack和若干个TaskTrack
2. 前者负责作业的调度和资源的管理，后者负责执行JobTracker指派的具体的任务
3. 这种架构很难克服的缺陷
	1. 存在单点故障
	2. JobTracker任务过重。
	3. 容易造成内存溢出
	4. 资源分配不合理
4. 图8-4
![MapReduce1.0 体系结构](https://lh3.googleusercontent.com/V0Cnn22Wu3uljYk8Omb8dTFs8Ts3tXbpR1FwlC2JrtuZB_c7QKmT3JjEU6b4GR1X7si6vdSoe9Pl "MapReduce1.0 体系结构")
#### 8.3.2 YARN设计思路
1. 基本思路就是放权，即不让JobTracker这一组件承担过多的任务,把原JobTraker三大功能（资源管理、任务调度、任务监控）进行拆分
2. YARN包括ResourceManager、ApplicationMaster和NodeManager
	1. ResourceManager：负责资源管理
	2. ApplicationManager：负责任务调度和监控
	3. NodeManager负责执行TaskTracker任务
	4. 图 8-5
![YARN 架构的设计思路](https://lh3.googleusercontent.com/jCbb1QgkVt8LUTvrU00B8TzeabonmFsckUdQo_PpEycK4LyRO9I3v8u0_FBboV4IrkeIzgAT4zs3 "YARN 架构的设计思路")
#### 8.3.3 YARN 体系结构
1. 图 8-6
![YARN 体系结构](https://lh3.googleusercontent.com/Vv3diXTJAJoif6ARj-SXTHwy0fULn150YOMedQNLHv4oTh6exPvzqfuPDT_ldUtEThjShNbTtylf "YARN 体系结构")
2. 表8-3
![YARN各个组件的功能](https://lh3.googleusercontent.com/YyK95fl4e6_brqdAV_JRRYBkeia1BYyntIGtKm2seMjMgI8WE0_owg4ulzxpcq21tV63kF25MVgr "YARN各个组件的功能")


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTIwMDk5NTMyOCw0MTQwMDk4OTgsOTU2Mj
g2NDMzLDYzMDQ1Mzk4Myw4MTMwMzA4MzMsLTUyNzQzMjI1NCwt
NDk3ODQ4MDQyLDE2NTg5NDk5MjksMzAyNTk1NDYwLDEzMDU0MD
M3NzcsLTM1NjQxMDA5MiwtMTgzMjgxMTk0OSwtMTcxODIwOTY3
MywtMjc0Mzc1NzU3LC0xNjU0MjcwMzAsLTEyNjMwMDk4MzZdfQ
==
-->
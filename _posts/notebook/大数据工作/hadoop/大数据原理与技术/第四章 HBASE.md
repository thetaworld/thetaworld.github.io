## 第四章
### 综述
1. HBASE 是Goggle BigTable的实现
2.  特点:
	1. HBASE 是bigTable的开源实现
	2. 特点: 是一个高可靠. 高性能 .面向列. 可伸缩的分布式数据库
	3. 用途: 用来存储非结构化和结构化的松散数据
### 4.1 综述
#### 4.1.1 从BigTable说起
1. BIgTable是一个分布式存储系统
2. 待写
#### 4.12 HBase 简介
1.  图4-1 表述了Hadoop 生态圈中HBase与其他部分的关系
	1. HBASE 利用 Hadoop MapReduce来处理HBASE的海量数据实现高性能的计算
	2. 利用Zookeeper 作为协同服务,实现稳定服务和失败服务
	3. 使用HDFS作为高可靠的底层存储,利用廉价的集群提供海量数据的存储能力
		1. 当然HBASE也可以直接利用本地文件系统而不用HDFS作为底层存储
		2. 不过为了系统的提高数据可靠性和系统的健壮性,发挥HBASE处理大数据等能力,一般用在HDFS上
	4.  为了方便在HBASE上进行数据处理,sqoop为HBASE提供高效的,便捷的RDBMS的的数据导入功能
	5. pig 为HBASE提供高层的语言支持
 图4-1
 ![Hadoop生态系统中HBase与其他部分对应关系](https://lh3.googleusercontent.com/97EtVMJ_vEk7cHLhIC5ZU2KcyqsPSuJ5L0r2D_eo3ZeVLAN_nHLVlTaU8MaVemY_5pzwtK6woxA "Hadoop生态系统中HBase与其他部分对应关系")
 #### 4.1.3 HBase与传统关系数据库的对比分析
 1. 数据类型:HBASE 则采取了更加简单的数据模型,他把数据存储为未经解释的字符串,用户可以把不同格式的结构化数据和非结构化数据都序列化为字符串保存在HBASE,用户需要自己编写程序把字符串解析为不同的数据类型
 2. 数据操作: HBase 则不存在复杂的表和表之间的复杂操作,只有简单的插入,查询.更新.查询等.因为HBase在设计上避免了复杂的表跟表之间的的关系,通常是只采取单表的主键查询,所以无法实现关系数据库中那样表与表之间的关联操作
 3. 存储模式: HBase 是基于列存储的,每个列都是由几个文件保存,不同的列族的文件是分离的,她的优点是:
	 1. 降低I/O开销,支持大量并发用户查询,因为仅需要处理可以回答的这些查询的列,而不需要处理和查询与查询无关的大量数据行;
 4. 数据索引: HBASE只有一个索引行健.HBase 位于Hadoop框架之上,因此,可以使用Hadoop MapReduce来快速.高效的生成索引列表
 5. 数据维护:HBase 在执行更新操作时,并不会删除数据旧的版本,而是生成一个新的版本,旧的版本依然保留
 6. 可伸缩性:可以实现灵活的的水平扩展
 7. 局限性:HBase 也有自身的局限性,如HBase 不支持事物,因此无法实现跨行的原子性
 ### 4.2 HBASE 访问接口
 8.  HBase 提供了Native Java API, HBase shell . Thrift Shell .Rest GateWay .Pig .Hive等多种访问方式
 9. 表 4-2
 ![enter image description here](https://lh3.googleusercontent.com/FONqFyhB3aAqoLJ4iWyAD5R5POYF0Ytq26QfI-XGLZyvtkApHIcsjhgGWviF2n05x9s6bXeN4Uw "HBase的访问接口")
 
 

 ### 4.3 HBase 数据模型
 本节介绍了HBase的列族数据模型,并阐述了Hbase 的数据库的概念视图和物理视图的差别
 #### 4.3.1 数据模型概述
 1. Hbase 是一个稀疏的.多维度.排序的映射表,这张表的索引是行健,列族,列限定符和时间戳
 2. 是一个是未经解释的字符串,没有数据类型
 3. 由于对于整个映射表的每行数据而言,有些列的值就是空的,所以说HBASE是稀疏的
 4. 在HBASE执行更新操作的时候,并不会删除旧版本的数据,而是生成一个新的版本
	 1.  Hbase 提供了二种数据版本回收的方式:
		 1.  保存数据的最后n个版本
		 2. 保存最近一段时间内的版本(如最近7天)
####   4.3.2 数据模型的相关概念
1. 表
	1. Hbase 采取表组织数据,表有行和列组成,组成若干个列族
2. 行
	1. 每个行由行键(Row Key)来标识
	2. 访问表 有三种方式
		1. 单个行键
		2. 一个行键的区间来访问
		3. 全表扫描
	3. 行键的为字符串,在内部保存为字节数组,最大长度为64KB 
	4. 在设计行键的,要充分考虑这个特性,将经常一起读取的行列存储在一起
3.  列族
	1. 
4. 列限定符
	1. 列限定符不用事先定义,也不要在不同行之间保持一致
5. 单元格
	1.  在Hbase 表中,通过行,列族和列限定符确定一个单元格
	2. 每个单元格中可以保存一个数据的多个版本
	3. 每个版本对于一个不同的时间戳
6.  时间戳
	1. 一般为64为整型
		1. 可以由用户自己赋值
		2. 也可以在HBASE在数据写入时自动赋值
7.  图 4-2

![enter image description here](https://lh3.googleusercontent.com/9fgXH3TI2qksazpRyD3g8Q-vi7GfjuVRrSrIkNPe9z51VQ71PD_nOxUifbapAZA6CImZwk6IMCQ)

####  4.3.3 数据坐标
1. HBASE 使用坐标来定位数据
2. HBASE 中根据行键.列族.列限定符和时间戳来确定一个单元格,因此可以视为一个"四维坐标",即[行键,列族,列限定符,时间戳]
3. 如果把所有坐标看做一个整体,视为"键",把四维坐标对应的数据单元格中的数据视为"值",那么HBASE 也可以看做一个键值数据库.
4.  图4-3

![Hbase 可以看做一个键值数据库](https://lh3.googleusercontent.com/zfFbU_pAIPCyBVHGnsg2sJtQSnEpSXP8_UUAsseOF717ke2NGZKz_mVdNjAIIe1FlkD_fLadZbg)

#### 4.3.4 概念视图
1. 图 4-5

![HBASE概念视图](https://lh3.googleusercontent.com/tvF7TqVwXiQeIzFg185Mxe0Np5_NuqxOq7YT6JgtPXuAk-JLlEvhzPjSJg1K7_e3lCxG2cs3vJc "Hbase 概念视图")
#### 4.3.5 物理视图
1. HBASE 会按照contents 和anchor 这两个列族分别存放,属于同一个列族的数据保存在一起,同时,和每个列族一起存放的还包括行键和时间戳
2.  图 4-5
	![HBASE 物理视图](https://lh3.googleusercontent.com/DI3G7a4J5FGieoe4AgqXGkKX78mKfMHYRTs-o0efnOB5hb2ONaovDj9NVRBemVVLYlcTEZLHgcw "HBASE 物理视图")
3. 在物理视图中,这些空的列不会被存储为null,而是根本不会存储,当请求这些这些空白单元格的时候,会返回null值
#### 4.3.6 面向列的存储
1. HBASE 并不是一个列存储的数据库,HBASE是以列族为单位进行分解(列族可以包含很多列)
2. 是关于NSM(N-ary Storage Moedel) 和DSM(Decomposition Storage Model)的不同
3. 图4-4 是一个关于行存储结构和列存储结构的实例
	
	![行存储和列存储结构](https://lh3.googleusercontent.com/lHsOL4O2Qc3-yXRSAIrOGTRyDglqefH64fJHfhdamUZEKlyayxTcqkgrwXxPz7ZeUzt1aYtjcVw)
### 4.4 HBase 的实现原理
####  4.4.1 HBase 的功能组件
1. Hbase 的功能组件
	1. 库函数,连接到每个客户端
	2. 一个Master服务器.主服务器Master 负责管理和维护HBase表的分区信息,比如一个表被分为哪些Region,每个Region被存放在哪台Region服务器上,同时也负责维护Region服务列表.因此,如果Mater主服务器死机,那么这个系统就会失效. Master 会实时监控集群中的Region服务器,把指定的Region分配到可用的Region服务器上,并确保整个集群内部不同Region服务器之间的负载均衡,当某个Region服务器失效时,Master会把该故障服务器上的Region重新分配到其他可用的Region服务器,除此之外,Master还有处理模式变化,如表和列族的创建.
	3. 许多个Region 服务器.Region 服务器负责存储和委会分配给自己的Region,处理来自客户端的读写请求.
	4.  系统架构:
		1. 客户端并不是直接从Master 主服务器上读取数据,而是在获得Region的存储位置后,直接从Region服务器读取数据.尤其需要指出的是,HBASE客户端并不依赖于Master而是借助Zookeeper来获得Region的位置信息,所以大多数客户端从来不和Master通信,这种设计使Master的负载很小.
#### 4.4.2 表和Region
1. 在一个HBASE中存储了很多表.对于每个HBASE表而言,表的行是根据行键的值字典序维护的,
2. 需要根据行键的值对表中的行进行分区,每个行区间构成一个分区,被称为"Region",包含了位于某个值域区间的所有数据,它是负载均衡的和数据分发的基本单位,这些Region会被分配到不同的Region服务器上.
3.  对于Region的理解:

![对于Region的理解](https://lh3.googleusercontent.com/DfmHDVo8OecO9623bBHnl3_dnn7Et9egXo4-TVnHhLfmrRzzvoxHrL1OVYjtqBHyFOpx0qlmvqY "Region的理解")
4. Region 服务器与Region的关系
![Region和Region服务器的关系](https://lh3.googleusercontent.com/D4RRHHAlJCydHcK9WQrtwE5nALtOVidkB853kPvWTvci6QtZdcEikiH6KbcfIXa-0YTZfzhsM8A "Region和Region服务器的关系")
#### 4.4.3 Region 的定位
设计相应的Region定位机制,保证客户端知道在哪里找到自己需要的数据.

1. 每个Region都有一个RegionID来标识它的唯一性,一个Region的标识符就可以表示为"表名+开始主键+RegionID"
2. 每个Region都有一个RegionID来标识它的唯一性
3. 构建一个映射表,映射表的每个条目(或者每行)包含两项内容
	1.  Region标识符
	2. Region服务器标识
4. 这个映射表包含了关于Region的元数据(即Region和Region服务器之间的对应关系),因此被称为"元数据表",又名".META表"
5. .META表也会被分裂为多个Region,为了定位这些Region,就需要在构建一个新的映射表,记录所有与元数据的具体位置.-"-ROOT-表"
6. HBASE 采取类似B+树的三层结构来保存Region位置信息.
7. 图 4-6  HBASE的三层结构中各层次的名称和作用
![HBASE的三层结构中各层次的名称和作用](https://lh3.googleusercontent.com/3TABmQaANjLmt6unZ7Xy_k_cyPSvixG4rMeKItVqIV72GSjX7T1uSgt55zK3gmOtdIkNziVFbM4 "HBASE的三层结构中各层次的名称和作用")
	HBASE的三层结构
![enter image description here](https://lh3.googleusercontent.com/c442hSdOdhN9pGGF3uVGiFy5Ors9-hutkdZ5aa03GpPTfeRb5GkUqqLoVgNHGplACALrZwbq_Pg "HBASE三层结构")
8. 为了加速寻址过程,一般都会在客户端进行缓存.在需要访问数据时,从缓存中获取Region位置信息却发现不存在,才会判断出缓存失效.
9.  当一个客户端从Zookeeper服务器拿到-ROOT_ 表地址后,就可以通过"三级寻址"找到用户数据所在的Region服务器,没有必要在连接主服务器Master.因此,主服务器的负载就小很多.
### 4.5 HBase 运行机制
#### 4.5.1 HBase 系统架构
1. 图4-9 
HBASE的系统架构
![enter image description here](https://lh3.googleusercontent.com/js8vbUsGqifbRtR5uspMxGcnRoimpcaFJXgi3L8ZSVn6SguBsE_N8Y-TgFPwovhFA6qcGZCn768)
2. 各个组件的功能介绍
	1. 客户端
		1. 客户端包含HBASE的接口
		2. 缓存维护着已经访问的Region位置信息,用来加快后续数据访问的数据访问过程.
		3. HBASE客户端利用PRC机制与Master和Region服务器进行通信
			1. 对于管理类操作,客户端与Master进行RPC;
			2. 对于数据读写类操作,客户端则与Region进行RPC.
	2. Zookeeper服务器:
		1. 可能是多台机器构成的集群来提供稳定可靠的协同服务
		2. Zookeeper帮助Master服务器了解Region服务器的工作状态
		3. Zookeeper服务器在多Master服务器的时候,可以选择一个Master作为集群的总管,并保证任何时刻总有唯一一个Master在运行,这样就避免了Master的"单点失效问题".
		4. Zookeeper 中保存了-ROOT-表的地址和Master的地址,客户端可以通过访问Zookeeper获得-ROOT-表的地址,并且最后通过"三级寻址"找到所需的数据.
		5. Zookeeper还存储了HBASE的模式.包括有哪些表,每个表有哪些列族
	3. Master: 
		1. 主要负责表和Region的管理工作
			1. 管理用户对表的增加,删除,修改,查询等操作
			2. 实现不同Region服务器的负载均衡
			3. 在Region分裂或者合并之后,从新调整Region的分布
		2. 对发生故障失效的Region服务器上Region进行迁移
		3. Master仅仅维护着表和Region的元数据信息,因此负载很低
		4. 任何时刻,一个Region只能分配一个Region服务器
	4. Region服务器
		1. Region服务器是HBASE中最核心的模块,负责维护分配给自己的Region,并且相应用户的读写请求.
		2. HBASE一般采用HDFS做底层存储,因此Region服务器需要向HDFS文件系统读写数据.
		3. 可以使用其他任何支持Hadoop接口的文件系统作为底层存储,比如本地文件系统或者云计算环境中的Amazon S3
####  4.5.2 Region 服务器的工作原理
1. 概述
	1. Region 服务器内部管理了一系列Region对象和一个HLog文件
		1. HLog 是磁盘记录的文件,他记录了所有的更新操作
	2.  每个Region又是由多个Store组成的
	3.  每个Store对应与表中的一个列族的存储.
	4. 每个Store又包含一个MemStore和若干个StoreFile,其中MemStore是内存中的缓存,保存最新的更新的数据.StoreFile是磁盘中的文件,这些文件是B树结构的,方便快速读取.
	5. StoreFile底层实现方式是HDFS文件系统的HFile,采取压缩的方式,减少网络IO和磁盘IO
![enter image description here](https://lh3.googleusercontent.com/H2SrVec8HtUuyq2XTmTUfHDtNtLClQEs4rP3KkAtbzbgByh_V1Fd5QduEwlhzQjBYoE7e4hrsGuG)
2. 用户读写数据的过程
	1. 写入数据: 会被分配到Region服务器去执行操作.用户数据首先被写入MemStore和HLog,当操作写入HLog之后,commit()调用后,才会将其返回客户端.
	2. 读取数据: Region服务器首先会访问MemStore的缓存,如果不在缓存中,才会到磁盘上面的StoreFile中去寻找 
3. 缓存的刷新
4. StoreFile的合并: 为了减少查询时间,系统一般会调用Store,compact()把多个StoreFile文件合并为一个大文件.由于合并操作比较耗消费资源,因此只会在StoreFile文件的数量到达一个阈值的时候才会触发合并操作. 
#### 4.5.3 Store的工作原理
1.  当多个StoreFile文件合并后,会逐渐越来越大的StoreFile文件,当单个StoreFile文件大小超过一定阈值时,就会文件分裂操作,同时一个Region就会被分裂为2个子Region,父Region下线,新分裂的 2个子Region就会被Master服务器分配到相应的Region服务器上. 
2. 示意图 图4-11
![enter image description here](https://lh3.googleusercontent.com/7ftGPCceqkHjohQPYmES4Iy2zMzMZN2UBNuacRJ_RXvw5yvlUk6gNHtWQ8ZjmQm7_yjVzr5hthF4)
#### 4.5.4 HLog的工作原理
1. 预写式日志(Write Ahead Log)
2. Region对象共用一个HLog,如果一个Region服务器发生故障,为了其上的Region对象,需要对Region服务器上HLog按照所属的Region对象进行拆分,然后发到其他Region服务器进行恢复.
### 4.6 HBase 编程实践
1. [HBASE的基础入门](https://dblab.xmu.edu.cn/blog/install-hbase/)
2. [HBASE的中文教程](https://www.yiibai.com/hbase/hbase_shell.html)

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExMjA4NTI3NiwtNDQ5MTU5NjI2LDEyMT
Y0OTEwMTUsLTEyNTYyODg0OTksMTMxOTcxMjk0NywtMzY1NjQ0
MjI3LC05Mjc5NDg5MjUsMjA1NDgyNDU1NSwtNzkzNTUyODIzLC
0xNjcxMTEzMjUxLC0yNTc0NTU3MzIsLTEzMjk3NDk3MzgsMTI4
MTgwNzc0MywtNjM3NTk3OTU2LC04NjE2ODkzODUsMTQ2NjczMD
M2MCwtNzEwNjMwMDEyLC0xNDM2Njk4MzQxLDQ5ODIyMjc4OSw1
NTMyNTM3NzNdfQ==
-->
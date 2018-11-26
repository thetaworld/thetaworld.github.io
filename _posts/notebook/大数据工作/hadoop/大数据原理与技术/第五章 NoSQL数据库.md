## 第五章
### 写在章节前面
1. 传统的关系型数据库的特点
	1. 可以较好的支持结构化数据存储和管理
	2. 支持事物的ACID的特性
	3. 借助索引实现高效查询
2. 在web 2.0 时代面临的问题
	1. 数据类型繁多,包含结构化数据和大量非结构化数据
	2. 由于关系型数据库模型不够灵活,水平扩展能力较差等局限,无法满足大规模非结构的话的数据的存储.
	3. 事物机制和支持复杂查询,在web 2.0 时代很多应用成为鸡肋
3. 章节组织的顺序
	1. NoSQL 兴起的原因
	2. NoSQL数据库的四大类型和NoSQL数据库的三大基石
	3. NewSQL 数据库
### 5.1 NoSQL 简介
1. 是对非关系型数据库的统称,采取的类似键/值,列族,文档等非关系型模型
2. 不存在固定表结构,通常不存在连接操作,也没有遵守ACID约束 因此,具有水平扩展性
3. NoSQL 的三个特点
	1. 灵活的可扩展性
		1. 升级硬件来实现"纵向扩展"
		2. NoSQL可以满足"横向扩展"
	2. 灵活的数据模型:采取键/值,列族等非关系模型
	3. 与云计算紧密融合:
### 5.2 NoSQL兴起的原因
1. 关系型数据库无法满足web 2.0 需求
	1. 无法满足海量数据的管理需求
	2. 无法满足数据的高并发需求
	3. 无法满足高扩展和高可用的需求
2. 关系数据库的关键特性在web 2.0 时代成为鸡肋
	1. web 2.0 网站并不不要求严格的数据库事务
	2. web 2.0 并不要求严格的读写实时型,对于关系数据库而言,一旦有一条记录成功插入数据库中,就可以立即被查询.
	3. web 2.0 通常不包含大量复杂的SQL 查询
### 5.3 NoSQL 与关系型数据可的比较
1. 两者各有优势,也存在不同层面的缺陷,又有市场分工
2. 对于一些复杂的查询分析型数据仓库产品,仍然比NoSQL数据库获得更好的性能
### 5.4 NoSQL的四大类型
1.  键值数据库
2. 列族数据库
3. 文档数据库
###  5.5 NoSQL的三大基石
1. CPA
	1. C: 一致性
	2. P:可用性
	3. A:分区容忍性
2. BASE:
	1. Basically Available 
	2. Soft-state:指状态可以有一段时间不同步
	3. Eventually consistency:
3. 最终一致性:参考分布式操作系统
### 5.6 从NoSQL到NewSQL数据库
1. 大数据引发数据数据处理架构变革

![大数据引发数据数据处理架构变革](https://lh3.googleusercontent.com/dvYuP8uC4gJHkVuI9RRYaf4wsIU9uAM-QvZFLp1kNeXZVOiLZaSwkkb6Ehm9bqAePefto3-IoyFl)

2. 关系型数据库,NoSQL,NewSQL数据库产品分类

![enter image description here](https://lh3.googleusercontent.com/8PBcmw8V5ui3iFmDZHxPHfU01iUeJoylLQ01fIcftxxa_dSuAK5Mlkj3dNB5Keb9sUuBrgs_CCNZ "关系型数据库,NoSQL,NewSQL数据库产品分类")
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkzNjI0MTg2NCwxNzMzNDQyODI1LDk4Mj
E5MzczMCw1MjY1MzM2MTcsNDQ4NDE0NjM4LC0xODIxNzcxODBd
fQ==
-->
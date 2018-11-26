# 第三篇 大数据处理与分析(重点篇)
## 本篇内容概览
1. 大数据包括静态数据和动态数据(流数据)
2. 静态数据适合采用批量数据处理.动态数据需要进行实时计算.
3. 本篇知识地图
![enter image description here](https://lh3.googleusercontent.com/OwN_SHDKQ2bQS9n4Aga081q1QcDRldzvoFsvwpkZh9IxqU7XhY1wwSQVQ5704w0inmsNkyRgLsq7)
## 第七章 MapReduce
###  7.1 概述
1. Map 和Reduce 函数
	1. MapReduce 编程比较容易,是因为程序员只要考虑如何实现Map和Reduce函数,而不需要处理并行编程中的其他各种复杂问题,如分布式存储,工作调度,负载均衡,容错处理和网络通信等等,这些MapReduce框架负责处理.
### 7.2 MapReduce的工作流程
#### 7.2.1 工作流程概述
1. MapReduce 的核心思想可以"分而治之"来描述
2. 
#### 7.3 实例分析: WordCount
1. 适合用MapReduce处理的数据集满足一个前提条件:待处理的数据集可以的分解为许多小的数据集,而且每个数据集都可以并行的进行处理.
#### 课外科普
1. **[MapReduce入门和编程指导(推荐仔细阅读)](http://dbaplus.cn/news-21-1277-1.html)**
 
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjY5NDA3NDMwLC0xMjgxMzM0OTIzLDE5OT
U2NjUxMjIsLTEzNzM0ODM5MzYsMzQzNzE3MDUsLTE3ODMzNzEx
OSwtMTAyMjUxMjIzOSw3OTc0NjY2ODAsMTQ1MDYyMzk1LC05OD
A4NTIwMjEsMzI2MzEzNTNdfQ==
-->
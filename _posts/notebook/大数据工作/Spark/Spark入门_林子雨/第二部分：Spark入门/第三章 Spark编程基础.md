## RDD编程
- RDD 创建
	- Spark采用textFile()方法来从文件系统中加载数据并创建RDD
	- 这种方法把文件的URI作为的参数，这个URI可以是本地文件系统、分布式文件系统HDFS的地址或者是Amazon S3的地址等
		- 从文件系统中加载数据创建RDD
		- 从分布式文件系统HDFS中加载数据
	- 通过并行集合（包括数组）创建RDD
- RDD操作
	- RDD操作包括两种类型
		- 转换（Transformation）
		- 行动（Action）
	- 转换操作
		- 对于RDD而言，每一次转换操作都会产生不同的RDD，供给下一个操作使用。
		- RDD的转化过程是惰性的，只有遇到行动操作，才会触发“从头到尾”的真正的计算。
		- 常用的RDD转化操作API
			* filter(func)：筛选出满足函数func的元素，并返回一个新的数据集  
				```
				// 筛选出满足函数func的元素，并返回一个新的数据集RDD
				val lines =sc.textFile("file:///home//hadoop/Desktop/word.txt")
				val lineWithSpark =lines.filter(line=>line.contains("Spark"))
				lineWithSpark.foreach(println)
				```
			* map(func)：将每个元素传递到函数func中，并将结果返回为一个新的数据集  
				```
				将每个元素传递到函数func中，并将结果返回一个新的数据集RDD
				val data =Array(1,2,3,4,5)
				val rdd1= sc.parallelize(data)
				val rdd2=rdd1.map(x=>x+1)
				result：
					2
					5
					6
					4
					3
				``` 
			* flatMap(func)：与map()相似，但每个输入元素都可以映射到0或多个输出结果  
			* groupByKey()：应用于(K,V)键值对的数据集时，返回一个新的(K, Iterable)形式的数据集  
			* reduceByKey(func)：应用于(K,V)键值对的数据集时，返回一个新的(K, V)形式的数据集，其中的每个值是将每个key传递到函数func中进行聚合
		- 
	- 行动操作
		- 常见的RDD行动操作API
			- count() 返回数据集中的元素个数  
			* collect() 以数组的形式返回数据集中的所有元素  
			* first() 返回数据集中的第一个元素  
			* take(n) 以数组的形式返回数据集中的前n个元素  
			* reduce(func) 通过函数func（输入两个参数并返回一个值）聚合数据集中的元素  
			* foreach(func) 将数据集中的每个元素传递到函数func中运行
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0Mzg0ODkzNjEsLTE2NDA1MDQzNTcsLT
EwMzIzOTI0OCwtMTEzNTM3NzczLC0yMDg4NzQ2NjEyXX0=
-->
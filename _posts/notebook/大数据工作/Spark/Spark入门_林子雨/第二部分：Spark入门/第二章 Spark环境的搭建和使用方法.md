## 2.1 安装Spark与使用
 - 参考[Spark安装和使用](http://dblab.xmu.edu.cn/blog/1307-2/)
 - ==WARN util.Utils: Service 'sparkDriver' could not bind on port 0. Attempting port 1.==[错误处理](https://stackoverflow.com/questions/44914144/error-sparkcontext-error-initializing-sparkcontext-java-net-bindexception-can?newreg=52a67c24b7074b9687a5d830fb1b82a5)
 - 上述错误和SPARK_LOCAL_IP的设置有关
### 在Spark Shell 中运行代码
 - 在Spark shell中，可以和分布存储在多台机器上的内存或者硬盘上的数据进行交互，并且控制过程由Spark自动完成控制完成，不需要用户参与
 - Spark-shell 命令
	1. spark-shell --master \<master-url>
	2. Spark 运行模式取决于传递给SparkContxt的\<master-url>的值
		* local 使用一个Worker线程本地化运行SPARK(完全不并行)  
		* local[*] 使用逻辑CPU个数数量的线程来本地化运行Spark  
		* local[K] 使用K个Worker线程本地化运行Spark（理想情况下，K应该根据运行机器的CPU核数设定）  
		* spark://HOST:PORT 连接到指定的Spark standalone master。默认端口是7077.  
		* yarn-client 以客户端模式连接YARN集群。集群的位置可以在HADOOP_CONF_DIR 环境变量中找到。  
		* yarn-cluster 以集群模式连接YARN集群。集群的位置可以在HADOOP_CONF_DIR 环境变量中找到。
		* master：这个参数表示当前的Spark Shell要连接到哪个master，如果是local[*]，就是使用本地模式启动spark-shell，其中，中括号内的星号表示需要使用几个CPU核心(core)；  
		* jars： 这个参数用于把相关的JAR包添加到CLASSPATH中；如果有多个jar包，可以使用逗号分隔符连接它们；
	3. spark-shell --help 获得完整的选项列表
### 开发Spark独立应用程序
1. 开发Spark独立应用程序的步骤
	- 安装编译打包程序（首次需要这样做）
		-  sbt
		- maven
	- 编写Spark 应用程序代码
	- 编译打包
	- 通过spark-submit运行程序
	- sbt和maven打包形成的jar包采取同样方式提交运行
		```
		spark-submit
		 -- class <main class>#需要运行的程序的主类，应用程序的入口点
		 --deploy-mode<deploy-mode> #部署模式
		 --master<master-url>#与之前提到的含义相同
		 ...#其他参数
		 <application-jar> #应用程序jar包
		 [application-arguments] #传递给主类的主方法的参数 
		```
	- Spark在Idea开发实例参考[idea+maven/sbt开发spark程序](http://dblab.xmu.edu.cn/blog/1327/)
	- 在idea导出jar包。选择Build->Build Artifacts…，在弹出的窗口选择Bulid就可以了
### Spark集群的搭建
参考[Spark集群搭建](http://dblab.xmu.edu.cn/blog/1187-2/)
### 在集群上运行Spark应用程序
参考[在集群上运行Spark应用程序](http://dblab.xmu.edu.cn/blog/1217-2/)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzNTg2MjE1OTUsMTg5MDg2NTgzLDE5Nj
gxMDk1MjcsLTE0NDE4OTQyLC0xNDQzNzYzNTY2LDE0NTUxNTE2
NzMsLTEzNjM0MTc5MTYsLTE0NzM3NjA2ODIsLTIwNDk1MDY2MT
csOTA3MjgzNjk4LDkyNDc1MTE2LC0xNTA3MDQxMTk3LDE5NTEy
NDI3NTNdfQ==
-->
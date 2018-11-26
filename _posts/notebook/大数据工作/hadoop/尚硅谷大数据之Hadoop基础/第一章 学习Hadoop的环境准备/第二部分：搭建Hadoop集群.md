# 搭建Hadoop环境
## 搭建Hadoop伪分布式环境
1. 在/opt 目录下创建文件夹 
	1. 在/opt 目录下创建 module、software 文件夹 [atguigu@hadoop101 opt]
		1. $ sudo mkdir module [atguigu@hadoop101 opt]$ sudo mkdir software 
	2.  修改 module、software 文件夹的所有者 [atguigu@hadoop101 opt]
		2. $ sudo chown atguigu:atguigu module/ software/ 
2. 安装JDK 
	1. 查询是否安装java软件
	2. 如果安装版本低于Hadoop所需版本，卸载该jdk
		1. 查询是否安装rpm -qa |grep java
		2. 卸载 sudo rpm -e 软件包
	3. 解压jdk到/opt/module下面
		1. tar -zxvf jdk-8u144-linux-x64.tar.gz -C /opt/module/ 
	4. 配置jdk的环境变量
		1. 配置/etc/profile并让其生效
			1. #JAVA_HOME
			2. export JAVA_HOME=/opt/module/jdk1.8.0_144 
			3. export PATH=$PATH:$JAVA_HOME/bin
		 2. 如果 java -version 不生效 
			 1. sync 把buffer里面写入磁盘
			 2. reboot
3.  安装Hadoop
	1. tar -zxvf hadoop-2.7.2.tar.gz -C /opt/module/
	2. sudo vi /etc/profile   
	3. 在 profie 文件末尾添加 jdk 路径：（shitf+g）
	4. ##HADOOP_HOME 
		-  export HADOOP_HOME=/opt/module/hadoop-2.7.2
		-  export PATH=\$PATH:\$HADOOP_HOME/bin 
		-  export PATH=\$PATH:\$HADOOP_HOME/sbin 
	7. 如果 hadoop version 不生效 
		1. sync 把buffer里面写入磁盘
		2. reboot
4. 配置伪分布式集群
	1. 配置hadoop-env.sh
		2. 修改JAVA_HOME 路径
			1. export JAVA_HOME=/opt/module/jdk1.8.0_144 
	2. 配置 core-site.xml
		```
		<!-- 指定HDFS 中 NameNode 的地址 --> 
		<property> 
			<name>fs.defaultFS</name>
			<value>hdfs://hadoop101:9000</value>
		</property>
		<!-- 指定 hadoop 运行时产生文件的存储目录 --> 
		<property> 
			<name>hadoop.tmp.dir</name>
			<value>/opt/module/hadoop-2.7.2/data/tmp</value> 		
		</property>
		```
	3. 配置hdfs-site.xml
		```
		<!-- 指定HDFS 副本的数量 --> 
		<property>
			 <name>dfs.replication</name> 
			 <value>1</value>  
		 </property> 
		```
5. 启动集群
	1. 格式化NameNode
		- hdfs namenode -format 
	2. 启动NameNode
		- hadoop-daemon.sh start namenode
	3. 启动DataNode
		- hadoop-daemon.sh start namenode 
### YARN上运行MapReduce任务
1. 配置YARN
	1. 配置yarn-site.xml
	```
		<!-- reducer 获取数据的方式 --> 
		<property>
			<name>yarn.nodemanager.aux-services</name>
			<value>mapreduce_shuffle</value> 
		</property> 
	 
		<!-- 指定YARN 的 ResourceManager 的地址 --> 
		<property> 
			<name>yarn.resourcemanager.hostname</name>
			<value>hadoop101</value> 
		</property> 
	```
	2. 配置 mapred-env.sh
		-  配置一下JAVA_HOME 
	3. 配置： (对 mapred-site.xml.template 重新命名为) mapred-site.xml 
	```
		<!-- 指定 mr 运行在 yarn 上 -->
		<property>
			<name>mapreduce.framework.name</name>
			<value>yarn</value>
		</property>
	```
	4. 启动集群
		1.   启动resourcemanager 
		2.   启动nodemanager
2. 配置历史服务器
	1.  配置 mapred-site.xml
	```
		<property>
			<name>mapreduce.jobhistory.address</name>
			<value>hadoop101:10020</value>
		</property>
		<property>
			<name>mapreduce.jobhistory.webapp.address</name>
			<value>hadoop101:19888</value>
		</property>	
	```
	2. mr-jobhistory-daemon.sh start historyserver 启动历史服务器 
	3. 查看 jobhistory http://192.168.1.101:19888/jobhistory
3. 配置日志的聚集，
	1. 日志聚集概念：应用运行完成以后，将日志信息上传到 HDFS 系统上
	2. 配置 yarn-site.xml
	```
		<!--开启日志聚集功能-->
		<property>
			<name>yarn.log-aggregation-enable</name>
			<value>true</value>
		</property>
		<!--设置日志保留的时间为7天-->
		<property>
			<name>yarn.log-aggregation.retain-seconds</name>
			<value>604800</value>
		</property>	
	```
## 完全分布式运行模式
1. 集群规划
![ioXSVx.png](https://s1.ax1x.com/2018/11/06/ioXSVx.png)
2. 配置集群
	1. 核心配置文件 
		-  core-site.xml 
	```
		<!--指定NameNode的地址-->
		<property>
			<name>fs.defaultFS</name>
			<value>hdfs://hadoop102:9000</value>
		</property>
		<!--指定 hadoop 运行时产生文件的存储目录-->
		<property>
			<name>hadoop.tmp.dir</name>
			<value>>/opt/module/hadoop-2.7.2/data/tmp</value>
		</property>	
	```
	2.  hdfs 配置文件
		-  hadoop-env.sh
		```
		export JAVA_HOME=/opt/module/jdk1.8.0_144
		```
		- hdfs-site.xml
		```
		<!--指定NameNode的地址-->
		<property>
			<name>dfs.replication</name>
			<value>3</value>
		</property>
		<!--指定 hadoop 运行时产生文件的存储目录-->
		<property>
			<name>dfs.namenode.secondary.http-address</name>
			<value>hadoop104:50090</value>
		</property>
		``
	3. yarn 配置文件
			-  yarn-env.sh
		```
		export JAVA_HOME=/opt/module/jdk1.8.0_144
		```
		- yarn-site.xm
		```
		<!--reducer获得数据的方式-->
		<property>
			<name>yarn.nodemanager.aux-services</name>
			<value>mapreduce_shuffle</value>
		</property>
		<!--指定 YARN 的 ResourceManager 的地址-->
		<property>
			<name>yarn.resourcemanager.hostname</name>
			<value>hadoop103</value>
		</property>
		```
	4. mapreduce 配置文件
			- mapred-env.sh
		```
		export JAVA_HOME=/opt/module/jdk1.8.0_144
		```
		- mapred-site.xml
		```
		<!--指定mr运行在yarn上-->
		<property>
			<name>mapreduce.framework.name</name>
			<value>yarn</value>
		</property>
		```
3. 在集群上分发配置好的 Hadoop 配置文件
	- xsync /opt/module/hadoop-2.7.2/
4.  测试
	1.  配置 slaves 
	2. /opt/module/hadoop-2.7.2/etc/hadoop/slaves 
	3. $ vi slaves
	 ```
	 hadoop102
	 hadoop103
	 hadoop104
	  ```
5. 启动集群
	1. 各个服务组件逐一启动/停止 
		- 分别启动/停止 hdfs 组件 
		- hadoop-daemon.sh start|stop namenode|datanode|secondarynamenode
		- 启动/停止
		-  yarn yarn-daemon.sh start|stop resourcemanager|nodemanager
	2. 各个模块分开启动/停止（配置 ssh 是前提）常用 
		- 整体启动/停止 hdfs start-dfs.sh stop-dfs.sh 
		- 整体启动/停止 yarn start-yarn.sh stop-yarn.sh
		- **注意** 
		- NameNode 和 ResourceManger 如果不是同一台机器，不能在 NameNode 上 启动 yarn，应该在 ResouceManager 所在的机器上启动 yarn。
	3. 全部启动/停止集群（不建议使用） 
		- start-all.sh
		- stop-all.sh
## 有关Hadoop其他笔记
1. Hadoop 运行模式
	1. 本地模式、伪分布式模式、完全分布式模式
2. [hadoop 50070 无法访问问题解决汇总](https://www.cnblogs.com/zlslch/p/6604189.html)
3.  **Hadoop无法正常启动的解决方法**

	1. 一般可以查看启动日志来排查原因，注意几点：

		-   启动时会提示形如 “DBLab-XMU: starting namenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-namenode-DBLab-XMU.out”，其中 DBLab-XMU 对应你的机器名，但其实启动日志信息是记录在 /usr/local/hadoop/logs/hadoop-hadoop-namenode-DBLab-XMU.log 中，所以应该查看这个后缀为 **.log** 的文件；
		-   每一次的启动日志都是追加在日志文件之后，所以得拉到最后面看，对比下记录的时间就知道了。
		-   一般出错的提示在最后面，通常是写着 Fatal、Error、Warning 或者 Java Exception 的地方。
		-   可以在网上搜索一下出错信息，看能否找到一些相关的解决方法。
	2. 此外，**若是 DataNode 没有启动**，可尝试如下的方法（注意这会删除 HDFS 中原有的所有数据，如果原有的数据很重要请不要这样做）：
		```
		针对 DataNode 没法启动的解决方法
		./sbin/stop-dfs.sh # 关闭
		rm -r ./tmp # 删除 tmp 文件，注意这会删除 HDFS 中原有的所有数据
		./bin/hdfs namenode -format # 重新格式化 NameNode
		./sbin/start-dfs.sh # 重启
		```
4. 单节点Hadoop集群的启动和关闭
	1. 关闭 nodemanager 、resourcemanager 和 historymanager
		 - sbin/yarn-daemon.sh stop resourcemanager 
		 - sbin/yarn-daemon.sh stop nodemanager
		 - sbin/mr-jobhistory-daemon.sh stop historyserver 
	2. 启动 nodemanager 、resourcemanager 和 historymanager
		-  sbin/yarn-daemon.sh start resourcemanager
		-  sbin/yarn-daemon.sh start nodemanager 
		- sbin/mr-jobhistory-daemon.sh start historyserver 
5. **配置文件说明**
	1.  Hadoop 配置文件分两类：默认配置文件和自定义配置文件，只有用户想修改某一默认 配置值时，才需要修改自定义配置文件，更改相应属性值。
		- 默认配置文件：存放在 hadoop 相应的 jar 包中
		-  **[core-default.xml]** hadoop-common-2.7.2.jar/ core-default.xml 
		- **[hdfs-default.xml]** hadoop-hdfs-2.7.2.jar/ hdfs-default.xml 
		- **[yarn-default.xml]** hadoop-yarn-common-2.7.2.jar/ yarn-default.xml
		-  **[mapred-default.xml]** hadoop-mapreduce-client-core-2.7.2.jar/ mapred-default.xml
	2. 自定义配置文件：存放在$HADOOP_HOME/etc/hadoop
		-  **core-site.xml** 
		-  **hdfs-site.xml**
		-   **yarn-site.xml** 
		-  **mapred-site.xml**
6. **集群分发脚本**
	1. scp 
		- scp 可以实现服务器与服务器之间的数据拷贝。
	2. rsync
		- rsync 远程同步工具，主要用于备份和镜像。具有速度快、避免复制相同内容和支持符 号链接的优点
		- 用 rsync 做文件的复制要比 scp 的速度快，rsync 只对差异文件做更 新。scp 是把所有文件都复制过去。![ioWaeH.png](https://s1.ax1x.com/2018/11/06/ioWaeH.png)
	3.  编写脚本
		1.  期望脚本： xsync 要同步的文件名称
		2. 在/home/atguigu/bin 这个目录下存放的脚本，atguigu 用户可以在系统任何地方直 接执行。
		3. 集群同步脚本 
		```
		#!/bin/bash #1 获取输入参数个数，如果没有参数，直接退出 
		pcount=$# 
		if((pcount==0)); then
		echo no args; 
		exit;
		fi 
		#2 获取文件名称
		p1=$1 
		fname=`basename $p1` 
		echo fname=$fname
		#3 获取上级目录到绝对路径 
		pdir=`cd -P $(dirname $p1);pwd`
		echo pdir=$pdir
		#4 获取当前用户名称 
		user=`whoami
		#5 循环
		for((host=103; host<105; host++));
			do echo --------------------- hadoop$host ---------------- 
			rsync -rvl $pdir/$fname $user@hadoop$host:$pdir
		done
	```
	4. chmod 777 xsync
	5. 调用脚本形式：，xsync 文件名称
7. **SSH 无密登录配置**
	1. ssh 另一台电脑的 ip 地址
	2. ssh无密码登录原理
![ioLJpQ.png](https://s1.ax1x.com/2018/11/06/ioLJpQ.png)
	3. 生成公钥和私钥：ssh-keygen -t rsa
	4. 将公钥拷贝到要免密登录的目标机器上 ssh-copy-id hadoop103
	5. .ssh 文件夹下（~/.ssh）的文件功能解释
		1. known_hosts ：记录 ssh 访问过计算机的公钥(public key)
		2. id_rsa ：生成的私钥 
		3. id_rsa.pub ：生成的公钥 
		4. authorized_keys ：存放授权过得无密登录服务器公钥
8. **集群时间同步**
	1. 时间同步的方式：找一个机器，作为时间服务器，所有的机器与这台集群时间进行定时 的同步，比如，每隔十分钟，同步一次时间
	2. 配置时间同步实操(必须是时间同步)
		1. 检查ntp是否安装 rpm -qa |grep ntp 
		2.  修改ntp配置文件
			- vi /etc/ntp.conf 修改内容如下 a）
				- 修改 1（设置本地网络上的主机不受限制。）
					-  #restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap 为
					-  restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap b）
				- 修改 2（设置为不采用公共的服务器） 
					- server 0.centos.pool.ntp.org iburst 
					- server  1.centos.pool.ntp.org iburst 
					- server 2.centos.pool.ntp.org iburst 
					- server 3.centos.pool.ntp.org iburst 为
					- #server 0.centos.pool.ntp.org iburst 
					- #server 1.centos.pool.ntp.org iburst
					-  #server 2.centos.pool.ntp.org iburst #server 3.centos.pool.ntp.org iburst 
				- 添加 3（添加默认的一个内部时钟数据，使用它为局域网用户提供服务。）
					- server 127.127.1.0 fudge 127.127.1.0 stratum 10 
		3. 修改/etc/sysconfig/ntpd 文件
			-  vim /etc/sysconfig/ntpd 增加内容如下（让硬件时间与系统时间一起同步）
				-  SYNC_HWCLOCK=yes 
		4. 重新启动 
			-  service ntpd status ntpd 
			-  service ntpd 
			-  chkconfig ntpd on
	3.  其他机器配置（必须 root 用户） 
		1. 在其他机器配置 10 分钟与时间服务器同步一次 
			-  [root@hadoop103 hadoop-2.7.2]# crontab -e 
			-  */10 * * * * /usr/sbin/ntpdate hadoop102 
			- 修改任意机器时间 
				- [root@hadoop103 hadoop]# date -s "2017-9-11 11:11:11" 
			- 十分钟后查看机器是否与时间服务器同步 
				- [root@hadoop103 hadoop]# date
9. 常见Hadoop常见错误
		1. ![iTS2GV.png](https://s1.ax1x.com/2018/11/06/iTS2GV.png)
		2. ![iTSxqH.png](https://s1.ax1x.com/2018/11/06/iTSxqH.png)
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTYyNDc1MTEzLC0xNzAyOTE5NDIxLC0zOD
k5OTkxOTYsMTY3MDMwNTg0MiwtMTQ1MDgxNzQxOSwtMTQ5ODcw
OTk2LC0xNTQ0MTU3NjAwLDY5MjU5MDAyMywxMzcxOTA0MjQ4LC
02MjA0NjI3MzMsLTE3MjQ5NjM5MTAsLTgxNjU0NjkzLC0xNzg0
MjYzNjY5LDQ2MjIzOTQxMSwtODM2MzM5ODU3LC0xNDEzMDM0OD
gxLDEwMTcyNjQzMDksLTMxMzk1NDk4MCwtMjQ2ODgyNTI1LC0x
NDA5MDczMjk5XX0=
-->
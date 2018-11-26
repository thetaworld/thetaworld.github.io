## Maven
### 第一章
1. Maven 的作用
	1.  添加第三方的jar包
		1. 之前的方法造成工具区大量的重复的文件
		2. 使用Maven后，每个jar包只在本地仓库中保存一份，需要jar包的工程维护一个文本的形式的jar包引用，我们称之为“坐标”。
	2. jar 之间的依赖关系
		1. maven 可以替代我们自动将当前jar包依赖的其他所有的jar包全部导进来，无需人工参与。
	3. 解决jar包之间的冲突
		1.   ![jar包冲突举例](https://lh3.googleusercontent.com/yNWmavh3hhI59cLNW1KJknyhTByhM-fJsWilKkZ_qht77O-d-G6HLBKj1ZooBx4Tx75U_CDu1e4v "jar包冲突举例")
		2. Maven 可以自动解决jar冲突的问题，因为Maven中内置两条依赖原则
			1. 最短路径者优先
			2. 先声明者优先
	4. 获取第三方的Jar包
		1. Maven 有一个完全统一规范的jar管理规范。
		2. 只要在项目中以坐标的方式依赖一个jar包，Maven就会自动从仓库中进行下载，并同时下载这个jar包的所依赖的其他jar包
	5.  将项目拆分为多个工程模块
		1.   特别大型的项目不能通过package结构划分模块，必须将项目拆分为多个工程协同开发
		2. 工程拆分后又如何进行互相调用和访问呢？这就需要用到Maven的依赖管理机制
	6. 实现项目的分布式部署
		1. 每个模块需要运行在独立的服务器上，称为分布式部署
		2. 分布式部署需要用到Maven
### 第二章 Maven 是什么
1. Maven 是自动化构建工具，专注服务于Java平台的项目构建和依赖
2. 构建的概念
	1.  构建就是以我们编写的Java代码、框架配置文件、国际化等其他资源文件、JSP页面和图片等静态资源作为“**原材料**”，去**“生产”**出一个可以运行的**项目**的过程
3. 构建的环节
	1. 清理：删除以前的编译结果，为重新编译做好准备。
	2. 编译：将Java源程序编译为字节码文件。
	3. 测试：针对项目中的关键点进行测试，确保项目在迭代开发过程中关键点的正确性。
	4. 报告：在每一次测试后以标准的格式记录和展示测试结果。
	5. 打包：将一个包含诸多文件的工程封装为一个压缩文件用于安装或部署。Java工程对应jar包，Web工程对应war包。
	6. 安装：在Maven环境下特指将打包的结果——jar包或war包安装到本地仓库中。
	7. 部署：将打包的结果部署到远程仓库或将war包部署到服务器上运行。
4. 自动构建
	1. “编译、打包、部署、测试”这些程式化的工作可以交由Maven自动构建
	2. 简单的说来就是它可以自动的从构建过程的起点一直执行到终点：
![Maven图片](https://lh3.googleusercontent.com/p1NEgzigj3xoRajG4fCRRjTmGFK_RrD5FsN1Ccm822x-TnUKkbMU33j33yyfVTojePcsy58IWUgd "Maven图片")
### 第三章 Maven 如何使用
1. 第一个Maven程序的目录结构
	1. 创建约定的目录结构
	2. 创建Maven核心配置文件pom.xml
	3. 编写主代码
	4. 编写测试代码
	5. 运行几个基本的Maven命令
		1. 进入pom.xml文件所在的目录
		2. mvn clean 
		3. mvn compile
		4. mvn test-compile
		5. mvn test
		6. mvn package
		7. mvn install 
2. Maven 联网问题
	1.  Maven的核心程序并不包含具体的功能，仅负责宏观调度。具体功能由插件完成。
	2. Maven 核心程序先从本地仓库查找插件，如果没有就从远程中央仓库下载。如果不能上网下载则无法执行Maven的具体功能。
	3. Maven的核心配置文件：
		1.   apache-maven-3.2.2\conf\settings.xml
		2. \<localRepository>以及准备好的仓库位置\</localRepository>
###  第四章 Maven 核心概念
1. Maven 的核心概念
	1. POM
	2. 约定的目录结构
	3. 坐标
	4. 依赖管理
	5. 仓库管理
	6. 生命周期
	7. 插件与目标
	8. 继承
	9. 聚合
2. POM 
	1. 项目对象模型  
	2. 将java工程信息封装为对象，作为便于操作和管理的模型。
	3. Maven工程的核心配置文件
3. 约定的目录结构
	1. JavaEE开发领域的普遍认同的一个观点是：约定>配置>编码
	2. Maven 正是指定了特定文件保存的目录才能够对我们的Java工程进行自动化构建。
4. 坐标
	1.   使用三维坐标在Maven的仓库中唯一确定一个Maven工程。
	2. 【1】groupId：公司或者组织的域名倒序+当前项目名称
	3. 【2】artifactId：当前项目的模块名称
	4. 【3】version：当前模块的版本
5. 如何通过坐标到仓库中查找jar
	1. 将gav三个维度连接起来
		1.  com.atguigu.maven+Hello+0.0.1-SNAPSHOT
	2.  以连接起来的字符串作为目录的结构在仓库中查找
		2. com/atguigu/maven/Hello/0.0.1-SNAPSHOT/Hello**-**0.0.1-SNAPSHOT.jar
	3. ==注意== 	自己的Maven工程必须执行安装操作才会进入仓库。安装命令是：mvn install
6. 依赖管理
	1. A.jar包需要用到B.jar包中的类，A对B有依赖
	2. 当前工程会到本地仓库中根据坐标查找它所依赖的jar包
	3.   配置的基本形式是使用dependency标签指定目标jar包的坐标
	```
		<dependencies>
			<dependency>
			<!—坐标 -->
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.10</version>
			<!-- 依赖的范围 -->
			<scope>test</scope>
			</dependency>
		</dependencies>
	```
	4. 直接依赖与间接依赖
		1.  如果A依赖B，B依赖C，那么A→B和B→C都是直接依赖
		2. A→C是间接依赖。
	5. 依赖的范围
		1. compile
		```
		[1]main目录下的Java代码**可以**访问这个范围的依赖

		[2]test目录下的Java代码**可以**访问这个范围的依赖

		[3]部署到Tomcat服务器上运行时**要**放在WEB-INF的lib目录下

		例如：对Hello的依赖。主程序、测试程序和服务器运行时都需要用到。
		```
		2. test
		```
		[1]main目录下的Java代码**不能**访问这个范围的依赖

		[2]test目录下的Java代码**可以**访问这个范围的依赖

		[3]部署到Tomcat服务器上运行时**不会**放在WEB-INF的lib目录下

		例如：对junit的依赖。仅仅是测试程序部分需要
		```
		3. provided
		 ```
		[1]main目录下的Java代码**可以**访问这个范围的依赖

		[2]test目录下的Java代码**可以**访问这个范围的依赖

		[3]部署到Tomcat服务器上运行时**不会**放在WEB-INF的lib目录下

		例如：servlet-api在服务器上运行时，Servlet容器会提供相关API，所以部署的时候不需要
		 ```
		 4. 其他：runtime、import、system等
		 5. 各个范围依赖的作用可以概括为下图：
		 ![依赖范围](https://lh3.googleusercontent.com/4RNDk2KzV3giiMPLZgL7vcszFXmAMJvYAm-hkVyh4Rp2qvKZ2uTCkHuTT9P_iQtZBa6s6VaaxUDO "依赖范围")
	6. 依赖的传递性
		1.   当存在间接依赖的情况时，主工程对间接依赖的jar可以访问吗？
		2. 这要看间接依赖的jar引入时的依赖范围--只有依赖范围为compile时，可以访问（对A的可见性）
	7. 依赖的原则：解决jar包冲突
		1. 路径短者优先
		2. 路径相同时先声明者（depency标签配置的先后顺序）优先
	8. 依赖的排除
		1.  有的时候为了确保程序正确可以将有可能重复的间接依赖排除
		2. 请看下面的例子：
		```
		假设当前工程为public，直接依赖environment。
		environment依赖commons-logging的1.1.1对于public来说是间接依赖。
		当前工程public直接依赖commons-logging的1.1.2
		加入exclusions配置后可以在依赖environment的时候排除版本为1.1.1的commons-logging的间接依赖
		<dependency>
			<groupId>com.atguigu.maven</groupId>
			<artifactId>Environment</artifactId>
			<version>0.0.1-SNAPSHOT</version>
			<!-- 依赖排除 -->
			<exclusions>
				<exclusion>
					<groupId>commons-logging</groupId>
					<artifactId>commons-logging</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>commons-logging</groupId>
			<artifactId>commons-logging</artifactId>
			<version>1.1.2</version>
		</dependency>
		``` 
	9. 统一管理目标jar包的版本
		1.  例子
		```
			<properties>

				<spring.version>4.1.1.RELEASE</spring.version>

			</properties>

		……

			<dependency>

				<groupId>org.springframework</groupId>

				<artifactId>spring-core</artifactId>

				<version>${spring.version}</version>

			</dependency>

			<dependency>

				<groupId>org.springframework</groupId>

				<artifactId>spring-context</artifactId>

				<version>${spring.version}</version>

			</dependency>

			<dependency>

				<groupId>org.springframework</groupId>

				<artifactId>spring-jdbc</artifactId>

				<version>${spring.version}</version>

			</dependency>

			<dependency>

				<groupId>org.springframework</groupId>

				<artifactId>spring-orm</artifactId>

				<version>${spring.version}</version>

			</dependency>

			<dependency>

				<groupId>org.springframework</groupId>

				<artifactId>spring-web</artifactId>

				<version>${spring.version}</version>

			</dependency>

			<dependency>

				<groupId>org.springframework</groupId>

				<artifactId>spring-webmvc</artifactId>

				<version>${spring.version}</version>

			</dependency>
		```
	10.  仓库
		1. 本地仓库：为当前本机电脑的所有的Maven工程服务
		2. 远程仓库：
			1. 私服：假设在当前局域网环境下，为当前局域网范围内的所有Maven工程服务。
			2. 中央仓库：假设在Internet上，为全世界的所有Maven项目服务。
			3. 中央仓库的镜像 ：架在各个大洲，为中央仓库分担流量。减轻中央仓库压力，同时更快的响应用户请求。
			4. 如图所示：![Maven 仓库架构](https://lh3.googleusercontent.com/tTI0775gzdZEGAHjwWTXiLJYmZL_qKWDvUSplOnZ2lQX8UxePsgyi_YwBpe1GyxI3L5YvYSv493X "Maven 仓库架构")
		3.  仓库里面的文件
			1. Maven 的插件
			2. 自己开发的项目的模块
			3. 第三方框架或者架构
			4. 不管什么样的jar包，在仓库中都是按照坐标生成的目录结构，所以可以通过统一的方式查询依赖
	11. 生命周期
		1. 什么是Maven的生命周期
			1. Maven 生命周期定义了各个环节的执行的顺序，有了这个清单，Maven就可以自动化执行构建命令
			2.    Maven 有三套相互独立的生命周期
					1. Clean Lifecycle 在进行真正的构建之前进行一些清理工作
					2. Default Lifecycle 构建的核心部分，编译，测试，打包，安装，部署等等
					3. Site Lifecycle 生成项目的报告，站点，发布站点
					4. ==注释== 再次强调一下它们是相互独立的，你可以仅仅调用clean来清理工作目录，仅仅调用site来生成站点。当然你也可以直接运行 **mvn clean install site** 运行所有这三套生命周期。
					5. 每套生命周期都有一组阶段（Phase）组成，我们的命令总是对应于一个特定的阶段。
					6. 三个生命周期：
						1. clean生命周期
							1. Clean生命周期一共包含了三个阶段：
							2. pre-clean 执行一些需要在clean之前完成的工作
							3. clean 移除所有上一次构建生成的文件
							4. post-clean 执行一些需要在clean之后立刻完成的工作
						2. Site生命周期
							1. pre-site 执行一些需要在生成站点文档之前完成的工作
							2. site 生成项目的站点文档
							3. post-site 执行一些需要在生成站点文档之后完成的工作，并且为部署做准备
							4. site-deploy 将生成的站点文档部署到特定的服务器上
							5. 这里经常用到的是site阶段和site-deploy阶段，用以生成和发布Maven站点，这可是Maven相当强大的功能，Manager比较喜欢，文档及统计数据自动生成，很好看。
						3. Default生命周期
							1. Default生命周期是Maven生命周期中最重要的一个，绝大部分工作都发生在这个生命周期中。这里，只解释一些比较重要和常用的阶段：
							2. validate
							3. generate-sources
							4. process-sources
							5. generate-resources
							6. process-resources 复制并处理资源文件，至目标目录，准备打包。
							7. **compile**  编译项目的源代码。
							8. process-classes
							9. generate-test-sources
							10. process-test-sources
							11. generate-test-resources
							12. process-test-resources 复制并处理资源文件，至目标测试目录。
							13. **test-compile**  编译测试源代码。
							14. process-test-classes
							15. test 使用合适的单元测试框架运行测试。这些测试代码不会被打包或部署。
							16. prepare-package
							17. **package**  接受编译好的代码，打包成可发布的格式，如JAR。
							18. pre-integration-test
							19. integration-test
							20. post-integration-test
							21. verify
							22. **install**将包安装至本地仓库，以让其它项目依赖。
							23. deploy将最终的包复制到远程的仓库，以让其它开发人员与项目共享或								部署到服务器上运行。
						4. 生命周期与自动化构建
							1. 运行任何一个阶段时候，它的前面所有的阶段都会被执行。
							2.  我们运行mvn install  的时候，代码会被编译，测试，打包。这就是Maven为什么能够自动执行构建过程的各个环节的原因。
							3. Maven的插件机制是完全依赖Maven的生命周期的，因此理解生命周期至关重要。
7. 插件和目标
	1.  Maven的核心仅仅定义了抽象的生命周期，具体的任务都是交由插件完成的
	2. 每个插件都能实现多个功能，每个功能就是一个插件目标。
	3. Maven的生命周期与插件目标相互绑定，以完成某个具体的构建任务。
	4. 例如：
		1. compile就是插件maven-compiler-plugin的一个功能
		2. pre-clean是插件maven-clean-plugin的一个目标。
### 第六章 Eclipse 创建Maven 项目
1. 修改工程默认的JDK版本
	1.  在Maven核心程序的settings.xml中加入如下配置:
	```
	<profile>
		<id>jdk-1.8</id>
		<activation>
			<activeByDefault>true</activeByDefault
			<jdk>1.8</jdk>
		</activation>
		<properties>
			<maven.compiler.source>1.8</maven.compiler.source>
			<maven.compiler.target>1.8</maven.compiler.target>
			<maven.compiler.compilerVersion>															  
				1.8
			</maven.compiler.compilerVersion>
		</properties>
	</profile>
	```
2. 解决项目测试乱码问题
		1. 在当前项目的pom.xml文件中增加插件配置
	```
	<build>
			<plugins>
			<!-- 解决maven test命令时console出现中文乱码乱码 -->
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-surefire-plugin</artifactId>
					<version>2.7.2</version>
				<configuration>
					<forkMode>once</forkMode><!--在一个进程中进行所有测试 ; 默认值:once -->
					<argLine>-Dfile.encoding=UTF-8</argLine>
				</configuration>
			</plugin>
		</plugins>
	</build>
	```
### 第七章 继承
1. 为什么需要继承
	1. 非compile范围的依赖依赖不能在“依赖链”中传递的，所以有需要的工程需要单独配置
	2. 此时如果项目需要将各个模块的junit版本统一为4.9，那么到各个工程中手动修改无疑是非常不可取的。使用继承机制就可以将这样的依赖信息统一提取到父工程模块中进行统一管理。 
2. 创建父工程
	1.   创建父工程和创建一般的Java工程操作一致
	2.  打包方式处要设置为pom
3.  在子工程中引用父工程
```
<parent>

	<!-- 父工程坐标 -->

	<groupId>...</groupId>

	<artifactId>...</artifactId>

	<version>...</version>

	<relativePath>从当前目录到父项目的pom.xml文件的相对路径</relativePath>

</parent>
```
4. 此时如果子工程的groupId和version如果和父工程重复则可以删除
5. 在父工程中管理依赖
	1. 将Parent项目中的dependencies标签，用dependencyManagement标签括起来
	```
	<dependencyManagement>

		<dependencies>

			<dependency>

				<groupId>junit</groupId>

				<artifactId>junit</artifactId>

				<version>4.9</version>

				<scope>test</scope>

			</dependency>

		</dependencies>

	</dependencyManagement>
	```
	2. 在子项目中重新指定需要的依赖，删除范围和版本号
	```
	<dependencies>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
		</dependency>
	</dependencies>
	```
### 第八章 聚合
1. 为什么要使用聚合
	1. 将多个工程拆分为模块后，需要手动逐个安装到仓库后依赖才能够生效。修改源码后也需要逐个手动进行clean操作。
	2. 而使用了聚合之后就可以==批量进行Maven工程的安装、清理工作。 ==
2. 如何使用聚合？
	1.  在总的聚合工程中使用modules/module标签组合，指定模块工程的相对路径即可
	```
	<modules>
		<module>../Hello</module>
		<module>../HelloFriend</module>
		<module>../MakeFriends</module>
	</modules>
	```
	
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc4MDc2Nzk0MywtNDY2MTE1ODQ3LDE1OT
g4MTQwNzcsMTYwNTUyNDM1Miw0NTEwNjA1MTQsLTcyMjM0NjE2
NCwxMjcxMjcwNjg3LDEwNTA0Mzg3OTMsLTE1Mjk4MzMyNDMsLT
QzODMyNzkwLDMwNTkzMTU4LC0xNTEwMjczOTI5XX0=
-->
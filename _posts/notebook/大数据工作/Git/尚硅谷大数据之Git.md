## Git
### 第一章 Git简介
### 第二章 Git操作
1. 理解Git 仓库的目录结构
	1. 工作区 就是你电脑的本地磁盘目录 
	2. 本地库：工作区有个隐藏的目录.git,它就是Git的本地仓库
	3. 暂存区：一般存在在“.git目录”下index文件中，所以我们把暂存区有时候也就是叫做索引（index）
![enter image description here](https://lh3.googleusercontent.com/EqsVKuylWmy_5GM0leOOROPF2nUmnI-4JrpZ6P7Ln3n06K9XenUXsFQT4adgQa6Qi0hvP5_u5qCm "Git目录")
2.  创建本地版本仓库 git init
3. 提交文件
	1. 输入命令：git add 文件名，将文件添加到暂存区
	2. 删除有两个命令git rm  - - cached <文件名> 、rm <文件名> 前者是删除暂存区的文件，后者和git没什么关系，就相当于linux命令
	3. 通过git status 进行查看工作目录（暂存区）状态
	4. 输入命令：git commit 提交文件到本地库
	5. 编写注释  ，完成提交
	6. 也可以git commit  –m “注释内容”, 直接带注释提交 
4. 回退、版本穿越、还原文件、删除某个文件
	1.  git log 文件名 查看仓库历史记录
	2. git log --pretty=online 查看简易信息
	3. git reset --hard HEAD^
	4. git reset --hard HEAD~n 回退n次操作
	5. git reset 文件名，撤销文件缓存区的状态
	6. git reflog 文件名 查看历史记录的版本号
	7. git reset --hard 版本号
	8. 删除项目文件夹的文件(未更新到本地仓库)
		1. git checkout 文件名
		2. 可以从本地仓库还原文件
	9. 删除某个文件
		1.  删除工作目录下的某个文件
		2. 输入命令：git  add 文件名 （这个add不是增加，而是把上面的操作添加进git）
		3. 输入命令：git commit, 真正地删除仓库中的文件
		4. 注意了，所谓的删除只是这一次操作的版本号没有了，其他的都可以恢复
	10. 分支
		1. Git采用分支的方式管理多个版本
		2.  输入命令：git branch  <分支名>
		3. 输入命令：git branch –v，查看分支
		4. 输入命令：git checkout  <分支名>
		5. 输入命令：git checkout  –b  <分支名>，将创建分支，切换分支一起完成
		6. 输入命令：git checkout  master，切换到主干
		7. 输入命令：git merge  <分支名>，合并分支
		8. 删除一个分支：
		9.	Git branch –d <分支名>
	11. 冲突
		1. 冲突一般指同一个文件同一位置的代码，在两种版本合并时版本管理软件无法判断到底应该保留哪个版本，因此会提示该文件发生冲突，需要程序员来手工判断解决冲突。
合并时冲突
		2. 程序合并时发生冲突系统会提示CONFLICT关键字，命令行后缀会进入MERGING状态，表示此时是解决冲突的状态。
		3. 解决冲突
			1. 此时通过git diff 可以找到发生冲突的文件及冲突的内容。
			2. 然后修改冲突文件的内容，再次git add \<file\> 和git commit 提交后，后缀MERGING消失，说明冲突解决完成。
### 第三章 GitHub
1. 示意图 ![github 示意图](https://lh3.googleusercontent.com/QpOn3ssP3lJq3rkIfVorF1ezRLE-x1EFlP7hf9Q8_moWfo6W7MBBONTPH17XPMBhQmD4nIt8q5M7 "github 示意图")
2. 增加远程地址
	1. git remote add  <远端代号>  <远端地址> 。
	2. <远端代号> 是指远程链接的代号，一般直接用origin作代号，也可以自定义。
	3. <远端地址> 默认远程链接的url
	4. •例： git  remote  add  origin  https://github.com/xxxxxx.git
3. 推送到远程库
	1. git  push  <远端代号>  <本地分支名称>。
	2. <远端代号> 是指远程链接的代号。
	3. • <分支名称> 是指要提交的分支名字，比如master。
	4. •例： git  push  origin  master
4. 从GitHub上克隆（复制）一个项目
	1. git  clone <远端地址>  <新项目目录名>。
	2.  <远端地址> 是指远程链接的地址。
	3. <项目目录名> 是指为克隆的项目在本地新建的目录名称，可以不填，默认是GitHub的项目名。
	4. 命令执行完后，会自动为这个远端地址建一个名为origin的代号。
	5. 例 git  clone  https://github.com/xxxxxxx.git 文件夹名 
5. 从GitHub更新项目
	1. git  pull <远端代号>  <远端分支名>。
	2.  <远端代号> 是指远程链接的代号。
	3. <远端分支名>是指远端的分支名称，如master。
	4. 例  git pull origin  master
6. 协作冲突
	1. 在上传或同步代码时，由于你和他人都改了同一文件的同一位置的代码，版本管理软件无法判断究竟以谁为准，就会报告冲突,需要程序员手工解决。
	2. 解决冲突三板斧：
		1. 修改合并
		2. git add
		3. git commit
7. github fork
![Github fork](https://lh3.googleusercontent.com/7I7nUacnCcq4GpJqEeVo2uCn8-i21FC1KnNWdkUHETYAwtntQMfCllTLOq5Zo9oTPk7gzG87E1c5 "Github fork")
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA5MjI2MDg2Myw5NDMyODg3NzksNjA5MD
A2OTI1LDE1OTY1MjgzNjMsLTQwNDQyMjA2MywtOTUzOTcyOTk5
LC0xNTM2Mjk5MTAzLC0yMDg4NzQ2NjEyXX0=
-->
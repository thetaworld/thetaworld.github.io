## Redis
### 第一章 NoSQL数据库简介
1. 待写
### 第二章Redis 介绍及安装
1. Redis 介绍
	1. Redis是一个开源的key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）。
	2. 这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。
	3. 在此基础上，Redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是Redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件.
	4. 在此基础上实现了master-slave(主从)同步。
2. Redis 应用场景
	1.  ![Redis应用场景1](https://lh3.googleusercontent.com/RYJjv3Cb5NLyZJf7Kipod1OBiE0A8HyHMZRY89q9KKW76oNYM8B4tFjwzFTuMaU--lZoqLPeu90X "Redis应用场景1")
	2. ![Redis应用场景1](https://lh3.googleusercontent.com/kve7lOtfK59pVY29uvRtMVZKB0Z7SMPMpECKstvwiIbNqB-AnDgbidX20L6INZr5Xv9dUl3PE2G5 "Redis应用场景2")
	3. Redis 安装
		1. 参考网络即可
	4. Redis 相关知识.
		1.   默认16个数据库，类似数组下标从0开始，初始默认使用0号库
		2. 使用命令  select  <dbid> 来切换数据库。如: select 8
		3. 统一密码管理，所有库都是同样密码，要么都OK要么一个也连接不上
		4. Redis是单线程+多路IO复用技术
			1.  多路复用是指使用一个线程来检查多个文件描述符（Socket）的就绪状态，比如调用select和poll函数，传入多个文件描述符，如果有一个文件描述符就绪，则返回，否则阻塞直到超时。得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行（比如使用线程池）。
			2. 串行 vs 多线程+锁（memcached） vs 单线程+多路IO复用(Redis)
### 第三章 Redis 五大数据类型
1. 这里可以参考Redis内部存储原理，了解Redis的实现原理
2. 本章内容组织如下：
	1. key
	2. string
	3.  set
	4. list
	5. hash
	6. zset
3. key 相关的操作
	1. keys  * 查询当前库的所有键
	2. exist  \<key\>  判断某个键是否存在
	3. type  \<key \>查看key==所对对象==的类型
	4. del \<key\> 删除某个键
	5.  expire \<key\> 为键值设置过期时间，单位秒
	6. ttl \<key\> 查看还有多少秒过期，-1 表示永不过期 ，-2表示已过期
	7. dbsize 查看当前数据库的key的数量
	8. flushdb 清空当前库
	9. flushall 通杀所有库
4.  String  
	1. String类型是二进制安全的。意味着Redis的string可以包含任何数据。比如jpg图片或者序列化的对象
	2.  String类型是Redis最基本的数据类型，一个Redis中字符串value最多可以是512M
	3. get  \<key\> 查询对应键值
	4. set \<key\>  \<value\> 添加键值对
	5. append  \<key\> \< value\> 将给定的\<value\>追加到原值的末尾
	6. strlen \<key\> 获得值的长度
	7. setnx \<key\> \<value\> 只有在key不存在的时候设置key的值
	8. value的值是数字值时
		1.  incr key 将value值加1
		2. decr key 将value值减1
		3. incrby/decrby  \<key\> \<step\> 将key中存储的数字增减。自定义步长。
		4. 原子性：所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）。
			1.   在单线程中， 能够在单条指令中完成的操作都可以认为是" 原子操作"，因为中断只能发生于指令之间。
			2. 在多线程中，不能被其它进程（线程）打断的操作就叫原子操作
			3. Redis单命令的原子性主要得益于Redis的单线程
	9. 同时操作多个key
		1. mset \<key1\>  \<value1\>  \<key2\>  \<value2\>  .....
		2. mget \<key1\>  \<key2\>  \<key3\> .....
		3.  msetnx \<key1\>  \<value1\>  \<key2\>  \<value2\>  
	10. getrange \<key\>  <起始位置>  <结束位置>
	11. setrange \<key\>  <起始位置>  \<value\>
	12. setex \<key\>  <过期时间>  \<value\>
	13. getset \<key\>  \<value\>
5.  List
	1. 单键多值
	2. Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。
	3. 它的底层实际是个双向链表，对两端的操作性能很高，通过索引下标的操作中间的节点性能会较差。 
	4. lpush/rpush \<key\>  \<value1\>  \<value2\>  \<value3\> 
	5. rpoplpush \<key1\>  \<key2\>
	6. lrange \<key\> \<start\> \<stop\>
	7. lindex \<key\> \<index\>
	8. llen \<key\>
	9. linsert \<key\>  before \<value\>  \<newvalue\>
	10. lrem \<key\> \<n\>  \<value\>
6.  Set
	1.  Redis set对外提供的功能与list类似是一个列表的功能，特殊之处在于set是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个==成员==是否在一个set集合内的重要接口，这个也是list所不能提供的
	2. Redis的Set是string类型的无序集合。它底层其实是一个value为null的hash表,所以添加，删除，查找的复杂度都是O(1)。
	3. 常用命令：
	```
	sadd <key>  <value1>  <value2> .....
	smembers <key>
	sismember <key>  <value>
	scard  <key>
	srem <key> <value1> <value2> ....
	spop <key>
	srandmember <key> <n>
	sinter <key1> <key2>
	sunion <key1> <key2>
	sdiff <key1> <key2>
	
	```
7.  Hash
	1. Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象
	2.  类似Java里面的Map<String,Object>
	3.  常用命令
	```
	hget <key1>  <field>
	hset <key>  <field>  <value>
	hmset <key1>  <field1> <value1> <field2> <value2>...
	hexists key  <field>
	hkeys <key>
	hvals <key>
	hincrby <key> <field>  <increment>
	hsetnx <key>  <field> <value>
	```
8. zset (sorted  set )
	1.  Redis有序集合zset与普通集合set非常相似，是一个没有重复元素的字符串集合。不同之处是有序集合的没有成员都关联了一个评分（score） ，这个评分（score）被用来按照从最低分到最高分的方式排序集合中的成员。集合的成员是唯一的，但是评分可以是重复了  。
	2. 因为元素是有序的, 所以你也可以很快的根据评分（score）或者次序（position）来获取一个范围的元素。访问有序集合的中间元素也是非常快的,因此你能够使用有序集合作为一个没有重复成员的智能列表。
	3. 常用命令
	```
	zadd <key> <score1> <value1>  <score2> <value2>...
	zrange <key>  <start> <stop>  [WITHSCORES]
	zrangebyscore key min max [withscores] [limit offset count]
	zrangebyscore key min max [withscores] [limit offset count]
	zincrby <key> <increment> <value>
	zrem <key>  <value>
	zcount <key>  <min>  <max>
	zrank <key>  <value>
	```
### 第四章 Redis的相关配置
1. 大小写不敏感
2. include 多实例的情况可以把公用的配置文件提取出来
3. ip地址绑定
	1.  默认情况bind=127.0.0.1只能接受本机的访问请求
	2. 不写的情况下，无限制接受任何ip地址的访问
	3. 生产环境肯定要写你应用服务器的地址
	4. 如果开启了protected-mode，那么在没有设定bind ip且没有设密码的情况下，Redis只允许接受本机的响应
4. tcp-backlog
	1.  可以理解是一个请求到达后至到接受进程处理前的队列
	2. backlog队列总和=未完成三次握手队列 + 已经完成三次握手队列
	3. 高并发环境tcp-backlog 设置值跟超时时限内的Redis吞吐量决定
5. timeout  一个空闲的客户端维持多少秒会关闭，0为永不关闭。
6. TCP keepalive 对访问客户端的一种心跳检测，每个n秒检测一次。官方推荐设为60秒。
7. daemonize 是否为后台进程
8. pidfile  存放pid文件的位置，每个实例会产生一个不同的pid文件
9. log level 四个级别根据使用阶段来选择，生产环境选择notice 或者warning
10. logfile  日志文件名称
11. syslog 是否将Redis日志输送到linux系统日志服务中
12. syslog-ident 日志的标志
13. syslog-facility 输出日志的设备
14. database 设定库的数量 默认16
15. security 在命令行中设置密码
16. maxclient 最大客户端连接数
17. maxmemory 
	1.  设置Redis可以使用的内存量。一旦到达内存使用上限，Redis将会试图移除内部数据，移除规则可以通过maxmemory-policy来指定。如果Redis无法根据移除规则来移除内存中的数据，或者设置了“不允许移除”
	2. •那么Redis则会针对那些需要申请内存的指令返回错误信息，比如SET、LPUSH等。
	3. ØMaxmemory-policy
		1.  volatile-lru：使用LRU算法移除key，只对设置了过期时间的键
		2. allkeys-lru：使用LRU算法移除key
		3. volatile-random：在过期集合中移除随机的key，只对设置了过期时间的键
		4. allkeys-random：移除随机的key
		5. volatile-ttl：移除那些TTL值最小的key，即那些最近要过期的key
		6. noeviction：不进行移除。针对写操作，只是返回错误信息
		7. Maxmemory-samples
			1. 设置样本数量，LRU算法和最小TTL算法都并非是精确的算法，而是估算值，所以你可以设置样本的大小
			2. 一般设置3到7的数字，数值越小样本越不准确，但是性能消耗也越小。
### 第七章 JRedis与实战
1. 参考Jedis 的java API
2. 实现手机验证码功能
### 第八章 Redis 事务
1. Redis事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
2. Redis事务的主要作用就是串联多个命令防止别的命令插队
3. 从输入Multi命令开始，输入的命令都会依次进入命令队列中，但不会执行，至到输入Exec后，Redis会将之前的命令队列中的命令依次执行。
4. 组队的过程中可以通过discard来放弃组队。
5. ![enter image description here](https://lh3.googleusercontent.com/y2lDzvWqg__bUeUxAzx2S0WP18w62VCWOY7vo-hoVhr0tMKq4WiVe75VmhezEffhxBoiGgthd06b "redis 事务")
6. 事务的错误处理
	1. 组队中某个命令出现了报告错误，执行时整个的所有队列会都会被取消。
![redis事务处理](https://lh3.googleusercontent.com/XMg7O3KxFhi-j_sQLb9gwL2sRtdrYZUXPOWJPnilsyI7aExnqXYnDgi9uFUmZn5cmt7xZSKzL-jv "redis事务处理")
	2. 如果执行阶段某个命令报出了错误，则只有报错的命令不会被执行，而其他的命令都会执行，不会回滚。
![iDjtcd.png](https://s1.ax1x.com/2018/10/23/iDjtcd.png)
	3. 事务冲突问题
		1.  redis 乐观锁
		2. 乐观锁适用于多读的应用类型，这样可以提高吞吐量。Redis就是利用这种check-and-set机制实现事务的。
		3. 基于版本的乐观锁
		4. 在执行multi之前，先执行watch key1 [key2],可以监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断
		5. unwatch
			1. 取消 [WATCH]  命令对所有 key 的监视。
			2. 如果在执行 [WATCH]命令之后， [EXEC] 命令或 [DISCARD] 命令先被执行了的话，那么就不需要再执行 [UNWATCH] 了
	4. Redis 事务的特性
		1.  单独的隔离操作。事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断
		2. 没有隔离级别的概念。队列中的命令没有提交之前都不会实际的被执行，因为事务提交前任何指令都不会被实际执行，也就不存在“事务内的查询要看到事务里的更新，在事务外查询不能看到”这个让人万分头痛的问题
		3. 不保证原子性。Redis同一个事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚
	5. Redis 秒杀案例
		1.  

### 第九章 Redis 持久化
1. Redis 提供了2个不同形式的持久化方式
	1. RDB
	2. AOF
2.  RDB
	1. 在指定的时间间隔内将内存中的数据集快照写入磁盘，也就是行话讲的Snapshot快照，它恢复时是将快照文件直接读到内存里。
	2.  Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。
	3. 在Linux程序中，fork()会产生一个和父进程完全相同的子进程，但子进程在此后多会exec系统调用，出于效率考虑，Linux中引入了“写时复制技术”，一般情况父进程和子进程会共用同一段物理内存，只有进程空间的各段的内容要发生变化时，才会将父进程的内容复制一份给子进程
	4. rdb的保存的文件 
		1. 在redis.conf中配置文件名称， dbfilename默认为dump.rdb
		2. rdb文件的保存路径，也可以修改。默认为Redis启动时命令行所在的目录下
	5. rdb的保存策略
		1. 自动保存
		2. 手动保存
			1. save   只管保存，其它不管，全部阻塞
			2. save vs bgsave
				1. stop-writes-on-bgsave-error yes
				2.  rdbcompression yes
				3. rdbchecksum yes
			3. rdb 备份
				1.  先通过config get dir  查询rdb文件的目录
				2. 将*.rdb的文件拷贝到别的地方
			4. rdb 恢复
				1.  关闭Redis
				2. 先把备份的文件拷贝到工作目录下
				3. 启动Redis, 备份数据会直接加载
			5. rdb 优点
				1. 节省磁盘空间
				2. 恢复速度快 
			6. rdb 缺点
				1. 虽然Redis在fork时使用了写时拷贝技术,但是如果数据庞大时还是比较消耗性能
				2. 在备份周期在一定间隔时间做一次备份，所以如果Redis意外down掉的话，就会丢失最后一次快照后的所有修改。
3. AOF
	1. 以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来(读操作不记录)，只许追加文件但不可以改写文件，Redis启动之初会读取该文件重新构建数据，换言之，Redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作。
	2. AOF的备份机制和性能虽然和RDB不同, 但是备份和恢复的操作同RDB一样，都是拷贝备份文件，需要恢复时再拷贝到Redis工作目录下，启动系统即加载
	3. AOF和RDB同时开启，系统默认取AOF的数据
	4. AOF文件故障恢复
		1. AOF文件的保存路径，同RDB的路径一致
		2. 如遇到AOF文件损坏，可通过redis-check-aof --fix appendonly.aof  进行恢复
	5. AOF同步频率设置
		1.  始终同步，每次Redis的写入都会立刻记入日志
		2. 每秒同步，每秒记入日志一次，如果宕机，本秒的数据可能丢失。
		3. 把不主动进行同步，把同步时机交给操作系统。
	6. 重写
		1.  AOF采用文件追加方式，文件会越来越大为避免出现此种情况，新增了重写机制,当AOF文件的大小超过所设定的阈值时，Redis就会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集.可以使用命令bgrewriteaof。
		2. AOF文件持续增长而过大时，会fork出一条新进程来将文件重写(也是先写临时文件最后再rename)，遍历新进程的内存中数据，每条记录有一条的Set语句。重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这点和快照有点类似
		3. 重写虽然可以节约大量磁盘空间，减少恢复时间。但是每次重写还是有一定的负担的，因此设定Redis要满足一定条件才会进行重写。
		4. 系统载入时或者上次重写完毕时，Redis会记录此时AOF大小，设为base_size,如果Redis的AOF当前大小>= base_size +base_size*100% (默认)且当前大小>=64mb(默认)的情况下，Redis会对AOF进行重写。
	7. AOF 优点
		1.  备份机制更稳健，丢失数据概率更低。
		2. 可读的日志文本，通过操作AOF稳健，可以处理误操作
	8. AOF 缺点
		1. 比起RDB占用更多的磁盘空间。
		2. 恢复备份速度要慢
		3. 每次读写都同步的话，有一定的性能压力
		4. 存在个别Bug，造成恢复不能。
	9. 如何使用
		1.  官方推荐两个都启用
		2. 如果对数据不敏感，可以选单独用RDB
		3. 不建议单独用 AOF，因为可能会出现Bug
		4. 如果只是做纯内存缓存，可以都不用
### 第十章 Redis 主从复制
1. 主从复制，就是主机数据更新后根据配置和策略，自动同步到备机的master/slaver机制，Master以写为主，Slave以读为主
2. 用处
	1. 读写分离，性能扩展
	2. 容灾快速恢复
### 第十一章 Redis的集群
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjEwOTk0MjU2NywtODE4NjI2NDg0LC05NT
EzNDE2NywtMTkxODYxNTM0NiwtODY1Njg0ODc3LC0xNDczMjIy
MjA5LC0xNTQxNTE1MTA5LDY5Nzc0OTk5MiwxODQxNTE4MzI0LC
0xNTc0OTY1MTIsNjk3NzQ5OTkyXX0=
-->
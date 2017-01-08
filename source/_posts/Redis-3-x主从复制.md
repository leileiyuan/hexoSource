---
title: Redis 3.x主从复制
date: 2017-01-08 18:06:06
tags: 
 - redis
categories: redis
---
Redsi3.x以后，提供了真正的集群，而不是像2.x只能使用分片的集群了(分片式集群是有问题的，但也有一定的应用)。windows下目前没的提供3.0的版本的redis，所以我们在linux下安装redis，做集群的配置，需要对linux有所熟悉。

**安装和配置**
创建安装目录，下载redis安装文件，解压缩，编译，安装

	[root@localhost ~]# mkdir /usr/local/src/redis
	[root@localhost ~]# cd /usr/local/src/redis/
	[root@localhost redis]# wget http://download.redis.io/releases/redis-3.2.6.tar.gz
	[root@localhost redis]# tar -xvf redis-3.2.6.tar.gz 
	[root@localhost redis]# cd redis-3.2.6
	[root@localhost redis]# make
	[root@localhost redis]# make install

到此安装完毕。
redis的启动，停止

	redis-server		// 启动redis服务
	redis-cli			// 连接erdis服务
	
	redis-cli shutdown       // 关闭服务


Redis的主从复制（读写分离）
> 避免单点故障
> 满足读多写少的应用场景。对缓存而言，读多写少，在Redis中优其重要。

我们使用Master/Slave的结构：安装一个redis，启动3个实例(6379、6380、6381)
![](/img/redis/redis-one.png)

在`/usr/local/src/redis/redis-3.2.6`目录下创建一个`redis`目录，这个`redis`目录下创建3个目录，一个作为master(6379)，两个作为slave(6380、6381)。

	[root@localhost redis]# pwd
	/usr/local/src/redis/redis-3.2.6/redis

创建`master-6379`目录，复制 `redis.conf`到该目录下

	[root@localhost redis]# mkdir 6379
	[root@localhost redis]# cd 6379/
	[root@localhost 6379]# cp  /usr/local/src/redis/redis-3.2.6/redis.conf  ./
	
	[root@localhost 6379]# ll
	总计 48
	-rw-r--r-- 1 root root 46695 01-08 15:30 redis.conf

编辑`redis.conf`文件

	daemonize yes						# 设置为后台运行
	pidfile /var/run/redis_6379.pid		# 记录redis运行的进程号，每个端口对应的文件是不一样的
	maxmemory 200m						# redis最大内存

保存退出，后台启动redis，指定redis.conf文件

	[root@localhost 6379]# redis-server ./redis.conf 
	[root@localhost 6379]# 
	
	[root@localhost 6379]# redis-cli 
	127.0.0.1:6379> ping
	PONG
	127.0.0.1:6379> 


创建`6380(slave)`和`6381(slave)`。直接复制`6379(master)`，修改配置文件，将6379全部替换成6380

	[root@localhost redis]# ll
	总计 4
	drwxr-xr-x 2 root root 4096 01-08 15:46 6379
	[root@localhost redis]# cp 6379/ 6380 -R
	[root@localhost redis]# cp 6379/ 6381 -R
	[root@localhost redis]# ll
	总计 12
	drwxr-xr-x 2 root root 4096 01-08 15:46 6379
	drwxr-xr-x 2 root root 4096 01-08 15:47 6380
	drwxr-xr-x 2 root root 4096 01-08 15:48 6381

	[root@localhost 6380]# vim redis.conf 
	:%s/6379/6380/g			# 将6379全部替换成6380
	
6381的redis.conf同样也替换成6381

启动3个redis实例：

	[root@localhost redis]# redis-server ./6379/redis.conf 
	[root@localhost redis]# redis-server ./6380/redis.conf 
	[root@localhost redis]# redis-server ./6381/redis.conf 
	[root@localhost redis]# ps -ef | grep redis
	root     12583     1  0 16:04 ?        00:00:00 redis-server 127.0.0.1:6379   
	root     12587     1  0 16:04 ?        00:00:00 redis-server 127.0.0.1:6380   
	root     12591     1  0 16:04 ?        00:00:00 redis-server 127.0.0.1:6381   
	root     12601 12413  0 16:07 pts/3    00:00:00 grep redis

**设置主从**
两种方式可以设置主从
1. 在redis.conf中设置slaveof
slaveof masterip  masterport
2.	使用redis-cli客户端连接到redis服务，执行slaveof命令
slaveof   masterip  masterport
第二种方式重启服务以后，将失去主从复制关系。

分别设置6380、6381两个Slave开始主从模式

	slaveof 127.0.0.1 6379

查看主从关系：

	[root@localhost redis]# redis-cli
	127.0.0.1:6379> info replication
	# Replication
	role:master
	connected_slaves:2
	slave0:ip=127.0.0.1,port=6380,state=online,offset=71,lag=1
	slave1:ip=127.0.0.1,port=6381,state=online,offset=71,lag=1
	master_repl_offset:71
	repl_backlog_active:1
	repl_backlog_size:1048576
	repl_backlog_first_byte_offset:2
	repl_backlog_histlen:70
	127.0.0.1:6379> 

当前角色是master(role:master)，有两个从的连接(connected_slaves:2)，分别是`slave0:ip=127.0.0.1,port=6380,state=online,offset=71,lag=1`和`slave1:ip=127.0.0.1,port=6381,state=online,offset=71,lag=1`

从`6380(Slave)`查看主从关系

	[root@localhost redis]# redis-cli -p 6380
	127.0.0.1:6380> info replication
	# Replication
	role:slave
	master_host:127.0.0.1
	master_port:6379
	master_link_status:up
	master_last_io_seconds_ago:4
	master_sync_in_progress:0
	slave_repl_offset:43
	slave_priority:100
	slave_read_only:1
	connected_slaves:0
	master_repl_offset:0
	repl_backlog_active:0
	repl_backlog_size:1048576
	repl_backlog_first_byte_offset:0
	repl_backlog_histlen:0
	127.0.0.1:6380> 

**测试：**
主库/写库(master)中写入数据，查看从库/读库（Slave）中的数据情况

主库：

	[root@localhost redis]# redis-cli -p 6379
	127.0.0.1:6379> set name leilei
	OK


从库:

	[root@localhost redis]# redis-cli -p 6380
	127.0.0.1:6380> keys *
	(empty list or set)
	127.0.0.1:6380> keys *
	1) "name"

从库是不能写入数据的

	127.0.0.1:6380> set test aaa
	(error) READONLY You can't write against a read only slave.

> 还有一种结构是主-从-从，从第二个从库开始，作为下级的主库（一般也是只读的）
> ![](/img/redis/redis-two.png)

**主从复制过程原理：**
1.  当从库与主库建立Master/Slave关系后，会向主库发送sync命令；
2. 主库接收到sync命令以后，会在后台保存快照（RDB持久化方式），并将期间接收到的命令缓存起来；
3. 当快照完成后，主库会将快照文件和所有缓存的`写命令`发送给从库；
4. 从库接收到后，载入快照文件并且执行接收到的缓存命令，保持数据一致；
5. 之后，主库每当接收到`写命令`时就会发送给从库，保存数据一致。 


**无磁盘复制**
redis从2.8.18实现了无磁盘复制功能（实验阶段），当磁盘出现IO性能问题时，主库在与从库复制数据时，不会将快照存储磁盘上，而是直接通过网络发送给从库。
通过配置开启：

	repl-diskless-sync yes

**复制架构中出现宕机情况**
如果在主从复制架构中出现宕机的情况，需要分情况看：
1. 从库宕机
a)	这个相对而言比较简单，在Redis中从库重新启动后会自动加入到主从架构中，自动完成同步数据；
b)	问题？ 如果从库在断开期间，主库的变化不大，从库再次启动后，主库依然会将所有的数据做RDB操作吗？还是增量更新？（从库有做持久化的前提下）
不会将所有数据做RDB的，因为在Redis2.8版本后就实现了，主从断线后恢复的情况下实现增量复制。
2. 主库宕机
a)	这个相对而言就会复杂一些，需要以下2步才能完成
第一步，在从数据库中执行SLAVEOF NO ONE命令，断开主从关系并且提升为主库继续服务；
第二步，将主库重新启动后，执行SLAVEOF命令，将其设置为其他库的从库，这时数据就能更新回来；
b)	这个手动完成恢复的过程其实是比较麻烦的并且容易出错，有没有好办法解决呢？当前有的，Redis提供的`哨兵（sentinel）`的功能。

**哨兵（sentinel）**
什么是哨兵
顾名思义，哨兵的作用就是对Redis的系统的运行情况的监控，它是一个独立进程。它的功能有2个：

1. 监控主库和从库是否运行正常；
2. 数据出现故障后自动将从库转化为主库；

**原理**
单个哨兵的架构：
![](/img/redis/redis-three.png)

多个哨兵的架构：
![](/img/redis/redis-four.png)
多个哨兵，不仅同时监控主从库，而且哨兵之间互为监控。

**配置哨兵**
启动哨兵进程首先需要创建哨兵配置文件：

	vim sentinel.conf

输入内容：

	sentinel monitor taotaoMaster 127.0.0.1 6379 1

说明：
`taotaoMaster`：监控主库的名称，自定义即可，可以使用大小写字母和“.-_”符号
`127.0.0.1`：监控的主库的IP
`6379`：监控的主库的端口
`1`：最低通过票数

启动哨兵进程：

	redis-sentinel ./sentinel.conf
![](/img/redis/redis-five.png)

由上图可以看到：
1、	哨兵已经启动，它的id为9059917216012421e8e89a4aa02f15b75346d2b7
2、	为master库添加了一个监控
3、	发现了2个slave（由此可以看出，哨兵无需配置slave，只需要指定master，哨兵会自动发现slave）

**从库宕机：**
kill掉从库进程后，30秒后哨兵的控制台输出：

2989:X 05 Jun 20:09:33.509 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379

说明已经监控到slave宕机了，那么，如果我们将3380端口的redis实例启动后，会自动加入到主从复制吗？
 
2989:X 05 Jun 20:13:22.716 * +reboot slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:13:22.788 # -sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379

可以看出，slave从新加入到了主从复制中。-sdown：说明是恢复服务。
![](/img/redis/redis-six.png)

**主库宕机**
哨兵控制台打印出如下信息：
2989:X 05 Jun 20:16:50.300 # +sdown master taotaoMaster 127.0.0.1 6379  `说明master服务已经宕机`
2989:X 05 Jun 20:16:50.300 # +odown master taotaoMaster 127.0.0.1 6379 #quorum 1/1  
2989:X 05 Jun 20:16:50.300 # +new-epoch 1
2989:X 05 Jun 20:16:50.300 # +try-failover master taotaoMaster 127.0.0.1 6379  `开始恢复故障`
2989:X 05 Jun 20:16:50.304 # +vote-for-leader 9059917216012421e8e89a4aa02f15b75346d2b7 1  `投票选举哨兵leader，现在就一个哨兵所以leader就自己`
2989:X 05 Jun 20:16:50.304 # +elected-leader master taotaoMaster 127.0.0.1 6379  选中leader
2989:X 05 Jun 20:16:50.304 # +failover-state-select-slave master taotaoMaster 127.0.0.1 6379 选中其中的一个slave当做master
2989:X 05 Jun 20:16:50.357 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  `选中6381`
2989:X 05 Jun 20:16:50.357 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  `发送slaveof no one命令`
2989:X 05 Jun 20:16:50.420 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379   `等待升级master`
2989:X 05 Jun 20:16:50.515 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379 ` 升级6381为master`
2989:X 05 Jun 20:16:50.515 # +failover-state-reconf-slaves master taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:50.566 * +slave-reconf-sent slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:51.333 * +slave-reconf-inprog slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:52.382 * +slave-reconf-done slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:52.438 # +failover-end master taotaoMaster 127.0.0.1 6379` 故障恢复完成`
2989:X 05 Jun 20:16:52.438 # +switch-master taotaoMaster 127.0.0.1 6379 127.0.0.1 6381  `主数据库从6379转变为6381`
2989:X 05 Jun 20:16:52.438 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6381 ` 添加6380为6381的从库`
2989:X 05 Jun 20:16:52.438 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  `添加6379为6381的从库`
2989:X 05 Jun 20:17:22.463 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381 `发现6379已经宕机，等待6379的恢复`

![](/img/redis/redis-seven.png)

可以看出，目前，6381位master，拥有一个slave为6380.

接下来，我们恢复6379查看状态：
2989:X 05 Jun 20:35:32.172 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  `6379已经恢复服务`
2989:X 05 Jun 20:35:42.137 * +convert-to-slave slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  `将6379设置为6381的slave`

![](/img/redis/redis-eight.png)

**配置多个哨兵**

	vim sentinel.conf

输入内容：

	sentinel monitor taotaoMaster 127.0.0.1 6381 2
	sentinel monitor taotaoMaster2 127.0.0.1 6381 1

3451:X 05 Jun 21:05:56.083 # +sdown master taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:56.083 # +odown master taotaoMaster2 127.0.0.1 6381 #quorum 1/1
3451:X 05 Jun 21:05:56.083 # +new-epoch 1
3451:X 05 Jun 21:05:56.083 # +try-failover master taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:56.086 # +vote-for-leader 3f020a35c9878a12d2b44904f570dc0d4015c2ba 1
3451:X 05 Jun 21:05:56.086 # +elected-leader master taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:56.086 # +failover-state-select-slave master taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:56.087 # +sdown master taotaoMaster 127.0.0.1 6381
3451:X 05 Jun 21:05:56.189 # +selected-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:56.189 * +failover-state-send-slaveof-noone slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:56.252 * +failover-state-wait-promotion slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:57.145 # +promoted-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:57.145 # +failover-state-reconf-slaves master taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:57.234 * +slave-reconf-sent slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:58.149 * +slave-reconf-inprog slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:58.149 * +slave-reconf-done slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:58.203 # +failover-end master taotaoMaster2 127.0.0.1 6381
3451:X 05 Jun 21:05:58.203 # +switch-master taotaoMaster2 127.0.0.1 6381 127.0.0.1 6380
3451:X 05 Jun 21:05:58.203 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster2 127.0.0.1 6380
3451:X 05 Jun 21:05:58.203 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster2 127.0.0.1 6380

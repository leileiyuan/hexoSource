---
title: mysql主从复制（读写分离）
date: 2017-01-09 09:55:50
tags: mysql
categories: mysql
---
mysql的主从复制或叫读写分离，适用于对mysql操作时读多写少的场景。读库（从库）只负责查询，写库（主库）负责数据的变更（新增、修改、删除），主库数据发生变化，同步到从库中去；其中还有个问题，同步的时候是有延迟的，从读库中查询到的数据，可能并不是最新的，但要保证数据的最终一致性。

<!-- more -->

> 读库和写库数据，最终一致性
> 写数据（新增、修改、删除）只能在写库中完成
> 读数据只能在读库中完成

**安装两个mysql**
安装两个mysql，配置不同的端口。使用绿色版的，只需要做一些配置即可安装好
绿色版mysql解压缩出来以后，编辑`你的mysql目录/data/my.ini`文件

	# pipe
	socket=0.0
	port=3380
	
	# The TCP/IP Port the MySQL Server will listen on
	port=3380
	
	# Path to the database root
	datadir=D:/mysql/3380/data/Data
	
	# General and Slow logging.
	log-output=FILE
	general-log=0
	general_log_file="WIN-A1JA6NUDTV4.log"
	slow-query-log=1
	# 慢查询日志文件，mysql执行的SQL超过了指定的时间（执行太慢了），记录在这里，方便优化
	slow_query_log_file="D:/mysql/3380/logs/mysql-slow.log"
	long_query_time=10
	
	# 二进制文件，记录mysql的操作
	# Binary Logging.
	log-bin="D:/mysql/3380/logs/mysql-bin"
	
	# 错误日志 方便查找问题
	# Error Logging.
	log-error="D:/mysql/3380/logs/mysql.err.log"

	# mysql实例的ID，不重复即可
	# Server Id.
	server-id=380


添加 mysql系统服务，在mysql的bin目录`D:\mysql\3380\bin`下执行命令：

	D:\mysql\3380\bin>.\mysqld.exe install MySQL-3380 --defaults-file="D:\mysql\3380\data\my.ini"
	Service successfully installed.

> 删除系统服务
> sc delete MySQl-3380

启动服务：

	D:\mysql\3380\bin>net start MySQL-3380
	MySQL-3380 服务正在启动 .
	MySQL-3380 服务已经启动成功。

安装上以后，默认用户名是`root`，没有密码。
设置或修改密码：

	D:\mysql\3380\bin>mysqladmin -u root -p password
	Enter password: ****
	New password: ****
	Confirm new password: ****

使用客户端测试连接
![](/img/mysql/mysql-three.png)


**安装从库**
1. 复制3380目录; 

		D:\mysql>dir
		2017/01/09  10:52    <DIR>          .
		2017/01/09  10:52    <DIR>          ..
		2015/10/26  17:04    <DIR>          3380
		2017/01/09  10:52    <DIR>          3381
				            0 个文件              0 字节
				            4 个目录 61,139,161,088 可用字节

2. 删除所有的日志文件
删除`3381/logs`目录下所有文件；
删除`3381\data`目录下所有日志文件
![](/img/mysql/mysql-four.png)

3. 修改`my.ini`配置文件的端口和目录。
4. 修改`server-id`，不要与3380的server-id重复

从库安装完毕。此时3380、3381两个不同端口的数据都可以正常连接。

**建立主从关系**
mysql的主从复制原理：
mysql是将主库(master/写库)的变更记录下来，记录到二进制文件中；
从库(slave/读库)读取该文件并执行，即可完成主从复制。
![](/img/mysql/mysql-five.png)

> 主-从存在数据延迟问题，不可避免

主从配置需要注意的地方：数据需要在同一基准下
1. 主库和从库数据库的版本一致
2. 主库和从库数据库数据一致[ 这里就会可以把主的备份在从上还原，也可以直接将主的数据目录拷贝到从的相应数据目录]
3. 主库开启二进制日志,主库和从库的server_id都必须唯一

**主从复制的配置：**
**主库配置**
主库`my.ini`配置文件中：

开启主从复制，主库的配置

	log-bin="D:/mysql/3380/logs/mysql-bin"

指定主库serverid

	server-id=380

指定同步的数据库，如果不指定则同步全部数据库 我们已经创建一个数据叫`taotao`

	binlog-do-db=taotao

执行SQL语句查询状态：

	 show master status;
	
![](/img/mysql/mysql-six.png)

`File:mysql-bin.000007`：二进制文件；
`Positiion:120`：位置，从哪个位置开始作同步；需要记录下Position值，需要在从库中设置同步起始值。
`Binlog_Do_DB:taotao`：需要同步的数据库。

**从库配置**
在`my.ini`修改：
\#指定serverid，只要不重复即可，从库也只有这一个配置，其他都在SQL语句中操作
server-id=381

以下执行SQL：

	CHANGE MASTER TO master_host = '127.0.0.1',
	 master_user = 'root',
	 master_password = 'root',
	 master_port = 3380,
	 master_log_file = 'mysql-bin.000007',
	 master_log_pos = 120;


\#启动slave同步

	START SLAVE;

\#查看同步状态

	SHOW SLAVE STATUS;

![](/img/mysql/mysql-seven.png)
`Slave_IO_Running`和`Slave_SQL_Runnimg`都为`Yes`表示同步设置成功。现在是有问题的

查看从库的错误日志`D:\mysql\3381\logs\mysql.err.log`：

	 [ERROR] Slave I/O: Fatal error: The slave I/O thread stops because master and slave have equal MySQL server UUIDs; these UUIDs must be different for replication to work. Error_code: 1593

Mysql的UUID相同了，因为我们上面是复制过来的。

打开`D:\mysql\3381\data\data\auto.cnf`文件：

	[auto]
	server-uuid=d71ec2ee-65ba-11e5-a4e3-000c29deb96f

随便改下，不相同即可。
重启3381的服务，查看`Slave_IO_Running`和`Slave_SQL_Runnimg`都为`Yes`

测试：
分别打开主库，从库；
修改主库（写库）数据，保存；
查看从库数据是否有同步过去。。。。。。

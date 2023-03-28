<h3>常用命令</h3>
```sql
show master status;
show slave status\G;

stop slave;
start slave;
reset slave;
reset slave all; -- 删掉当前的slave配置

show variables like '%locks%';

show binlog events; -- 默认显示第一个binlog的内容
show binlog events in 'mysql-bin.000001'; -- 指定binlog
```



<h3>MySQL常用的配置文件</h3>
```sql
# /etc/my.cnf for mysql 5.6.x
# add by cd.net on 2022-01-11
# ---------------------------------
[mysqld]

datadir = /home/dbcenter/mysql-server

# 最好加上这个，否则连接会巨慢无比，大量超时待认证的连接
skip_name_resolve
skip_host_cache

max_connections = 500
max_connect_errors = 500

innodb_buffer_pool_size = 32G
# mysql 8.x 以上不能有下面这句
innodb_file_format = Barracuda
innodb_file_per_table = ON

performance_schema = OFF
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

join_buffer_size = 6G
sort_buffer_size = 4G
read_rnd_buffer_size = 1G 

# 允许接收的数据包最大值
max_allowed_packet = 128M

# query slow
long_query_time = 1
slow-query-log = 1
slow-query-log-file = /home/dbcenter/mysql-server/query-slow.log

# replication +++++++++++++++++++++++++++++++++++++
# 不同服务器的server-id一定不能一样
server-id = 1158

# 开启binlog日志
log_bin = mysql-bin
binlog_format = mixed

# 忽略几个mysql自带的DB变动日志
binlog-ignore-db = mysql
binlog-ignore-db = information_schema
binlog-ignore-db = performance_schema
# 同时也不复制这些表
replicate_wild_ignore_table = mysql.%
replicate_wild_ignore_table = information_schema.%
replicate_wild_ignore_table = performance_schema.%

# 这里的参数根据需要调节，控制MySQL高可用安全级别，根据业务特定在性能和数据一致性方面做决策
# InnoDB在复制中最大的持久性和一致性，[每次commit都 write cache & flush disk]
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
# 事务日志文件大小
innodb_log_file_size = 512M

[mysqld_multi]
mysqld     = /usr/bin/mysqld_safe
mysqladmin = /usr/bin/mysqladmin
log        = /var/log/mysqld_multi.log

!includedir /etc/my.cnf.d
```
MySQL主从同步和主主同步的要点就是两台服务器的配置文件差不多一样，差异是server-id不一样。



<h3>下面介绍主要的配置步骤：（假设主服务器是A-IP，从服务器是B-IP）</h3>

第一步：配置主服务器A

```sql
-- 在主A-IP上面给从B-IP加一个访问用户并设置密码
grant replication slave,replication client on *.* to hot_backup@'B-IP' identified by 'you_password';
flush privileges;
注意：

# 如果是 mysql 8.0 以上版本，创建用户和授权必须分开设置。后面从服务器的设置同理。
create user hot@'B-IP' identified by 'xxx';
grant replication slave,replication client on *.* to hot@'B-IP' with grant option;
flush privileges;
```

第二步：配置从服务器B

```sql
-- 在从B-IP也加入主A-IP用的账号(目前这个没有使用，主主模式会用到）
grant replication slave,replication client on *.* to hot_backup@'A-IP' identified by 'you_password';
flush privileges;

-- 在从B-IP上加一个账号，让主A-IP能访问，将数据备份推送到从B-IP服务器。（备份数据的过程还有其他方式）
grant all privileges on *.* to root_dump@'A-IP' identified by 'your_password';
flush privileges;

-- 删除从B上所有的binlog历史日志（为将来主主同步做准备）
reset master;
show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      120 |              | mysql            |
+------------------+----------+--------------+------------------+
```

第三步：主A的当前数据还原到从B
```sql
-- 登录主A做如下的配置，清空当前的binlog
reset master;

-- 锁住主A数据库，不让数据继续写入：
flush tables with read lock;

-- 查看当前主A的日志状态
show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      924 |              | mysql            |
+------------------+----------+--------------+------------------+

-- 一定要记住这里日志的位置，将作为从B同步数据的起点。

-- 接下来将主A当前数据快照，推送到从B（把想要备份的数据库都推送一遍）
mysqldump dbname1 -uroot -plocalpassword --opt | mysql dbname1 -uroot_dump -pyour_password -h B-IP
mysqldump dbname2 -uroot -plocalpassword --opt | mysql dbname2 -uroot_dump -pyour_password -h B-IP
...
-- 可以指定库中单独的一张或多张表名
mysqldump dbname table1 table2 -uroot -ppass --opt | mysql dbname -uroot_dump -ppass -h B-IP


-- 将所有的DB都推送给从B之后，到主A命令行解除写锁：
unlock tables;

```


第四步：从服务器B启动同步

```sql
-- 从B上登录mysql，给从B指定主服务器A
CHANGE MASTER TO MASTER_HOST='A-IP',MASTER_USER='hot_backup',MASTER_PASSWORD='you_password',
	MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=924;

-- 启动同步
start slave;
-- 可以在从服务上查看当前同步状态
show slave status\G 
```

查看从服务器的状态就会看到类似下图的结果
![image](https://user-images.githubusercontent.com/15883558/228136423-6a8dc779-02ab-456a-a4e1-48484c3cf7ea.png)

看到Slave_IO_running 和 Slave_SQL_Running 的值都是 Yes，代表同步一切正常。恭喜MySQL的主从同步就完成了，更多状态的含义可以上网查找资料。


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

家发现没有，虽然我们说的是主从同步，而实际主主同步很容易实现。因为我们在第二步第一条SQL语句已经在从服务器B中加入了主服务器A的访问授权，主服务器A也能抽取得到从服务器B的binlog日志文件，这样任何对从B的修改也可以同步到主A并还原。于是可以有下面第五步。

第五步：主A上启动对B的同步

```sql
-- 从A上登录mysql，给A指定主服务器B
CHANGE MASTER TO MASTER_HOST='B-IP',MASTER_USER='hot_backup',MASTER_PASSWORD='you_password',
	MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=120;
-- 这里的120就是服务器B在reset master;之后，当时binlog日志文件的编号和对应字节位置。

-- 启动同步
start slave;
-- 可以在A上查看当前同步状态
show slave status\G 
```


<h3>问题一：主服务器清除Binlog</h3>

```sql
-- 如果要 reset master，记住如下操作：
a. master 	lock table
b. slave 	finished the binlog replication
c. master 	reset master
d. master 	show master status  | MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=120;
e. slave  	slave stop
f. master 	unlock table
g. slave  	change master to ***
h. slave  	slave start;
```

问题二：同步出现错误

```sql
-- Slave SQL错误 导致同步卡住的一种处理方法
-- 就是从库上跳过1条语句的执行，这个一定要慎重，看这条语句是否可以跳过，否则两边数据库就不一致了
mysql>stop slave;
mysql>set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
mysql>start slave;
mysql>show slave status\G
```

问题三：如果只想同步特定的表

果想在从服务器只同步主服务器中某个特定库的某几张表，可以在从服务器/etc/my.cnf配置文件中加入如下的配置：

```sql
# 忽略要同步的库
replicate-ignore-db=mysql
replicate-ignore-db=performance_schema
replicate-ignore-db=information_schema
replicate-ignore-db=your_ignore_db

# 指定要复制的库和相应的几张表名
replicate-do-db=your_db
replicate-do-table=your_db.table_name1
replicate-do-table=your_db.table_name2
```


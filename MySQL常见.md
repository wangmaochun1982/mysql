表被锁定

-- 查询系统库中什么表被锁定了。

show OPEN TABLES where In_use > 0;

mysqldump
```sql
# 备份单张表，同时启用gzip压缩
mysqldump -uroot -pdb191@b.x fund_trade account_share_logs | gzip > \ 
191_dump_20210918_104401.fund_trade.account_share_logs.sql.gz
```

查询当前连接情况

```sql
# show 所有的连接
show processlist;
# 筛选条件
select * from information_schema.processlist where user = 'root';
```

表空间占用统计

```sql

-- 整个库的所有表全部统计出来了

select table_schema,table_name, table_rows, (data_length + index_length) as len,
concat(round((data_length + index_length)/1024/1024, 2), 'MB') as datas
from information_schema.tables
order by len desc;
```

<h3>MySQL配置参数</h3>

涉及参数 –skip-name-resolve ,–skip-host-cache ,–skip-networking

当新的客户连接mysqld时，mysqld创建一个新的线程来处理请求。该线程先检查是否主机名在主机名缓存中。如果不在，线程试图解析主机名：

如果操作系统支持线程安全gethostbyaddr_r ()和gethostbyname_r()调用，线程使用它们来执行主机名解析。
如果操作系统不支持线程安全调用，线程锁定一个互斥体并调用gethostbyaddr()和gethostbyname()。在这种情况下，在第1个线程解锁互斥体前，没有其它线程可以解析不在主机名缓存中的主机名。
你可以用**–skip-name-resolve**选项启动mysqld来禁用DNS主机名查找。然而，在这种情况下，你只可以使用MySQL中的授权表中的IP号。如果你有一个很慢的DNS和许多主机，你可以通过用–skip-name-resolve禁用DNS查找或增加HOST_CACHE_SIZE定义(默认值：128)并重新编译mysqld来提高性能。

你可以用**–skip-host-cache**选项启动服务器来禁用主机名缓存。要想清除主机名缓存，执行**FLUSH HOSTS**语句或执行**mysqladmin flush-hosts**命令。

如果你想要完全禁止TCP/IP连接，用**–skip-networking**选项启动mysqld。


<h4>Use Database很卡</h4>
```sql
shell > mysql -A -uroot -ppassword
```

<h2>InnoDB性能优化</h2>

transaction_isolation

解读：事务隔离级别，Oracle, SQL Server等商业知名数据库默认级别为READ-COMMITTED，而MySQL默认为REPEATABLE-READ，它利用自身独有的Gap Lock解决了"幻读"。但也因为Gap Lock的缘故，相比于READ-COMMITTED级别的Record Lock，REPEATABLE-READ的事务并发插入性能受到很大的限制。

设置：隔离级别的选择取决于实际的业务需求（安全与性能的权衡），如果不是金融、电信等事务级别要求很高的业务，完全可以设置成transaction_isolation=READ-COMMITTED。

innodb_buffer_pool_size

解读：InnoDB缓冲池大小，它决定了MySQL可以在内存中缓存多少数据和索引，而不是每次都从磁盘上读取。

设置：如果是专用的MySQL服务器，一般设置为操作系统内存的75%左右，但至少保留2G内存用于操作系统维护和MySQL异常事件处理。

innodb_buffer_pool_instances

解读：InnoDB缓冲池实例个数，InnoDB缓冲池是通过一整个链表的方式来管理页面（段、簇、页）的，由于互斥锁的存在（保护链表中的页面），高并发事务下，页面的读取和写入就需要锁的竞争和等待。通过设置innodb_buffer_pool_instances，将一整个链表划分为多个，每个缓冲池实例管理自己的页面和互斥，从而提高效率。

设置：如果缓冲池比较大（8G以上），可以按照innodb_buffer_pool_size / innodb_buffer_pool_instances = 1G进行设置，但如果缓冲池特别大（32G以上），可以按照每个实例2~3G进行划分，实例数不是越多越好，多实例代表多线程，线程的开销（CPU、MEM）也得考虑。

innodb_log_file_size

解读：InnoDB日志文件大小（Redo Log），它将事务对InnoDB表的修改记录保存在ib_logfile0、ib_logfile1中。innodb_log_file_size越大，缓冲池中的脏数据需要检查点（checkpoint）进行刷盘的频率就越少，从而减少磁盘IO来降低高并发负载造成的峰值。但日志文件也不是越大约好，由于内存中脏数据刷盘的频率减少，一旦数据库发生异常崩溃，数据库重启时从innodb_log_file中读取数据进行恢复的时间越长。

设置：一般选取业务高峰期一个小时的日志量作为标准，计算过程如下：
```sql
# 命令
$ mysql -uuser -p -e 'show engine innodb status\G'|grep 'Log sequence number' \
&& sleep 60 \
&& mysql -uuser -p -e 'show engine innodb status\G'|grep 'Log sequence number'

# 输出
Log sequence number 149949388055
Log sequence number 149959622102

# 计算
( 149959622102 - 149949388055 ) / 1024 / 1024 = 10M
10 / 60 * 3600 = 600M
600 / 2 = 300M

# 解释
Log sequence number代表InnoDB运行至今写入日志的总字节数，两次打印之间线程休眠60秒
得到一分钟之内事务日志记录的总量10M，再转换成一个小时的总量600M
因为`ib_logfile0、ib_logfile1`两个文件循环写入，一个文件为300M
最终，innodb_log_file_size=300M
```

innodb_flush_log_at_trx_commit

解读：InnoDB事务日志刷盘时机

当0时，事务提交到日志缓冲区，后台Write线程每隔一秒将缓冲区的日志写入系统缓冲区，实际写入物理日志文件的时机取决于操作系统。
当1时，事务提交到日志缓冲区，Master线程同步将缓冲区的日志直接写入物理日志文件，这完全符合InnoDB ACID事务标准，数据不会丢失。
当2时，事务提交到系统缓冲区，Master线程每隔一秒将系统缓冲区的日志写入物理日志文件。
设置：安全1 > 2 > 0，速度0 > 2 > 1，根据实际业务需求（安全与速度权衡）选择合理的刷盘时机。

sync_binlog

解读：二进制事务日志刷盘时机，需要配合log-bin选项才能记录二进制日志。区别于InnoDB事务日志，二进制事务日志是针对整个MySQL Server的，而InnoDB事务日志只针对InnoDB存储引擎。二进制事务日志的作用一是用于主从复制，二是用于数据恢复。区别于InnoDB事务日志恢复，二进制事务日志是用于误操作的数据恢复，而InnoDB事务日志是用于InnoDB存储引擎的崩溃恢复。

当0时，将由操作系统控制binlog_cache的刷盘时机。

当1时，所有事务开始、提交阶段，都会同步写入磁盘，这是最安全的方式。如果设置innodb_flush_log_at_trx_commit = 1, sync_binlog = 1，这是使用InnoDB事务最安全可靠的方式。

当N时，事务每提交N次，同步写入一次二进制日志。设置：如果MySQL是单机，可以考虑sync_binlog=0；如果是主从，且每秒事务并发量低，考虑sync_binlog=1；事务并发量很高，考虑sync_binlog=N，N的选取可以通过统计业务正常时期的OPS。

![image](https://user-images.githubusercontent.com/15883558/228123294-75a916cb-1fd9-4799-b186-c655b71f0060.png)

innodb_file_per_table

解读：InnoDB独立表空间，innodb_file_per_table = ON表示每张表在独立的物理文件中（.ibd）存储数据和索引，innodb_file_per_table = OFF表示所有表都共享表空间即一个物理文件（ibdata1）。如果通过drop/truncate table操作，独立表空间的物理存储会立即被回收（删除/初始化），而共享表空间不会被回收且只会一直增大。

设置：innodb_file_per_table = ON，但需要注意的是，独立表空间只存储数据和索引，如回滚日志缓冲（Undo Log）、插入索引缓冲（Insert Buffer）、二次写缓冲（Doublewrite Buffer）等还是放在共享表空间。

query_cache_size

解读：查询缓存大小，它是为了在追踪表的数据未发生变化时，本次查询命中之前的查询语句，从而跳过解析、优化、执行阶段，直接返回缓冲池中的数据。但实际在OLTP系统中，极少能命中查询缓存（前提是数据库中的数据变化频率很小），因为一旦数据有变则缓存失效。且因为查询缓存会跟踪所有表的变化，它也会成为整个数据库的瓶颈（资源竞争点）。

设置：query_cache_size = 0，同时配合设置query_cache_type = 0，MySQL5.7.20以上、MySQL8.0会直接弃用所有查询缓存配置项。

max_connections

解读：最大连接数，当max_connections设置太小时（默认151），MySQL可能会报错Too many connections。当max_connections设置太大时（1000以上），操作系统可能忙于线程间的切换而失去响应。

设置：每个连接都会消耗一定内存，计算过程如下：





  

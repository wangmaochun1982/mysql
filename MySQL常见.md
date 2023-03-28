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



  

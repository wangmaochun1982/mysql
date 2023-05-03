1.计算MYSQL在负载高峰时占用的总内存

```sql
select (@@key_buffer_size + @@innodb_buffer_pool_size +@@innodb_log_buffer_size + @@binlog_cache_size + 
@@max_connections*(@@read_buffer_size+@@read_rnd_buffer_size+@@sort_buffer_size+@@join_buffer_size+@@thread_stack+@@tmp_table_size))/(1024*1024*1024) as max_memory_gb;
```
max_connect后面的是单个查询连接的参数，前面的是全局参数

2.INNODB的总数据量（table & index)

```sql
select count(*) as tables,concat(round(sum(table_rows)/10000000,2),'M') numrows,
       concat(round(sum(data_length)/(1024*1024*1024),2),'G') DATA,
	   concat(round(sum(index_length)/(1024*1024*1024),2),'G') IDX,
	   concat(round(sum(data_length+index_length)/(1024*1024*1024),2),'G') total_size
	   from information_schema.tables where engine='InnoDB';
```     
备注：把参数 innodb_buffer_pool_size设置成超过InnoDB的总数据量是没有意义的，通常设置能容纳InnoDB的活跃数据就够了

3.InnoDB缓存池命中率

```sql
show status like 'Innodb_buffer_pool_read%s';

MariaDB [(none)]> show status like 'Innodb_buffer_pool_read%s';
+----------------------------------+---------------+
| Variable_name                    | Value         |
+----------------------------------+---------------+
| Innodb_buffer_pool_read_requests | 2094796734609 |
| Innodb_buffer_pool_reads         | 990980        |
+----------------------------------+---------------+
2 rows in set (0.001 sec)

MariaDB [(none)]> select (2094796734609-990980)/2094796734609*100;
+------------------------------------------+
| (2094796734609-990980)/2094796734609*100 |
+------------------------------------------+
|                                 100.0000 |
+------------------------------------------+
1 row in set (0.000 sec)

```
Innodb_buffer_pool_reads:代表Mysql不能从InnoDB缓存池读到需要的数据而不得不从硬盘中进行读的次数
使用下面的命令查询MYSQL每秒从磁盘读的次数

```sql
$ mysqladmin extended-status -ri1|grep Innodb_buffer_pool_reads
```
把这个值和硬盘的I/O能力进行对比，如果接近了硬盘处理I/O的上限，那么从OS查看到CPU用于等待I/O的时间(I/O wait)
eg:vmstat中的CPU和Iostat 中的%iowait,这时硬盘就是瓶颈


4.innodb_buffer_pool_instances系统参数

一个和相关innodb_buffer_pool_size的参数是innodb_buffer_pool_instance,它设定把innodb缓存池分成几个区

当innodb_buffer_pool_size>1G时，这个参数起作用，可以设大一点，以提高并发度

5.查询参数文件

```sql
select variable_path,variable_source,count(*) from performance_schema.variables_info where length(variable_path)!=0 group by variable_path,variable_source
```






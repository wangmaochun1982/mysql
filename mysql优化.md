1.计算MYSQL在负载高峰时占用的总内存

```sql
select (@@key_buffer_size + @@innodb_buffer_pool_size +@@innodb_log_buffer_size + @@binlog_cache_size + @@max_connections*(@@read_buffer_size+@@read_rnd_buffer_size+@@sort_buffer_size+@@join_buffer_size+@@thread_stack+@@tmp_table_size))/(1024*1024*1024) as max_memory_gb;
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



子查询优化

1.将子查询改变为表连接，尤其是在子查询的结果集较大的情况下；

2.添加复合索引，其中复合索引的包含的字段应该包括 where 字段与关联字段；

3.复合索引中的字段顺序要遵守最左匹配原则；

4.MySQL 8 中自动对子查询进行优化；

现有两个表

```sql
create table Orders
(
    id         integer AUTO_INCREMENT PRIMARY KEY,
    name       varchar(255) not null,
    order_date datetime     NOT NULL
) comment '订单表';

create table OrderDetails
(
    id           integer AUTO_INCREMENT PRIMARY KEY,
    order_id     integer     not null,
    product_code varchar(20) not null,
    quantity     integer     not null
) comment '订单详情表';
```

子查询

```sql
select * from orders where id in (select order_id from OrderDetails where product_code = 'PC50');

```

优化1：改为表连接

```sql
select * from orders as t1 inner join orderdetails o on t1.id = o.order_id where product_code='PC50';
```

优化2：给 order_id 字段添加索引

优化3：给 product_code 字段添加索引

结果证明：给 product_code 字段添加索引 的效果优于给 order_id 字段添加索引，因为不用对索引列进行全表扫描

![image](https://user-images.githubusercontent.com/15883558/234503771-2ac3ca38-352b-4693-9f29-ef9d12622fe0.png)


优化4：给 order_id 和 product_code 添加复合索引

优化5：给 product_code 和 order_id 添加复合索引

![image](https://user-images.githubusercontent.com/15883558/234503868-996a67e4-d6f6-40b6-8acd-b6d3e07629c7.png)


对于复合索引 idx(order_id, product_code)，因为查询中需要判断 product_code 的值是否为 PC51，所以要对 order_id 该列进行全索引扫描，性能较低 
[ 因为 product_code 不是有序的，先根据 order_id 进行排序，再根据 product_code 进行排序 ]；

对于复合索引 idx(product_code, order_id) ，因为 product_code 本身是有序的，所以可以快速定位到该 product_code 然后快速获取该 order_id，性能较高；

 
 <h1>
  
  待排序的分页查询的优化
  
  现有一个电影表
    
  ```sql
  create table film
(
    id           integer auto_increment primary key,
    score        decimal(2, 1) not null,
    release_date date          not null,
    film_name    varchar(255)  not null,
    introduction varchar(255)  not null
) comment '电影表';  
 ```
    
 对于浅分页
    
 select score, release_date, film_name from film order by score limit 0, 20;

耗时 825ms
    
对于深分页
    
select score, release_date, film_name, introduction from film order by score limit 1500000, 20;

耗时 1s 247ms
    
若不加处理，浅分页的速度快，limit 的深度越深，查询的效率越慢

给排序字段添加索引
给 score 字段添加索引

create index idx_score on film(score);
    
结果

浅分页的速度为 60 ms，深分页的速度为 1s 134ms

浅分页的情况得到了优化，而深分页依然很慢

查看深分页的执行情况
![image](https://user-images.githubusercontent.com/15883558/234504977-26dcc2e4-b7d3-43ac-83f7-d3405afdbd36.png)

其并没有走 score 索引，走的是全表的扫描，所以给排序字段添加索引只能优化浅分页的情况
   
解释

只给 score 添加索引，会造成回表的情况

![image](https://user-images.githubusercontent.com/15883558/234505255-6e55956c-6727-4114-b281-a6f4ad15e2c0.png)

    
对于浅分页，回表的性能消耗小于全表扫描，故走 score 索引；

对于深分页，回表的性能消耗大于全表扫描，故走 全表扫描；
    
给排序字段跟 select 字段添加复合索引 
    
给 score, release_date, film_name 添加复合索引
    
create index idx_score_date_name on film(score, release_date, film_name);
    
浅分页的速度为 58 ms，深分页的速度为 357 ms，两者的速度都得到了提升
查看深分页的执行情况

![image](https://user-images.githubusercontent.com/15883558/234505944-d1cd5241-a2e6-49bc-bfbb-a4cf18b8795a.png)
    
![image](https://user-images.githubusercontent.com/15883558/234506036-eb49f26c-48b9-41de-b2aa-3e5ed27c789b.png)

    
对于该复合索引，排序的值和查询的值都在索引上，没有进行回表的操作，效率很高。唯一的不足是：若要添加新的查询列，就要更改该索引的列，不够灵活。
    

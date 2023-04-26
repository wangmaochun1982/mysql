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
  
  



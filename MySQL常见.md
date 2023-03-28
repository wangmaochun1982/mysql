-- 查询系统库中什么表被锁定了。

show OPEN TABLES where In_use > 0;


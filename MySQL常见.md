<h2>表被锁定</h2>

-- 查询系统库中什么表被锁定了。

show OPEN TABLES where In_use > 0;


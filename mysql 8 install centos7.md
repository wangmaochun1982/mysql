一 下载

选择社区版，https://dev.mysql.com/downloads/mysql/ 和对应的操作系统版本。选择下载RPM Bundle，包含所有的软件包。

![image](https://user-images.githubusercontent.com/15883558/228449563-774efce0-cddd-4993-90a2-4f3f59ef5778.png)

二 解压安装

```sql
#解压
tar -xvf mysql-8.0.29-1.el7.x86_64.rpm-bundle.tar 
#安装
for  f in `ls *.rpm`;do rpm -ivh $f;done

#根据提示，可能需要卸载冲突的低版本的mariadb或者是mysql的包，安装缺失的软件包：
rpm -e mariadb-libs-5.5.56-2.el7.x86_64 --nodeps
yum install perl-DUMP*
yum install perl-Data*
yum install perl-Test*
```

三 启动MySQL

```sql
[root@node-2 ~]# systemctl start mysqld
[root@node-2 ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2022-07-14 09:47:24 CST; 4s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 31759 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 31904 (mysqld)
   Status: "Server is operational"
    Tasks: 38
   Memory: 481.6M
   CGroup: /system.slice/mysqld.service
           └─31904 /usr/sbin/mysqld
​
7月 14 09:47:08 node-2 systemd[1]: Starting MySQL Server...
7月 14 09:47:24 node-2 systemd[1]: Started MySQL Server.
[root@node-2 ~]# 
```


四 修改root口令

```sql
[root@node-2 ~]# grep "temporary" /var/log/mysqld.log 
2022-07-14T01:47:18.082124Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: RIskaSByr7_k
[root@node-2 ~]# 

[root@node-2 ~]# mysql -uroot -p 
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.29
​
Copyright (c) 2000, 2022, Oracle and/or its affiliates.
​
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
​
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
​
mysql> alter user 'root'@'localhost' identified by 'Onlyou168';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
mysql> alter user 'root'@'localhost' identified by 'Onlyou_168';
Query OK, 0 rows affected (0.01 sec)
​
mysql> 
```


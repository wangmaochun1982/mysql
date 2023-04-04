生产环境：CentOS Linux release 7.9.2009 (Core)，mysql Ver 14.14 Distrib 5.7.33


包括：MySQL5.7基础安装、修改数据库存储位置、配置允许远程连接访问、创建与删除用户&数据库等。


系统通用配置

```sql
# 禁用防火墙与SELINUX
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 开启防火墙启用MySQL 3306端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# 重载配置
firewall-cmd --reload
# 查看端口号是否开启(返回yes开启)
firewall-cmd --query-port=3306/tcp
# 获取防火墙列表
firewall-cmd --list-all

# 磁盘分区(数据库数据盘)
mkfs.xfs /dev/sdb
mkdir /data
mount /dev/sdb /data

# 设置自动挂载
# 获取磁盘UUID
[root@mysql8 ~]# blkid | grep sdb
/dev/sdb: UUID="dc71382a-b53a-40ff-a0fe-2c0422105ddd" TYPE="xfs"
# 追加至/etc/fstab
echo "UUID=dc71382a-b53a-40ff-a0fe-2c0422105ddd       /data           xfs            defaults        0 0" >> /etc/fstab


```

配置安装源

首先，您需要在系统上启用MySQL 5.7社区版的yum存储库，对应的RPM包可在MySQL官方网站上找到。

```sql
yum localinstall https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
```
安装MySQL5.7

```sql
yum install mysql-community-server
# 启动服务&设置开机自启动
systemctl start mysqld && systemctl enable mysqld && systemctl status mysqld
```

初始化数据库

```sql
# 获取数据库临时密码
grep 'A temporary password' /var/log/mysqld.log | tail -1
# 初始化数据库
mysql_secure_installation

Securing the MySQL server deployment.

Enter password for user root: **********

The 'validate_password' plugin is installed on the server.
The subsequent steps will run with the existing configuration
of the plugin.
Using existing password for root.

Estimated strength of the password: 100
# 修改root管理员密码
Change the password for root ? ((Press y|Y for Yes, any other key for No) : y

New password: ******************

Re-enter new password: ******************

Estimated strength of the password: 100
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : y
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

# 删除匿名用户
Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.


Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

# 禁止root账号远程登录
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
Success.

By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.

# 删除测试库
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
 - Dropping test database...
Success.

 - Removing privileges on test database...
Success.

Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

# 重新加载特权表，生效配置
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
Success.

All done!
```

修改数据库存放位置

```sql
# 创建数据库存放目录
mkdir /data/mysql
# 配置权限
chown -vR mysql:mysql /data/mysql/
chmod -vR 700 /data/mysql/
# 停止数据库服务
systemctl stop mysqld
# 复制原数据库至新目录
cp -av /var/lib/mysql/* /data/mysql/

# 修改/etc/my.cnf配置，如下所示：
[root@localhost ~]# cat /etc/my.cnf | grep -v "^#" | grep -v "^$"
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

[client]
port=3306
socket=/data/mysql/mysql.sock

[mysqld_safe]
socket=/data/mysql/mysql.sock
nice=0

# 启动数据库服务
systemctl start mysqld && systemctl status mysqld

# 确保数据库启动成功以后，我们就可以删除不再需要的原数据库文件了
rm -rf /var/lib/mysql
```

设置允许远程连接

远程连接可以设置root账号，也可以单独创建一个账号，专门用于远程管理，毕竟root账号的权限过大，一旦密码泄漏严重影响数据库安全。

```sql
# 登录MySQL
[root@localhost ~]# mysql -u root -p
# 进入mysql库
mysql> use mysql;
# 更新域属性，'%'表示允许任何主机访问
mysql> update user set host="%" where user ="root";
# 全局特权所有数据库
mysql> grant all privileges on *.* to 'root'@'%' with grant option;
# 生效配置
mysql> FLUSH PRIVILEGES;
```

MySQL基础操作

```sql
# 登录MySQL
[root@localhost ~]# mysql -u root -p

# 创建数据库
mysql> CREATE DATABASE test;

# 创建用户db_test并允许远程连接
mysql> CREATE USER 'db_test'@'%' IDENTIFIED BY 'password';

# 授权允许远程访问test数据库的所有表
mysql> GRANT ALL ON test.* TO 'db_test'@'%';

# 查询所有用户
mysql> SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
+------------------------------------+
| query                              |
+------------------------------------+
| User: 'db_test'@'%';               |
| User: 'mysql.session'@'localhost'; |
| User: 'mysql.sys'@'localhost';     |
| User: 'root'@'localhost';          |
+------------------------------------+
6 rows in set (0.01 sec)

# 查询数据库
mysql> show databases;
+-----------------------+
| Database              |
+-----------------------+
| information_schema    |
| mysql                 |
| performance_schema    |
| sys                   |
| test                  |
+-----------------------+
6 rows in set (0.00 sec)

# 删除数据库
mysql> drop database test;
Query OK, 0 rows affected (0.00 sec)

# 更新指定用户密码
mysql> UPDATE user SET authentication_string=PASSWORD('密码') WHERE User='db_test' and Host='%';
Query OK, 0 rows affected, 1 warning (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 1

# 删除用户
mysql> drop user 'db_test'@'%';
Query OK, 0 rows affected (0.00 sec)

# 生效配置
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

# 退出数据库
mysql> exit
Bye
```


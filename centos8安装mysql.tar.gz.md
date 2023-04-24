一、下载MySql8.0.26

https://dev.mysql.com/downloads/mysql/

![image](https://user-images.githubusercontent.com/15883558/233890705-b84a97a1-5867-4677-89fd-0e84589eb9b8.png)


二、安装MySql8.0.26

添加mysql用户组和用户

```sql
[root@localhost ~]# groupadd mysql
[root@localhost ~]# useradd -r -g mysql -s /bin/false mysql
```


因为用户只需要用于授权而不是登录，所以useradd命令使用 -r和-s /bin/false选项来创建一个对服务器主机没有登录权限的用户。

2、安装依赖包
（1）安装lrzsz工具包，该工具包用于上传文件

```sql
[root@localhost opt]# yum install -y lrzsz
```

（2）安装libaio依赖包  安装libncurses*依赖包

```sql
[root@localhost ~]# yum install libaio
[root@localhost ~]# yum install libncurses*
```

3、上传mysql安装包

使用rz命令将下载的mysql-8.0.26-linux-glibc2.12-x86_64.tar.xz上传到服务器

解压mysql-8.0.26-linux-glibc2.12-x86_64.tar.xz到目录/opt/module下面

```sql
[root@localhost software]# tar -xvf mysql-8.0.26-linux-glibc2.12-x86_64.tar.xz -C /opt/module/
[root@localhost software]# cd /opt/module/
[root@localhost module]# mv mysql-8.0.26-linux-glibc2.12-x86_64 mysql8
进入mysql解压后的目录mysql8目录，并创建存放mysql数据的目录

[root@localhost ~]# cd /opt/module/mysql8/
[root@localhost mysql8]# mkdir data
```

修改创建的data文件夹的权限

```sql
[root@localhost mysql8]# chown mysql:mysql data
[root@localhost mysql8]# chmod 750 data
```

配置mysql环境变量

```sql
[root@localhost ~]# vim /etc/profile
export MYSQL_HOME=/opt/module/mysql8
export PATH=$PATH:$MYSQL_HOME/bin

[root@localhost ~]# source /etc/profile

```

5、初始化mysql

注意：如果需要数据库的表对大小写不敏感，在初始化mysql时务必加上--lower-case-table-names=1这个参数，
因为官网明文提示只有在初始化的时候设置 lower_case_table_names=1才有效。

```sql
[root@localhost ~]# mysqld --user=mysql --basedir=/opt/module/mysql8 --datadir=/opt/module/mysql8/data --initialize --lower-case-table-names=1

```

![image](https://user-images.githubusercontent.com/15883558/233891354-13ac52da-c70e-461c-9a70-7641327ca73a.png)


配置mysql

```sql
[root@localhost ~]# cd /opt/module/mysql8/support-files/
[root@localhost support-files]# vim mysql.server 

basedir=/opt/module/mysql8
datadir=/opt/module/mysql8/data

# Default value, in seconds, afterwhich the script should timeout waiting
# for server start. 
# Value here is overriden by value in my.cnf. 
# 0 means don't wait at all
# Negative numbers mean to wait indefinitely
service_startup_timeout=900

# Lock directory for RedHat / SuSE.
lockdir='/var/lock/subsys'
lock_file_path="$lockdir/mysql"

# The following variables are only set for letting mysql.server find things.

# Set some defaults
mysqld_pid_file_path=/opt/module/mysql8/data/mysqld_pid
if test -z "$basedir"
then
  basedir=/opt/module/mysql8
  bindir=/opt/module/mysql8/bin
  if test -z "$datadir"
  then
    datadir=/opt/module/mysql8/data
  fi
  sbindir=/opt/module/mysql8/bin
  libexecdir=/opt/module/mysql8/bin
else
  bindir="$basedir/bin"
  if test -z "$datadir"
  then
    datadir="$basedir/data"
  fi
  sbindir="$basedir/sbin"
  libexecdir="$basedir/libexec"
fi
修改的值如下图所示：
```
![image](https://user-images.githubusercontent.com/15883558/233891469-0d34baff-4d1d-49d4-aaad-d85338e211a4.png)


设置mysql开机启动

1）将上述修改的配置文件拷贝至/etc/init.d文件夹下并命名为mysqld

```sql
[root@localhost ~]# cp /opt/module/mysql8/support-files/mysql.server /etc/init.d/mysqld
```

2）给新复制的文件授权

```sql
[root@localhost ~]# chmod 755 /etc/init.d/mysqld

```

（3）将mysql服务加到系统服务中

```sql
[root@localhost ~]# chkconfig --add mysqld
[root@localhost ~]# chkconfig mysqld on

```

修改my.cnf文件

[root@localhost ~]# vim /etc/my.cnf

```sql
[client]
port=3306
socket=/tmp/mysql.sock

[mysqld]
basedir=/opt/module/mysql8
datadir=/opt/module/mysql8/data
socket=/tmp/mysql.sock
user=mysql
port=3306
character_set_server=utf8
# symbolic-links=0
# bind-address=0.0.0.0
lower_case_table_names=1

[mysqld_safe]
log-error=/opt/module/mysql8/data/error.log
pid-file=/opt/module/mysql8/data/mysqld.pid
tmpdir=/tmp
```

7、启动mysql

[root@localhost ~]# service mysqld start
[root@localhost ~]# service mysqld status
[root@localhost ~]# service mysqld stop


[root@localhost ~]# mysql -uroot -p

alter user 'root'@'localhost' identified by 'root';
flush privileges;




---
layout: post
title:  "Mariadb Installation & Master-Master Replication"
date:   2018-01-16 08:43:59
author: Lin Shi & Peng Xiao
categories: database
---

本文记录了MYSQL双主备份的配置过程。

## 版本和安装

### 1. 文档使用Mariadb 最新稳定版本 10.2.8

```
[cisco@dev1 ~]$ mysql -V
mysql  Ver 15.1 Distrib 10.2.8-MariaDB, for Linux (x86_64) using  EditLine wrapper
```
​

### 2. 安装教程

https://downloads.mariadb.org/mariadb/repositories/#mirror=neusoft&distro=CentOS&distro_release=centos7-amd64--centos7&version=10.2


## HA配置与部署

以下操作建议在数据库安装完毕、插入数据前进行，以避免出现问题


### 1 编辑配置文件

```
sudo vim /etc/my.cnf.d/server.cnf
```


编辑如下部分，注意2台设备的server-id和自增offset设置为不一致


设备1

```
[mysqld]

log-bin=mysql-bin // Replication要求必须打开 bin log功能 
server-id=101      // 两台设备分别设置不同的 id
    ​

auto-increment-increment=2  //id自增间隔，设置成2 
auto-increment-offset=1         //其中一台设置为1，另外一台设置为2，这主要为了避免2台MySQL的id自增发生重复


replicate-do-db=demo     // 需要同步的数据库，多个数据库写多行
relay-log=mysqld-relay-bin
log-slave-updates 
```


​设备2

```
​[mysqld]

log-bin=mysql-bin // Replication要求必须打开 bin log功能 
​server-id=102      // 两台设备分别设置不同的 id

​auto-increment-increment=2  //id自增间隔，设置成2 
​auto-increment-offset=2         //其中一台设置为1，另外一台设置为2，这主要为了避免2台MySQL的id自增	发生重复

​replicate-do-db=demo     // 需要同步的数据库，多个数据库写多行
​relay-log=mysqld-relay-bin
log-slave-updates 
```

配置完两台server以后，分别启动mysql.

```
systemctl start mariadb.service
```

如果有防火墙，主要把mysql 3306端口配置成允许访问

```
firewall-cmd --zone=public --add-service=mysql --permanent
firewall-cmd --reload
```

### 2 进入设备A 的MySQL CLI 进行本机Master角色配置

mysql shell配置：另外一台机器访问此master的权限，用户和密码以及以及所在的机器地址。

```
$ mysql 
> GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'copyuser'@'172.16.1.%' IDENTIFIED BY 'root@123';
> FLUSH PRIVILEGES;
> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql_bin.000001 |      497 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```

### 3 进入设备B配置slave

```
$ mysql 
> CHANGE MASTER TO MASTER_USER='copyuser',MASTER_PASSWORD='root@123',MASTER_HOST='172.16.1.100',MASTER_PORT=3306,MASTER_LOG_FILE='mysql_bin.000001',MASTER_LOG_POS=497;
```


启动slave

```
> START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

查看slave的状态

```
> show slave status\G; 验证slave状态,注意一下两行是否为Yes


Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

### 4 把2，3的配置过程反过来再做一次


设备B

```
MariaDB [(none)]> GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'copyuser'@'172.16.1.%' IDENTIFIED BY 'root@123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> FLUSH PRIVILEGES
    -> ;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> 
MariaDB [(none)]> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      816 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

MariaDB [(none)]> 
```

设备A


```
$ mysql 
> CHANGE MASTER TO MASTER_USER='copyuser',MASTER_PASSWORD='root@123',MASTER_HOST='172.16.1.200',MASTER_PORT=3306,MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=816;
```


启动slave

```
> START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

查看slave的状态

```
> show slave status\G; 验证slave状态,注意一下两行是否为Yes


Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```


## 验证

先在设备A上create数据库demo

```
MariaDB [(none)]> create database demo;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> 
```

然后到设备B上查看，demo也会自动产生

```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| demo               |
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
5 rows in set (0.00 sec)

MariaDB [(none)]> 
```

在设备A的demo里创建一个table

```
MariaDB [(none)]> use demo;
Database changed
MariaDB [demo]> CREATE TABLE student (id INT UNSIGNED NOT NULL PRIMARY KEY AUTO_INCREMENT,Name CHAR(30));
Query OK, 0 rows affected (0.07 sec)

MariaDB [demo]> show columns from student;
+-------+------------------+------+-----+---------+----------------+
| Field | Type             | Null | Key | Default | Extra          |
+-------+------------------+------+-----+---------+----------------+
| id    | int(10) unsigned | NO   | PRI | NULL    | auto_increment |
| Name  | char(30)         | YES  |     | NULL    |                |
+-------+------------------+------+-----+---------+----------------+
2 rows in set (0.00 sec)

MariaDB [demo]> 
```

在设备B上也能看到这个table

```
MariaDB [(none)]> use demo;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [demo]> show tables;
+----------------+
| Tables_in_demo |
+----------------+
| student            |
+----------------+
1 row in set (0.00 sec)

MariaDB [demo]> 
```

在设备B上插入数据

```
MariaDB [demo]> INSERT INTO student (Name) VALUES ('stu1'),('stu2');
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0

MariaDB [demo]> 
```

在设备A上能看到数据

```
MariaDB [demo]> select * from student;
+----+------+
| id | Name |
+----+------+
|  2 | stu1 |
|  4 | stu2 |
+----+------+
2 rows in set (0.00 sec)

MariaDB [demo]> 
```
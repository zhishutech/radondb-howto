# radon支持语句

[toc]

## 介绍
本文主要是参照官方给出API接口以及RadonDB支持的SQL文档进行测试整理的，使用过程中存在一些注意点体现在具体内容中。
**官方文档：**
* https://github.com/radondb/radon/blob/master/docs/api.md
* https://github.com/radondb/radon/blob/master/docs/radon_sql_support.md
* https://github.com/radondb/radon/blob/master/docs/how_to_import_and_export_data.md
## 用户权限
RadonDB没有自己的用户管理，主要是继承MySQL的用户以及权限
### 查看RadonDB用户

```
# curl http://192.168.1.27:8080/v1/user/userz
```

### 添加用户

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"user": "radon_test", "password": "radon_test"}' http://192.168.1.27:8080/v1/user/add
```

Response:

```
HTTP/1.1 200 OK
Date: Sun, 18 Nov 2018 07:17:28 GMT
Content-Length: 0
```

RadonDB会在每个backend节点上创建一个username@'localhost'的账户：

```
[root@sg-master-18:/root 5.7.23-log_Instance1 root@localhost:radondb 15:19:55]>select user,host from mysql.user where user='radon_test';
+------------+-----------+
| user | host |
+------------+-----------+
| radon_test | localhost |
+------------+-----------+
1 row in set (0.00 sec)
```

通过API接口创建用户只能指定用户名和密码，host无法指定，默认只有localhost。看了下Radon的代码，这个localhost是在代码中写死的。没有理解开发人员提供这个接口的想法。

### 修改密码

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"user": "radon_test", "password": "radon_new"}' http://192.168.1.27:8080/v1/user/update
```

Response:

```
HTTP/1.1 200 OK
Date: Sun, 18 Nov 2018 07:22:35 GMT
Content-Length: 0
```

RadonDB会在每个backend节点上修改username@'localhost'账户的密码。

### 删除用户

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"user": "radon_test"}' http://192.168.1.27:8080/v1/user/remove
```

Response:

```
HTTP/1.1 200 OK
Date: Sun, 18 Nov 2018 07:25:05 GMT
Content-Length: 0
```

RadonDB会在每个backend节点上删除username@'localhost'账户。


## DDL操作
### 创建库 CREATE DATABASE

```
mysql> create database radondb;
Query OK, 2 rows affected (0.00 sec)
```

会在每个backend节点上创建一个指定名称的schema

### 删除库 DROP DATASBASE

```
mysql> DROP DATABASE radondb;
Query OK, 0 rows affected (0.02 sec)
```

会将每个backend节点上指定名称的schema删除

### 创建表  CREATE TABLE
#### 创建全局表
当使用CREATE TABLE语法创建表的时候，如果没有指定分区键，那么RadonDB会认为创建的是一张全局表，会在每个backend节点上创建一张相同名称的表。并且每次对全局表的操作，会在每个节点上都执行。

```
mysql> create table sg_test_global(id int,name varchar(20));
Query OK, 0 rows affected (0.05 sec)

mysql> show tables;
+-------------------+
| Tables_in_radondb |
+-------------------+
| sg_test_global |
+-------------------+
1 row in set (0.00 sec)
```

#### 创建分区表
当使用CREATE TABLE 语法创建表的时候，通过指定PARTITION BY HASH()来表明创建的是一张分区表。RadonDB默认会创建32个分区，将这32个分区平均分布到所有backend节点上。
**注意**
* 创建分区表时，分区表字段无法设置auto_increment属性
* 创建分区表时，datetime字段无法设置default now()，但是可以设置指定默认值
* 创建分区表时，shardkey不能是函数

```
mysql> create table sg_test_partition(id int ,name varchar(20)) partition by hash(id);
Query OK, 0 rows affected (0.43 sec)

mysql> show tables;
+-------------------+
| Tables_in_radondb |
+-------------------+
| sg_test_global |
| sg_test_partition |
+-------------------+
2 rows in set (0.00 sec)
```

在backend1节点上可以看到

```
[root@sg-master-18:/root 5.7.23-log_Instance1 root@localhost:radondb 14:36:15]>show tables;
+------------------------+
| Tables_in_radondb |
+------------------------+
| sg_test_global |
| sg_test_partition_0000 |
| sg_test_partition_0001 |
| sg_test_partition_0002 |
| sg_test_partition_0003 |
| sg_test_partition_0004 |
| sg_test_partition_0005 |
| sg_test_partition_0006 |
| sg_test_partition_0007 |
| sg_test_partition_0008 |
| sg_test_partition_0009 |
| sg_test_partition_0010 |
| sg_test_partition_0011 |
| sg_test_partition_0012 |
| sg_test_partition_0013 |
| sg_test_partition_0014 |
| sg_test_partition_0015 |
+------------------------+
17 rows in set (0.00 sec)

```

在backend2节点上可以看到

```
[root@sg-shadow-19:/root 5.7.23-log_Instance1 root@localhost:radondb 14:36:13]>show tables;
+------------------------+
| Tables_in_radondb |
+------------------------+
| sg_test_global |
| sg_test_partition_0016 |
| sg_test_partition_0017 |
| sg_test_partition_0018 |
| sg_test_partition_0019 |
| sg_test_partition_0020 |
| sg_test_partition_0021 |
| sg_test_partition_0022 |
| sg_test_partition_0023 |
| sg_test_partition_0024 |
| sg_test_partition_0025 |
| sg_test_partition_0026 |
| sg_test_partition_0027 |
| sg_test_partition_0028 |
| sg_test_partition_0029 |
| sg_test_partition_0030 |
| sg_test_partition_0031 |
+------------------------+
17 rows in set (0.00 sec)
```

### 删除表 DROP TABLE

```
mysql> drop table sg_test_global;
Query OK, 0 rows affected (0.03 sec)

mysql> drop table sg_test_partition;
Query OK, 0 rows affected (0.30 sec)
```

### 修改存储引擎 ALTER TABLE ... ENGINE={InnoDB|TokuDB}

```
mysql> show create table sg_test_partition;
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| sg_test_partition | CREATE TABLE `sg_test_partition` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(20) COLLATE utf8_bin DEFAULT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_bin |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> alter table sg_test_partition engine=InnoDB;
Query OK, 0 rows affected (0.61 sec)

mysql> show create table sg_test_partition;
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| sg_test_partition | CREATE TABLE `sg_test_partition` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(20) COLLATE utf8_bin DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```


### 清空表数据 TRUNCATE TABLE 

```
mysql> TRUNCATE TABLE sg_test_partition;
Query OK, 0 rows affected (16.79 sec)
```

RadonDB会将所有的分片表的数据清空（如果是分片表）

### 添加字段 ALTER TABLE table_name ADD COLUMN (col_name column_definition,...)

```
mysql> ALTER TABLE sg_test_partition ADD COLUMN(address varchar(50));
Query OK, 0 rows affected (0.69 sec)
```

RadonDB会将所有的分片表上添加字段（如果是分片表）

### 删除字段 ALTER TABLE table_name DROP COLUMN col_name

```
mysql> ALTER TABLE sg_test_partition DROP COLUMN address;
Query OK, 0 rows affected (0.75 sec)
```

RadobDB会将所有分片表上删除字段（如果是分片表）

### 修改字段 ALTER TABLE table_name MODIFY COLUMN col_name column_definition

```
mysql> ALTER TABLE sg_test_partition MODIFY COLUMN name varchar(50);
Query OK, 0 rows affected (0.21 sec)
```

RadonDB会在所有分片表上修改字段（如果是分片表）
### 添加索引 CREATE INDEX index_name ON table_name (index_col_name,...)

```
mysql> CREATE INDEX name_index ON sg_test_partition(name);
Query OK, 0 rows affected (0.38 sec)
```

RadonDB会在所有的分片表中添加索引（如果是分片表）
### 删除索引 DROP INDEX index_name ON table_name

```
mysql> DROP INDEX name_index ON sg_test_partition;
Query OK, 0 rows affected (0.30 sec)
```

RadonDB会在所有的分片表中删除索引（如果是分片表）

## DML操作
RadonDB支持的SQL语法与MySQL基本一致，不需要单独学习新的语法
* INSERT
 * INSERT 需要显示声明列
* UPDATE
* DELETE
* SELECT
* REPLACE
* SHOW ENGINES
* SHOW DATABASES
* SHOW TABLES
* SHOW COLUMNS
* SHOW CREATE TABLE
* SHOW PROCESSLIST
* SHOW VARIABLES
* USE DATABASE
* KILL 

## 备份与恢复
go-dumper是用Go语言写的多线程备份、恢复工具，兼容MySQL以及mydumper
### 安装go-dumper

```
# git clone https://github.com/XeLabs/go-mydumper
# cd go-mydumper
# make
```

### mydumper导出数据
* 查看导出数据前的数据量

```
mysql> select * from sg_test_partition;
+------+----------+---------------------+
| id | name | gmt_create |
+------+----------+---------------------+
| 2 | lisi | 2019-11-06 01:02:03 |
| 1 | zhangsan | 2018-11-06 01:02:03 |
| 3 | wangwu | 2020-11-06 01:02:03 |
| 4 | zhaoliu | 2021-11-06 01:02:03 |
+------+----------+---------------------+
4 rows in set (0.01 sec)
```

* 使用mydumper导出数据

```

# bin/mydumper -h 192.168.1.27 -P 3308 -u qbench -p qbench -db radondb -o radondb.sql
 2018/11/30 21:57:46.310016 dumper.go:35: [INFO] dumping.database[radondb].schema...
 2018/11/30 21:57:46.312360 dumper.go:45: [INFO] dumping.table[radondb.sg_test_partition].schema...
 2018/11/30 21:57:46.312582 dumper.go:171: [INFO] dumping.table[radondb.sg_test_partition].datas.thread[1]...
 2018/11/30 21:57:46.375319 dumper.go:123: [INFO] dumping.table[radondb.sg_test_partition].done.allrows[4].allbytes[0MB].thread[1]...
 2018/11/30 21:57:46.375368 dumper.go:173: [INFO] dumping.table[radondb.sg_test_partition].datas.thread[1].done...
 2018/11/30 21:57:46.375436 dumper.go:191: [INFO] dumping.all.done.cost[0.07sec].allrows[4].allbytes[137].rate[0.00MB/s]
```

* 查看mydumper导出结果 

```
# ll radondb.sql/
总用量 12
-rw-r--r--. 1 root root 0 11月 30 21:57 metadata
-rw-r--r--. 1 root root 40 11月 30 21:57 radondb-schema-create.sql
-rw-r--r--. 1 root root 210 11月 30 21:57 radondb.sg_test_partition.00001.sql
-rw-r--r--. 1 root root 208 11月 30 21:57 radondb.sg_test_partition-schema.sql
```

* radondb-schema-create.sql 文件中存放的是建库语句
* radondb.sg_test_partition.00001.sql 文件中存放的是INSERT语句，即数据
* radondb.sg_test_partition-schema.sql 文件中存放是建表语句


### myloader导入数据
* 删除原表

```
mysql> drop database radondb;
Query OK, 32 rows affected (0.21 sec)
```

* 修改 radondb.sg_test_partition-schema.sql 文件
 * 修改前

```
CREATE TABLE `sg_test_partition` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(20) COLLATE utf8_bin DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

 * 修改后

**注意**
PARTITION BY HASH(gmt_create) 放到COLLATE部分后面，恢复时会报错

```
CREATE TABLE `sg_test_partition` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(20) COLLATE utf8_bin DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL
) ENGINE=InnoDB PARTITION BY HASH(gmt_create) DEFAULT CHARSET=utf8 COLLATE=utf8_bin ;
```

 * 使用myloader导入数据

```
# bin/myloader -h 192.168.1.27 -u qbench -p qbench -P 3308 -d radondb.sql/
 2018/11/30 22:14:54.502723 loader.go:76: [INFO] restoring.database[radondb]
 2018/11/30 22:14:54.502878 loader.go:87: [INFO] working.table[radondb.sg_test_partition]
 2018/11/30 22:14:54.989947 loader.go:111: [INFO] restoring.schema[radondb.sg_test_partition]
 2018/11/30 22:14:54.990372 loader.go:127: [INFO] restoring.tables[sg_test_partition].parts[00001].thread[2]
 2018/11/30 22:14:55.000834 loader.go:145: [INFO] restoring.tables[sg_test_partition].parts[00001].thread[2].done...
 2018/11/30 22:14:55.000900 loader.go:202: [INFO] restoring.all.done.cost[0.01sec].allbytes[0.00MB].rate[0.00MB/s]
```

* 检查数据

```
mysql> select * from sg_test_partition;
+------+----------+---------------------+
| id | name | gmt_create |
+------+----------+---------------------+
| 2 | lisi | 2019-11-06 01:02:03 |
| 1 | zhangsan | 2018-11-06 01:02:03 |
| 3 | wangwu | 2020-11-06 01:02:03 |
| 4 | zhaoliu | 2021-11-06 01:02:03 |
+------+----------+---------------------+
4 rows in set (0.01 sec)
```
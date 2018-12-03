# radon安装

#### 安装Radon
* 从GitHub上下载Radon源码包到/usr/local目录下

```
# cd /usr/local
# git clone https://github.com/radondb/radon
```

* 编译安装Radon

```
# cd /usr/local/radon
# make build
```

在编译完成之后，在radon目录下会生成一个bin目录

```
# ll /usr/local/radon/bin
-rwxr-xr-x. 1 root root 13939364 11月 16 21:28 radon
-rwxr-xr-x. 1 root root 9929223 11月 16 21:28 radoncli
```
#### 运行Radon
* 准备配置文件
Radon的运行通过配置文件的方式进行管理，在源码包的conf目录下有一个默认的配置文件`radon.default.json`，我们利用这个默认配置文件来启动Radon

```
# cp /usr/local/radon/conf/radon.default.json /usr/local/radon/bin
```

* 启动Radon

```
# /usr/local/radon/bin/radon -c /usr/local/radon/bin/radon.default.json > /tmp/radon.log 2>&1 &
# ps -ef | grep radon
root 17001 16334 0 01:12 pts/2 00:00:00 /usr/local/radon/bin/radon -c /usr/local/radon/bin/radon.default.json
root 17017 16334 0 01:13 pts/2 00:00:00 grep --color=auto radon
# lsof -i :3308
COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
radon 17001 root 8u IPv6 93491 0t0 TCP *:tns-server (LISTEN)
```

#### 向Radon中添加MySQL节点
目前Radon是通过开放API接口的方式进行集群管理，这样方便开发人员进行定制化的开发，所以我们在配置过程中也是通过调用Radon开放的API接口进行MySQL节点的添加。
Radon默认采用8080为管理端口，3308为访问端口
* 通过管理端口开放API添加MySQL节点

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"name": "backend1", "address": "10.10.30.218:3306", "user":"qbench", "password": "xxxxxx", "max-connections":1024}' http://127.0.0.1:8080/v1/radon/backend
```

 `name`表示后端节点的名称，可自定义；`address`表示要添加的MySQL的连接地址以及端口；`user`和`password`表示用于连接MySQL的用户名和密码，`max-connections`表示最大连接数
如果添加节点成功会返回一下内容：

```
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 06:23:49 GMT
Content-Length: 0
```

#### 通过RadonDB访问MySQL
通过Radon默认创建的密码为空的root用户，通过3308端口访问Radon看能否访问成功

```
# /usr/local/mysql/bin/mysql -uroot -h127.0.0.1 -P3308
Welcome to the MySQL monitor. Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.7-Radon-1.0 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> \s
--------------
/usr/local/mysql/bin/mysql Ver 14.14 Distrib 5.7.24, for linux-glibc2.12 (x86_64) using EditLine wrapper

Connection id: 1
Current database:
Current user: qbench@192.168.1.27
SSL: Not in use
Current pager: stdout
Using outfile: ''
Using delimiter: ;
Server version: 5.7-Radon-1.0 MySQL Community Server (GPL)
Protocol version: 10
Connection: 127.0.0.1 via TCP/IP
Server characterset: utf8
Db characterset: utf8
Client characterset: utf8
Conn. characterset: utf8
TCP port: 3308
--------------

mysql>
```

## Radon集群安装
### 环境介绍
* 操作系统：redhat-7.4
* MySQL版本：5.7.24
* Go版本：1.11.2
* MySQL地址：10.10.30.218
* Radon Master:192.168.1.27
* Radon Slave:192.168.1.29
* MySQL node1:10.10.30.18
* MySQL node2:10.10.30.19
* Backup node:192.168.1.35
### 安装步骤
#### Radon安装
按照上述单节点Radon的安装步骤在192.168.1.27和192.168.1.29上安装Radon，并启动，但不配置后端MySQL节点。
步骤省略。

#### 通过add peer接口搭建Radon集群
* 在Radon Master 192.168.1.27节点上添加Master和Slave节点，都返回200 OK表示成功

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"address": "192.168.1.27:8080"}' http://192.168.1.27:8080/v1/peer/add
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 07:02:08 GMT
Content-Length: 0

# curl -i -H 'Content-Type: application/json' -X POST -d '{"address": "192.168.1.29:8080"}' http://192.168.1.27:8080/v1/peer/add
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 07:02:19 GMT
Content-Length: 0
```

* 在Radon Slave 192.168.1.29节点上添加Master和Slave节点，都返回200 OK表示成功

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"address": "192.168.1.27:8080"}' http://192.168.1.29:8080/v1/peer/add
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 07:02:59 GMT
Content-Length: 0

# curl -i -H 'Content-Type: application/json' -X POST -d '{"address": "192.168.1.29:8080"}' http://192.168.1.29:8080/v1/peer/add
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 07:03:09 GMT
Content-Length: 0
```
* 查看各个Radon节点/usr/local/radon/bin/radon-meta目录下会有三个文件:backend.json peers.json version.json

```
# ll bin/radon-meta/
总用量 12
-rw-r--r--. 1 root root 218 11月 17 02:03 backend.json
-rw-r--r--. 1 root root 49 11月 17 02:03 peers.json
-rw-r--r--. 1 root root 31 11月 17 02:03 version.json
```
#### 在Radon Master 192.168.1.27节点上添加MySQL节点
* 通过管理端口开放API添加MySQL节点1 10.10.30.18

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"name": "backend1", "address": "10.10.30.18:3306", "user":"qbench", "password": "xxxxxx", "max-connections":1024}' http://192.168.1.27:8080/v1/radon/backend
```

返回

```
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 07:18:24 GMT
Content-Length: 0
```

* 通过管理端口开放API添加MySQL节点2 10.10.30.19

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"name": "backend2", "address": "10.10.30.19:3306", "user":"qbench", "password": "xxxxxx", "max-connections":1024}' http://192.168.1.27:8080/v1/radon/backend
```

返回

```
HTTP/1.1 200 OK
Date: Sat, 17 Nov 2018 07:18:35 GMT
Content-Length: 0
```

* 查看Radon Master节点192.168.1.27 /usr/local/radon/bin/radon-meta/backend.json文件

```
# cat /usr/local/radon/bin/radon-meta/backend.json
{
        "backup": null,
        "backends": [
                {
                        "name": "backend1",
                        "address": "10.10.30.18:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                },
                {
                        "name": "backend2",
                        "address": "10.10.30.19:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                }
        ]
}
```

* 查看Radon Slave节点192.168.1.29 /usr/local/radon/bin/radon-meta/backend.json文件，发现Master上的配置被同步到Slave上

```
# cat /usr/local/radon/bin/radon-meta/backend.json
{
        "backup": null,
        "backends": [
                {
                        "name": "backend1",
                        "address": "10.10.30.18:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                },
                {
                        "name": "backend2",
                        "address": "10.10.30.19:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                }
        ]
}
```

#### 添加Backup节点
* 通过管理端口开放API添加Backup节点：192.168.1.35

```
# curl -i -H 'Content-Type: application/json' -X POST -d '{"name": "backup", "address": "192.168.1.35:3306", "user": "backup", "password": "backup", "max-connections":1024}' http://192.168.1.27:8080/v1/radon/backup
```
返回
```
HTTP/1.1 200 OK
Date: Sun, 18 Nov 2018 09:28:36 GMT
Content-Length: 0
```

* 查看Radon Master节点192.168.1.27 /usr/local/radon/bin/radon-meta/backend.json文件

```
# cat /usr/local/radon/bin/radon-meta/backend.json
{
        "backup": {
                "name": "backup",
                "address": "192.168.1.35:3306",
                "user": "backup",
                "password": "xxxxxx",
                "database": "",
                "charset": "utf8",
                "max-connections": 1024
        },
        "backends": [
                {
                        "name": "backend1",
                        "address": "10.10.30.18:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                },
                {
                        "name": "backend2",
                        "address": "10.10.30.19:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                }
        ]
}
```

* 查看Radon Slave节点192.168.1.29 /usr/local/radon/bin/radon-meta/backend.json文件，发现Master上的配置被同步到Slave上

```
# cat /usr/local/radon/bin/radon-meta/backend.json
{
        "backup": {
                "name": "backup",
                "address": "192.168.1.35:3306",
                "user": "backup",
                "password": "xxxxxx",
                "database": "",
                "charset": "utf8",
                "max-connections": 1024
        },
        "backends": [
                {
                        "name": "backend1",
                        "address": "10.10.30.18:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                },
                {
                        "name": "backend2",
                        "address": "10.10.30.19:3306",
                        "user": "qbench",
                        "password": "xxxxxx",
                        "database": "",
                        "charset": "utf8",
                        "max-connections": 1024
                }
        ]
}
```

#### 访问Radon

```
# /usr/local/mysql/bin/mysql -uqbench -p -P3308 -h192.168.1.27
Enter password:
Welcome to the MySQL monitor. Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.7-Radon-1.0 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database |
+--------------------+
| information_schema |
| gangshen |
| mysql |
| orchestrator |
| performance_schema |
| qdata_mysql |
| sbtest |
| sys |
| zzz |
+--------------------+
9 rows in set (0.00 sec)

mysql>
```/Users/wubx/02-vm/01-radondb/ebook/radon/2-radon.md

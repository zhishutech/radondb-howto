# xenon配置文件

\#cat /etc/xenon/xenon.json
下面这个配置中我加一些注释，使用中为了避免错误，可以去掉注释(万恶的json不支持添加注释）
```
{
 "server":
 {
    "节点":"本机IP",
    "endpoint":"172.18.0.10:8801"    
 },

 "raft":
 {
    "meta-datadir":"raft.meta",
    "heartbeat-timeout":2000,  
    "election-timeout":6000,
    "Raft中成为leader后要执行系统命令":"这个地方也可以支持脚本调用",
    "leader-start-command":"ip a a 172.18.0.100/16 dev eth0 && arping -c 3 -A  172.18.0.100  -I eth0",
    "Raft从Leader切换为follower时执行的命令":"这个地方也可以支持脚本调用",
    "leader-stop-command":"ip a d 172.18.0.100/16 dev eth0"
 },

 "mysql":
 {
    "admin":"root",
    "passwd":"",
    "host":"127.0.0.1",
    "port":3306,
    "basedir":"/usr/local/mysql",
    "MySQL启动的配置文件":"",
    "defaults-file":"/data/mysql/mysql3306/my3306.cnf",
    "切换为master时执行的命令":"",
    "master-sysvars":"super_read_only=0,read_only=0,innodb_flush_log_at_trx_commit=1,sync_binlog=1",
    "切换为slave时执行的命令":"",
    "slave-sysvars":"super_read_only=1,read_only=1,innodb_flush_log_at_trx_commit=2,sync_binlog=0"
 },

 "replication":
 {
    "复制用的帐号":"",
    "user":"repl",
    "passwd":"repl4slave"
 },

 "backup":
 {
        "xenon对宣布的IP":"写本地的IP",
        "ssh-host":"172.18.0.10",
        "ssh-user":"root",
        "ssh-passwd":"",
        "basedir":"/usr/local/mysql",
        "从远程备份过来的文件直接放的位置":"指定为datadir",
        "backupdir":"/data/mysql/mysql3306/data",
        "xtrabackup-bindir":"/usr/bin/"
 },

 "rpc":
 {
    "request-timeout":500
 },

 "log":
 {
    "level":"INFO"
 }
}
```


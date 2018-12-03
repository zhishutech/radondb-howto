

[toc]

### 每个节点上启动xenon
#### 启动xenon：
\#su - mysql
\$cd /data/xenon
\$nohup ./bin/xenon -c /etc/xenon/xenon.json >./xenon.log 2>&1 &

推荐使用： screen 或是supervisor

\-> screen  -dm ./bin/xenon -c /etc/xenon/xenon.json >./xenon.log 2>&1 &

\-> supervisor的配置使用，略

#### 关闭: xenon:
pkill xenon



#### xenon的raft节点间通信 

./bin/xenoncli cluster add 172.18.0.10:8801,172.18.0.11:8801,172.18.0.12:8801


使用中注意事项
#### 状态查询

![-w1264](image/15433875881869/15434622339606.jpg)

#### 集群GTID状态查询
![-w873](image/15433875881869/15434622891023.jpg)

查看VIP是不是绑定成功：
![-w759](image/15433875881869/15434623067100.jpg)

#### 高可用切换
kill掉leader节点上的MySQL，观查是不是进行切换。
可以通过:
\#\`pwd\`
/data/xenon
\#./bin/xenoncli cluster status 确认谁是leader
然后用ip addr show 确认VIP是不是在他身上。


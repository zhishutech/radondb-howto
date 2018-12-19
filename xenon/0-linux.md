# linux准备

xenon节点前重建需要使用xtrabackup进行备份。 节点之间需要ssh连通。xenon基于用户名和基于ssh key信任实现。

这里使用ssh key无密码登录。


基于安装节点：

| 机器名 | ip | 安装角色 |
| --- | --- | --- |
| dzst141 | 172.18.0.10 | xenon,mysql |
| dzst142 | 172.18.0.11 | xenon,mysql |
| dzst143 | 172.18.0.12 | xenon,mysql | 

### 主机间ssh信任建立

\#ssh-keygen  

一路回车

![-w706](/image/15433908436018.jpg)

执行: authorized_keys

![-w653](/image/15433909245137.jpg)

注意上面的目结构，没有knows_host的情况,分发到其它机器上

\#scp -r .ssh 172.18.0.11:~/

\#scp -r .ssh 172.18.0.12:~/

测试ssh信任是不是工作ok。

\#ssh 172.18.0.11

\#ssh 172.18.0.12

\#ssh 172.18.0.10

\#ssh 172.18.0.11
...

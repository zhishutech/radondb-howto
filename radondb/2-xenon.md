# xenon结构

xenon是MySQL的一个Agent结构如下：

<img src="image/xenon.png" width="500" height="500" align="center">

** 主要功能如下 **

* MySQL高可用选主
* 基于Raft（依赖于GTID）选主
* 数据一致性依赖于增强半同步(semi-sync)
* 故障切换动作
* 借助于配置中leader-start-command  & leader-stop-command  调用相应的脚本完成
* 这一块也可以自已扩展，结合Consul，ZK来玩
* MySQL故障后切为从节点后，自动拉起并修复制关系，也可以自动重建（需要配置）
* 集成Xtrabackup备份调度实现
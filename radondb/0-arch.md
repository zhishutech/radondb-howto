# 整体结构
[toc]
## RadonDB整体架构

<img src="image/radondb.jpg" width="500" height="300" align="center">

### Radon作用
* SQL解析及路由，混合OLTP和OLAP
* 分布式事务支持
* 用户验证, 在Radon中验证，不需要MySQL端创建
* 连接池功能
* SQL审计日志记录（默认没开启，开启后会记录全量的请求，用于审计使用）
* 记录全量SQL的binlog，用于计算节点数据实时复制
* 原生集群支持，配置在节点间自动同步
   
### xenon(MySQL Plus)作用
MySQL的高可用组件，这个也是我一直觉的增加半同步出现后，是MHA的一个最佳替换产品。 还有很多好玩的功能可以去在上面扩展，主要功能如下:

* MySQL高可用选主
* 基于Raft（依赖于GTID）选主
* 数据一致性依赖于增强半同步(semi-sync)
* 故障切换动作
* 借助于配置中leader-start-command  & leader-stop-command  调用相应的脚本完成
* 这一块也可以自已扩展，结合Consul，ZK来玩
* MySQL故障后切为从节点后，自动拉起并修复制关系，也可以自动重建（需要配置）
* 集成Xtrabackup备份调度实现

### MySQL存储节点

利用MySQL的增强半同步构建，一主两从。
主要用于存储数据中的某个分片，有点类似于Redis Cluster结构中的一个主从分组。 官方使用三个节点，为了高可用，推荐至少两个节点。 实验环境，也可以使用一个节点（在单节点结构下MySQL Plus不是必须的）

### 计算节点

目前利用作者优化过的TokuDB版本存储分库分表后的全量数据，这样复杂的SQL请求可以转到该节点上运行，官方目前该节点配置是三个，也可以是1-2个， 如果复制SQL比较多，这个地方需要增多一点，实现多个从节点上的SQL运算。 官方反馈，这个地方也需在找新的技术替代，如： Greenplum或是ClickHouse，也可能是MariaDB的ColumnDB。
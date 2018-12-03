# radon结构

Radon可以理解为一个MySQL的Proxy，主要功能：

<img src="image/radon.png" width="500" height="300" align="center">

分布式事务处理方面借助于：
Innodb本身的XA事务实现

Radon本身是无状态服务，多个Radon做成Cluster后可以自动同步配置。
# RadonDB介绍

RadonDB 是青云开源的基于 MySQL 研发的新一代分布式关系型数据库，可无限水平扩展，支持分布式事务，具备金融级数据强一致性，满足企业级核心数据库对大容量、高并发、高可靠及高可用的极致要求。

RadonDB整体架构

RadonDB官网： http://radondb.io

github开源地址： https://github.com/radondb

3306π大会上RadonDB相 [参考资料](/data/)


对于RadonDB一篇文章带整体识一下: RadonDB [RadonDB架构解析](http://wubx.net/radondb/)

对于RadonDB如果你想了解更多，也可以加我微信，回复： 加入RadonDB组织，我拉你进一个非官方组织。
<img src="image/wubx.jpeg" width="300" height="400" align="center">


* [RadonDB介绍](README.md)
* [RadonDB架构]
    * [整体结构](radondb/0-arch.md)
    * [radon结构](radondb/1-radon.md)
    * [xenon结构](radondb/2-xenon.md)
* [安装环境准备]
    * [golang安装](glang.md)
* [xenon]
    * [linux准备](xenon/0-linux.md)
    * [mysql安装](xenon/1-mysql.md)
    * [xenon安装](xenon/2-xenon-install.md)
    * [xenon配置文件](xenon/3-xenon-conf.md)
    * [xenon初始化](xenon/4-xenon-init.md)
    * [xenon高可用实现](xenon/5-xenon-ha.md)
    * [xenon故障切换](xenon/6-xenon-rebuildme.md)
* [radon]
    * [radon安装](radon/0-radon-install.md)
    * [radon支持语句](radon/1-radon-sql.md)
    * [radon测试感受](radon/2-radon.md)
* [总结]
    * [使用建议](suggest.md)
* [感谢]
    * [DB测试小队](team.md)

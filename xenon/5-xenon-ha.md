[toc]

### xenon高可用实现

xenon 是基于Raft的高可用实现。大家听到Raft可能的误区是Xenon参与了数据同步。 这是城的Xenon只是有两个作用：
1. 探测本地的MySQL是不是存活，如果没存活，利用： xenon.json中的配置

```
 "mysql":
 {
    "admin":"root",
    "passwd":"",
    "host":"127.0.0.1",
    "port":3306,
    "basedir":"/usr/local/mysql",
    "MySQL启动的配置文件":"",
    "defaults-file":"/data/mysql/mysql3306/my3306.cnf"
``` 
    调用:
    /usr/local/mysql/bin/mysqld_safe --defaults-file=/data/mysql/mysql/mysql3306/my3306.cnf &  
    把数据库拉起来。
    
2. 监控本地的gtid,做为raft选举的index,选举出来新的Leader后。 把其它节点change自已身上， 然后执行： xenon.json中raft定义的： "leader-start-command"部分。
   流程如下：
   
  https://github.com/radondb/xenon/blob/master/src/raft/leader.go#L332
   
   ```
	go func() {
		defer r.wg.Done()
		gtid, err := r.mysql.GetGTID()
		if err != nil {
			r.ERROR("mysql.get.gtid.error[%v]", err)
		} else {
			r.WARNING("my.gtid.is:%v", gtid)
		}

		// MySQL1. wait relay log replay done
		r.WARNING("1. mysql.WaitUntilAfterGTID.prepare")
		r.SetRaftMysqlStatus(model.RAFTMYSQL_WAITUNTILAFTERGTID)
		if err := r.mysql.WaitUntilAfterGTID(gtid.Retrieved_GTID_Set); err != nil {
			r.ERROR("mysql.WaitUntilAfterGTID.error[%v]", err)
			// TODO(array)here we should change state to FOLLOWR
		}
		r.ResetRaftMysqlStatus()
		r.WARNING("mysql.WaitUntilAfterGTID.done")

		// MySQL2. change to master
		r.WARNING("2. mysql.ChangeToMaster.prepare")
		if err := r.mysql.ChangeToMaster(); err != nil {
			// WTF, what can we do?
			r.ERROR("mysql.ChangeToMaster.error[%v]", err)
		}
		r.WARNING("mysql.ChangeToMaster.done")

		// MySQL3. enable semi-sync on master
		// wait slave ack
		r.WARNING("3. mysql.EnableSemiSyncMaster.prepare")
		if err := r.mysql.EnableSemiSyncMaster(); err != nil {
			// WTF, what can we do?
			r.ERROR("mysql.EnableSemiSyncMaster.error[%v]", err)
		}
		r.WARNING("mysql.EnableSemiSyncMaster.done")

		// MySQL4. set mysql master system variables
		r.WARNING("4.mysql.SetSysVars.prepare")
		r.mysql.SetMasterGlobalSysVar()
		r.WARNING("mysql.SetSysVars.done")

		// MySQL5. set mysql to read/write
		r.WARNING("5. mysql.SetReadWrite.prepare")
		if err := r.mysql.SetReadWrite(); err != nil {
			// WTF, what can we do?
			r.ERROR("mysql.SetReadWrite.error[%v]", err)
		}
		r.WARNING("mysql.SetReadWrite.done")
		//consul
		
		r.WARNING("6. start.vip.prepare")
		if err := r.leaderStartShellCommand(); err != nil {
			// TODO(array): what todo?
			r.ERROR("leader.StartShellCommand.error[%v]", err)
		}
		r.WARNING("start.vip.done")
		r.WARNING("async.setting.all.done....")
	}()
	
	```
	
	核心原理就是成为Leader后，会把其它节点change到自已身上，然后关闭super_read_only, read_only, 调用leader-start-command 绑定vip.
	 

###  Xenon故障切换流程

当原理的主节点故障或是crash, xenon会触发一次新的选举，原始的leader释放vip, 成为follower其它节点提义成为新的Leader，当成为leader后，还是走的上面的流程。

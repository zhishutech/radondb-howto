[toc]

###  故障节点重建方法
节点重建是使用的xtrabackup实现
调用命令: xenoncli mysql rebuildme
这里调用的配置是xenon.json中backup部分：

```
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
```
其中： backupdir 需要指定为MySQL的datadir
这块处理非常爆力，直接把datadir给rm后，从远程备份一个过来。 还好对于master不支持重建，该命令需要手工调用。

https://github.com/radondb/xenon/blob/master/src/cli/cmd/mysql.go#L164

```
	// 1. first to check I am leader or not
	{
		log.Warning("S1-->check.raft.leader")
		leader, err := callx.GetClusterLeader(self)
		ErrorOK(err)
		if leader == self {
			log.Panic("[%v].I.am.leader.you.cant.rebuildme.sir", self)
		}
	}

	// 2. find the best to backup
	{
		if fromStr != "" {
			bestone = fromStr
		} else {
			bestone, err = callx.FindBestoneForBackup(self)
			ErrorOK(err)
		}
		log.Warning("S2-->prepare.rebuild.from[%v]....", bestone)
	}

	// 3. check bestone is not in BACKUPING
	{
		rsp, err := callx.GetMysqldStatusRPC(bestone)
		ErrorOK(err)
		if rsp.BackupStatus == model.MYSQLD_BACKUPING {
			log.Warning("S3-->check.bestone[%v].is.backuping....", bestone)
			log.Panic("bestone[%v].is.backuping....", bestone)
		}
		log.Warning("S3-->check.bestone[%v].is.OK....", bestone)
	}

	// 4. disable raft
	{
		log.Warning("S4-->disable.raft")
		if _, err := callx.DisableRaftRPC(self); err != nil {
			log.Error("disableRaftRPC.error[%v]", err)
		}
	}

	// 5. stop monitor
	{
		log.Warning("S5-->stop.monitor")
		callx.StopMonitorRPC(self)
	}

	// 6. force kill mysqld
	{
		log.Warning("S6-->kill.mysql")
		err := callx.KillMysqldRPC(self)
		ErrorOK(err)

		// wait
		err = callx.WaitMysqldShutdownRPC(self)
		ErrorOK(err)
	}

	// 7. check bestone is not in BACKUPING again
	{
		rsp, err := callx.GetMysqldStatusRPC(bestone)
		ErrorOK(err)
		if rsp.BackupStatus == model.MYSQLD_BACKUPING {
			log.Warning("S7-->check.bestone[%v].is.backuping....", bestone)
			log.Panic("bestone[%v].is.backuping....", bestone)
		}
		log.Warning("S7-->check.bestone[%v].is.OK....", bestone)
	}

	// 8. remove data files 非常爆力的执行了  rm -rf datadir
	{
		datadir := conf.Backup.BackupDir
		cmds := "bash"
		args := []string{
			"-c",
			fmt.Sprintf("rm -rf %s/*", datadir),
		}

		_, err := common.RunCommand(cmds, args...)
		ErrorOK(err)
		log.Warning("S8-->rm.datadir[%v]", datadir)
	}

	// 9. do backup from bestone
	{
		log.Warning("S9-->xtrabackup.begin....")
		rsp, err := callx.RequestBackupRPC(bestone, conf, conf.Backup.BackupDir)
		ErrorOK(err)
		RspOK(rsp.RetCode)
		log.Warning("S9-->xtrabackup.end....")
	}

	// 10. do apply-log
	{
		log.Warning("S10-->apply-log.begin....")
		err := callx.DoApplyLogRPC(conf.Server.Endpoint, conf.Backup.BackupDir)
		ErrorOK(err)
		log.Warning("S10-->apply-log.end....")
	}

	// 11. start mysqld
	{
		log.Warning("S11-->start.mysql.begin...")
		if _, err := callx.StartMonitorRPC(self); err != nil {
			log.Error("start.mysql..error[%v]", err)
		}
		log.Warning("S11-->start.mysql.end...")
	}

	// 12. wait mysqld running
	{
		log.Warning("S12-->wait.mysqld.running.begin....")
		callx.WaitMysqldRunningRPC(self)
		log.Warning("S12-->wait.mysqld.running.end....")
	}

	// 13. wait mysql working
	{
		log.Warning("S13-->wait.mysql.working.begin....")
		callx.WaitMysqlWorkingRPC(self)
		log.Warning("S13-->wait.mysql.working.end....")
	}

	// 14. stop slave and reset slave all
	{
		log.Warning("S14-->stop.and.reset.slave.begin....")
		if _, err := callx.MysqlResetSlaveAllRPC(self); err != nil {
			log.Error("mysql.stop.adn.reset.slave.error[%v]", err)
		}
		log.Warning("S14-->stop.and.reset.slave.end....")
	}

	// 15. set gtid_purged
	{

		log.Warning("S15-->reset.master.begin....")
		callx.MysqlResetMasterRPC(self)
		log.Warning("S15-->reset.master.end....")

		gtid, err := callx.GetXtrabackupGTIDPurged(self, conf.Backup.BackupDir)
		ErrorOK(err)

		log.Warning("S15-->set.gtid_purged[%v].begin....", gtid)
		rsp, err := callx.SetGlobalVarRPC(self, fmt.Sprintf("SET GLOBAL gtid_purged='%s'", gtid))
		ErrorOK(err)
		RspOK(rsp.RetCode)
		log.Warning("S15-->set.gtid_purged.end....")
	}

	// 16. enable raft
	{
		// check whether the state is IDLE or not
		if conf.Raft.StartAsIDLE {
			log.Warning("S16-->enable.raft.skiped.since.StartAsIDLE=true...")
			log.Warning("S16-->run.as.IDLE...")
		} else {
			log.Warning("S16-->enable.raft.begin...")
			if _, err := callx.EnableRaftRPC(self); err != nil {
				log.Error("enbleRaftRPC.error[%v]", err)
			}
			log.Warning("S16-->enable.raft.done...")
		}
	}

	// 17. wait change to master
	{
		log.Warning("S17-->wait[%v ms].change.to.master...", conf.Raft.ElectionTimeout)
		time.Sleep(time.Duration(conf.Raft.ElectionTimeout))
	}

	// 18. start slave
	{
		log.Warning("S18-->start.slave.begin....")
		if _, err := callx.MysqlStartSlaveRPC(self); err != nil {
			log.Error("mysql.start.slave.error[%v]", err)
		} else {
			log.Warning("S18-->start.slave.end....")
		}
	}
```
其中这个地方容易遇到卡到第11步：
S11-->start.mysql.end...
...
一直在输出这个。

这个地方大多的原因是备份过来的权限问题。 可以手工改一下datadir权限或是
在11步聚拉起MySQL前面添加代码如下：

```
	// patch by wubx.  change datadir owner
	// 如果运行的用户不是mysql自行修改成需要的用户
	{
		datadir := conf.Backup.BackupDir
		cmds := "bash"
		args := []string{
			"-c",
			fmt.Sprintf("chown -R mysql:mysql %s", datadir),
		}

		_, err := common.RunCommand(cmds, args...)
		ErrorOK(err)
		log.Warning("SPre11-->chown owner to mysql[%v]", datadir)
	}
```
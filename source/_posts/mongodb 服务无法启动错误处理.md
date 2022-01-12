---
layout: ops
title: mongodb 服务无法启动错误处理
uuid: 12595d02e865
date: 2022-01-12 14:20:23
tags:
---

今日，修改 MongoDB ,运行外网访问，重启之后，发生如下错误。

```shell
Job for mongod.service failed because the control process exited with error code. See "systemctl status mongod.service" and "journalctl -xe" for details.
```

通过 journalctl -xe ，详细错误如下：

```shell
● mongod.service - MongoDB Database Server
   Loaded: loaded (/usr/lib/systemd/system/mongod.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2022-01-12 14:04:50 CST; 7s ago
     Docs: https://docs.mongodb.org/manual
  Process: 29261 ExecStart=/usr/bin/mongod $OPTIONS (code=exited, status=14)
  Process: 29259 ExecStartPre=/usr/bin/chmod 0755 /var/run/mongodb (code=exited, status=0/SUCCESS)
  Process: 29256 ExecStartPre=/usr/bin/chown mongod:mongod /var/run/mongodb (code=exited, status=0/SUCCESS)
  Process: 29255 ExecStartPre=/usr/bin/mkdir -p /var/run/mongodb (code=exited, status=0/SUCCESS)

Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: Starting MongoDB Database Server...
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ mongod[29261]: about to fork child process, waiting until server is ready for connections.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ mongod[29261]: forked process: 29264
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: mongod.service: control process exited, code=exited status=14
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: Failed to start MongoDB Database Server.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: Unit mongod.service entered failed state.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: mongod.service failed.
[root@iZwz92s22j9ysy0jmizizgZ ~]# journalctl -xe
Jan 12 14:02:09 iZwz92s22j9ysy0jmizizgZ systemd[1]: mongod.service: control process exited, code=exited status=14
Jan 12 14:02:09 iZwz92s22j9ysy0jmizizgZ systemd[1]: Failed to start MongoDB Database Server.
-- Subject: Unit mongod.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit mongod.service has failed.
--
-- The result is failed.
Jan 12 14:02:09 iZwz92s22j9ysy0jmizizgZ systemd[1]: Unit mongod.service entered failed state.
Jan 12 14:02:09 iZwz92s22j9ysy0jmizizgZ systemd[1]: mongod.service failed.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ sudo[29238]:     root : TTY=pts/0 ; PWD=/root ; USER=root ; COMMAND=/sbin/service mongod restart
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ sudo[29238]: pam_unix(sudo:session): session opened for user root by root(uid=0)
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ polkitd[514]: Registered Authentication Agent for unix-process:29239:422071355 (system bus name :1.1667
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: Starting MongoDB Database Server...
-- Subject: Unit mongod.service has begun start-up
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit mongod.service has begun starting up.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ mongod[29261]: about to fork child process, waiting until server is ready for connections.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ mongod[29261]: forked process: 29264
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ mongod[29261]: ERROR: child process failed, exited with 14
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ mongod[29261]: To see additional information in this output, start without the "--fork" option.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: mongod.service: control process exited, code=exited status=14
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: Failed to start MongoDB Database Server.
-- Subject: Unit mongod.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit mongod.service has failed.
--
-- The result is failed.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: Unit mongod.service entered failed state.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ systemd[1]: mongod.service failed.
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ sudo[29238]: pam_unix(sudo:session): session closed for user root
Jan 12 14:04:50 iZwz92s22j9ysy0jmizizgZ polkitd[514]: Unregistered Authentication Agent for unix-process:29239:422071355 (system bus name :1.16
```

通过 Google 找到解决方案，[全文链接](https://askubuntu.com/questions/823288/mongodb-loads-but-breaks-returning-status-14)。

我采用第三种解决方案：

```shell
sudo rm -rf /tmp/mongodb-27017.sock
sudo systemctl start mongod
```

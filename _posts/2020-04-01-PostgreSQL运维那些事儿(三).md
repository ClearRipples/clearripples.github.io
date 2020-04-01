```
---
layout:     post
title:      PostgreSQL运维那些事儿(三)
subtitle:   pgpool 常用命令
date:       2020-04-01
author:     Wright Liang
header-img: img/post-bg-debug.png
catalog: true
tags:
    - PostgreSQL
---
```

### 执行在线恢复

```
[pgpool@gs-server-7149 bin]$ pcp_recovery_node -v -d -h 10.200.50.117 -p 9898 -U postgres -n 0
```
### 提升节点

```
[pgpool@gs-server-7149 bin]$ pcp_promote_node -v -d -h 10.200.50.117 -p 9898 -U pgpool -n 0
```

### 移除节点

```
[pgpool@gs-server-7149 bin]$ pcp_detach_node -d -h 10.200.50.117 -p 9898 -U pgpool -n 0
```

### 关联节点

```
[pgpool@gs-server-7149 bin]$ pcp_attach_node -d -h 10.200.50.117 -p 9898 -U pgpool -n 0
```

### 检查主从状态

```
[lianghongbo01@gs-server-7149 ~]$ sudo su - pgpool
上一次登录：五 3月 20 14:32:14 CST 2020pts/5 上
[pgpool@gs-server-7149 pgpool-II]$ pcp_watchdog_info -h 10.200.50.231 -p 9898 -U pgpool
Password: 
2 YES 10.200.50.231:9999 Linux gs-server-7149 10.200.50.231

10.200.50.231:9999 Linux gs-server-7149 10.200.50.231 9999 9000 4 MASTER
10.200.50.117:9999 Linux gs-server-6974 10.200.50.117 9999 9000 7 STANDBY
```
### 获取节点信息

```
[pgpool@gs-server-7149 bin]$ pcp_node_info -v -d -n 0 -U pgpool
Password: 
DEBUG: recv: tos="m", len=8
DEBUG: recv: tos="r", len=21
DEBUG: send: tos="I", len=6
DEBUG: recv: tos="i", len=73
Hostname          : 10.200.50.55
Port              : 5432
Status            : 2
Weight            : 0.500000
Status Name       : up
Role              : standby
Replication Delay : 0
Last Status Change: 2019-11-22 10:36:15
DEBUG: send: tos="X", len=4

```

### 获取连接池信息

```
[pgpool@gs-server-7149 bin]$ pcp_pool_status -v -d -U pgpool
Password: 
DEBUG: recv: tos="m", len=8
DEBUG: recv: tos="r", len=21
DEBUG pcp_pool_status: send: tos="B", len=4
DEBUG: recv: tos="b", len=18

```
### 获取进程列表
```
[pgpool@gs-server-7149 bin]$ pcp_proc_count -v -d -U pgpool
Password: 
DEBUG: recv: tos="m", len=8
DEBUG: recv: tos="r", len=21
DEBUG: send: tos="N", len=4
DEBUG: recv: tos="n", len=8937
No       |       PID
_____________________
0        |       12569
1        |       16376
2        |       30393
3        |       37806
4        |       22936
5        |       42242
6        |       27825
7        |       42658
8        |       42248
9        |       20236
10       |       62282
11       |       10977
12       |       27288
13       |       30308
14       |       45112
15       |       37795

```

---
layout:     post
title:      PostgreSQL 集群之主从安装配置
subtitle:   Centos 7 主从安装配置
date:       2020-03-19
author:     Wright Liang
header-img: img/post-bg-debug.png
catalog: true
tags:
    - PostgreSQL
---

## 目录
- [单机安装](https://hongbo.tech/2020/03/18/PostgreSQL-%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%8D%95%E6%9C%BA%E5%AE%89%E8%A3%85/)
- [基于流式的 wal 数据复制功能搭建主/热备数据库集群](https://hongbo.tech/2020/03/19/PostgreSql-%E9%9B%86%E7%BE%A4%E4%B9%8B%E6%B5%81%E5%A4%8D%E5%88%B6%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/)
- pgpool工具高可用
- 配置调优

---

## 1. 集群配置
> 两台服务器， IP 设置如下，并将其配置加入到 `/etc/host` 中

```
10.200.50 55 gs-server-7297
10.200.50.65 gs-server-7296
```

> 两台服务器的角色分别如下：

```
gs-server-7297: 主服务器
gs-server-7296: 从服务器
```

## 2. 主从配置
### 2.1 主节点（gs-server-7297)
#### 2.1.1  创建一个用于流复制的账号

```
CREATE ROLE pgrepuser REPLICATION LOGIN ENCRYPTED PASSWORD 'pgreppass'; # 正式的为 nQbk09JKEwoq
```
#### 2.1.2 修改 postgresql.conf 

```
sudo vim /var/lib/pgsql/10/data/postgresql.conf
```
#### 2.1.3 修改配置项如下，另外可以参照[这里](https://pgtune.leopard.in.ua/#/)根据使用场景进行配置的优化：

```
listen_addresses = '*'
max_connections = 1024 # 这个设置，从库的必须大于主库的
password_encryption = md5
wal_level = logical # logical包含replica的功能，这样可使主节点同时具备流复制来源和逻辑复制发布者
archive_mode = on   #允许归档
archive_command = 'cp %p /data1/pgdata/pg_archive/%f'   # 归档目录 按实际情况填写
max_wal_senders = 32 # 这个设置了可以最多有几个流复制连接，差不多有几个从，就设置几个
wal_keep_segments = 256 ＃ 设置流复制保留的最多的xlog数目
wal_sender_timeout = 60s ＃ 设置流复制主机发送数据的超时时间
synchronous_standby_names = '*' #数据同步模式，默认为异步，可能导致数据不一致，修改为同步模式
```

#### 2.1.4 在  pg_hba.conf 文件中为 pgreppass 设置权限规则。允许 pgrepuser 从所有地址连接到主节点，并使用基于 MD5 的加密方式

```
sudo vim /var/lib/pgsql/10/data/pg_hba.conf
```

> 配置文件最后添加一行

```
host    replication     pgrepuser       0.0.0.0/0               md5
```

#### 2.1.5 主服务器配置好后，**需要重启数据库**

```
sudo systemctl restart postgresql-10.service
```
#### 2.1.6 **生产服务器上，则执行如下**
>若在生产环境中没有条件进行数据库重启，也可以使用 pg_ctl reload 指令重新加载配置
  
```
sudo systemctl reload postgresql-10
```
#### 2.1.7 重置用户 postgres 密码, 需要记住后面需要用到

```
sudo  passwd -d postgres
sudo -u postgres passwd
记录好密码  M2ssSRMm 
```

### 2.2 从节点
#### 2.2.1 停止从机上的 pgsql 服务（非启动过，则忽略该步骤）

```
sudo systemctl stop postgresql-10
```

#### 2.2.2 使用 pg_basebackup 生成备库
> 首先清空 $PGDATA 目录

```
sudo su - postgres
cd /var/lib/pgsql/10/data
rm -rf *
```
> 使用 pg_basebackup 命令生成备库, 需要输入密码

```
pg_basebackup -D /data1/pgdata/data -Fp -Xstream -R -c fast -v -P -h 10.200.50.55 -U pgrepuser -W
```
> 输入密码后正常会显示

```
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/6000028 on timeline 1
pg_basebackup: starting background WAL receiver
24474/24474 kB (100%), 1/1 tablespace                                         
pg_basebackup: write-ahead log end point: 0/60000F8
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: base backup completed
```

#### 2.2.3 调整 postgresql.conf

```
sudo vim /var/lib/pgsql/10/data/postgresql.conf
```
> 变更一下配置

```
hot_standby = on  说明这台机器不仅仅是用于数据归档，也用于数据查询
max_connections = 1280 ＃ 一般查多于写的应用从库的最大连接数要比较大
max_standby_streaming_delay = 30s # 数据流备份的最大延迟时间
wal_receiver_status_interval = 1s  # 多久向主报告一次从的状态，当然从每次数据复制都会向主报告状态，这里只是设置最长的间隔时间
hot_standby_feedback = on # 如果有错误的数据复制，是否向主进行反馈
```
#### 2.2.4 添加 recovery.conf 配置

```
standby_mode = on  # 这个说明这台机器为从库
primary_conninfo = 'host=10.200.50.55 port=5432 user=pgrepuser password=pgreppass'  # 这个说明这台机器对应主库的信息
recovery_target_timeline = 'latest' # 这个说明这个流复制同步到最新的数据
trigger_file='recovery.fail'
restore_command = 'cp %p /data1/pgdata/archivetemp/%f' #restore_command 如果发现从属服务器处理事务日志的速度较慢，跟不上主服务器产生日志的速度，为避免主服务器产生积压，你可以在从属服务器上指定一个路径用于缓存暂未处理的日志。请在 recovery.conf 中添加如下一个代码行，该代码行在不同操作系统下会有所不同。
```

#### 2.2.5 重启从库

```
sudo systemctl restart postgresql-10.service
```
**启动复制进程注意事项**

> 一般情况下，我们建议先启动所有从属服务器再启动主服务器，如果顺序反过来，会导致主服务器已经开始修改数据并生成事务日志了，但从属服务器却还无法进行复制处理，这会导致主服务器的日志积压。如果在未启动主服务器的情况下先启动从属服务器，那么从属服务器日志中会报错，说无法连接到主服务器，但这没有关系，忽略即可。等所有从属服务器都启动完毕后，就可以启动主服务器了。
>
>此时所有主从属服务器应该都是能访问的。主服务器的任何修改，包括安装一个扩展包或者是新建表这种对系统元数据的修改，都会被同步到从属服务器。从属服务器可对外提供查询服务。
>
>如果希望某个从属服务器脱离当前的主从复制环境，即此后以一台独立的 PostgreSQL 服务器身份而存在，请直接在其 data 文件夹下创建一个名为 failover.now 的空文件。从属服务器会在处理完当前接收到的最后一条事务日志后停止接收新的日志，然后将 recovery.conf 改名为 recovery.done。此时从属服务器已与主服务器彻底解除了复制关系，此后这台PostgreSQL 服务器会作为一台独立的数据库服务器存在，其数据的初始状态就是它作为从属服务器时处理完最后一条事务日志后的状态。一旦从属服务器脱离了主从复制环境，就不可能再切换回主从复制状态了，要想切回去，必须按照前述步骤一切从零开始。

## 3. 测试主/备服务
### 3.1 流复制验证
> **分别登陆主/从服务器**，使用 \l 命令查看数据库列表

```
sudo su - postgres
psql
\l
```
> 显示如下：

```
                                List of databases
  Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
          |          |          |             |             | postgres=CTc/postgres
template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
          |          |          |             |             | postgres=CTc/postgres
(3 rows)

```

> 切换到**主库**有，创建一个数据库 test

```
create database test template=template1;
```
> 然后切换到从库中查看，出现如下信息说明流同步已完成

```
                              List of databases
  Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
          |          |          |             |             | postgres=CTc/postgres
template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
          |          |          |             |             | postgres=CTc/postgres
test      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 

```
> 在从库进行 DDL 操作，可以发现从库被正确的设置为只读模式

```
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# CREATE TABLE test (id BIGSERIAL PRIMARY KEY, name VARCHAR(255), age INT);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
test=# 
```
> 在主库上，可以正常的进行读写操。同时主节点的恢复（recovery）模式为 false

```
ostgres=# \c test
You are now connected to database "test" as user "postgres".
test=# CREATE TABLE test (id BIGSERIAL PRIMARY KEY, name VARCHAR(255), age INT);
CREATE TABLE
test=# INSERT INTO test(name, age) VALUES('中文', 31), ('test', 31);
INSERT 0 2
test=# select * from test;
id | name | age 
----+------+-----
  1 | 中文 |  31
  2 | test |  31
(2 rows)

test=# select pg_is_in_recovery();
pg_is_in_recovery 
-------------------
f
(1 row)

test=# 
```
> 再切换到从库，执行查询语句可以看到之前在主库上写入的两条记录已被成功复制过来

```
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# select * from test;
id | name | age 
----+------+-----
  1 | 中文 |  31
  2 | test |  31
(2 rows)
```

- 数据库复制状态，在主节点上可以看到客户端地址（client_addr）为：10.200.50.65，使用流式复制（state），同步模式（sync_state）为同步复制


```
test=# \x
Expanded display is on.
test=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 32967
usesysid         | 16384
usename          | pgrepuser
application_name | walreceiver
client_addr      | 10.200.50.65
client_hostname  | 
client_port      | 47039
backend_start    | 2019-11-08 17:25:35.331942+08
backend_xmin     | 560
state            | streaming
sent_lsn         | 0/801EC50
write_lsn        | 0/801EC50
flush_lsn        | 0/801EC50
replay_lsn       | 0/801EC50
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 1
sync_state       | sync
```

### 3.2 主备切换验证
> 按照以下说明执行

```
# 1. 关闭主节点数据库服务 centos7-001 建议采用 `pg_ctl stop -m fast`
sudo systemctl stop postgresql-10.6
# 2. 将从节点变为主节点
sudo su - postgres
pg_ctl promote -D /var/lib/pgsql/10/data
#3 将老主库变更为备库，在老的主库的$PGDATA目录下也创建recovery.conf文件（创建方式跟之前介绍的一样，内容可以和原从库pghost2的一样，只是primary_conninfo的IP换成对端pghost2的IP）
同时备库和主库修改 postgressql.conf，将老主库的 hot_standby = on，同时 max_connections 改成比现在主库大
```
此时在节点 centos7-002 上，PostgreSQL数据库已经从备节点转换成了主节点。同时，recovery.conf 文件也变为了 recovery.done 文件，表示此节点不再做为从节点进行数据复制。
至此，流复制集群建好。







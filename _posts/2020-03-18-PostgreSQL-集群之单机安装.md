---
layout:     post
title:      PostgreSQL 集群之单机安装
subtitle:   Centos 7 单机安装
date:       2020-03-18
author:     Wright Liang
header-img: img/post-bg-debug.png
catalog: true
tags:
    - PostgreSQL
---

## 目录
- [单机安装](https://hongbo.tech/2020/03/18/PostgreSQL-%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%8D%95%E6%9C%BA%E5%AE%89%E8%A3%85/)
- [基于流式的 wal 数据复制功能搭建主/热备数据库集群]()
- [pgpool工具高可用]()
- [配置调优]()

---
## 1. 目标
搭建一个全新的集群，用于业务系统的分库，预计需要支持到的数据库为 1000+。需要达到一下目标
- 高可用：数据库服务器可以一起工作，这样如果主要的服务器失效则允许一个第二服务器快速接手它的任务
- 负载均衡：允许多台计算机提供相同的数据
- 统一开发、QA 和线上环境目录不对应问题，减少排查问题的干扰

**对于第三个目标，由于历史原因导致现有的环境中，开发和QA 环境的 pgsql 不是通过 yum 进行安装，表现在目录不一致**

## 2. 资源配置以及集群基本结构
- 系统环境：Centos Core 7 x64 2 台
- pgsql 版本 10.10

## 3. 系统安装、配置
- Centos 7 安装

### 3.1 配置 yum 源
> 到官网选择对应的版本，得到相关repos的连接
https://www.postgresql.org/download/linux/redhat/

```
# proxychains4: 为代理客户端
sudo proxychains4 yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
- 检查 yum 源

```
sudo proxychains4 yum search postgresql10
```
正常应该显示如下信息：

```
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
============================================================================================== N/S matched: postgresql10 ===============================================================================================
postgresql10-debuginfo.x86_64 : Debug information for package postgresql10
postgresql10-tcl-debuginfo.x86_64 : Debug information for package postgresql10-tcl
postgresql10.x86_64 : PostgreSQL client programs and libraries
postgresql10-contrib.x86_64 : Contributed source and binaries distributed with PostgreSQL
postgresql10-devel.x86_64 : PostgreSQL development header files and libraries
postgresql10-docs.x86_64 : Extra documentation for PostgreSQL
postgresql10-libs.x86_64 : The shared libraries required for any PostgreSQL clients
postgresql10-odbc.x86_64 : PostgreSQL ODBC driver
postgresql10-plperl.x86_64 : The Perl procedural language for PostgreSQL
postgresql10-plpython.x86_64 : The Python procedural language for PostgreSQL
postgresql10-pltcl.x86_64 : The Tcl procedural language for PostgreSQL
postgresql10-server.x86_64 : The programs needed to create and run a PostgreSQL server
postgresql10-tcl.x86_64 : A Tcl client library for PostgreSQL
postgresql10-test.x86_64 : The test suite distributed with PostgreSQL

  Name and summary matches only, use "search all" for everything.
```

### 3.2 centos 系统配置(一般运维已经处理掉，但要查看一下)
#### 3.2.1 Selinux配置

编辑 /etc/sysconfig/selinux，设置 SELINUX 为 disabled：

```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
 修改完后，==重启系统==

### 3.3 安装 pgsql 包

#### 3.3.1 安装服务端
```
sudo proxychains4 yum install -y postgresql10-server
```
#### 3.3.2 检查安装的软件包

```
rpm -qa *postgresql*
```
显示内容如下

```
postgresql10-libs-10.10-1PGDG.rhel7.x86_64
postgresql10-10.10-1PGDG.rhel7.x86_64
postgresql10-server-10.10-1PGDG.rhel7.x86_64
```
### 3.4 配置 pgsql
#### 3.4.1 配置环境变量

切换到 root 

```
sudo su - root
```
执行如下脚本，进行环境变量设置

```
sudo echo 'export PGSQL_HOME=/usr/pgsql-10' > /etc/profile.d/pgsql.sh
sudo echo 'export PATH=${PGSQL_HOME}/bin:$PATH' >> /etc/profile.d/pgsql.sh
source /etc/profile
```
#### 3.4.2 初始化数据库

```
postgresql-10-setup initdb
```
如果没有意外，则显示以下信息，

```
Initializing database ... OK
```

#### 3.4.3 启动服并配置服务自启动

```
sudo systemctl start postgresql-10.service
sudo systemctl enable postgresql-10.service
```
#### 3.4.4 检查服务的启动

```
sudo netstat -antp | grep postmaster
```
正常会显示如下信息

```
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      45251/postmaster    
tcp6       0      0 ::1:5432                :::*                    LISTEN      45251/postmaster   
```

也可以通过进程的方式查看

```
sudo pgrep -a postmaster
```
显示如下：

```
45251 /usr/pgsql-10/bin/postmaster -D /var/lib/pgsql/10/data/
45253 postgres: logger process                               
45255 postgres: checkpointer process                         
45256 postgres: writer process                               
45257 postgres: wal writer process                           
45258 postgres: autovacuum launcher process                  
45259 postgres: stats collector process                      
45260 postgres: bgworker: logical replication launcher       
You have new mail in /var/spool/mail/root
```
#### 3.4.5 检查服务的版本

```
sudo psql -V
```
可见显示：

```
psql (PostgreSQL) 10.10
```

### 3.5 连接到 pgsql
#### 3.5.1 切换到 postgressql 用户

```
sudo su - postgres
```
#### 3.5.2 登录 pgsql

```
psql
```
显示如下

```
psql (10.10)
Type "help" for help.
```

#### 3.5.3 显示当前的数据库

```
\l
```
显示如下：
```
List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
```

* 其他指令

```
\h 获取帮助
\q 退出
```

### 3.6 配置 TCP/IP 用户登录
#### 3.6.1 修改管理员密码

```
sudo su - postgres
psql
ALTER USER postgres PASSWORD '123qwe!'; 
\q # 退出
sudo systemctl restart postgresql-10.service #重启服务
```
#### 3.6.2 开启协议的连接方式

安装 postgresql10-contrib.x86_64

```
sudo yum search contrib
sudo proxychains4 yum install postgresql10-contrib.x86_64
```

```
sudo vim /var/lib/pgsql/10/data/postgresql.conf
```
去掉这个几行前面的 # 号，保存
```
listen_addresses = '*'
port = 5432
password_encryption = md5
shared_buffers = 4096MB
wal_keep_segments = 128 #不宜设置过大，这个涉及到wal日志的存量，一般设置为128，不要超过200
shared_preload_libraries = 'pg_stat_statements '
track_activity_query_size = 4096
```
* 重要配置

```
#EOD生产环境使用1个小时，因为可能有大的事务存在。
#启用这个配置项主要是因为有代码失误，会导致类似问题:
idle_in_transaction_session_timeout = 0        # in milliseconds, 0 is disabled

```



重启服务使其生效

```
sudo systemctl restart postgresql-10.service
```

#### 3.6.3 配置用户连接权限

```
sudo vim /var/lib/pgsql/10/data/pg_hba.conf
```
加入如下配置

```
host    all             all             0.0.0.0/0               md5
```
重启服务
```
sudo systemctl reload postgresql-10.service
```
#### 3.6.6 修改 pgsql 数据文件保存位置为 /data1/pgdata

创建目录并修改权限
```
cd /data1/
 sudo mkdir pgdata
 cd pgdata/
 sudo mkdir data
 sudo mkdir pg_archive  
 sudo mkdir archivetemp
 sudo chown -R postgres /data1/pgdata
 sudo chmod -R 700 /data1/pgdata
```

添加软链到服务

```
sudo su - postgres
mv /var/lib/pgsql/10/data/* /data1/pgdata/data/
rm -rf /var/lib/pgsql/10/data
ln -s /data1/pgdata/data /var/lib/pgsql/10/
exit
```
重启服务

```
sudo systemctl restart postgresql-10.service
```



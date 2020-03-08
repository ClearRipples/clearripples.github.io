---
layout:     post                    # 使用的布局（不需要改）
title:      PostgresSQL 运维那些事儿(二)       # 标题 
subtitle:   数据库表空间迁移测试
date:       2020-03-08              # 时间
author:     Wright Liang            # 作者
header-img: img/post-bg-debug.png    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Postgres SQl
---

## 1 背景
随着业务的不断增长，eod后端的爬虫在网络抓取的数据越来越多，数据库的磁盘空间也消耗越来越多。单一磁盘无法承受住postgresql多个实例数据库的数据增长，因此需要对一些业务增长比较快的数据库进行迁移，有多种迁移方式，全库迁移，表空间迁移。从成本考虑来看，表空间迁移是最简单易行的。

## 2 过程

## 2.1 环境  

目的：将pg_default里的某个大库迁移到另外一个表空间中，新建的表空间是/data2磁盘下，这样可以缓解单一磁盘空间不足的情况，同时测试是否对同步库有影响  

环境：10.202.80.85    
测试库：eodseoquery  
默认表空间：pg_default  
迁移表空间：tbs1_data  

## 2.2 详细步骤  
这里从EOD线上数据库拉取一个真实数据库来进行模拟操作（主要是验证大数据量，库大小12G左右）。

- 创建一个的数据库eodseoquery  

这是一个空的数据库，默认表空间为pg_default
```
postgres=# create database eodseoquery tablespace pg_default;
CREATE DATABASE
```  
可以看到已经出现了数据库的详细信息
```
postgres=# \l+
                                                                            List of databases
             Name             |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   |  Size   | Tablespace |                Description              
   
------------------------------+----------+----------+------------+------------+-----------------------+---------+------------+-----------------------------------------
---
 AdDissextor_TencentSocialAds | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7911 kB | pg_default | 
 dlp                          | dlpUser  | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7503 kB | pg_default | 
 eodseoquery                  | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7343 kB | pg_default | 
 exampledb                    | db_name  | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 51 MB   | pg_default | 
 kong                         | kong     | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7343 kB | pg_default | 
 postgres                     | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7343 kB | pg_default | default administrative connection databa
se
 template0                    | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 | =c/postgres          +| 7233 kB | pg_default | unmodifiable empty database
                              |          |          |            |            | postgres=CTc/postgres |         |            | 
 template1                    | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 | =c/postgres          +| 7343 kB | pg_default | default template for new databases
                              |          |          |            |            | postgres=CTc/postgres |         |            | 
 user_profiledb               | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 2949 MB | pg_default | 
(9 rows)

```
- 使用sql恢复数据库，模拟真实数据，该过程大概消耗了5分钟，数据库sql大小是12G

```
[postgres@gs-server-5354 ~]$ psql eodseoquery < eodseoquery.sql 
```

- 创建一个新的表空间，并指定路径在/data3/pg_tbs1，注意，该路径是需要迁入库的新路径，是物理路径。此前已经让ophelp挂载了一块100G的磁盘在/data3下，pg_tbs1文件夹是自己创建出来的，同时，需要将该文件夹的所属者和组改成postgres，并将权限改为700 。`注意，如果有备库的话，也需要在备库存在一个相同的文件夹，包括所属者和权限，保证主库和备库的同步能够高度一致，不然备库会同步出错。`

```
create tablespace tbs_data location '/data3/pg_tbs1';
postgres=# create tablespace tbs_data location '/data3/pg_tbs1';
CREATE TABLESPACE
```
- 此时查询pg_default表空间的大小，同时在备库中可以看到同步。  

```
postgres=# \db+
                                     List of tablespaces
    Name    |  Owner   |    Location    | Access privileges | Options |  Size   | Description 
------------+----------+----------------+-------------------+---------+---------+-------------
 pg_default | postgres |                |                   |         | 15 GB   | 
 pg_global  | postgres |                |                   |         | 521 kB  | 
 tbs_data   | postgres | /data3/pg_tbs1 |                   |         | 0 bytes | 
(3 rows)

```

- *迁移eodseoquery*库，将该库的表空间从pg_default移动到tbs1_data，此过程由于是物理迁移，迁移时间和数据库的大小有关，测试的12G数据大概花费1分半钟，总体来说还是比较快速的。迁移过程：是先复制一份到tbs1_data，然后迁移完毕后，再删除pg_default里的库，如果迁移过程出现问题会进行回滚，保证库的完备性。  

```
alter database eodseoquery set tablespace tbs1_data;

```

- 迁移完毕后，进行验证，发现pg_default的表空间大小确实变小了，tbs_data变多了,同时库eodseoquery的表空间也从pg_default变为了tbs1_data。  

```
postgres=# \db+
                                     List of tablespaces
    Name    |  Owner   |    Location    | Access privileges | Options |  Size   | Description 
------------+----------+----------------+-------------------+---------+---------+-------------
 pg_default | postgres |                |                   |         | 3043 MB | 
 pg_global  | postgres |                |                   |         | 521 kB  | 
 tbs_data   | postgres | /data3/pg_tbs1 |                   |         | 12 GB   | 


postgres=# \l+
                                                                            List of databases
             Name             |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   |  Size   | Tablespace |                Description              
   
------------------------------+----------+----------+------------+------------+-----------------------+---------+------------+-----------------------------------------
---
 AdDissextor_TencentSocialAds | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7801 kB | pg_default | 
 dlp                          | dlpUser  | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7393 kB | pg_default | 
 eodseoquery                  | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 12 GB   | tbs_data   | belongs to Gridsum.Eod.SeoQuery
 exampledb                    | db_name  | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 51 MB   | pg_default | 
 kong                         | kong     | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7233 kB | pg_default | 
 postgres                     | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 7343 kB | pg_default | default administrative connection databa
se
 template0                    | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 | =c/postgres          +| 7233 kB | pg_default | unmodifiable empty database
                              |          |          |            |            | postgres=CTc/postgres |         |            | 
 template1                    | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 | =c/postgres          +| 7233 kB | pg_default | default template for new databases
                              |          |          |            |            | postgres=CTc/postgres |         |            | 
 user_profiledb               | postgres | UTF8     | zh_CN.UTF8 | zh_CN.UTF8 |                       | 2949 MB | pg_default | 
(9 rows)

```

## 3 测试  

主库查询：  

![image](/uploads/c1cb8ace1c75180aff695eef4cbe4ee7/image.png)

备库查询：

![image](/uploads/611936b5c08014ba20e6772be4eb0b51/image.png)  

结论：两个库都能正常查询。  

## 4 其他
针对linux系统单一磁盘报警，上面的方法还是非常简便高效的，测试中没有发现太多的问题，唯一就是备库需要同步创建目录和新磁盘，不然会同步失效导致备库宕机。  

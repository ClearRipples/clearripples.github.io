---
layout:     post                    # 使用的布局（不需要改）
title:      PostgresSQL 运维那些事儿（一）
subtitle:   创建角色，建库,修改 owner
date:       2020-03-08              # 时间
author:     Wright Liang            # 作者
header-img: img/post-bg-postgres.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Postgres SQl
    - SQL
---

# 写在开始
> 项目中使用 PostgreSQL 作为数据的存储。项目开发中经常会有进行数据库创建或修改的情况。很不幸因为集群是我搭建的，然后项目数据库相关的东西，都要我来处理（基础设施让开发人员来搞就是吃力不讨好的活，各种原因，哎~~）后面我会陆续将我项目中使用的常用 SQL 整理出来，也是个总结吧。也希望能帮到别人。集群的构建上我会花个时间全部整理出来。

# 关于建库
> 在项目的需求、系统设计讨论清楚后，假定项目选定 Postgre SQL 作为基本的关系数据库，那么建库将会是第一件事。那么是不是直接
`CREATE DATABASE xxxx` 就可以了呢? 当然不是，没这么简单。先来搞清楚 CREATE DATABASE 的几个参数的含义吧。

- **创建数据库**
```
CREATE DATABASE yysg        -- 根据实际项目起名
WITH OWNER = yysg           -- 数据库的所属人
ENCODING = 'UTF8'           -- 内容的排序格式
TABLESPACE = pg_default     -- 表空间  
LC_COLLATE = 'en_US.UTF-8'  -- 指定数据库语言环境，初始化后不可更改
LC_CTYPE = 'en_US.UTF-8'    -- 代表了区域中的字符分类，比如哪些字符是字母，哪些是数字，大小写等，初始化不可修改
CONNECTION LIMIT = -1       -- 连接数限制，-1 表示不限制

TEMPLATE template0;         -- 模板引擎

GRANT CONNECT, TEMPORARY ON DATABASE yysg TO public; -- 根据实际
GRANT ALL ON DATABASE yysg TO yysg; -- 根据实际
GRANT ALL ON DATABASE yysg TO postgres; -- 根据实际

COMMENT ON DATABASE yysg
IS 'XXX database name';
```

> 建库完成后, 得给数据库添加个角色，用于配置到连接串代码里

- **创建角色**

```
create user XXX with password 'XXXXXX';

--或者 

create role XXX with password 'XXXXX' login;
```
==注意：使用create role时，需要携带  login参数，否则将无法登录，如果忘记可使用命令修改：==

```
alter role XXX  login;
```

## 修改整个库的 owner
> 如果很不幸的建好库后，没有给库设置对的角色，并且还有表，函数的 owner 也不是期望的，可以通过下面的匿名 function 进行一次性修改完成

```
DO $$
DECLARE
  r record;
  i int;
  v_schema text[] := '{public}';
  v_new_owner varchar := 'yysg';
BEGIN
  FOR r IN
      SELECT 'ALTER TABLE "' || table_schema || '"."' || table_name || '" OWNER TO ' || v_new_owner || ';' AS a FROM information_schema.tables WHERE table_schema = ANY (v_schema)
     UNION ALL
     SELECT 'ALTER TABLE "' || sequence_schema || '"."' || sequence_name || '" OWNER TO ' || v_new_owner || ';' AS a FROM information_schema.sequences WHERE sequence_schema = ANY (v_schema)
     UNION ALL
     SELECT 'ALTER TABLE "' || table_schema || '"."' || table_name || '" OWNER TO ' || v_new_owner || ';' AS a FROM information_schema.views WHERE table_schema = ANY (v_schema)
     UNION ALL
    SELECT 'ALTER FUNCTION "' || nsp.nspname || '"."' || p.proname || '"(' || pg_get_function_identity_arguments(p.oid) || ') OWNER TO ' || v_new_owner || ';' AS a FROM pg_proc p JOIN pg_namespace nsp ON p.pronamespace = nsp.oid WHERE nsp.nspname = ANY (v_schema)
     UNION ALL
     SELECT 'ALTER DATABASE "' || current_database() || '" OWNER TO ' || v_new_owner
 LOOP
     EXECUTE r.a;
 END LOOP;
 FOR i IN array_lower(v_schema, 1)..array_upper(v_schema, 1)
 LOOP
     EXECUTE 'ALTER SCHEMA "' || v_schema[i] || '" OWNER TO ' || v_new_owner;
 END LOOP;
END
$$;
```




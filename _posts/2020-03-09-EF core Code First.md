---
layout:     post
title:      Ef Core Code First
subtitle:   Ef Core 基本操作
date:       2020-03-09
author:     Wright Liang
header-img: img/post-bg-efcore.png
catalog: true
tags:
    - .net core
    - EfCore
---


# 背景
> 2017 年的时候，从媒体监测团队接手过来的一个对爬虫数据进行分析的一个系统，基本结构是采用 DDD 的模式，具体的后续我会将其整理处理，这里不多说，ORM 采用 EF 6.0， 基于 .Net Framework 4.5 进行开发，但根据这边的技术栈，希望将其升级到 .NETCore 中来，也就涉及到 EFCore 的使用

# 基本操作
> 两种方式进行 EFCore 的配置

#### 在现有项目中 startup 中调整为如下方式

```
services.AddDbContext<ZhiyuDbContext>(op => op
		.UseMySQL(Configuration.GetConnectionString("ZhiyuCommendation"),b => b
			.MigrationsAssembly(typeof(Startup).Assembly.FullName)));

```

- 运行命令 `dotnet ef migrations add InitialCreate -o "../Zhiyu.Platform.Repositories.EntityFramework/Migrations"` 进行初始化

- 运行 `dotnet ef database update` 进行数据库的更新操作，如果需要取消则使用 ``

- Change target assembly. 所有命令都需要增加 dotnet ef  --startup-project ../Zhiyu.Platform.WebApi/ XXXXX

```
cd Project.Data/
dotnet ef --startup-project ../Project.Api/ migrations add Initial

// code doesn't use .MigrationsAssembly...just rely on the default
options.UseSqlServer(connection)
```


- 对于已经存在的数据库
    - 正常执行迁移命令 `migrations add`
    - **在Migrations目录中找到生成的 xxx_init.cs 文件，进去把 Up() Down()方法中的代码都去掉，只留下空方法体**
    - 执行 Update-Database命令。这样就完成了迁移的初始化。完成这一步后，以后再修改实体就能顺利的使用迁移命令了

# EF Core迁移更新到生产环境
EF Core将迁移更新到生产环境可以使用Script-Migration命令生成sql脚本，然后到生产数据库执行

```
dotnet ef migrations script 0 InitialCreate
dotnet ef migrations script 20180904195021_InitialCreate
```


# ef 常用命令
- 查看当前连接的数据库 `dotnet ef dbcontext info`
- 查看当前可用的连接 `dotnet ef dbcontext list`
- 查看当前可用的更新 `dotnet ef migrations list`
- [更多](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dotnet#dotnet-ef-migrations-list)
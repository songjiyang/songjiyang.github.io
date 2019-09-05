---
bg: 'mongodb.jpg'
layout: post
title:  "mongodb工具，安全，故障"
crawlertitle: "mongodb"
summary: ""
date:   2019-09-05 09:00:00 +0700
categories: posts
tags: 'mongodb'
author: 宋天
---


mongodb导入和导入工具，监控工具，授权认证，故障排除



### 安全


#### 用户
- 创建第一个用户


```
> db.createUser(
... {
... user: "myUserAdmin",
... pwd: "passwd",
... roles: [ "userAdminAnyDatabase"]
... }
... )
Successfully added user: { "user" : "myUserAdmin", "roles" : [ "userAdminAnyDatabase" ] }
```


#### 认证

- 在mongod启动时加入--auth参数开启登录验证
- 使用用户名和密码进行身份验证


    mongo -u "myUserAdmin" -p "passwd" --authenticationDatabase "admin" 


- authenticationDatabase表示创建用户使用的数据库，但不是指定该用户权限的。
- 如果目的既是登陆test数据库，而用户也是test数据库中创建，那么可以不用写后面的参数 --authenticationDatabase


#### 授权

- 权限 = 在哪里 + 做什么 


    {resource:{db:"test", collections: ""}, actions:["find", "update"]}
    
    
- 角色，一组权限的集合
    1. read, 读取当前数据库中所有非系统集合
    2. readWrite, 读写当前数据库中所有非系统集合
    3. dbAdmin, 管理当前数据库
    4. userAdmin, 管理当前数据库中的用户和角色
    5. read/readWrite/dbAdmin/userAdminAnyDatabase 对所有数据库执行操作（只在admin数据库提供）

- 创建一个只能读取test数据库的用户


``` 
use test;
db.createUser(
    {
        user: "testReader",
        pwd: "passwd",
        roles: [{role: "read", db: "test"}]
    }
)

Successfully added user: {
	"user" : "testReader",
	"roles" : [
		{
			"role" : "read",
			"db" : "test"
		}
	]
}
```

- 创建一个只能读取accounts集合的用户


```
use test;
db.createRole(
    {
        role: "readAccount",
        privileges: [
            {
                resource: {
                    db:"test", 
                    collection: "accounts"
                },
                actions: ["find"]
            }
        ],
        roles: []
    }
)
use test;
db.createUser(
    {
        user: "accountsReader",
        pwd: "passwd",
        roles: ["readAccount"]
    }
)
```
- roles字段可以从已有角色中继承权限


## 工具

#### 数据导入导出
##### [mongoexport](https://docs.mongodb.com/manual/reference/program/mongoexport/)
##### [mongoimport](https://docs.mongodb.com/manual/reference/program/mongoimport/index.html)
- upsertFileds 根据特定字段去对比从而选择去更新还是插入
- stopOnError 发送错误停止
- maintainInsertionOrder 维持csv的顺序导入


#### 监控

##### [mongostat](https://docs.mongodb.com/manual/reference/program/mongostat/index.html)，统计mongodb数据库当前的状态

- 需要用户有clusterMonitor的角色
- 可以指定打印间隔和次数
- o指定需要查看的指标
    1. commman - 每秒执行的命令数
    2. dirty, used - 数据库引擎缓存的使用量百分比
    3. vsize - 虚拟内存使用量（MB)
    4. res - 常驻内存使用量
    5. conn - 连接数

##### [mongotop](https://docs.mongodb.com/manual/reference/program/mongotop/index.html),显示每个集合上的读写时间

## 故障




##### 响应时间过长

- 合适的索引，使用explain()查看索引的有效性 
- 工作集超出RAM的大小，使用mongostat查看服务的状态，used百分比远远大过dirty的百分比时，表示mongo内存不够使用


##### 连接失败

- 默认情况下，mongod进程可以支持多达65536个连接，不恰当的**配置**可能限制连接数
    1. db.serverStatus().connections可查看连接数
- ulimt -a配置中间的open files也会限制mongodb的连接数
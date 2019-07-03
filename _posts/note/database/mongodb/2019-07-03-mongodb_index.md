---
bg: 'mongodb.jpg'
layout: post
title:  "mongodb索引"
crawlertitle: "mongodb"
summary: ""
date:   2019-07-03 17:00:00 +0800
categories: posts
tags: 'mongodb'
author: 宋天
---


mongodb的索引的基本使用 


## 索引

- 索引分为单键索引和复合索引
- mongodb采用b-tree做索引
- 复合索引只能支持前缀子查询
- explain()函数可以进行查询分析


#### 创建索引

db.collection.createIndex(`<keys>`, `<options>`), `<keys>`文档指定了创建索引的字段

- 单键索引keys文档只有一个字段
- 复合索引keys文档有多个字段
- 多键索引，对一个数组字段设置索引

#### 查询索引

db.collection.getIndexes()

#### 索引的效果

db.collection.explain().`<method(...)>`,

- 查看winningPlan的stage， COLLSCAN表示线性扫描，IXSCAN代表索引扫描
- 当查询到索引且只要返回索引字段，则不用去FETCH，最大化利用了索引

#### 删除索引

db.collection.dropIndex()

- 如果要更改某些字段上已经创建的索引，必须首先删除原有索引，再重新创建索引，否则，新索引不会包含原有文档


#### 索引的特性

##### 索引的唯一性 
- {unique: true}
- 如果某个字段出现了重复值，就不可以在字段上创建唯一索引
- 如果新增的文档不包含唯一性索引字段，只有第一篇缺失该字段的文档可以被写入数据库，索引中该文档的键值被默认为null


##### 索引的稀疏性 （只将包含索引键字段的文档加入索引中)
- {sparse: true}
- 如果同一个索引既具有唯一性，又具有稀疏性，那么可以存储多篇缺失索引字段的文档

##### 索引的生存时间 （针对日期字段，设定生存时间，自动删除超过生存时间的文档）
- {expireAfterSeconds: 20} 
- 复合键索引不具备生存时间特性
- 当索引键是包含日期元素的数组字段时，数组中的最小日期将被用来计算文档是否已经过期
- 过期删除操作有一定的延迟

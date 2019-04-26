---
bg: 'mongodb.jpg'
layout: post
title:  "mongodb聚合操作"
crawlertitle: "mongodb"
summary: ""
date:   2019-04-26 09:00:00 +0700
categories: posts
tags: 'mongodb'
author: 宋天
---


mongodb的一些聚合（aggregate）操作的基本使用 

## 聚合

`{ <operator>: [<argument1>, <argument2>...]}`

### 聚合操作

`db.collection.aggregate(<pipeline>, <optinos>)`

#### 字段路径表达式

| 格式                   | 作用                           |
| ---------------------- | ------------------------------ |
| `$<field>`             | 使用$来指示字段路径            |
| `$<field>.<sub-field>` | 使用$和.来指示内嵌文档字段路径 |

#### 系统变量表达式

| 格式           | 作用                 |
| -------------- | -------------------- |
| `$$<variable>` | 使用$$来指示系统变量 |


#### 常量表达式

| 格式                   | 作用                                                                    |
| ---------------------- | ----------------------------------------------------------------------- |
| `$literal: <variable>` | 如果variable带$开头会被当成字段路径表达式，此种写法可以有类似转义的功能 |


#### 聚合管道阶段

##### $project
```
    db.acccounts.aggregate( [
        {
            $project: {
                _id: 0,    // 不输出_id
                balance: 1, // 输出balance
                clientName: "$name.firstName" //将name.firstName映射到clientName字段上面
            }
        }
    ])
```

```
    db.acccounts.aggregate( [
        {
            $project: {
                _id: 0,    // 不输出_id
                balance: 1, // 输出balance
                nameArray: [ "$name.firstName", "$name.middleName", "$name.lastName"] 
                //输出一个数组,取文档中的值，当文档对应字段不存在时输出null
            }
        }
    ])
```

##### $match
和前面的条件筛选一致

- 应该在最开始的使用match,一是剔除不关心的数据，二是提高聚合的效率

##### $limit
##### $skip

##### $unwind

```
    db.acccounts.aggregate( [
        {
            $unwind: {
              path: "$currency",
              // 将一个数组展开
              includeArrayIndex: "ccyIndex"
              // 在展开时添加元素位置
            }
        }
    ])
```
- 当unwind字段不存在，或者为null,或者为空数组的时候不会显示出来，设置preserveNullAndEmptyArrays为true的时候会将这个文档显示出来

##### $sort
- 1为从小到大排序，-1为从大到小排序

##### $lookup
- 格式一
```
    $lookup: {
        from: <collection to join>,
        localFiedld: <field from the input documents>,
        foreignField: <field from the documents of the "from" collection>,
        as: <output array field>
    }

```
- 将查询到的外汇汇率写入银行账户文档

```
    db.accounts.aggregate([
        $lookup: {
            from: forex,   // 另一集合的名称
            localField: currency,   //本集合要匹配的字段
            foreignField: ccy,    // 另一个集合要匹配的字段
            as: forexData  //匹配上之后，forex将被当做内嵌文档插入本集合，名称则为该值
        }
    ])
```
- 格式二
```
    $lookup: {
        from: <collection to join>,
        let: { <var_1>: <expression>,..., <var_n>: <expression>},
        pipeline: <pipeline to execute on the collection to join>,
        // 满足pipeline的文档才会被插入本文档
        as: <output array field>
    }

```
 - 将特定日期外汇汇率写入余额大于100的银行账户文档
 
```
    $lookup: {
        from: "forex"
        let: { bal: "$balance"},
        pipeline:[
        {
            $match: {
                $expr: {  //使用let中的变量的前提
                    $and: [
                        {$eq: ["date", new Date("2018-12-21")]},
                        {$gt: ["$$bal", 100]}  //使用let中声名的变量
                    ]
                }
            }
        }
        ]
        as: "forexData"
    }

```

##### $group

```
    $group: {
        _id: <expression>,
        <field1>: { <accumulator1>: <expression1> }
    }
```
- 按照交易货币来分组交易
```
    db.transaction.aggregate([
        {
            $group: {
                _id: "$currency"
            }
        }
    ])
```
- 使用聚合操作符计算分组聚合值
```
    db.transactions.aggregate([
        {
            $group: {
                // 按照currency分组
                _id: "$currency",
                // qty字段的和，赋值到新字段totalQty
                totalQty: { $sum: "$qty" },
                // price和qty相乘，然后求和再赋值到totalNotional
                totalNotional: {$sum: {$multiply [ "$price", "$qty"]}},
                // price的平均值
                avgPrice: { $avg: "$price"}
                // 文档的数量
                count: {$sum : 1},
                // 最大金额
                maxNotional: {$max: {$multiply: ["$price", "$qty"] }},
                // 最小金额 
                minNotional: {$min: {$multiply: ["$price", "$qty"] }}
            }
        }
    ])
```
- 如果想聚合但不分组时，可以将_id设置为null,这是所有的文档会当做一个组
- 可以和push配合创建一个新的数组
- $out可以将查询的内容写入一个新的集合，当新的集合存在时，会将文档覆盖，当管道阶段遇到错误，则新集合不会被创建，旧集合也不会被覆盖


#### 聚合选项

- `allowDiskUse: <boolean>`, 每个聚合管道阶段使用的内存不能超过100MB,如果数据量较大，为了防止聚合管道阶段超出内存上限并且抛出错误，可以用该选项


#### 聚合操作优化
##### 顺序优化
- $match阶段会尽量在$project阶段之前运行，但有些条件必须在project之后则不能被优化
- $match阶段会在$sort阶段之前运行
- $skip阶段会在$project阶段之前运行
##### 合并优化
- $sort和$limit之间没有夹杂着改变文档数量的聚合阶段，阶段可以合并，unwind和match时可以改变文档数量的操作
- 相同的阶段可以合并，limit, skip, match
- lookup和unwind， 如果应用在lookup阶段创建的as字段上面，两则可以合并
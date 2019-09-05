---
bg: 'mongodb.jpg'
layout: post
title:  "mongodb技巧"
crawlertitle: "mongodb"
summary: ""
date:   2019-09-05 09:00:00 +0700
categories: posts
tags: 'mongodb'
author: 宋天
---


mongodb的一些技巧





## 按日期分组

- 要分组的日期是Date类型的
- 注意timezone, +0800代表中国时区
- group的_id可以进行转字符串，然后连接处理


```
db.suggestions.aggregate([
    {
        $project: {
          year: {$year: {date:'$submitDate', timezone:'+0800'}},
          month: {$month: {date:'$submitDate', timezone:'+0800'}},
          dayOfMonth: {$dayOfMonth: {date:'$submitDate', timezone:'+0800'}},
          submitDate:1
        }
    },
    {
        $group: {
            _id:{year:'$year', month:'$month', day:'$dayOfMonth'},
            
            count: {
                $sum: 1
            }
        }
    },
    {   
        $sort : { _id : 1 }
    }   
])
```


- 要分组的日期是timeStamp，并且为string, 网上查找的方法


```
db.getCollection('wechat_message').aggregate(  
    [     
        {   $project : { day : {$substr: ["$sendTime", 0, 10] }}},          
        {   $group   : { _id : "$day",  number : { $sum : 1 }}},  
        {   $sort    : { _id : -1 }}          
    ]  
)  

```
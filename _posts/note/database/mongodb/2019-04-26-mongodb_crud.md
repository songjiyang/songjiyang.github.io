---
bg: 'mongodb.jpg'
layout: post
title:  "mongodb增删改查"
crawlertitle: "mongodb"
summary: ""
date:   2019-04-26 09:00:00 +0700
categories: posts
tags: 'mongodb'
author: 宋天
---


mongodb的基本增删改查操作

## 查询

#### 字段操作符


| 格式                             | 作用                             |
| -------------------------------- | -------------------------------- |
| `{field : {$type : <BSON type>}` | 匹配对应类型的字段的文档         |
| `{field : {$exists : bool }`     | 匹配对应字段存在或者不存在的文档 |

#### 数组操作符

| 格式                                          | 作用                     |
| --------------------------------------------- | ------------------------ |
| `{field : {$all :[<value1>, <value2>]}`       | 所以value都在field数组中 |
| `{field : {$elemMatch :{<query1>, <query2>}}` | 数组中元素满足多个query  |


#### 运算操作符


| 格式                                    | 作用                    |
| --------------------------------------- | ----------------------- |
| `{field : {: /pattern/, : '<options>'}` | 正则匹配                |
| `{field : {: /pattern/<options>}`       | 正则匹配，可以和$in使用 |



- find方法会返回一个游标对象Cursor，可以使用索引直接获取对应位置的文档，游标会在遍历完所有的文档或者10分钟之后关闭
- 游标常用方法，`hasNext(), next(), forEach(function), limit(number), skip(offset)， count(applySkipLimit)`, limit传入0时会返回所有文档, applySkipLimit设置为true时会考虑skip和limit来计算count,默认为false
- 在不提供筛选条件的时候，cursor.count()会从集合的元数据Metadata中获取结果，当数据库分布式结构比较复杂时，元数据中的文档数量可能不准确，这种情况下应该使用局和管道来计算文档数量
- sort({field:1, field:-1}) 1表示从小到大排序，-1逆向
- skip()命令在limit()之前执行，即时命令的顺序是相反的， sort()命令肯定在skip和limti命令之前执行

#### 投影

- { field: 1} 1表示希望出现的字段， 0表示不希望出现，文档主键是默认出现的可以使用0来去除
- 不能同时使用包含和不包含，除了文档主键之外
- $slide，可以对于数组使用，接收一个数字或者数组，表示这个数组对应索引的值，可以使用负数，负索引和python中功能类似
- $elemMatch, 数组字段中满足筛选条件的第一个元素
- $,  使用filter中的筛选条件


## 更新

`db.collection.update(<query>, <update>, <options>)`

- `<update>`文档不包含任何更新操作符， db.collection.update会将`<update>`文档替换数据库中的文档
- 主键_id是不可以更改的
- 如果在`<update>`文档中想包含_id字段，则_id值一定要和被更新的文档_id值保持一致
- 使用`<update>`替换整篇被更新文档时，只有==第一篇==符合`<query>`的文档筛选条件的文档会被更新

#### 文档更新操作符


| 格式                                      | 作用                   |
| ----------------------------------------- | ---------------------- |
| `{$set: { <field1>: <value1>, ...}}`      | 设置一个字段的值       |
| `{$unset: { <field1>: "", ...}} `         | 设置一个字段的值       |
| `{$rename: { <field1>: <newName1>, ...}}` | 对字段重命名           |
| `{$inc: { <field1>: <amonut1>, ...}} `    | 增加或者减少           |
| `{$mul: { <field1>: <number1>, ...}}`     | 相乘                   |
| `{$min: { <field1>: <value1>, ...}}  `    | 选择原值和新值最小的值 |
| `{$max: { <field1>: <value1>, ...}}`      | 选择原值和新值最大的值 |



- set可以更新普通字段，内嵌文档（使用.property)，数组(使用.index),可以使用index大于数据库中的长度来增加元素，跳过某些index增加元素
- unset中field的值没有任何影响
- unset如果字段在文档中不存在，将不会产生影响
- unset对数组删除操作时，数组的长度不会被改变，而是对于位置的值被赋为null
- rename的原字段不存在，将不会对文档产生影响
- rename的新字段在文档中存在，则新字段原本的内容将消失，成为原字段的内容，相当于先unset再set操作
- rename可以将一个字段从一个内嵌文档中取出来或者放进去，但是不可用于数组内的对象，无论是拿出来还是放进去
- inc和mul只能用于number类型的
- inc如果字段在文档中不存在，将会将值赋给定值，mul不存在会将值赋0
- min和max可以用于其他可以比较的类型字段，如日期等
- min和max的字段在文档中不存在时，和set效果相同
- min和max的更新的字段类型和原字段类型不一致时，会按照BSON数据类型排序规则进行比较，Null ---> Regular Expression


#### 数组更新操作符

| 格式                                                | 作用                                 |
| --------------------------------------------------- | ------------------------------------ |
| `{$addToSet: { <field1>: <value1>, ...}} `          | 向数组增加一个值，如果重复，则不添加 |
| `{$pop: { <field1>: <-1| 1>, ...}}`                 | 从头或者尾移除一个数组元素           |
| `{$pull: { <field1>: <value|condition>, ...}}`      | 从数组字段删除特定元素               |
| `{$pullAll: { <field1>: [<value1>, <value2>] ...}}` | 从数组字段删除特定元素               |
| `{$push: {<field1>: <value1>, ...} } `              | 向数组增加一个元素                   |


- addToSet增加内嵌文档时，内嵌文档的属性和值完全一样时才不会增加进去，顺序颠倒内嵌文档还是会被插入进数组
- addToSet可以和each关键字配合将多个字段插入数组，`{$addToSet:{field: {$each:[value1,value2]}}}` , `{$addToSet: {field:[value1, value2]}}`这种写法会讲一个内嵌数组插入field数组
- pop 1删除结尾元素， -1删除开头元素， 可以使用在内嵌数组上面
- pop删除完所有元素会留下一个空数组
- pull是要删除field的元素，不用使用elemMatch，使用elemMatch表示在field类型的数组的元素进行筛选
- pullAll相当于pull加in
- pull和pullAll删除内嵌数组时，需要内嵌数组的顺序和值完全相同
- pullAll删除内嵌文档，内嵌文档必须顺序和值完全相同， 而pull删除内嵌文档，主要内嵌文档部分匹配，顺序也可以不一致。
- push和addToSet一样，如果字段不存在，这个字段会被增加到原文档中，也可以each命令配合将多个值增加到数组
- push可以和each命令，position命令搭配使用从指定位置开始插入，position可以接受负数，-1表示最后一个元素的前面
- push可以和each命令， sort命令搭配使用在插入新元素后并按从小到大排序，或者从大到小。在插入内嵌文档时，sort可以对内嵌文档的字段进行排序。如果只想排序的话，each的值可以设置为空数组就可以达到要求的效果
- push可以和each命令，slice命令截取部分数组
- postion,sort,slice命令的顺序执行顺序从大到小
- set中$指定匹配条件的元素， $[]执行所有元素


#### 更新文档选项


| header 1               | header 2                                                       |
| ---------------------- | -------------------------------------------------------------- |
| `{multi: <boolean>}`   | 更新多个文档，默认所有更新都只针对一篇文档                     |
| `{upsert: <boolean> }` | 更新或者创建文档，update没有匹配到文档的时候，就会创建一篇文档 |


- 更新文档只能保证单个文档操作的原子性，不能保证多个文档操作的原子性
- upsert的时候当匹配条件是一个精确的内容时会将匹配条件当成文档的一部分，而匹配的条件是一个模糊的内容时，如大于，小于等，就不会将匹配条件插入文档
- save命令， 文档中包含_id字段时，相当于执行一个带upsert选项的update命令

## 删除

`db.collection.remove({query},{option})`

`db.colletion.drop({writeConcern: <document>)`

- 在默认情况下，remove命令会删除所有符合筛选条件的文档
- 如果只想删除一篇，可以使用justOne
- 在文档数较多时drop的效率比remove的效率高



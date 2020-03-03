---
bg: 'rabbitmq.jpeg'
layout: post
title:  "RabbitMQ消息属性"
crawlertitle: "rabbitmq"
summary: ""
date:   2020-03-03 18:26:00 +0700
categories: posts
tags: 'rabbitmq'
author: 宋天
---

RabbitMQ消息属性介绍




## 使用属性

- 消息头帧中的消息属性时一系列被Basic.Properties数据结构预定义好的值，有些属性例如delivery-mode, 已经有具体的含义，有的属性例如type没有具体属性

![消息属性](http://pic.sjoe.top/blog/03fig02_alt.jpg
)

### content-type

创建显示消息格式契约，就好像http中的content-type，告诉消费者你是以这种方式来序列化消息的，消费者能选择性的根据这个字段进行反序列化（取决于消费者端所用的Driver是否包含此功能)

### content-encoding

处理XML或者数量较大的JSON或YAML文件时，会出现的问题，可以在发布消息进行压缩，接收消息解压缩

- 存的是MIME(多用途互联用邮件扩展类型)，而不是UTF-8
- 注意区分content-type, content-encoding, charset的区别



| 属性             | 含义                                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------ |
| content-type     | 内容的类型，一般是text/plain, application/json, text/xml这些，发送方和接收方会根据此字段解析内容 |
| content-encoding | 压缩格式，一般是br, gzip, x-gzip指定数据被压缩                                                   |
| charset          | 字符编码，一般是UTF-8, ISO-8859-1，GBK, ASCII，指明是文本内容时采用的编码                        |

- Base64编码， 将二进制数据转成文本，会使用数据变长原来的1/3,之所以使用这样的编码是因为在某些文本协议中存在特殊字符，如果直接将二进制放入很可能破坏其结构，例如XML中的图片，XML的`<>`等都是特殊字符，所以需要Base64编码转为安全的文本，即不带特殊字符的文本，这样也可以在文本中存放二进制数据，接收方解码之后可以使用，STMP邮件协议就是这么做的

### message-id和correlation-id

message-id可以作为消息的唯一标识，correlation-id可以作为关联消息，表明该消息是另外一个消息的响应

### timestamp

表示消息的创建时间，是一个Unix时间戳

### expiration

表示消息的过期时间，是一个时间戳字符串，当把一个过期的消息发布到服务器，该消息不会被路由到任何队列，而是直接被丢弃

- RabbitMQ还有其他让消息过期的功能，例如队列的x-message-ttl参数

### delivery-mode

有两个值，1表示非持久消息，2表示持久化消息

- 1使用纯内存队列，具有较低的延迟性，2开启磁盘存储的队列，能保证RabbitMQ在宕机之后重新启动还能继续发送存储的消息
- 具体的使用要和业务相关，影响重大的业务例如交易数据使用2，即使丢失的数据也不会造成很大影响可以使用1
- 消息的持久化和队列的持久化需要区别，要是消息能够持久化，队列也必须是持久化的，队列的持久化是RabbitMQ重启后会不会重新声明这些队列。
- 持久化消息会有潜在的性能和伸缩性的问题

### app-id和user-id

应用id，255个UTF-8字符串

用户id, RabbitMQ会校验应用使用的认证用户和user-id是否一致， 可能用于聊天室这样的程序，每个用户的有自己的RabbitMQ用户

### type

消息类型属性，更像一个自定义属性，可以用于指定额外的消息类型，例如指定使用thrift或者protobuf（两种二进制编码的消息，具有较小的空间使用)时所需要文件, 或者其他类型，只要消费者能识别并处理

### reply-to
定义不明确，谨慎使用
### headers
自定义属性，是一个键值对, 并且RabbitMQ可以根据header的值来路由，而不依赖路由键

### priority

优先级，介于0到9的整数，数字越小具有越高的优先级，可以先比值大的收到(感觉打破消息队列里面队列这个概念，但毕竟是一个功能)

### cluster-id/reserved

不能使用

## 总结

合理利用消息队列的属性可以创建有价值的元数据，来创建复杂的路由和事务机制，而不用污染消息体本身
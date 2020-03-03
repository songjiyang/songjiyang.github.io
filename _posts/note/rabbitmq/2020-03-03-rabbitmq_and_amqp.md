---
bg: 'rabbitmq.jpeg'
layout: post
title:  "RabbitMQ和AMQP"
crawlertitle: "rabbitmq"
summary: ""
date:   2020-03-03 18:26:00 +0700
categories: posts
tags: 'rabbitmq'
author: 宋天
---

RabbitMQ基本概念和AMQ协议简介


- RabbitMQ使用RPC进行客户端和服务端的通信，与普通的Web API不同的是，在AMQP规范中，服务器和客户端都可以发出命令
- 要完全连接到RabbitMQ需要完成由三个同步RPC请求所组成的序列，这三个RPC请求分别对应启动、调整和打开连接操作
- 一个AMQP连接可以有多个信道，允许客户端和服务器之间进行多次会话，这被称为多路复用。


# AMQP帧(frame)
- 帧类型
- 信道编号
- 以字节为单位的帧大小
- 帧有效载荷
- 结束字节标记（ASCII值206）

![rabbitmq帧](http://pic.sjoe.top/blog/rabbitmq_frame.jpg)

- 五种帧类型
    1. 协议头帧，用于连接RabbitMQ, 仅使用一次
    2. 方法帧，RPC请求或者响应
    3. 内容头帧，包含一条消息的大小和属性
    4. 消息体帧，包含消息的内容
    5. 心跳帧，一种校验机制确保两端都可用且在正常工作
- AMQP中的心跳行为用于确保客户端和服务器之间相互响应，这是一个AMQP作为一种双向RPC协议的完美示例，如果RabbitMQ发生心跳到客户端没有得到响应会断开连接，客户端可以在建立连接时将心跳时间设置为0来关闭它们。也可以更改rabbitmq.config的heartbat值更改RabbitMQ的最大心跳间隔。之前在我们线上发生过一个问题，Nodejs客户端经常报一个Channel Closed的错，后来发现是因为一个canvas的作图程序将Nodejs进程给阻塞导致不能及时返回心跳给RabbitMQ
- 使用方法帧、内容头帧和消息体帧向RabbitMQ发布消息，先是携带命令和参数的方法帧，然后是内容帧，包含消息属性以及消息体的大小，最后是消息体帧，包含具体的消息，如果消息体超过上限，消息内容会被拆分为多个消息体帧

### 方法帧结构

![方法帧](http://pic.sjoe.top/blog/02_05.png)

- Basic和Publish这两个字段位置是类和方法的ID,使用数字表示RPC命令
- 参数包括交换器名称和路由键值
- mandatory标志告知RabbitMQ必须完成消息路由，否则应该返回一个Basic.Return帧用于指明消息无法路由

### 内容头帧结构

![内容头帧](http://pic.sjoe.top/blog/02_06.png)

- 包含消息的一些属性

### 消息体帧

![内容头帧](http://pic.sjoe.top/blog/02_07.png)


- 消息体对AMQ协议是不透明的，不被RabbitMQ解码，检查

## 使用协议

### 声明一个交换机(exchange)

- 使用Exchange.Declare命令来向服务端请求声明一个创建一个交换机，如果成功将返回Exchange.DeclareOk, 否则RabbitMQ将返回Channel.Close来关闭信道，包含一个状态码和失败原因

### 声明一个队列(queue)

- 使用Queue.Declare来声明一个队列，成功将返回Queue.DecalreOk,否则将关闭信道，重复声明一个队列是可以的，RabbitMQ将会对后续的声明请求返回一些有用的信息，如现在的消息数量，消费者的数量

> 当对一个已存在的队列声明不同的属性时，会关闭信道， 应用程序正确处理这个错误，有的客户端库会捕获一个异常而有的客户端库会注册一个监听器来监听Channel.Close命令，对于生产者向一个关闭的信道发送消息，RabbitMQ将关闭连接，而对于消费者可能并不会感到异常，会以为RabbitMQ中没有消息。

### 绑定队列和交换机

- 使用Queue.Bind和Queue.BindOk来绑定队列到交换机上面，一次只能有一个队列

### 发布一个消息到RabbitMQ

- 如上所述，最少要发送三个帧，方法帧是Basic.Publish, 携带着交换机的名称和路由键，RabbitMQ会将交换机的名称和已配置的交换机比较（如果找不到的话，默认会丢弃消息，可以通过配置mandatory标识或者消息确认机制来保证消息肯定能发送，不过都会对性能有影响)
- 当找到交换机的时候，会根据路由键来找特定的队列，RabbitMQ会将消息以FIFO的方式放入任何绑定的队列，消息并不是被复制放入队列，而是存放的引用，这样能减少物理内存的使用，当消息被取走时，根据引用找到对应的消息然后发送。
- 默认情况，没有消费者消费，消息会一直保存在队列，取决于delivery-mode属性，RabbitMQ选择将不断增长的消息保存在内存中还是写入硬盘
![发送消息](http://pic.sjoe.top/blog/02_11.png)

### 从RabbitMQ消费

- 使用Basic.Consume和Basic.ConsumeOk来建立消费连接，在此之后，RabbitMQ服务端将不断使用Basic.Deliver来发送消息到客户端
- 当客户端想停止接收消息的时候，发送Basic.Cancel到RabbitMQ, 在接收到服务端的Basic.CancelOk之前，这个阶段还是可以一直接收消息的。

![发送消息](http://pic.sjoe.top/blog/02_12.png)
- 当配置Basic.Consume的参数no_ack为false的时候，客户端必须通知RabbitMQ每一个消息它都收到了，当配置为true的时候，RabbitMQ将不断的发送给客户端直到它发送Basic.Cancel或者断开连接

![发送消息](http://pic.sjoe.top/blog/02_13.png)
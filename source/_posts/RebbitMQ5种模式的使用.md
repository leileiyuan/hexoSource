---
title: RebbitMQ5种模式的使用
date: 2016-12-29 13:42:37
tags:
 - RabbitMQ
categories: 
 - RabbitMQ
---

MQ(Message Quequ)：消息队列是应用程序和应用程序之间的通信方法。
RabbitMQ是一个开源的，基于AMQP(Advanced Mseeage Queuing Protocol)的消息队列。

RabbitMQ官网[http://www.rabbitmq.com](http://www.rabbitmq.com)提供了6种消息队列
**1 "Hello World!"**
![](/img/rabbitmq/java-one.png)

**2 Work queues**
![](/img/rabbitmq/java-two.png)

**3 Publish/Subscribe**
![](/img/rabbitmq/java-three.png)

**4 Routing**
![](/img/rabbitmq/java-four.png)

**5 Topics**
![](/img/rabbitmq/java-five.png)

**6 RPC**
远程调用，这种模式，严格意义上来讲，不算是消息队列。可以使用专门的RPC服务框架(比如dubbo：[http://dubbo.io/](http://dubbo.io/))
![](/img/rabbitmq/java-six.png)

---
title: RabbitMQ配置用户角色/权限
date: 2016-12-29 14:45:17
tags: RabbitMQ
---

在RabbitMQ中添加用户，配置权限，设置Virtual Hosts

- 用户角色说明
1. 超级管理员(`administrator`)
可登陆管理控制台，可查看所有的信息，并且可以对用户，策略(policy)进行操作。
1. 监控者(`monitoring`)
可登陆管理控制台，同时可以查看rabbitmq节点的相关信息(进程数，内存使用情况，磁盘使用情况等)
3. 策略制定者(`policymaker`)
可登陆管理控制台, 同时可以对policy进行管理。但无法查看节点的相关信息(上图红框标识的部分)。
4. 普通管理者(`management`)
仅可登陆管理控制台，无法看到节点信息，也无法对策略进行管理。
5. 其他
无法登陆管理控制台，通常就是普通的生产者和消费者。

- 添加用户角色
我们添加一个用户，设置角色为`administrator`
![Alt text](/img/rabbitmq/user-one.png)

- 创建Virtual Hosts
![Alt text](/img/rabbitmq/user-two.png)

- 设置权限
![Alt text](/img/rabbitmq/user-three.png)
![Alt text](/img/rabbitmq/user-four.png)

现在就可以使用我们新增加的用户来管理维护MQ了。

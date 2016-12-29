---
title: RabbitMQ在windows下的安装问题
date: 2016-12-29 13:57:51
tags: RabbitMQ
categories: RabbitMQ
---
RabbitMQ在windows安装，有些问题会导致安装失败，安装的注意事项：
- 使用默认的安装路径
- 系统用户名必须是英文
- 计算机名必须是英文

> 或者 直接在linux 中安装。

1. 安装过程有错误，忽略
![Alt text](/img/rabbitmq/install-one.png)

2. 安装完成后，系统服务中会有RabbitMQ的服务在
![Alt text](/img/rabbitmq/install-two.png)
3. 找到RabbitMQ的安装目录，启动管理插件
		rabbitmq-plugins enable rabbitmq_management
![Alt text](/img/rabbitmq/install-four.png)
4. 	在浏览器中输入地址查看：http://127.0.0.1:15672/。使用默认的账户登录：guest/ guest
![Alt text](/img/rabbitmq/install-five.png)

5. 登录成功才算安装成功
![Alt text](/img/rabbitmq/install-six.png)

 
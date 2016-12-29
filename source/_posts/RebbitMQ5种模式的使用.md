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
简单队列生产者与消费者一对一
P：Producer，生产者；
C：Consumer，消费者
![](/img/rabbitmq/java-one.png)

简单队列生产者：
```java
package com.belongtou.rabbitmq.simple;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import cn.itcast.rabbitmq.util.ConnectionUtil;
/** 生产者 */
public class Producer {
	private final static String QUEUE_NAME = "hello";
	public static void main(String[] args) throws Exception {
		// 获取MQ连接及通道
		Connection connection = ConnectionUtil.getConnection();
		Channel channel = connection.createChannel();
		// 声明（创建）队列
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);
		// 发送消息
		String message = "Hello World!";
		channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
		System.out.println(" [x] Sent '" + message + "'");

		// 关闭通道和连接
		channel.close();
		connection.close();
	}
}
```
简单队列消费者：
```java
package com.belongtou.rabbitmq.simple;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.QueueingConsumer;
import cn.itcast.rabbitmq.util.ConnectionUtil;
/** 消费者 */
public class Consumer {
	private final static String QUEUE_NAME = "hello";
	public static void main(String[] args) throws Exception {
		// 获取连接及通道
		Connection connection = ConnectionUtil.getConnection();
		Channel channel = connection.createChannel();
		// 声明队列
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);

		// 定义队列的消费者
		QueueingConsumer consumer = new QueueingConsumer(channel);
		// 监听队列
		channel.basicConsume(QUEUE_NAME, true, consumer);

		// 获取消息
		while (true) {
			QueueingConsumer.Delivery delivery = consumer.nextDelivery();
			String message = new String(delivery.getBody(), "UTF-8");
			System.out.println(" [x] Received '" + message + "'");
		}
	}
}
```

**2 Work queues**
Work模式的队列是一个生产者对应多个消费者，消息是竞争的，谁抢到消息算谁的
![](/img/rabbitmq/java-two.png)
为了说明问题，采用重复代码来提供两个消费者。设置的两个消费者的休眠时间一个长一个短。也就是说一个任务可能执行多，一个任务可能执行的少，来争抢消息。
生产者：
```java
/** 生产者 */
public class Producer {
	private final static String QUEUE_NAME = "queue_work";
	public static void main(String[] args) throws Exception {
		// 获取MQ连接及通道
		Connection connection = ConnectionUtil.getConnection();
		Channel channel = connection.createChannel();
		// 声明（创建）队列
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);
		// 发送消息

		for (int i = 0; i < 50; i++) {
			// 消息内容
			String message = "" + i;
			channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
			System.out.println(" [x] Sent '" + message + "'");

			Thread.sleep(i * 10);
		}

		// 关闭通道和连接
		channel.close();
		connection.close();
	}
}
```
消费者一：
```java
/** 消费者一 */
public class Consumer {
	private final static String QUEUE_NAME = "queue_work";
	public static void main(String[] args) throws Exception {
		// 获取连接及通道
		Connection connection = ConnectionUtil.getConnection();
		Channel channel = connection.createChannel();
		// 声明队列
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 同一时刻服务器只会发一条消息给消费者
        //channel.basicQos(1);

		// 定义队列的消费者
		QueueingConsumer consumer = new QueueingConsumer(channel);
		// 监听队列,设置为false，手动返回状态
		channel.basicConsume(QUEUE_NAME, false, consumer);

		// 获取消息
		while (true) {
			QueueingConsumer.Delivery delivery = consumer.nextDelivery();
			String message = new String(delivery.getBody(), "UTF-8");
			System.out.println(" Consumer111 [x] Received '" + message + "'");
            // 休眠10毫秒
            Thread.sleep(10);
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
		}
	}
}
```

消费者二：
```java
/** 消费者二 */
public class Consumer2 {
	private final static String QUEUE_NAME = "queue_work";
	public static void main(String[] args) throws Exception {
		// 获取连接及通道
		Connection connection = ConnectionUtil.getConnection();
		Channel channel = connection.createChannel();
		// 声明队列
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 同一时刻服务器只会发一条消息给消费者
        //channel.basicQos(1);

		// 定义队列的消费者
		QueueingConsumer consumer = new QueueingConsumer(channel);
		// 监听队列,设置为false，手动返回状态
		channel.basicConsume(QUEUE_NAME, false, consumer);

		// 获取消息
		while (true) {
			QueueingConsumer.Delivery delivery = consumer.nextDelivery();
			String message = new String(delivery.getBody(), "UTF-8");
			System.out.println("Consumer2 [x] Received '" + message + "'");
			
            // 休眠1秒
            Thread.sleep(1000);
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
		}
	}
}
```

运行结果：
消费者一：

	Consumer111 [x] Received '1'
	Consumer111 [x] Received '3'
	Consumer111 [x] Received '5'
	 
消费者二：

	Consumer2 [x] Received '0'
	Consumer2 [x] Received '2'
	Consumer2 [x] Received '4'
	Consumer2 [x] Received '6'
	Consumer2 [x] Received '8'
实际结果是，消费者一和消费者二，交替获取消息。并非"能者多劳"，我们要对通道设置一个值，同一时刻只发一条消息给消费者：

	// 同一时刻服务器只会发一条消息给消费者
	channel.basicQos(1);
如此，消息便会产生争抢。

**消息确认**
消费者从队列中获取消息，服务端如何知道消息已经被消费呢。
有两种确定模式：
自动确认：只要消息从队列中获取，无论消费者获取到消息是否成功消费，都认为消息已经成功消费。
手动确认：消费从队列中获取消息后，服务器将该消息标记为不可用状态，等待消费者的反馈，如果消息者一直没有反馈，那么该消息一直处于不可用状态。

手动模式：
消费者收到消息，给服务器一个反馈。
```java
	// 定义队列的消费者
	QueueingConsumer consumer = new QueueingConsumer(channel);
	// 监听队列,设置为false，手动返回完成
	channel.basicConsume(QUEUE_NAME, false, consumer);

	// 获取消息
	while (true) {
		QueueingConsumer.Delivery delivery = consumer.nextDelivery();
		String message = new String(delivery.getBody(), "UTF-8");
		System.out.println("Consumer2 [x] Received '" + message + "'");
		
           // 休眠1秒
           Thread.sleep(1000);
           channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
	}
```

自动模式：
```java
	// 定义队列的消费者
	QueueingConsumer consumer = new QueueingConsumer(channel);
	// 监听队列
	channel.basicConsume(QUEUE_NAME, true, consumer);
```

**3 Publish/Subscribe**
订阅模式。 
生产者把消息发送到交换机，队列绑定到交换机上，消费者即可通过队列获取消息，可以达到一个消息被多个消息者获取的目的，不需要再从队列中获取消息，解除队列与交换机的绑定关系即可。
X(Exchanges)：交换机
![](/img/rabbitmq/java-three.png)

> 消息发送到交换机，没有队列绑定到交换机时，消息将丢失，交换机没有存储消息的能力。

场景：商品进行了添加或更新，需要通知给某前台系统刷新缓存，通知给某搜索系统更新数据刷新缓存
生产者：
```java
public class Send {
    private final static String EXCHANGE_NAME = "test_exchange_fanout";
    public static void main(String[] argv) throws Exception {
        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明exchange
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        // 消息内容
        String message = "商品已经被更新，id=1001";
        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        System.out.println(" 后台系统： '" + message + "'");

        channel.close();
        connection.close();
    }
}
```

消费者一：
```java
public class Recv {
    private final static String QUEUE_NAME = "test_queue_ps_1";
    private final static String EXCHANGE_NAME = "test_exchange_fanout";
    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 绑定队列到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 同一时刻服务器只会发一条消息给消费者
        channel.basicQos(1);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列，手动返回完成
        channel.basicConsume(QUEUE_NAME, false, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" 前台系统： '" + message + "'");
            Thread.sleep(10);

            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    }
}
```

消费者二：
```java
public class Recv2 {
    private final static String QUEUE_NAME = "test_queue_ps_2";
    private final static String EXCHANGE_NAME = "test_exchange_fanout";
    public static void main(String[] argv) throws Exception {

        // 获取到连接以及mq通道
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 绑定队列到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 同一时刻服务器只会发一条消息给消费者
        channel.basicQos(1);

        // 定义队列的消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        // 监听队列，手动返回完成
        channel.basicConsume(QUEUE_NAME, false, consumer);

        // 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println("搜索系统： '" + message + "'");
            Thread.sleep(10);

            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    }
}
```

运行结果：消费者一和消费者二 都会收到该消息

	 后台系统： '商品已经被更新，id=1001'//生产者
	 搜索系统： '商品已经被更新，id=1001'
	 前台系统： '商品已经被更新，id=1001'

**4 Routing**
![](/img/rabbitmq/java-four.png)

**5 Topics**
![](/img/rabbitmq/java-five.png)

**6 RPC**
远程调用，这种模式，严格意义上来讲，不算是消息队列。可以使用专门的RPC服务框架(比如dubbo：[http://dubbo.io/](http://dubbo.io/))
![](/img/rabbitmq/java-six.png)

获取连接的类 `ConnectionUtil`:
```java
public class ConnectionUtil {
	public static Connection getConnection() throws Exception {
		// 定义连接工厂
		ConnectionFactory factory = new ConnectionFactory();
		// 设置服务地址
		factory.setHost("localhost");
		// 端口
		factory.setPort(5672);
		// 设置账号信息，用户名、密码、vhost
		factory.setVirtualHost("/taotao");
		factory.setUsername("taotao");
		factory.setPassword("taotao");
		// 通过工程获取连接
		Connection connection = factory.newConnection();
		return connection;
	}
}
```
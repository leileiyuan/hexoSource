---
title: spring AMQP对RabbitMQ的支持
date: 2017-01-03 10:44:30
tags: 
 - RabbitMQ
 - spring
categories: RabbitMQ
---

AMQP(Adanced Mesage Queuing Protocol)：高级消息队列协议。是一个异步消息传递所使用的应用层协议规范。AMQP客户端能够无视消息的来源任意发送和接受消息。spirng-AMQP简化了对MQ的使用，目前还只是支持RabbitMQ。
<!-- more -->
与spirng整合使用:
消费者：
```java
public class Foo {
    //具体执行业务的方法
    public void listen(String foo) {
        System.out.println("消费者： " + foo);
    }
}
```
生产者：
```java
public class SpringMain {
	public static void main(final String... args) throws Exception {
		AbstractApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:spring/rabbitmq-context.xml");
		// RabbitMQ模板
		RabbitTemplate template = ctx.getBean(RabbitTemplate.class);
		// 发送消息
		template.convertAndSend("Hello, world!");
		Thread.sleep(1000);// 休眠1秒
		ctx.destroy(); // 容器销毁
	}
}
```

代码很简单，那么所有关联关系，队列，交换机等，都在spring配置文件中处理：
1. 定义RabbitMQ的连接工厂`connectionFactory`，连接到RabbitMQ服务器；
2. 定义Rabbit模板`amqpTemplate`，该模板就是操作RabbitMQ的入口，用来发送消息等
3. Rabbit模板`amqpTemplate`所依赖的内容有：连接工厂`connectionFactory`，交换机`fanoutExchange`
4. 将队列`myQueue`绑定到交换机`fanoutExchange`
5. 定义消费者`foo`来监听消息，一有消息马上接受处理。
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:rabbit="http://www.springframework.org/schema/rabbit"
	xsi:schemaLocation="http://www.springframework.org/schema/rabbit
	http://www.springframework.org/schema/rabbit/spring-rabbit-1.4.xsd
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-4.1.xsd">

	<!-- 定义RabbitMQ的连接工厂 -->
	<rabbit:connection-factory id="connectionFactory"
		host="127.0.0.1" port="5672" username="taotao" password="taotao" virtual-host="/taotao" />

	<!-- 定义Rabbit模板，指定连接工厂以及定义exchange -->
	<rabbit:template id="amqpTemplate" connection-factory="connectionFactory"
		exchange="fanoutExchange" />
	<!-- <rabbit:template id="amqpTemplate" connection-factory="connectionFactory" 
			exchange="fanoutExchange" routing-key="foo.bar" /> -->

	<!-- MQ的管理，包括队列、交换器等 -->
	<rabbit:admin connection-factory="connectionFactory" />

	<!-- 定义队列，自动声明 -->
	<rabbit:queue name="myQueue" auto-declare="true" durable="false" />

	<!-- 定义交换器，自动声明 -->
	<rabbit:fanout-exchange name="fanoutExchange"
		auto-declare="true" durable="false">
		<rabbit:bindings>
			<rabbit:binding queue="myQueue" />
		</rabbit:bindings>
	</rabbit:fanout-exchange>

	<!-- <rabbit:topic-exchange name="myExchange">
			<rabbit:bindings> 
				<rabbit:binding  queue="myQueue" pattern="foo.*" /> 
			</rabbit:bindings> 
		 </rabbit:topic-exchange> -->

	<!-- 队列监听 -->
	<rabbit:listener-container connection-factory="connectionFactory">
		<rabbit:listener ref="foo" method="listen" queue-names="myQueue" />
	</rabbit:listener-container>

	<bean id="foo" class="com.belongtou.rabbitmq.spring.Foo" />

</beans>
```


---
title: redis分片式集群，整合spring
date: 2016-11-15 15:29:28
tags:
 - redis
 - spring
categories: redis

---

redis3.0之前使用的集群方式为 分片式集群。分片集群采用key的哈希值，来决定数据存储在那一台服务器上。分片集群依赖服务器的数量来进行key的哈希值计算，无法动态增加或减少服务节点。
<!-- more -->
#### 分片式集群测试
```java
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisShardInfo;
import redis.clients.jedis.ShardedJedis;
import redis.clients.jedis.ShardedJedisPool;

/** 集群式的连接池 */
public class ShardedJedisPoolDemo {

    public static void main(String[] args) {
        // 构建连接池配置信息
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        // 设置最大连接数
        poolConfig.setMaxTotal(50);

        // 定义集群信息
        List<JedisShardInfo> shards = new ArrayList<JedisShardInfo>();
        // 一个服务，多个节点。。
        shards.add(new JedisShardInfo("127.0.0.1", 6379));
        shards.add(new JedisShardInfo("192.168.0.20", 6379));

        // 定义集群连接池
        ShardedJedisPool shardedJedisPool = new ShardedJedisPool(poolConfig, shards);
        ShardedJedis shardedJedis = null;
        try {
            // 从连接池中获取到jedis分片对象
            shardedJedis = shardedJedisPool.getResource();
            /* for (int i=1;i<=20;i++) {
				shardedJedis.set("key" + i, "vlaue" + i);
			}*/
            // 从redis中获取数据
           String value = shardedJedis.get("key6");
           System.out.println(value);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (null != shardedJedis) {
                // 关闭，检测连接是否有效，有效则放回到连接池中，无效则重置状态
                shardedJedis.close();
            }
        }

        // 关闭连接池
        shardedJedisPool.close();
    }
}
```

存储的数据会以key计算哈希，存储到两台服务器上。获取的时候会以key计算哈希从两台服务器上取

#### 整合spring
与spring整合，就是构建分片式集群的上下分环境，创建出相应的数据对象，配置对象等，通过spring注入的方式来获取到即可。
按照上面的代码逻辑，外围执行环境是**ShardedJedisPool**，ShardedJedisPool构建时需要有两个参数，配置信息 和 集群信息

	JedisPoolConfig poolConfig = new JedisPoolConfig(); // 配置信息
	List<JedisShardInfo> shards = new ArrayList<JedisShardInfo>(); // 集群信息
	shards 中有多个JedisShardInfo
	
	ShardedJedisPool shardedJedisPool = new ShardedJedisPool(poolConfig, shards);

applicationContext-redis.xml配置文件
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd">

	<!-- redis连接池配置 -->
	<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
		<property name="maxTotal" value="${redis.maxTotal}"></property>
	</bean>

	<!-- redis 分片式集群 连接池 -->
	<bean class="redis.clients.jedis.ShardedJedisPool">
		<constructor-arg index="0" ref="jedisPoolConfig" />
		<constructor-arg index="1">
			<list>
				<bean class="redis.clients.jedis.JedisShardInfo">
					<constructor-arg index="0" value="${redis.node1.host}" />
					<constructor-arg index="1" value="${redis.node1.port}" />
				</bean>
			</list>
		</constructor-arg>
	</bean>
</beans>
```

string类型的  set 和 get 操作
```java
@Service
public class RedisService {

	@Autowired
	private ShardedJedisPool jedisPool;

	/**
	 * set 操作
	 * @param key
	 * @param value
	 * @return
	 */
	public String set(String key, String value) {
		ShardedJedis shardedJedis = null;
		try {
			shardedJedis = jedisPool.getResource();
			return shardedJedis.set(key, value);
		} finally {
			if (shardedJedis != null) {
				// 关闭，检测连接是否有效，有效则放回到连接池中，无效则重置状态
				shardedJedis.close();
			}
		}
	}
	
	/**
	 * get 操作
	 * @param key
	 * @return
	 */
	public String get(String key) {
		ShardedJedis shardedJedis = null;
		try {
			shardedJedis = jedisPool.getResource();
			return shardedJedis.get(key);
		} finally {
			if (shardedJedis != null) {
				// 关闭，检测连接是否有效，有效则放回到连接池中，无效则重置状态
				shardedJedis.close();
			}
		}
	}
}
```

#### 模板性代码封装
上的的代码，重复太多，模板性的代码提出来。
定义接口**Function**
```java
public interface Function<T, E> {
	/**
	 * T 返回 类型，不确定，E 入参 不确定
	 * @param e
	 * @return
	 */
	public T calback(E e);
}
```
RedisService中。模板性代码提到**execute**方法中，入参为**Function**类型对象，可以确定入参类型为**ShardedJedis**。
```java
	private <T> T execute(Function<T, ShardedJedis> fun){
		ShardedJedis shardedJedis = null;
		try {
			shardedJedis = jedisPool.getResource();
			return fun.calback(shardedJedis);
		} finally {
			if (shardedJedis != null) {
				// 关闭，检测连接是否有效，有效则放回到连接池中，无效则重置状态
				shardedJedis.close();
			}
		}
	}
```

进一步确定返回类型
**set**方法中调用execute，直接返回，那么返回类型确定 为String。
别外，用人话讲，是因为作用域的问题，**set**方法的入参 需要用**final**关键字修饰。官方话讲，为了防止在调用外部变量的时候，该变量引用被修改，导致出现无法预料的问题。
```java
	/** set 操作 */
	public String set(final String key,final String value) {
		return this.execute(new Function<String, ShardedJedis>() {
			@Override
			public String calback(ShardedJedis e) {
				return e.set(key, value);
			}
		});
	}
```

内部类中引用外部类的局部变量，需要用**final**修饰，这是合乎逻辑的：内部类执行完，局部变量销毁，外部类回调的时候该变量已经不存在了。“防止该引用被修改”。

#### 代码预览
```java
@Service
public class RedisService {

	@Autowired
	private ShardedJedisPool jedisPool;

	/**
	 * set 操作
	 * 
	 * @param key
	 * @param value
	 * @return
	 */
	public String set(final String key, final String value) {
		return this.set(key, value, null);
	}

	/**
	 * set 操作 ，并设置 生存时间
	 * 
	 * @param key
	 * @param value
	 * @param seconds
	 * @return
	 */
	public String set(final String key, final String value, final Integer seconds) {
		return this.execute(new Function<String, ShardedJedis>() {
			@Override
			public String calback(ShardedJedis e) {
				String str = e.set(key, value);
				if (seconds != null) {
					e.expire(key, seconds);
				}
				return str;
			}
		});
	}

	/**
	 * get 操作
	 * 
	 * @param key
	 * @return
	 */
	public String get(final String key) {
		return this.execute(new Function<String, ShardedJedis>() {
			@Override
			public String calback(ShardedJedis e) {
				return e.get(key);
			}
		});
	}

	/**
	 * 删除 操作
	 * 
	 * @param key
	 * @return
	 */
	public Long del(final String key) {
		return this.execute(new Function<Long, ShardedJedis>() {
			@Override
			public Long calback(ShardedJedis e) {
				return e.del(key);
			}
		});
	}

	/**
	 * 设置生存时间
	 * 
	 * @param key
	 * @param seconds
	 *            生存时间
	 * @return
	 */
	public Long expire(final String key, final int seconds) {
		return this.execute(new Function<Long, ShardedJedis>() {
			@Override
			public Long calback(ShardedJedis e) {
				return e.expire(key, seconds);
			}
		});
	}

	private <T> T execute(Function<T, ShardedJedis> fun) {
		ShardedJedis shardedJedis = null;
		try {
			shardedJedis = jedisPool.getResource();
			return fun.calback(shardedJedis);
		} finally {
			if (shardedJedis != null) {
				// 关闭，检测连接是否有效，有效则放回到连接池中，无效则重置状态
				shardedJedis.close();
			}
		}
	}

}

```

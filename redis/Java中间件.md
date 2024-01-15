## Jedis

Jedis 是 Redis 官方推荐的 java 连接开发工具。使用 Java 操作 Redis 中间件!（就是一个jar包）。

jedis的特点：本身的方法就是对应的redis的命令，操作起来十分方便。



### 概述

#### 依赖

```java
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>4.3.2</version>
        </dependency>
```

#### 连接

```java
        // 建立连接 ip 端口
        jedis = new Jedis("xxx.xxx.xxx.xxx", 6379);
        // 密码
        jedis.auth("xxxxxx");
        // 选择数据库
        jedis.select(0);

        // 命令操作
        String result = jedis.set("name", "apple");
        System.out.println(result);
        System.out.println(jedis.get("name"));

        // 断开连接
        if (jedis != null) {
            jedis.close();
        }
```

注意：在 Jedis 实例创建好之后，Jedis 底 层实际会创建一个到指定 Redis 服务器的 Socket 连接。所以，为了节省系统资源与网络带宽， 在每次使用完 Jedis 实例之后，**需要立即调用 close()方法将连接关闭**。

### 连接池

#### JedisPool

Jedis本身是**线程不安全**的，并且频繁的创建和销毁连接会有性能损耗，因此我们推荐大家使用Jedis连接池代替Jedis的直连方式。

**创建连接池工厂类**

```java
class JedisConnectionFactory {
    private static JedisPool jedisPool;
    static {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(8);
        poolConfig.setMaxIdle(8);
        poolConfig.setMinIdle(0);
        jedisPool = new JedisPool(poolConfig, "xxx.xxx.xxx.xxx", 6379, 1000, "xxxxxx");
    }
    public static Jedis getJedis() {
        return jedisPool.getResource();
    }
}
```

**使用连接池**

```java
// 获取连接
try(Jedis jedis = JedisConnectionFactory.getJedis()) {
    // 执行操作
}
// 关键连接
jedis.close();
```

#### JedisPooled

对 Redis 的操作都需要使用 try-with-resource 块是比较麻烦的，可以使用 JedisPooled。

```java
private JedisPooled jedisPooled = new JedisPooled("xxx.xxx.xxx.xxx", 6379);
```

## SpringDataRedis

SpringData是Spring中数据操作的模块，包含对各种数据库的集成，其中对Redis的集成模块就叫做SpringDataRedis。

- 提供了对不同Redis客户端的整合（Lettuce和Jedis）；
- 提供了RedisTemplate统一API来操作Redis；
- 支持Redis的发布订阅模型；
- 支持Redis哨兵和Redis集群；
- 支持基于Lettuce的响应式编程；
- 支持基于JDK、JSON、字符串、Spring对象的数据序列化及反序列化；
- 支持基于Redis的JDKCollection实现；

SpringDataRedis中提供了`RedisTemplate`工具类，其中封装了各种对Redis的操作。并且将不同数据类型的操作API封装到了不同的类型中：

![](typora文档图片/UFlNIV0.png)




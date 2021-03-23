# Spring-Data-Redis 2.X以上版本使用心得和一些坑 #

  最近在修改之前旧项目的时候，将spring-data-redis的版本升级到了2.X以上，查看了官方的文档之后，发现新版本有一些新特性和新的使用方法，这里记录整理一下，并附上自己在使用的时候遇到的一点坑。 spring-data-redis最新版官方文档

spring-redis.xml配置（spring整合spring-data-redis）

```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.2.xsd">

    <!-- Redis连接池的配置 -->
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <!-- 控制一个pool可以分配多少个jedis实例 -->
        <property name="maxTotal" value="${redis.pool.maxActive}" />
        <!-- 连接池中最多可空闲链接个数，这里取值20，表示即使没有用数据库链接依然保持20个空闲链接 -->
        <property name="maxIdle" value="${redis.pool.maxIdle}" />
        <!-- 最大等待时间，当没有可用连接时，连接池等待链接被归还的最大时间ms，超过时间就抛出异常 -->
        <property name="maxWaitMillis" value="${redis.pool.maxWait}" />
        <!-- 在获取连接的时候检查链接的有效性 -->
        <property name="testOnBorrow" value="${redis.pool.testOnBorrow}" />
    </bean>

    <!-- 配置redis连接密码 -->
    <bean id="redisPassword" class="org.springframework.data.redis.connection.RedisPassword">
        <constructor-arg name="thePassword" value="${redis.auth}"></constructor-arg>
    </bean>

    <!-- redis单机配置，地址等在这配置 2.0以上的新特性 -->
    <bean id="redisStandaloneConfiguration" class="org.springframework.data.redis.connection.RedisStandaloneConfiguration">
        <property name="hostName" value="${redis.hostname}"/>
        <property name="port" value="${redis.port}"/>
        <property name="password" ref="redisPassword"/>
        <property name="database" value="${redis.database}"/>
    </bean>

    <!--配置jedis链接工厂 spring-data-redis2.0中
       建议改为构造器传入一个RedisStandaloneConfiguration  单机
                           RedisSentinelConfiguration  主从复制
                           RedisClusterConfiguration  集群-->
    <bean id="connectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <constructor-arg name="standaloneConfig" ref="redisStandaloneConfiguration"></constructor-arg>
    </bean>

    <!--手动设置 key  与 value的序列化方式-->
    <!-- 序列化器：能够把我们储存的key与value做序列化处理的对象，是一个类似于converter的工具。
           可以实现传入的java对象->redis可以识别的数据类型。 如：字符串。
           默认的Serializer是StringRedisSerializer。
           设定默认的序列化器是字符串序列化器，原因是redis可以存储的数据只有字符串和字节数组。
           一般来说，我们代码中操作的数据对象都是java对象。
           如果代码中，使用的数据载体就是字符串对象，那么使用Jackson2JsonRedisSerializer来做序列化器是否会有问题？
           如果jackson插件的版本不合适，有错误隐患的话，可能将字符串数据转换为json字符串 -> {chars:[], bytes:[]}
           使用StringRedisSerializer就没有这个问题。不处理字符串转换的。认为代码中操作的key和value都是字符串。
        -->
    <!-- 配置默认的序列化器 -->
    <!-- keySerializer、valueSerializer 配置Redis中的String类型key与value的序列化器 -->
    <!-- HashKeySerializer、HashValueSerializer 配置Redis中的Hash类型key与value的序列化器 -->
    <bean id="keySerializer" class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
    <bean id="valueSerializer" class="org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer"/>

    <!-- 配置jedis模板 -->
    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="keySerializer" ref="keySerializer"/>
        <property name="valueSerializer" ref="valueSerializer"/>
        <property name="hashKeySerializer" ref="keySerializer"/>
        <property name="hashValueSerializer" ref="valueSerializer"/>
    </bean>

</beans>
```

  这里首先有几点注意的地方：

对redis.properties文件的扫描我放到了spring-dao.xml中载入，由于web.xml中加载了，数据是共享，这里属于SSM整合的知识了。

```
    <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:redis.properties</value>
                <value>classpath:jdbc.properties</value>
            </list>
        </property>
    </bean>
```

 2.在这里我只配置了一个RedisTemplate用来操作redis，而且序列化器也只使用了一种String类型的，因为在后续的业务中，将使用Jackson来对list，set和hash等格式的数据进行处理，先转换成json，再转换成String，取的时候将String利用jackson封装成List等即可。

RedisUtil封装操作

  虽然RedisTemplate提供了很多操作的API，但毕竟只是API，这里将一些常用操作进行封装并加入logger日志记录，方便后面进行定位。

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Component;

import java.util.Set;
import java.util.concurrent.TimeUnit;


/**
 * 对SDR接口进行封装的工具类，来对redis进行操作
 */
@Component
public class RedisUtil {

    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 缓存value
     *
     * @param key
     * @param value
     * @param time
     * @return
     */
    public boolean cacheValue(String k, String value, long time) {
        String key = k;
        try {
            ValueOperations<String, String> valueOperations = redisTemplate.opsForValue();
            valueOperations.set(key, value);
            if (time > 0) {
                // 如果有设置超时时间的话
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return true;
        } catch (Throwable e) {
            logger.error("缓存[" + key + "]失败, value[" + value + "] " + e.getMessage());
        }
        return false;
    }

    /**
     * 缓存value，没有设置超时时间
     *
     * @param k
     * @param value
     * @return
     */
    public boolean cacheValue(String k, String value) {
        return cacheValue(k, value, -1);
    }

    /**
     * 判断缓存是否存在
     *
     * @param key
     * @return
     */
    public boolean containsKey(String key) {
        try {
            return redisTemplate.hasKey(key);
        } catch (Throwable e) {
            logger.error("判断缓存是否存在时失败key[" + key + "]", "err[" + e.getMessage() + "]");
        }
        return false;
    }

    /**
     * 根据key，获取缓存
     *
     * @param key
     * @return
     */
    public String getValue(String key) {
        try {
            ValueOperations<String, String> valueOperations = redisTemplate.opsForValue();
            return valueOperations.get(key);
        } catch (Throwable e) {
            logger.error("获取缓存时失败key[" + key + "]", "err[" + e.getMessage() + "]");
        }
        return null;
    }

    /**
     * 移除缓存
     *
     * @param key
     * @return
     */
    public boolean removeValue(String key) {
        try {
            redisTemplate.delete(key);
            return true;
        } catch (Throwable e) {
            logger.error("移除缓存时失败key[" + key + "]", "err[" + e.getMessage() + "]");
        }
        return false;
    }

    /**
     * 根据前缀移除所有以传入前缀开头的key-value
     *
     * @param pattern
     * @return
     */
    public boolean removeKeys(String pattern) {
        try {
            Set<String> keySet = redisTemplate.keys(pattern + "*");
            redisTemplate.delete(keySet);
            return true;
        } catch (Throwable e) {
            logger.error("移除key[" + pattern + "]前缀的缓存时失败", "err[" + e.getMessage() + "]");
        }
        return false;
    }

}
```

使用演示

```
	@Override
    @Transactional
    public List<Area> getAreaLsit() {
        String key = AREALISTKEY;
        List<Area> areaList = null;
        // 将list转换成string，利用jackson
        ObjectMapper objectMapper = new ObjectMapper();
        if (!redisUtil.containsKey(key)) {
            // 如果不存在这个缓存,就从数据库取
            areaList = areaDao.queryArea();
            // 转换成string缓存
            String jsonValue = null;
            try {
                jsonValue = objectMapper.writeValueAsString(areaList);
            } catch (JsonProcessingException e) {
                e.printStackTrace();
                logger.error("Json转换失败" + e.toString());
                // 由于开启了事务，这里需要抛异常来触发回滚
                throw new AreaOperationException(e.getMessage());
            }
            // 没有问题就缓存
            redisUtil.cacheValue(key, jsonValue);
        } else {
            // 如果缓存中存在
            String jsonValue = redisUtil.getValue(key);
            // 定义要将json转换成的类型
            JavaType javaType = objectMapper.getTypeFactory()
                    .constructParametricType(ArrayList.class, Area.class);
            try {
                areaList = objectMapper.readValue(jsonValue, javaType);
            } catch (JsonProcessingException e) {
                e.printStackTrace();
                logger.error("Json转换失败" + e.toString());
                // 由于开启了事务，这里需要抛异常来触发回滚
                throw new AreaOperationException(e.getMessage());
            }
        }
        return areaList;
    }
```

遇到的问题

1.首先遇到了一个版本不兼容的问题，一开始我用的版本如下

    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.9.0</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-redis</artifactId>
      <version>2.3.0.RELEASE</version>
    </dependency>

但是这两个版本会报一个找不到类的异常

    NoClassDefFoundError: redis/clients/jedis/util/Pool

但是jar包中其实是有这个类的，查询了之后认为是版本不兼容的问题，这里我选择将Spring-Data-Redis降级，改成2.1.9之后就可以正常工作了。

    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.9.0</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-redis</artifactId>
      <version>2.1.9.RELEASE</version>
    </dependency>
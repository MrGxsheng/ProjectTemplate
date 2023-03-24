### 依赖

```xml
        <!--    redis    -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

### RedisConfig

redisConnectionFactory 有红线修改 依赖版本（基本Template 出现红线都是这玩意）

```java
package com.te_springboot.controller;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.RedisSerializer;

/**
 * <p>
 * fw练习生
 * </p>
 *
 * @author sheng
 * @since 2023/3/22 15:33:14
 */

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String,Object> redisTemplate(RedisConnectionFactory redisConnectionFactory){
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        // 连接，这逼东西得放第一位
        template.setConnectionFactory(redisConnectionFactory);

        // hash 用 string
        template.setHashKeySerializer(RedisSerializer.string());
        // key
        template.setKeySerializer(RedisSerializer.string());

        // 默认用json
//        template.setDefaultSerializer(getRedisJsonSerializer());
        template.setDefaultSerializer(RedisSerializer.json());

        // 赋默认值
        template.afterPropertiesSet();
        return template;
    }
}

```

### RedisUtil

```java
package com.te_springboot.utils;

import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.time.Duration;
import java.util.Map;
import java.util.Set;
import java.util.function.Consumer;
/**
 * Redis
 *
 * <p>
 * fw练习生
 * </p>
 *
 * @author sheng
 * @since 2023/3/22 15:11:07
 */



@Component
@Slf4j
public class RedisUtil {

    // 自增带过期的默认值
    public static final int DEFAULT_INCR_INT_VALUE = 1;

    // 默认锁失效时间
    public static final int DEFAULT_LOCK_TIMEOUT = 233;

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    // 获取键
    public Set<String> getKeys(String prefix) {
        return redisTemplate.keys(prefix + "*");
    }

    // 取值
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    // 类型转换取值
    public <T> T get(String key, Class<T> t) {
        Object o = get(key);
        if (t.isInstance(o)) {
            return t.cast(o);
        }
        return null;
    }

    // 设置值
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    // 定时缓存
    public void set(String key, Object value, long second) {
//        redisTemplate.opsForValue().set(key, value, timeout);
        redisTemplate.opsForValue().set(key, value, Duration.ofSeconds(second));
    }

    // 自增定时数据，首次创建定时
    public void setIncrExp(String key, long timeout) {
        // TODO 这样写太不讲究了，回头改了
        setIncrExp(key, timeout, (k) -> log.info("创建自增定期键：{}，过期时间{}秒", k, timeout));
    }

    // 同上，创建时带一个回调函数，不清楚是不是叫回调，回头从新看一遍java吧
    public void setIncrExp(String key, long timeout, Consumer<String> consumer) {
        if (GeneralUtil.isNull(get(key))) {
            // 不存在创建
            set(key, DEFAULT_INCR_INT_VALUE, timeout);
            consumer.accept(key);
            return;
        }

        // 存在直接自增
        redisTemplate.opsForValue().increment(key);
    }

    // 设置过期时间
    public Boolean expire(String key, long second) {
        return redisTemplate.expire(key, Duration.ofSeconds(second));
    }

    // 获取hash
    public Map<Object, Object> getHash(String key) {
        return redisTemplate.opsForHash().entries(key);
    }

    // 获取hash
    public Object getHash(String key, Object index) {
        return redisTemplate.opsForHash().get(key, index);
    }

    // 获取hash 转类型
    public <T> T getHash(String key, Object index, Class<T> clazz) {
        return clazz.cast(getHash(key, index));
    }

    // 设置hash
    public void setHash(String key, Object k, Object v) {
        redisTemplate.opsForHash().put(key, k, v);
    }

    // 自增hash
    public Long addHash(String key, Object k, long step) {
        return redisTemplate.opsForHash().increment(key, k, step);
    }

    // 获取全部
    public Set<Object> getSet(String key) {
        return redisTemplate.opsForSet().members(key);
    }

    // 成员是否存在set
    public Boolean getSet(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }

    public void setSet(String key, Object... value) {
        redisTemplate.opsForSet().add(key, value);
    }

    // 删除key
    public Boolean delKey(String key){
        return redisTemplate.delete(key);
    }

    // 加锁
    public Boolean lock(String key) {
        return lock(key, DEFAULT_LOCK_TIMEOUT);
    }

    // 带过期的加锁
    public Boolean lock(String key, long ms) {
        return redisTemplate.opsForValue().setIfAbsent(key, System.currentTimeMillis(), Duration.ofMillis(ms));
    }

    // 释放锁
    public Boolean unlock(String key){
        return delKey(key);
    }

}

```

### application.yaml

```yaml
  redis:
    port: 6379
    host: localhost
```


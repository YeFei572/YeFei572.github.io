---
layout: post
tags: 小程序 SpringBoot Redis
categories: 技术分享
title:  "springboot项目集成redis服务器(小程序的access_token存储到中控服务器上)记"
---


> 使用springboot框架集成, 因为涉及到业务方面的代码, 本篇博文没有写怎么获取access_token ,获取access_token的方法网上一大片, 随便copy一个就可以了, 本文主要讲解如何集成redis, 然后写入,查询,实现多个地方共享access_token

### 一、引进依赖

 `springboot`框架集成任何技术都需要从导入依赖开始, `redis`也不会例外, 在 **common** 模块中的`gradle.build` 文件的`dependencies`下面引入 `redis`的依赖:
 
 ```
  dependencies {
  
	// https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-redis
	compile group: 'org.springframework.boot', name: 'spring-boot-starter-data-redis', version: '2.0.4.RELEASE'
	// https://mvnrepository.com/artifact/org.springframework.data/spring-data-redis
	compile group: 'org.springframework.data', name: 'spring-data-redis', version: '2.0.7.RELEASE'
  }
 ```
 一共是2个依赖包, 一个是 `spring-boot-starter-data-redis`, 另外一个是 `spring-data-redis`;
 
 ### 二、配置 **redisConfig.java** 文件
 
  在 **api** 模块中建立包名`config`, 并在其下面创建`redis`的配置文件:
  
  ![_20181204173943png](https://pic.v2ss.cn/qiniuyun/12730c329ec349778c48695f6d78812d__20181204173943.png) 
  
  ```java
  package com.keppel.config;

  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.data.redis.connection.RedisConnectionFactory;
  import org.springframework.data.redis.core.*;
  import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
  import org.springframework.data.redis.serializer.StringRedisSerializer;

  @Configuration
  public class RedisConfig {
  
	@Autowired
	RedisConnectionFactory redisConnectionFactory;

	@Bean
	public RedisTemplate, Object> functionDomainRedisTemplate() {
		RedisTemplate, Object> redisTemplate = new RedisTemplate<>();
		initDomainRedisTemplate(redisTemplate, redisConnectionFactory);
		return redisTemplate;
	}

	private void initDomainRedisTemplate(RedisTemplate, Object> redisTemplate, RedisConnectionFactory factory) {
		redisTemplate.setKeySerializer(new StringRedisSerializer());
		redisTemplate.setHashKeySerializer(new StringRedisSerializer());
		redisTemplate.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
		redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
		redisTemplate.setConnectionFactory(factory);
	}

	@Bean
	public HashOperations, String, Object> hashOperations(RedisTemplate, Object> redisTemplate) {
		  return redisTemplate.opsForHash();
	}

	@Bean
	public ValueOperations, Object> valueOperations(RedisTemplate, Object> redisTemplate) {
		  return redisTemplate.opsForValue();
	}

	@Bean
	public ListOperations, Object> listOperations(RedisTemplate, Object> redisTemplate) {
		  return redisTemplate.opsForList();
	}

	@Bean
	public SetOperations, Object> setOperations(RedisTemplate, Object> redisTemplate) {
		  return redisTemplate.opsForSet();
	}

	@Bean
	public ZSetOperations, Object> zSetOperations(RedisTemplate, Object> redisTemplate) {
		  return redisTemplate.opsForZSet();
	}
  }
  
  ```
  
### 三、配置application.yml文件
  配置文件加入以下配置
  ```
  server:
	port: 8080

  spring:
	redis: 
	  host: 220.49.145.221
	  port: 6379
	  password: xxxxxxx
	  database: 1
	  timeout: 5000
  ```
### 四、实际代码操控
首先, 定义一个AccessToken实体类, 注意,以下代码均使用了lombok插件,自动生成get/set方法:

```java
package com.keppel.user.entity;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;
 
/**
* @Author: YeFei
* @Date: Created in 18:33 2018/11/27
* @Description:
*/
@Data
public class AccessToken implements Serializable {
    private String redisKey;
    private Date createAt;
    private String accessToken;
    private Date expireAt;
}
```

创建基本的IRedisService,实现增删改查等乱七八糟的方法
```java
  package com.keppel.user.service.redis;

  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.data.redis.core.HashOperations;
  import org.springframework.data.redis.core.RedisTemplate;

  import javax.annotation.Resource;
  import java.util.List;
  import java.util.Set;
  import java.util.concurrent.TimeUnit;

  /**
   * @Author: YeFei
   * @Date: Created in 18:35 2018/11/27
   * @Description: 
   */
  public abstract class IRedisService<T> {
	  @Autowired
	  protected RedisTemplate, Object> redisTemplate;
	  @Resource
	  protected HashOperations, String, T> hashOperations;

	  /**
	  * 存入redis中的key
	  * @return
	  */
	  protected abstract String getRedisKey();

	 /**
	  * 添加
	  * @param key key
	  * @param doamin 对象
	  * @param expire 过期时间(单位:秒),传入 -1 时表示不设置过期时间
	  */
	  public void put(String key, T doamin, long expire) {
		  hashOperations.put(getRedisKey(), key, doamin);
		  if (expire != -1) {
				redisTemplate.expire(getRedisKey(), expire, TimeUnit.SECONDS);
		  }
	  }

	  public void remove(String key) {
			hashOperations.delete(getRedisKey(), key);
	  }

	  public T get(String key) {
			return hashOperations.get(getRedisKey(), key);
	  }

	  public List<T> getAll() {
			return hashOperations.values(getRedisKey());
	  }

	  public Set getKeys() {
			return hashOperations.keys(getRedisKey());
	  }

	  public boolean isKeyExists(String key) {
			return hashOperations.hasKey(getRedisKey(), key);
	  }

	  public long count() {
			return hashOperations.size(getRedisKey());
	  }

	  public void empty() {
		  Set set = hashOperations.keys(getRedisKey());
		  set.stream().forEach(key -> hashOperations.delete(getRedisKey(), key));
	  }
  }
```
接着, 创建redis的接口, 使接口实现IReadService接口
```java
package com.keppel.user.service.redis;

import com.keppel.user.entity.AccessToken;
import org.springframework.stereotype.Service;

/**
* Created by Administrator on 2017/3/1 16:00. 
*/
@Service
public class RedisServiceImpl extends IRedisService {
    private static final String REDIS_KEY = "TEST_REDIS_KEY";

    @Override
    protected String getRedisKey() {
        return this.REDIS_KEY;
    }
}
```

### 五、最后新增api添加入口

  ```java
  package com.keppel.api.user;

  import com.keppel.user.entity.AccessToken;
  import com.keppel.user.entity.User;
  import com.keppel.user.service.UserService;
  import com.keppel.user.service.redis.RedisServiceImpl;
  import com.keppel.util.DateUtil;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.web.bind.annotation.*;

  import java.util.Date;

  /**
  * @Author: YeFei
  * @Date: Created in 0:43 2018/11/24
  * @Description:
  */
  @RestController
  @RequestMapping(value = "/api/v1")
  public class UserCtrl {

	@Autowired
	private UserService userService;
	@Autowired
	private RedisServiceImpl redisService;

	@RequestMapping(value = "/save", method = RequestMethod.POST)
	public String saveUser(
		@RequestBody User user
	) {
		return userService.saveUser(user);
	}

	//查询所有对象
	@RequestMapping(value = "/redis/saveRedis", method = RequestMethod.GET)
	public Object saveRedis() {
		AccessToken accessToken = new AccessToken();
		accessToken.setRedisKey("access_token");
		accessToken.setAccessToken("this is accessToken content1");
		accessToken.setCreateAt(new Date());
		accessToken.setExpireAt(DateUtil.getDateAfterMinutes(1, new Date()));
		redisService.put(accessToken.getRedisKey(), accessToken, -1);
		return redisService.getAll();
	}

	//查询所有对象
	@RequestMapping(value = "/redis/getAll", method = RequestMethod.GET)
	public Object getAll() {
		return redisService.getAll();
	}

	//查询所有对象
	@RequestMapping(value = "/redis/getByKey", method = RequestMethod.GET)
	public Object getByKey(@RequestParam(value = "key")String key) {
		return redisService.get(key);
	}
  }
  ```

大概就是这样了! `AccessToken`存到了中控服务器, 就在也不用担心多个服务器同时更新的时候导致`accessToken`变更请求失败!


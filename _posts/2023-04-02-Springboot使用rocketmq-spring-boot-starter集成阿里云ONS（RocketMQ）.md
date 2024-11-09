---
layout: post
tags: RocketMQ
categories: 技术分享
title:  "Springboot使用rocketmq-spring-boot-starter集成阿里云ONS（RocketMQ）"
---
> 最近在抽离专用工具模块，发现一个问题，就是使用阿里云的队列产品ONS，也就是`rocketmq`,因为`ons-client`有臭名昭著的`fastjson`依赖，且最新版的`ons-clinet`还用着老版本的`fastjson`,有一两个漏洞，为了安全起见，就用`apache`官方的组件`rocketmq-spring-boot-starter`依赖，这里记录一下。

### 一、准备依赖
使用最新依赖，没有漏洞的：
```xml
<!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-spring-boot-starter -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.3</version>
</dependency>
```

### 二、编写配置文件
这里编写配置文件有个很大的坑，一不留神就G了：
```yaml
server:
  port: 10222
rocketmq:
  name-server: MQ_INST_111111111111111_AAAAAAA.mq-internet-access.mq-internet.aliyuncs.com:80
  producer:
    group: MQ_INST_111111111111111_AAAAAAA%GID_ffffff_local_test
    access-key: aaaaaaaAAAAAAAAAAAAAAAA
    secret-key: AAAAAAAAAAAAAAaaaaaaaaaaaaaaaa
  consumer:
    group: MQ_INST_111111111111111_AAAAAAA%GID_ffffff_local_test
    access-key: aaaaaaaAAAAAAAAAAAAAAAA
    secret-key: AAAAAAAAAAAAAAaaaaaaaaaaaaaaaa
  access-channel: CLOUD
```
讲一下吧：
- `name-server`这个属性是公网`TCP`地址，在后台管理可以找到，是一个`http://`开头的，把它删喽，只保留后面一部分即可。
- `producer`生产者组配置，`AK`和`SK`就不多少了，直接按照阿里云的配置给就好了，但是有一定要写对，你写错了控制台也不会报错，但是一直显示未上线（消费者），有点坑的是这个群组名称，群组名称如果连接普通`rocketmq`,直接写群组名称即可，但是你连`ONS`就必须要带上实例的`ID`，也就是`name-server`的子域名第一段，然后用`%`连接，再接群组名称，下面消费者亦是如此，这点必须要写对，不然也连不上。

### 三、编写生产者端的代码
生产者代码网上一大堆，随便找一下就可以搜到，这里放个简单`demo`即可:
```java
import lombok.RequiredArgsConstructor;
import org.apache.rocketmq.spring.core.RocketMQTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequiredArgsConstructor
public class TestCtrl {

    private static final String TOPIC = "MQ_INST_111111111111111_AAAAAAA%ffffff_local_test";
    private final RocketMQTemplate rocketMQTemplate;

    @GetMapping("/test")
    public void test() {
        rocketMQTemplate.convertAndSend(TOPIC.concat(":tag01"), "testMessage");
    }
}
```
上面的`TOPIC`也要`实例的ID + % + topic名字`,否则也有问题，发送消息的时候记得带上`tag`即可，其余的就没啥，该咋咋整。

### 四、编写消费者端代码
监听代码也很简单，但是配置一定要整对，不然在阿里云后台的GROUP会一直处于离线，导致消费不到消息：
```java
import lombok.extern.slf4j.Slf4j;
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.springframework.stereotype.Service;

/**
 * mq
 *
 * @author Protected User
 * @date 2023/04/01 10:00
 **/
@Slf4j
@Service
@RocketMQMessageListener(consumerGroup = "MQ_INST_111111111111111_AAAAAAA%GID_ffffff_local_test", topic = "MQ_INST_111111111111111_AAAAAAA%ffffff_local_test")
public class RocketListener implements RocketMQListener<String> {

    @Override
    public void onMessage(String message) {
        log.info("message:==========> {}", message);
    }
}
```

### 五、总结
所有配置要严格按照格式编写，否则容易出不可预知错误。
---
layout: post
tags: WebSocket 微服务
categories: 技术分享
title:  "微服务Websocket（stomp）使用注意点"
---

> 最近公司使用微服务做一个办公系统，其中涉及即时推送技术，采用了基于 `websocket` 的 `stomp` 协议，其中遇到了不少坑儿，写个博文记录一下。

### 1、先上后端代码:

```java
package com.xxx.notice.config;

import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.util.StrUtil;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.provider.OAuth2Authentication;
import org.springframework.security.oauth2.provider.token.RemoteTokenServices;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

/**
 * Author: keppelfei@gmail.com
 * Datetime: 2020/8/17  15:20
 * Description:
 */
@RequiredArgsConstructor
@Slf4j
@Configuration
@EnableWebSocketMessageBroker
public class WebsocketConfig implements WebSocketMessageBrokerConfigurer {

    private final RemoteTokenServices tokenService;

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS();
    }


    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setUserDestinationPrefix("/notice/");
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
                // 判断是否首次连接请求
                if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                    String tokens = accessor.getFirstNativeHeader("Authorization");
                    log.info("webSocket token is {}", tokens);
                    if (StrUtil.isBlank(tokens)) {
                        return null;
                    }
                    // 验证令牌信息
                    OAuth2Authentication auth2Authentication = tokenService.loadAuthentication(tokens.split(" ")[1]);
                    if (ObjectUtil.isNotNull(auth2Authentication)) {
                        SecurityContextHolder.getContext().setAuthentication(auth2Authentication);
                        accessor.setUser(() -> auth2Authentication.getName());
                        return message;
                    } else {
                        return null;
                    }
                }
                //不是首次连接，已经成功登陆
                return message;
            }
        });
    }
}
```

以上是配置文件，下面是主动推送消息：

```java
package com.dikar.imp.notice.handler.rabbitmq;

import cn.hutool.core.collection.CollectionUtil;
import com.alibaba.fastjson.JSON;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.time.LocalDateTime;
import java.util.*;

/**
 * Author: keppelfei@gmail.com
 * Datetime: 2020/8/17  18:13
 * Description:
 */
@Slf4j
@Component
@RequiredArgsConstructor
@RabbitListener(queues = RabbitmqConstants.DIRECT_NOTICE_QUEUE)
public class DirectExchangeHandler {

    private final Sequence idWorker;


    @SuppressWarnings("unchecked")
    @RabbitHandler
    public void receiveMsg(String message) {
        log.info("------------------------------------直连---------------------------------------{}", message);
        NoticeRecordMqDto noticeRecord = JSON.parseObject(message, NoticeRecordMqDto.class);
        if (Objects.isNull(noticeRecord.getId())) {
            noticeRecord.setId(idWorker.nextValue());
        }
        SimpMessagingTemplate simpMessagingTemplate = SpringContextHolder.getBean(SimpMessagingTemplate.class);
        if (Objects.nonNull(noticeRecord.getUserId())) {
            // 开始存储到noticeRecord中去
            noticeRecordMapper.insert(noticeRecord);
            // 开始推送
            if (Objects.equals(NoticeTargetEnum.SYS.getCode(), noticeRecord.getSendTarget())) {
                log.info("----------------------------开始发送推送（直连）-----------------------------");
                try {
                    simpMessagingTemplate.convertAndSendToUser(noticeRecord.getUserId().toString(), "/remind", message);      
                } catch (Exception e) {
                    e.printStackTrace();
                }
               
            }
           
        }
    }
}
```

### 2、再上前端代码：

```js
initWebSocket() {
             this.connection()
             const self = this
             // 断开重连机制,尝试发送消息,捕获异常发生时重连,使用14s尝试一次
             this.timer = setInterval(() => {
                 try {
                     self.stompClient.send('test')
                 } catch (err) {
                     console.log('断线了: ' + err)
                     self.connection()
                 }
             }, 14000)
         },
             connection: function () {
                 const token = store.getters.access_token
                 const TENANT_ID = getStore({name: 'tenantId'}) ? getStore({name: 'tenantId'}) : '1'
                 const headers = {
                     'Authorization': 'Bearer ' + token
                 }
                 // 建立连接对象
                 this.socket = new SockJS('/notice/ws')// 连接服务端提供的通信接口，连接以后才可以订阅广播消息和个人消息
                 // this.socket = new SockJS('/act/ws')// 连接服务端提供的通信接口，连接以后才可以订阅广播消息和个人消息
                 // 获取STOMP子协议的客户端对象
                 this.stompClient = Stomp.over(this.socket)
                 this.stompClient.debug = null
                 // 向服务器发起websocket连接
                 this.stompClient.connect(headers, () => {
                     this.stompClient.subscribe('/notice/' + this.userInfo.id + '/remind', (msg) => { // 订阅服务端提供的某个topic;
                         console.log(msg)
                         // this.stompClient.subscribe('/task/' + this.userInfo.username + '-' + TENANT_ID + '/remind', (msg) => { // 订阅服务端提供的某个topic;
                         let result = JSON.parse(msg.body);
                         this.$notify({
                             title: result.title,
                             type: 'warning',
                             dangerouslyUseHTMLString: true,
                             message: result.content,
                             offset: 60,
                             duration: 10000,        // 10s自动关闭
                         })
                     });
                 }, (err) => {
                     console.error("error ============================> " + err);
                 })
             },
         disconnect() {
             if (this.stompClient != null) {
                 this.stompClient.disconnect()
                 console.log('Disconnected')
             }
         }
```

### 3、对后端代码做一个说明（以下代码全部是上面截取的）：

- 首先后端配置对 `websocket` 的适配，其中要引入 `websocket` 的依赖：

```xml
<!--websocket-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
```

- 然后注册前端 `websocket` 路径 `/ws`

```java
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS();
}
```

- 接着就是 `websocket` 连接路径 `/notice`:

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.setUserDestinationPrefix("/notice/");
}
```

- 再就是配置连接鉴权信息: 重写 `configureClientInboundChannel`方法，开始鉴权。

### 4、对前端代码做一个说明：

- 重点说明一下，如果使用 `nginx` 做代理服务器要注意连接超时时间要设置**大于**此处监听重连时间，否则会报错：
  
  **Whoops! Lost connection to xxxxxx**
- 例子中的连接路径是：`https://localhost:8080/ws/notice/{uid}/remind`
  
  


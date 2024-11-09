---
layout: post
tags: SpringBoot RabbitMQ
categories: 技术分享
title:  "springboot项目集成rabbitmq发送邮件功能"
---


### 一、背景:
> 最近公司有个项目需要使用rabbitmq来做优化, 之前一直听说没有实战, 然后自己在网上找了一些资料, 资料真的很多, 但是实例基本没有, 都是log一句话什么的, 很头痛... 该博文抄了很多博主然后结合自己的了解做了一些改动,实现了我要的效果. 自己折腾一下,权当记录一下!

### 二、rabbitmq介绍

介绍就没有什么必要了, 网上资料一大堆, 说的都很好. 这里主要是使用了 `topic` 类型的交换机, 该类型灵活多变,能满足大部分业务场景.

我这边场景是: 用户注册的时候需要发送一个带公司简介的附件邮件给用户 , 如果附件偏大没有使用rabbitmq就很蛋疼, 要一直等着邮件发送完毕才会返回数据给用户, 体验很差.

说的再多也不如 SHOW ME YOUR CODE.

### 三、项目引入依赖
依次引入rabbitmq和javaMail的依赖

```xml
<!--rabbitMq-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<!--邮件系统-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```


### 四、`application.yml` 配置文件

```yml
# 集成邮件系统, 需要注意的是要开启邮箱的SMTP服务, 网易的163, qq的都需要开启后才能使用-----------------
spring:
  mail:
    host: smtp.qq.com
    port: 587
    username: xxxxxx@qq.com
    password: xxxxxxx
    protocol: smtp
    default-encoding: UTF-8
    
 # 集成rabbitmq------------------------
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    connection-timeout: 60000ms
    # 支持发布确认
    publisher-confirms: true
    # 支持发布返回
    publisher-returns: true
    cache:
      channel:
        size: 1
```

说明一下, 目前测试只是用了qq邮箱, 其他邮箱尚未测试, 应该都ok!

### 五、`rabbitmq` 配置内容

配置内容是rabbitmq的关键点, 主要是两个配置内容,也可以合在一起写, 为了清晰明了就分开写了, 第一个是`rabbitConfig`:

```java
package com.keppel.config.rabbitmq;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

/**
 * @Author: YeFei
 * @Date: Created in 15:03 2019/2/13
 * @Description:
 */
@Configuration
@Slf4j
@Data
@ConfigurationProperties(prefix = "spring.rabbitmq")
public class RabbitConfig {
    private String host;
    private String port;
    private String username;
    private String password;
    private String virtualHost;
    private String connectionTimeout;
    private boolean publisherConfirms;
    private boolean publisherReturns;

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
        connectionFactory.setAddresses(host + ":" + port);
        connectionFactory.setUsername(username);
        connectionFactory.setPassword(password);
        connectionFactory.setVirtualHost(virtualHost);
        connectionFactory.setConnectionTimeout(Integer.parseInt(connectionTimeout
                .substring(0, connectionTimeout.length()-2)));
        // 如果要进行消息回调,则这里必须要设置为true
        connectionFactory.setPublisherConfirms(publisherConfirms);
        connectionFactory.setPublisherReturns(publisherReturns);
        return connectionFactory;
    }

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public RabbitTemplate rabbitTemplate() {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory());
        rabbitTemplate.setEncoding("UTF-8");
        rabbitTemplate.setMandatory(true);
        // TODO 相应交换机接收后异步回调  关闭了回调功能, 此处暂时不知道什么原因,如果开启后就变成同步了
        /*rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("交换机接收信息成功,id:{}", correlationData.getId());
            } else {
                log.error("交换机接收信息失败:{}", cause);
            }
        });
        无相应队列与交换机绑定异步回调
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            String msg = new String(message.getBody());
            log.error("消息:{} 发送失败, 应答码:{} 原因:{} 交换机:{} 路由键:{}", msg, replyCode, replyText, exchange, routingKey);
        });*/
        return rabbitTemplate;
    }
}

```

以上是基本配置, 其中`@Slf4j`和`@Data` 是`lombok`插件, IDEA安装即可, 引入相关依赖, 然后再就是交换机类型配置 `TopicExchageConfig`:

```java
package com.keppel.config.rabbitmq;

import com.keppel.common.data.RabbitConstant;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @Author: feige
 * @Date: Created in 15:57 2019/2/13
 * @Description:
 */
@Configuration
public class TopicExchangeConfig {
    /**
     * 声明队列
     *
     * @return
     */
    @Bean
    public Queue emailQueue() {
        return new Queue(RabbitConstant.EMAIL_QUEUE, true);//true表示持久化该队列
    }

    @Bean
    public Queue httpRequestQueue() {
        return new Queue(RabbitConstant.MESSAGE_QUEUE, true);
    }

    /**
     * 声明交换机
     *
     * @return
     */
    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(RabbitConstant.CONTROL_EXCHANGE);
    }

    /**
     * 绑定
     *
     * @return
     */
    @Bean
    public Binding bindingEmail() {
        return BindingBuilder.bind(emailQueue()).to(topicExchange()).with(RabbitConstant.EMAIL_ROUTING_KEY);//*表示一个词,#表示零个或多个词
    }

    @Bean
    public Binding bindingHttpRequest() {
        return BindingBuilder.bind(httpRequestQueue()).to(topicExchange()).with(RabbitConstant.MESSAGE_ROUTING_KEY);
    }

}
```

最后配置内容再上一个常量类`RabbitConstant`:

```java
package com.keppel.common.data;

/**
 * @Author: feige
 * @Date: Created in 15:56 2019/2/13
 * @Description:
 */
public class RabbitConstant {
    /**
     * 邮件队列
     */
    public static final String EMAIL_QUEUE = "email";
    /**
     * 短信队列
     */
    public static final String MESSAGE_QUEUE = "message";
    /**
     * 邮件队列路由键（*表示一个词,#表示零个或多个词）
     */
    public static final String EMAIL_ROUTING_KEY = "email.key";
    /**
     * 短信队列路由键
     */
    public static final String MESSAGE_ROUTING_KEY = "message.key";
    /**
     * 交换机
     */
    public static final String CONTROL_EXCHANGE = "control.exchange";
}

```

### 六、生产者和消费者

然后就是生产者和消费者的相关代码了, 生产者的使用场景就是保存新用户的时候将消息写到mq中, 消费者写一个对应队列的监听方法即可, 先放上生产者的相关代码:

```java
    @Override
    public int saveUser(User user) {
        long l = System.currentTimeMillis();
        String[] userIds = new String[]{user.getId()};
        send(userIds, RabbitConstant.CONTROL_EXCHANGE, RabbitConstant.EMAIL_ROUTING_KEY);
        log.info("mq消费时间: " + (System.currentTimeMillis() - l));
        int res = userMapper.insert(user);
        log.info("总消耗时间为:" + (System.currentTimeMillis() - l));
        return res;
    }
    private void send(String[] msg, String exchange, String routingKey) {
        rabbitTemplate.convertAndSend(exchange, routingKey, msg);
        log.info("rabbitmq消息已经发送到交换机, 等待交换机接受..." + msg.toString());
    }
```

消费者相关代码:

```java
/**
 * 监听mq队列的消息
 * @param msg
 */
@RabbitListener(queues = RabbitConstant.EMAIL_QUEUE)
public void processEmail(String[] msg) {
    try {
        log.info("准备开始发送邮件......" + msg.toString());
        sendEmail(msg.toString());
    } catch (Exception e) {
        log.error("邮件发送失败了!");
    }
}
```

### 七、另外放一些关于邮件相关代码:

```java
package com.keppel.common.utils.mail;

import lombok.Data;
import org.springframework.core.io.FileSystemResource;

/**
 * @Author: feige
 * @Date: Created in 12:00 2019/2/13
 * @Description:
 */
@Data
public class MailBean {
    /**
     * 邮件主题
     */
    private String subject;

    /**
     * 邮件内容
     */
    private String text;

    /**
     * 附件
     */
    private FileSystemResource file;

    /**
     * 附件名称
     */
    private String attachmentFilename;

    /**
     * 内容ID，用于发送静态资源邮件时使用
     */
    private String contentId;

    public static MailBean getMailBean() {
        return new MailBean();
    }

}
```

控制层方法：

```java
package com.keppel.common.utils.mail;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Component;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;

/**
 * @Author: YeFei
 * @Date: Created in 12:01 2019/2/13
 * @Description:
 */
@Component
public class MailUtil {
    @Autowired
    private JavaMailSender mailSender; // 自动注入的Bean

    @Value("${spring.mail.username}")
    private String sender; // 读取配置文件中的参数

    /**
     * 发送邮件测试方法
     */
    public void sendMail() {
        SimpleMailMessage mimeMessage = new SimpleMailMessage();
        mimeMessage.setFrom(sender);
        mimeMessage.setTo(sender);
        mimeMessage.setSubject("SpringBoot集成JavaMail实现邮件发送");
        mimeMessage.setText("SpringBoot集成JavaMail实现邮件发送正文");
        mailSender.send(mimeMessage);
    }

    /**
     * 发送简易邮件
     * @param mailBean
     */
    public void sendMail(MailBean mailBean) {
        MimeMessage mimeMessage = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage);

        try {
            helper.setFrom(sender);
            helper.setTo(sender);
            helper.setSubject(mailBean.getSubject());
            helper.setText(mailBean.getText());
        } catch (MessagingException e) {
            e.printStackTrace();
        }

        mailSender.send(mimeMessage);
    }

    /**
     * 发送邮件-邮件正文是HTML
     * @param mailBean
     */
    public void sendMailHtml(MailBean mailBean) {
        MimeMessage mimeMessage = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage);

        try {
            helper.setFrom(sender);
            helper.setTo(sender);
            helper.setSubject(mailBean.getSubject());
            helper.setText(mailBean.getText(), true);
        } catch (MessagingException e) {
            e.printStackTrace();
        }

        mailSender.send(mimeMessage);
    }
    /**
     * 发送邮件-附件邮件
     * @param mailBean
     */
    public void sendMailAttachment(MailBean mailBean) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
            helper.setFrom(sender);
            helper.setTo(sender);
            helper.setSubject(mailBean.getSubject());
            helper.setText(mailBean.getText(), true);
            // 增加附件名称和附件
            helper.addAttachment(mailBean.getAttachmentFilename(), mailBean.getFile());
            mailSender.send(mimeMessage);
        } catch (MessagingException e) {
            e.printStackTrace();
        }
    }

    /**
     * 内联资源（静态资源）邮件发送
     * 由于邮件服务商不同，可能有些邮件并不支持内联资源的展示
     * 在测试过程中，新浪邮件不支持，QQ邮件支持
     * 不支持不意味着邮件发送不成功，而且内联资源在邮箱内无法正确加载
     * @param mailBean
     */
    public void sendMailInline(MailBean mailBean) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
            helper.setFrom(sender);
            helper.setTo(sender);
            helper.setSubject(mailBean.getSubject());

            /*
             * 内联资源邮件需要确保先设置邮件正文，再设置内联资源相关信息
             */
            helper.setText(mailBean.getText(), true);
            helper.addInline(mailBean.getContentId(), mailBean.getFile());

            mailSender.send(mimeMessage);
        } catch (MessagingException e) {
            e.printStackTrace();
        }
    }

    /**
     * 模板邮件发送
     * @param mailBean
     */
    public void sendMailTemplate(MailBean mailBean) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
            helper.setFrom(sender);
            helper.setTo(sender);
            helper.setSubject(mailBean.getSubject());
            helper.setText(mailBean.getText(), true);
            mailSender.send(mimeMessage);
        } catch (MessagingException e) {
            e.printStackTrace();
        }
    }
}
```

发送邮件方法：

```java
public InfoResponse sendEmail(String userId) {
    MailBean mailBean = MailBean.getMailBean();

    // 简易邮件发送测试
    // simpleEmail(mailBean);

    // HTML邮件正文发送测试
    // htmlEmail(mailBean);

    // 附件邮件发送测试
    attachEmail(mailBean);
    log.info(userId);
    return new InfoResponse("发送成功!");
}

private void attachEmail(MailBean mailBean) {
    String filePath="C:\\Users\\keppel\\Pictures\\测试一下下.jpg";
    FileSystemResource file = new FileSystemResource(new File(filePath));
    String attachmentFilename = filePath.substring(filePath.lastIndexOf(File.separator));
    mailBean.setSubject("SpringBoot集成JavaMail实现邮件发送");
    mailBean.setText("SpringBoot集成JavaMail实现附件邮件发送");
    mailBean.setFile(file);
    mailBean.setAttachmentFilename(attachmentFilename);
    mailUtil.sendMailAttachment(mailBean);
}
```

大致情况就是这样了!

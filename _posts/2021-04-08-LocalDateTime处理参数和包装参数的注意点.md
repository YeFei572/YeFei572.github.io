---
layout: post
tags: Java
categories: 技术分享
title:  "LocalDateTime处理参数和包装参数的注意点"
---

> 随着`java`的版本迭代更新，1.8已经成为了最火热的、使用最频繁的版本，而在1.8版本上更新的`LocalDateTime`也进入了大众视野，此处记录一下接参和出参包装的配置说明！

### 单独接参说明

此处接参主要说明是`@RequestBody`传参，此时要使用一个注解去格式化参数：

```
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;
```

此时要求前端传到后端的参数就是上面那种格式，不可以是带T什么乱遭的

### 单独出参包装说明

出参包装意思就是说在json包装类上面将格式规定好，然后发给前端，否则前端拿到的数据（时间）就是乱的。此时在包装类上面用一个注解来解决这个问题：

```
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
private LocalDateTime createTime;
```

### 统一配置说明

上面两种方式虽说可以解决，但是很不友好，每个属性都要去搞搞，下面要给配置类将所有的全部改好，无需单独配置：

```java
@Configuration
public class LocalDateTimeConfig {
 
    @Bean
    public ObjectMapper getObjectMapper() {
 
        ObjectMapper om = new ObjectMapper();
        om.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        om.disable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE);
 
        JavaTimeModule javaTimeModule = new JavaTimeModule();
 
        //日期序列化
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm:ss")));
 
        //日期反序列化
        javaTimeModule.addDeserializer(LocalDateTime.class,new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        javaTimeModule.addDeserializer(LocalDate.class,new LocalDateDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        javaTimeModule.addDeserializer(LocalTime.class,new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm:ss")));
        
        om.registerModule(javaTimeModule);
        return om;
    }
}
```

### `@RequestParam`传参配置一下：

两个配置（@requestbody和@requestParam)写在一起

```java
//不要听信idea的自动提示将代码转化成lambda方式，会报错
    @Bean
    public Converter<String, LocalDateTime> LocalDateTimeConvert() {
        return new Converter<String, LocalDateTime>() {
            @Override
            public LocalDateTime convert(String source) {
 
                DateTimeFormatter df = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                LocalDateTime date = null;
                try {
                    date = LocalDateTime.parse(source, df);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return date;
            }
        };
    }
 
    @Bean
    public Converter<String, LocalDate> LocalDateConvert() {
        return new Converter<String, LocalDate>() {
            @Override
            public LocalDate convert(String source) {
 
                DateTimeFormatter df = DateTimeFormatter.ofPattern("yyyy-MM-dd");
                LocalDate date = null;
                try {
                    date = LocalDate.parse(source, df);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return date;
            }
        };
    }
 
    @Bean
    public Converter<String, LocalTime> LocalTimeConvert() {
        return new Converter<String, LocalTime>() {
            @Override
            public LocalTime convert(String source) {
 
                DateTimeFormatter df = DateTimeFormatter.ofPattern("HH:mm:ss");
                LocalTime date = null;
                try {
                    date = LocalTime.parse(source, df);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return date;
            }
        };
    }
 
```


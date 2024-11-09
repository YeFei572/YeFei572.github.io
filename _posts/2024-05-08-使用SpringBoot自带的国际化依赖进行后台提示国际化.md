---
layout: post
tags:
  - SpringBoot
categories: 技术分享
title: 2024-05-08-使用SpringBoot自带的国际化依赖进行后台提示国际化
---

> 最近项目要做到海外去，后台的提示需要做成国际方言化，此处记录一下如何操作

### 一、准备依赖
由于项目依赖的是`SpringBoot`自带的国际化功能，所以一些必要依赖需要添加进来：
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--自定义校验器，当用户入参不合法的时候需要提示错误信息-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### 二、设置配置信息
先设置默认的配置地址，在application.yml里面加入下面的配置信息：
```yaml
spring:
  messages:
    basename: i18n/messages
```
上面是什么意思呢？就是我们把所有的语音提示配置设置一个路径，所有的配置全部放到`resources/i18n/`下面，比方说这个目录下有三个文件，依次是`messages.properties`、`message_en_US.properties`、`message_zh_CN.properties`。它们所代表的是通用默认配置、英文配置、中文配置。随便找一个打开看看【messages.properties】：
```properties
not.null=* 必须填写
```
上面表示这个key对应的错误是要返回给客户端的

另外还有几个配置需要写一下：
```java
package cn.v2ss.cache.config;

import cn.hutool.core.util.StrUtil;
import org.springframework.context.annotation.Bean;
import org.springframework.web.servlet.LocaleResolver;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Locale;

/**
 * 国际化配置
 *
 * @author Protected User
 * @date 2024/05/08 14:00
 **/
public class I18nConfig {

    @Bean
    public LocaleResolver localeResolver() {
        return new I18nLocaleResolver();
    }

    static class I18nLocaleResolver implements LocaleResolver {

        @Override
        public Locale resolveLocale(HttpServletRequest request) {
            String language = request.getHeader("Accept-Language");
            Locale locale = Locale.getDefault();
            if (StrUtil.isNotBlank(language)) {
                String[] split = language.split("-");
                locale = new Locale(split[0], split[1]);
            }
            return locale;
        }

        @Override
        public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {

        }
    }
}


package cn.v2ss.cache.config;

import lombok.RequiredArgsConstructor;
import org.hibernate.validator.HibernateValidator;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;

import javax.validation.Validator;
import java.util.Properties;

/**
 * @author Protected User
 * @date 2024/05/08 14:30
 **/
@Configuration
@RequiredArgsConstructor
public class ValidatorConfig {

    private final MessageSource messageSource;

    @Bean
    public Validator validator() {
        try (LocalValidatorFactoryBean factoryBean = new LocalValidatorFactoryBean()) {
            factoryBean.setValidationMessageSource(messageSource);
            factoryBean.setProviderClass(HibernateValidator.class);
            Properties properties = new Properties();
            properties.setProperty("hibernate.validator.fail_fast", "true");
            factoryBean.setValidationProperties(properties);
            factoryBean.afterPropertiesSet();
            return factoryBean.getValidator();
        }
    }

}


package cn.v2ss.cache.exception;

import cn.v2ss.cache.entity.R;
import cn.v2ss.cache.utils.StreamUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.support.DefaultMessageSourceResolvable;
import org.springframework.validation.BindException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;

/**
 * @author Protected User
 * @date 2024/05/08 14:54
 **/
@Slf4j
@RestControllerAdvice
public class GlobalException {

    @ExceptionHandler(BindException.class)
    public R<Void> handleBindException(BindException e) {
        log.error(e.getMessage());
        String message = StreamUtils.join(e.getAllErrors(), DefaultMessageSourceResolvable::getDefaultMessage, ", ");
        return R.fail(message);
    }

    /**
     * 自定义验证异常
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public R<Void> constraintViolationException(ConstraintViolationException e) {
        log.error(e.getMessage());
        String message = StreamUtils.join(e.getConstraintViolations(), ConstraintViolation::getMessage, ", ");
        return R.fail(message);
    }

}


```
### 三、使用方法

类入口加一个注解 `@Validated`

```Java

    /**
     * 通过code获取国际化内容
     * code为 messages.properties 中的 key
     * <p>
     * 测试使用 user.register.success
     *
     * @param code 国际化code
     */
    @GetMapping()
    public R<Void> get(String code) {
        return R.ok(MessageUtils.message(code));
    }

    /**
     * Validator 校验国际化
     * 不传值 分别查看异常返回
     * <p>
     * 测试使用 not.null
     */
    @GetMapping("/test1")
    public R<Void> test1(@NotBlank(message = "{not.null}") String str) {
        return R.ok(str);
    }

    /**
     * Bean 校验国际化
     * 不传值 分别查看异常返回
     * <p>
     * 测试使用 not.null
     */
    @GetMapping("/test2")
    public R<TestI18nBo> test2(@Validated TestI18nBo bo) {
        return R.ok(bo);
    }

    @Data
    public static class TestI18nBo {

        @NotBlank(message = "{not.null}")
        private String name;

        @NotNull(message = "{not.null}")
        @Range(min = 0, max = 100, message = "{length.not.valid}")
        private Integer age;
    }
```

### 四、总结
参考了[RuoYi-Vue-Plus](https://plus-doc.dromara.org/#/ruoyi-vue-plus/framework/association/i18n)项目，感谢开源
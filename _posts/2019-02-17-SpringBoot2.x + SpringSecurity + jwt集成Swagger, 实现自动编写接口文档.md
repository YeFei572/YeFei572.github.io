---
layout: post
tags: SpringBoot Swagger
categories: 技术分享
title:  "SpringBoot2.x + SpringSecurity + jwt集成Swagger, 实现自动编写接口文档"
---


### 一、初始工作
`Swagger`的相关介绍我就不多BB了, 百度介绍的很清晰, 首先国际惯例直接引入相关依赖

```xml
<!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger2 -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.8.0</version>
</dependency>

<!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

依赖是放在根`pom`文件中, 如果子模块需要使用就直接引入无需填写版本号, 如果是单模块直接引入上面的即可!

### 二、编写配置

先放上`Swagger`的配置:

```java
package com.keppel.config.swagger;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * @Author: feige
 * @Date: Created in 10:35 2019/2/19
 * @Description: swagger2配置类
 */
@Configuration
@EnableSwagger2
public class Swagger2 {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.keppel.api.user"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("个人脚手架,sb2.x + SpringSecurity + jwt")
                .description("")
                .termsOfServiceUrl("https://keppel.fun")
                .version("1.0")
                .build();
    }
}
```

然后编写`Controller`:

```java
package com.keppel.api.user;

import com.keppel.common.response.InfoResponse;
import com.keppel.common.utils.DateUtil;

import com.keppel.user.entity.User;
import com.keppel.user.service.UserService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import org.apache.commons.lang3.RandomUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.UUID;

/**
 * Created by keppel on 2018/11/23.
 */
@Api(value = "用户的控制器")
@RestController
@RequestMapping(value = "/api/v1")
public class UserCtrl {
    @Autowired
    private UserService userServiceImpl;
    private final static Logger logger = LoggerFactory.getLogger(UserCtrl.class);

    @ApiOperation(value = "保存新用户!", notes = "只保存一个用户!")
    @RequestMapping(value = "/public/sayHello", method= RequestMethod.GET)
    public InfoResponse saveUser() {
        long l1 = System.currentTimeMillis();
        for (int i = 0; i < 1; i++) {
            User user = new User();
            user.setId(UUID.randomUUID().toString().replace("-", ""));
            user.setCreateAt(new Date());
            user.setPassword("123456789");
            user.setLastLoginAt(DateUtil.getRandomNow2Tomorrow(new Date()));
            user.setUsername("32001923919");
            user.setPhone("15896272531");
            user.setOpenId("kdjfasjdfsjki32834281kjfsjfasdfdsja");
            user.setPlatform(RandomUtils.nextInt(1, 5));
            user.setUpdateAt(new Date());
            user.setEnabled(true);
            userServiceImpl.saveUser(user);
        }
        logger.info("花费时间: " + (System.currentTimeMillis() - l1));
        return new InfoResponse((System.currentTimeMillis() - l1) + "");
    }
}
```

启动一下, 浏览器输入: `http://localhost:8083/swagger-ui.html`, 发现报错404, 这是因为没有配置资源, 于是在`WebConfig`中配置:

```java
package com.keppel.config.web;

import com.keppel.security.AdminTokenAuthenticationInterceptor;
import com.keppel.security.SignAuthenticationInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.mobile.device.DeviceHandlerMethodArgumentResolver;
import org.springframework.mobile.device.DeviceResolverHandlerInterceptor;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.List;

/**
 * Author: feige
 * CreateAt: 2018/12/2 20:51
 * Description: springBoot2.x已经废弃了WebMvcConfigurerAdapter这个类, 2.x版本后直接实现WebMvcConfigurer接口
 * 46行代码开始都是为了兼容springboot-starter-mobile这个包的内容所做的配置
 **/
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    // 省略了部分无关代码
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");

        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
}
```

再访问一次, 发现还是不行, 报了401错误, 没有权限, 应该是没有设置静态资源权限:

```java
    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.cors().and()
                // 不需要csrf
                .csrf().disable()
                .exceptionHandling().authenticationEntryPoint(unauthorizedHandler).and()
                // 不创建session
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .authorizeRequests()
                .antMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                // 公开接口,无需token即可请求
                .antMatchers(
                        "/api/v1/test/*",
                        "/api/v1/public/sayHello",
                        "/api/v1/selectByUsername",
                        "/api/v1/findBySharePicUrl"
                ).permitAll()
                .antMatchers(
                        "/swagger-ui.html",
                        "/webjars/**",
                        "/v2/**",
                        "/swagger-resources/**"
                ).permitAll()
                .antMatchers(
                        "/admin/login",
                        "/admin/api/**",
                        "/admin/menu/menuList"
                ).permitAll()
                .anyRequest().authenticated();

        // Custom JWT based security filter
        httpSecurity
                .addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class);

        // disable page caching
        httpSecurity.headers().cacheControl();
    }
```

在`WebSecurityConfig.java`中添加
`swagger`的相关配置, 设置为放行状态即可!

最后贴一下相关结构图, 有个坑, 不要把Swagger2.java文件放在ApiApplication.java同目录, 扫描扫不到, 如果非要放,那么需要在启动类上添加`@EnabledSwagger2`标签.

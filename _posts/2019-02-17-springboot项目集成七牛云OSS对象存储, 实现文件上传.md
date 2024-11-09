---
layout: post
tags: SpringBoot OSS
categories: 技术分享
title:  "SpringBoot项目集成七牛云OSS对象存储, 实现文件上传"
---

### 一、背景
> 随着互联网技术发展, 老的分布式文件系统HDFS逐步已经被商家的对象存储替代了,完全包装好的技术, 完善的API文档, 用的不要太舒服, 本文记录`SpringBoot`项目集成七牛云对象存储.

#### 二、准备工作

首先要去七牛云注册账号, 这个流程我就不多说了, 注册完成后, 开一个`bucket`, 相当于一个仓库吧!

![imagepng](https://pic.v2ss.cn/qiniuyun/d3e4b9756fcf4571aacfadfbe2f3c3c3_image.png) 

上面有2个东西要记下, 一个是仓库名字, 一个是访问域名(七牛云会赠送30天免费域名)
然后在设置页面记录下2个东西, 一个是`AK`, 另一个是`SK`.

![imagepng](https://pic.v2ss.cn/qiniuyun/4b50b6974cdf43c6baca4cbbf1c289f5_image.png) 

### 三、引入jar包

```xml
<!-- https://mvnrepository.com/artifact/com.qiniu/qiniu-java-sdk -->
<dependency>
    <groupId>com.qiniu</groupId>
    <artifactId>qiniu-java-sdk</artifactId>
    <version>7.2.18</version>
</dependency>

```

### 四、配置文件走一波

配置文件先放`application.yml`里面的配置:

```yml
# 七牛云配置------------------------------------
# 七牛密钥，配上自己申请的七牛账号对应的密钥
qiniu.AccessKey: 0CQcXKb0Mjti_WrxZTjxxSUNpbhtx1yfffka1dGq
qiniu.SecretKey: fWGgBM8ddddE6SBd6TLdvEgVR9JYqYHXjffdxx9V
# 七牛空间名
qiniu.Bucket: domain
# 外链访问路径
qiniu.cdn.prefix: blog.domain.com
```

再放配置文件:

```java
package com.keppel.config.oss;

import com.google.gson.Gson;
import com.qiniu.common.Zone;
import com.qiniu.storage.BucketManager;
import com.qiniu.storage.UploadManager;
import com.qiniu.util.Auth;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @Author: feige
 * @Date: Created in 17:43 2019/2/25
 * @Description:
 */
@Configuration
public class QiniuUploadFileConfig {
    /**
     * 华东机房,配置自己空间所在的区域
     */
    @Bean
    public com.qiniu.storage.Configuration qiniuConfig() {
        return new com.qiniu.storage.Configuration(Zone.zone2());
    }

    /**
     * 构建一个七牛上传工具实例
     */
    @Bean
    public UploadManager uploadManager() {
        return new UploadManager(qiniuConfig());
    }

    @Value("${qiniu.AccessKey}")
    private String accessKey;
    @Value("${qiniu.SecretKey}")
    private String secretKey;

    /**
     * 认证信息实例
     * @return
     */
    @Bean
    public Auth auth() {
        return Auth.create(accessKey, secretKey);
    }

    /**
     * 构建七牛空间管理实例
     */
    @Bean
    public BucketManager bucketManager() {
        return new BucketManager(auth(), qiniuConfig());
    }

    @Bean
    public Gson gson() {
        return new Gson();
    }

}

```
### 五、编写上传代码
```java
package com.keppel.user.service.impl;

import com.keppel.user.service.QnUploadService;
import com.qiniu.common.QiniuException;
import com.qiniu.http.Response;
import com.qiniu.storage.BucketManager;
import com.qiniu.storage.UploadManager;
import com.qiniu.util.Auth;
import com.qiniu.util.StringMap;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.File;
import java.io.InputStream;

/**
 * @Author: feige
 * @Date: Created in 17:47 2019/2/25
 * @Description:
 */
@Service
public class QnUploadServiceImpl implements QnUploadService, InitializingBean {
    @Autowired
    private UploadManager uploadManager;

    @Autowired
    private BucketManager bucketManager;

    @Autowired
    private Auth auth;

    @Value("${qiniu.Bucket}")
    private String bucket;

    /**
     * 定义七牛云上传的相关策略
     */
    private StringMap putPolicy;

    /**
     * 以文件的形式上传
     * @param file
     * @return
     * @throws QiniuException
     */
    @Override
    public String uploadFile(File file, String fileName) throws QiniuException {
        Response response = this.uploadManager.put(file, fileName, getUploadToken());
        int retry = 0;
        while (response.needRetry() && retry < 3) {
            response = this.uploadManager.put(file, fileName, getUploadToken());
            retry++;
        }
        if (response.statusCode == 200) {
            return new StringBuffer().append("http://blog.domain.com/").append(fileName).toString();
        }
        return "上传失败!";
    }

    /**
     * 以流的形式上传
     *
     * @param inputStream
     * @return
     * @throws QiniuException
     */
    @Override
    public String uploadFile(InputStream inputStream, String fileName) throws QiniuException {
        Response response = this.uploadManager.put(inputStream, fileName, getUploadToken(), null, null);
        int retry = 0;
        while (response.needRetry() && retry < 3) {
            response = this.uploadManager.put(inputStream, fileName, getUploadToken(), null, null);
            retry++;
        }
        if (response.statusCode == 200) {
            return new StringBuffer().append("http://blog.domain.com/").append(fileName).toString();
        }
        return "上传失败!";
    }

    /**
     * 删除七牛云上的相关文件
     * @param key
     * @return
     * @throws QiniuException
     */
    @Override
    public String delete(String key) throws QiniuException {
        Response response = bucketManager.delete(this.bucket, key);
        int retry = 0;
        while (response.needRetry() && retry++ < 3) {
            response = bucketManager.delete(bucket, key);
        }
        return response.statusCode == 200 ? "删除成功!" : "删除失败!";
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        this.putPolicy = new StringMap();
        putPolicy.put("returnBody", "{\"key\":\"$(key)\",\"hash\":\"$(etag)\",\"bucket\":\"$(bucket)\",\"width\":$(imageInfo.width), \"height\":${imageInfo.height}}");
    }

    /**
     * 获取上传凭证
     * @return
     */
    private String getUploadToken() {
        return this.auth.uploadToken(bucket, null, 3600, putPolicy);
    }

}
```

大概就是这样, 然后在控制器里面调用就完事儿了.其实很简单!
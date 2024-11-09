---
layout: post
tags: Maven
categories: 技术分享
title:  "maven项目解决部分私服依赖问题, 以github上面开源项目paascloud为例!"
---

> 最近在研究一个github上面的 [SpringCloud开源项目](https://github.com/paascloud/paascloud-master), 该项目使用到了一些不是中央仓库的包, 这些个包是作者自己搞的, 虽说提供了, 但是没有走仓库下载总感觉少了点啥, 为此搭建了一个私人仓库来解决依赖的问题, 话不多说,开整!

* 想要使用私服必须要做2件事, 第一件事:在项目的根 `pom` 文件中添加私服仓库的配置地址等相关信息:

```xml
<repositories>
    <repository>
        <id>keppel</id>
        <name>keppel</name>
        <url>http://45.78.44.134:8081/nexus/content/repositories/keppel/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>

<pluginRepositories>
    <pluginRepositories>
        <id>keppel</id>
        <name>keppel</name>
        <url>http://45.78.44.134:8081/nexus/content/repositories/keppel/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </pluginRepository>
</pluginRepositories>
```

* 做完以上2点后就可以开始修改子项目的 `pom.xml` 文件了, 比如说下面的这个子项目缺少mybatis生成jar包

![imagepng](https://pic.v2ss.cn/qiniuyun/a6de8267bd814a4b9a2ad3d06e702d10_image.png) 

```xml
<dependency>
    <groupId>com.liuzm.mybatis</groupId>
    <artifactId>mybatis-generator</artifactId>
    <version>1.0</version>
</dependency>
```

我们直接将该地方的 `groupId` 给修改掉, 让系统自动从私服里面去下(私服里面的包我已经上传好了). 我们改成:

```xml
<dependency>
    <groupId>com.keppel.mybatis</groupId>
    <artifactId>mybatis-generator</artifactId>
    <version>1.0</version>
</dependency>
```

![imagepng](https://pic.v2ss.cn/qiniuyun/47015c00cc304c528bf6e435bdda6712_image.png) 


* 最后其他依赖都可以按照这种办法解决, 提供一下依赖的相关信息

```xml
<!-- mybatis-generator的依赖地址: -->
<dependency>
    <groupId>com.keppel.mybatis</groupId>
    <artifactId>mybatis-generator</artifactId>
    <version>1.0</version>
</dependency>

<!-- elastic-job-lite-starter的依赖地址: -->
<dependency>
    <groupId>com.keppel.paascloud</groupId>
    <artifactId>elastic-job-lite-starter</artifactId>
    <version>1.0</version>
</dependency>

<!--阿里支付的相关包依赖: -->
<dependency>
    <groupId>com.keppel.alipay</groupId>
    <artifactId>alipay-sdk-java</artifactId>
    <version>20170725114550</version>
</dependency>
<dependency>
    <groupId>com.keppel.alipay</groupId>
    <artifactId>alipay-trade-sdk</artifactId>
    <version>20161215</version>
</dependency>
```

目前暂时就发现这么多, 如果有什么缺失可以留言, 我补上去.

---
layout: post
tags: Docker
categories: 技术分享
title:  "使用docker创建私服（Nexus3）"
---

> 记录一下使用`docker`创建maven私服的过程。

### 1、安装`docker`

此次 `docker`安装不是重点，安装教程百度一搜索大堆。

### 2、拉取镜像并创建容器

```bash
# 拉取镜像
docker pull docker.io/sonatype/nexus3
# 创建挂载文件夹
mkdir -p /data/nexus/nexus-data/
# 授权文件夹
chmod 777 /data/nexus/nexus-data/
# 启动实例
docker run -id --privileged=true --name=nexus3 --restart=always -p 5807:8081 -p 5808:8082 -p 5809:8083 -v /data/nexus/nexus-data:/var/nexus-data sonatype/nexus3
```

### 3、处理登陆事宜

随后访问ip:5807,第一次登陆会要求重新设置密码，初始登陆密码在 `nexus-data`文件夹下面的`admin.password`文件中

### 4、处理仓库创建事宜

![image.png](https://pic.v2ss.cn/JMu_image.png)

其中中央库可以做代理，如下图配置：

![image.png](https://pic.v2ss.cn/ziB_image.png)

合集的配置如下：

![image.png](https://pic.v2ss.cn/CkQ_image.png)




---
layout: post
tags: Gradle
categories: 个人笔记
title:  "关于gradle无法正常拉取jar包，设置爬墙代理去解决问题"
---

> 有时候国内的镜像仓库没有映射完全，所有必须要从国外中央仓库、谷歌仓库进行拉取依赖，此时去修改gradle.build文件。

单独去修改gradle.build文件非常麻烦，又要改地址，又要改乱七八糟的配置，直接用爬墙工具算了，记录一下全局配置：

打开路径：`C:\Users\keppel\.gradle\gradle.properties`，增加如下配置：

```
systemProp.http.proxyHost=127.0.0.1
systemProp.https.proxyHost=127.0.0.1
# 代理配置端口号，一定要写准确
systemProp.https.proxyPort=4780         
systemProp.http.proxyPort=4780
```

最后确保代理软件在写代码的时候开启了，不然有时候依赖就拉不下来。
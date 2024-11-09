---
layout: post
tags: Flutter
categories: 技术分享
title:  "flutter编写的app打包抓包记录"
---

> 默认情况下，`flutter`因为采用了走自己代理的办法所以编写的包不能通过平常的配置去抓包，他的抓包方式分为两种，一种是代码级别的抓包，第二种是路由级别的抓包，此文记载一下。

### 1、代码级别的抓包

要修改`Dio`的配置（如果是采用Dio包发送http请求的话）

```
dio.onHttpClientCreate = (HttpClient client) { client.findProxy = (uri) { 
    // localhost改为电脑所在的ip（抓包工具所在电脑的ip）
    return "PROXY localhost:8888"; 
}; 
// 你也可以自己创建一个新的HttpClient实例返回。 
// return new HttpClient(SecurityContext); };
```

### 2、使用`wireshark`进行路由级别抓包

此种办法区别于普通抓包模式，中间加了一层路由级别，步骤如下：

- 电脑安装`wireshark`，直接去官网下载最新版本的软件进行安装即可
- 电脑保证能开热点，不管是win10自带的热点还是猎豹wifi、360wifi都可以，让被抓包的手机连接这个wifi，无需配置代理什么的。
- 打开`wireshark`，选择对的抓包路由，鼠标放上去会有ipv4和ipv6出现，ipv4和本机的ip一直就表示该路由是需要抓包的对象
- 双击后开始抓包，此时开始配置抓包过滤器，一般配置以下两个东西

```js
# 表示目标服务器的ip是这个的包都会被抓
ip.src == "192.168.88.250"
# 也可以是这样，ip可以通过ping域名获取到
ip.dst == "192.168.88.250"
```

- 选中一条tcp协议的包，使用快捷键`ctrl + shift + alt + t`既可以追踪流，获取到包的内容，记得改编码到UTF-8，否则会部分乱码


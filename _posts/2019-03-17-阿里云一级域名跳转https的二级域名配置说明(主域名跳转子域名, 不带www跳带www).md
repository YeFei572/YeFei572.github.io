---
layout: post
tags: Nginx
categories: 技术分享
title:  "阿里云一级域名跳转https的二级域名配置说明(主域名跳转子域名, 不带www跳带www)"
---

> 阿里云的**免费域名证书**目前不支持泛解析, 不支持通配符解析, 所有的证书只针对二级域名生效;
很多官网如果只对二级域名做配置https, 比如说, https://www.domain.com,  这种方式是可以正常跳转, 但是用户一般喜欢直接输入
domain.com进行访问, 此时如果没做配置,这种访问是不会跳转的!

**解决方法:**

- 1、 在阿里云后台域名解析的操作台上添加一条解析记录, 如下图所示

![imagepng](https://pic.v2ss.cn/qiniuyun/a3ffa321438e4ebc8a5631f7b46dc5fa_image.png) 

CNAME类型, 前缀直接不填或者@即可

- 2、 服务器`nginx`配置如下:

```
server {
    listen 443;
    server_name www.domain.com;
    ssl on;
    root /var/www/html;
    index index.html index.htm;
    ssl_certificate  cert/1ddddddddddddd.pem;
    ssl_certificate_key cert/1ddddddddddddd.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
    	index index.html;
    }
}
server {
    listen 80;
    server_name www.domain.com domain.com;
    rewrite ^/(.*) https://$server_name$request_uri? permanent;
}

```

配置完以上直接运行

```
nginx -s reload
```

然后清理一遍缓存即可以正常访问了!



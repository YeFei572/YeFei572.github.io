---
layout: post
tags: Nginx
categories: 技术分享
title:  "java获取客户端真实ip(通过反向代理nginx), 获取的不是服务器的ip"
---

> 最近做客户统计, 涉及到统计用户所在地, 网上很多例子获取用户的真实ip,但是我们的服务器使用的是nginx做的反向代理, 如此使用网上的办法就一直获取的是服务器的ip, 经过一番测试和配置终于拿到了客户端的真实ip.

### 1. 误区说明

- 获取到用户的ip是`0.0.0.0.0.1`, 这是因为项目在本地跑, 访问是`localhost:8080`, 把项目地址改成`127.0.0.1`就会获取Ipv4的地址

- 获取到用户的ip是`192.168.x.x`, 这是因为项目在本地跑, 而本地使用的是局域网, 如果把项目放在公网服务器上就会获取真实公网ip

- 将项目部署到了公网服务器上, 获取到用户的ip是服务器的ip, 这是因为使用nginx服务器反向代理了请求地址, 此时需要配置nginx, 后面会说到

#### 2. show me your code

不多说,上代码了, 先copy一下网上到处流行的代码:

```java
package com.moyutang.common.utils;

import javax.servlet.http.HttpServletRequest;
import java.net.InetAddress;
import java.net.UnknownHostException;

/**
 * @Author: feige
 * @Date: Created in 15:25 2019/6/15
 * @Description:
 */
public class IpUtil {
    public static String getIpAddr(HttpServletRequest request) {
        String ipAddress = null;
        try {
            ipAddress = request.getHeader("x-forwarded-for");
            if (ipAddress == null || ipAddress.length() == 0 || "unknown".equalsIgnoreCase(ipAddress)) {
                ipAddress = request.getHeader("Proxy-Client-IP");
            }
            if (ipAddress == null || ipAddress.length() == 0 || "unknown".equalsIgnoreCase(ipAddress)) {
                ipAddress = request.getHeader("WL-Proxy-Client-IP");
            }
            if (ipAddress == null || ipAddress.length() == 0 || "unknown".equalsIgnoreCase(ipAddress)) {
                ipAddress = request.getRemoteAddr();
                if (ipAddress.equals("127.0.0.1")) {
                    // 根据网卡取本机配置的IP
                    InetAddress inet = null;
                    try {
                        inet = InetAddress.getLocalHost();
                    } catch (UnknownHostException e) {
                        e.printStackTrace();
                    }
                    ipAddress = inet.getHostAddress();
                }
            }
            // 对于通过多个代理的情况，第一个IP为客户端真实IP,多个IP按照','分割
            if (ipAddress != null && ipAddress.length() > 15) { // "***.***.***.***".length()
                // = 15
                if (ipAddress.indexOf(",") > 0) {
                    ipAddress = ipAddress.substring(0, ipAddress.indexOf(","));
                }
            }
        } catch (Exception e) {
            ipAddress="";
        }
        return ipAddress;
    }
}
```

以上是静态工具类，然后是使用的代码：

```java
/**
 * 初始化用户设备信息
 * @param device4User
 */
@PostMapping(
        value = "/getIp",
        produces = MediaType.APPLICATION_JSON_VALUE
)
public void getIp(
        HttpServletRequest request
) {
    try {
        String ipAddr = IpUtil.getIpAddr(request);
        System.out.println(ipAddr);
    }catch (Exception e) {
        e.printStackTrace();
    }
}
```

到此时如果是nginx反向代理的话, 正常情况获取到的应该是nginx服务器所在的ip,而并非是客户端的ip
此时需要配置nginx的配置:

```js
server {
    listen 443;
    server_name www.domain.com;
    ssl on;
    root /var/www/html;
    index index.html index.htm;
    ssl_certificate   cert/2347148_www.domain.com.pem;
    ssl_certificate_key  cert/2347148_www.domain.com.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        // 配置此处用于获取客户端的真实IP
        proxy_set_header Host $http_host;
    	proxy_set_header X-Real-IP $remote_addr;
    	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    	proxy_set_header X-Forwarded-Proto $scheme;
    	proxy_pass http://localhost:8080;
    }
}

server {
    listen 80;
    server_name www.domain.com;
    rewrite ^/(.*) https://$server_name$request_uri? permanent;
}
```
如此这般, 再去看看, 应该就是获取到了客户端的真实IP.
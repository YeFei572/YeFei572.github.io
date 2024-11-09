---
layout: post
tags:
  - Linux
categories: 技术分享
title: 2024-05-10-Vmware虚拟机安装CentOS7系统.md
---

> 记录一下VM安装centOS系统

### 一、准备环境
需要注意的是要求安装VM和安装镜像，正常来说要搞对架构，如果架构不对，镜像安装会失败，正常intel系列的芯片直接选择X86-64,苹果或者amd系列用`https://mirrors.aliyun.com/centos-altarch/7/isos/aarch64/`

### 二、安装过程
> 如果不按照下面这些个注意点，可能会出现无法加入网络
安装的时候有几个地方需要注意一下：
- 1、选择网络类型的时候，如果要加入当前网络的局域网，就需要`使用桥接网络`。如果是想自创网络，就使用`网络地址转换NAT模式`。
- 2、配置centOS系统的时候，先选择语言为Chinese,再开始选择时区（亚洲/上海）、然后软件选择。重点来了，网络和主机名需要设置，最后点击右上角的打开按钮后可以立刻看见网络连接信息，如下图所示
![]({{ site.url }}/assets/20.png)
![]({{ site.url }}/assets/21.png)

### 三、后续工作
测试一下网络是否通：
```bash
# 安装网络工具
yum install -y net-tools
# 查询本机ip
ifconfig
```
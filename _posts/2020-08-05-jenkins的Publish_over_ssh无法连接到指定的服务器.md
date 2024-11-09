---
layout: post
tags: Jenkins
categories: 犯错记录
title:  "Jenkins的Publish_over_ssh无法连接到指定的服务器"
---

> 最近公司在搞持续集成，使用jenkins的publish over ssh插件来执行远程服务器的docker镜像拉取和执行，完了配置好了就是无法连接到远程服务器，搞了老长时间， 这个坑真的很让人无语，特此记录一下


### 环境说一下：
使用的是`Jenkins 2.235.3`版本，安装了`Publish Over SSH`, `jenkins`部署在`192.168.88.246`上面， 准备把项目部署到`192.168.88.250`上面

### 1. 问题
在 `192.168.88.246`上面先生成密钥对，然后将公钥复制给`192.168.88.250

```
# 生成密钥对，放到/root/.ssh/目录下面， 输入下面命令，狂按enter即可
ssh-keygen -t rsa -b 2048 -C "keppelfei@gmail.com"

# 将本机的公钥从本机复制到192.168.88.250上面去
ssh-copy-id 192.168.88.250

# 上面的一步会有很多乱七八糟的提示，最后要求输入250服务器的密码，输入即可
```
完成上面的那一步后开始配置插件

![15965932923106.png](https://pic.v2ss.cn/5DxqrECOHf)


最后点击测试按钮的时候就一直报错，说连接不上。

### 2. 解决方案
以上整个流程没毛病，但是连不上250服务器就很让人困惑了， 最后在[jenkins官网的issue](https://issues.jenkins-ci.org/browse/JENKINS-57495)上看到解决方案
先说明问题原因，因为新版本的生成秘钥方式插件暂时还不支持，所以就用老方式来生成秘钥吧：
```
# 删除之前的秘钥, 同时在两台服务器执行该命令
rm -rf /root/.ssh/*
# 重新生成秘钥对 一路狂按enter即可
ssh-keygen -t rsa -b 4096 -m PEM
# 重复
ssh-copy-id 192.168.88.250
```
完成上面不住再去test一下会发现ok了


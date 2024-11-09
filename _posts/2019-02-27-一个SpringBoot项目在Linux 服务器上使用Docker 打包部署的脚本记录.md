---
layout: post
tags: SpringBoot Docker
categories: 个人笔记
title:  "一个SpringBoot项目在Linux 服务器上使用Docker 打包部署的脚本记录"
---

```bash
cd keppelfei
# 拉一下新代码
git pull
# 切换主分支
git checkout master
# 开始打包镜像
./gradlew clean build dockerBuilder --info
# 停止当前运行的docker镜像
docker stop gradle-boot
# 删除docker镜像
sudo docker rm $(sudo docker ps -a -q)
# 查看当前所有的镜像
docker images
# 运行打好的镜像
docker run -d --name gradle-boot -p 8081:8081 keppel/spring-boot-gradle-for-docker:1.0
# 批量清空REPOSITORY, TAG为none的镜像
docker images|grep none|awk '{print $3}'|xargs docker rmi
# 查看docker容器对应的日志
docker logs -f -t --tail=40 gradle-boot
```
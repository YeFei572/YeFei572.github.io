---
layout: post
tags: Github
categories: 个人笔记
title:  "使用GitHub的actions进行在线打包flutter并发布到release！"
---

> 自己电脑有点卡，直接使用GitHub的工作流进行线上打包，记录一下流程

### 一、环境说明
就是普通的仓库，目前所有的GitHub仓库都支持工作流`Actions`了，都可以使用，只要配置完整。

### 二、配置工作流文件

#### 1、创建配置文件
配置文件如下，将该文件推送到仓库（所在目录：`/.github/workflow/dart.yml`)，大致说明见注释信息：

```yml
name: Flutter CI

# 创建触发打包条件：只要推送tag，就会触发版本打包

on:
  push:
    tags:
      - v*
    
jobs:
  build:
    # This job will run on ubuntu virtual machine
    runs-on: ubuntu-latest
    steps:
    
    # Setup Java environment in order to build the Android app.
    - uses: actions/checkout@v1
      with:
    # 使用office分支的代码
        path: office
    - uses: actions/setup-java@v1
      with:
        java-version: '12.x'
    
    # Setup the flutter environment.
    - uses: subosito/flutter-action@v1
      with:
        channel: 'stable' # 'dev', 'alpha', default to: 'stable'
        # flutter-version: '1.12.x' # you can also specify exact version of flutter
    
    # Get flutter dependencies.
    - run: flutter pub get    
    # Build apk
    - run: flutter build apk --target-platform android-arm
    # 发包
    - name: Release apk
      uses: ncipollo/release-action@v1.5.0
      with:
        artifacts: "build/app/outputs/apk/release/*.apk"
        token: ${{ secrets.ACTION_TOKEN }}
```

#### 2、创建仓库的secret
创建仓库的secret，其名字为：`ACTION_TOKEN`
![创建Action_token]({{ site.url }}/assets/1.png)

#### 3、推送触发打包
以上所有的操作完善后，本地执行如下代码：

```bash
# 根据具体情况修改tag版本号
git tag v1.0.0
git push --tags


# 删除远程标签
git push origin :refs/tags/v1.0.4
# 删除本地标签
git tag -d v1.0.4
```
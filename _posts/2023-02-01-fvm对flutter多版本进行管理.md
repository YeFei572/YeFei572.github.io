---
layout: post
tags: Flutter
categories: 技术分享
title:  "fvm对flutter多版本进行管理"
---

> 背景说明：公司的项目还在用`2.8.0`版本，线上`stable`版本已经到了`3.7.0`，一直想体验一下新版本的效果，但是修改环境变量个人觉得很麻烦，正好有现成的管理工具[fvm](https://fvm.app/docs/getting_started/overview)，记录一下使用方法。

- ### 一、安装方法
为了避免麻烦，这里记录一种通用方法，适用`windows`、`macOS`、`Linux`。方法是直接挂梯子去[github](https://github.com/fluttertools/fvm/releases)上去下载压缩包，找到对应的包，然后download下来 > 解压 > 配置环境变量。这种办法的好处是以后不用了清理也好清理，没什么注册表、配置文件等乱七八糟的东西。

校验是否安装成功：
```
fvm -h
```
- ### 二、配置说明
这里配置分两种，第一种是对`fvm`这个软件的配置，包括sdk放在哪里，另外一种配置是对项目的配置。下面依次对其讲解。<br/>
#### 1、对fvm进行配置：
```bash
# 查看当前配置情况
fvm config
# 设置flutter的SDK本地地址（缓存）, 如：C:\dev\sdk\
fvm config --cache-path C:\dev\sdk\
```
完成上述配置后，可以对`flutter`不同版本进行下载，不建议这么下载，直接挂梯子下载会快很多。下载完后直接以版本号丢在上述路径中，这里最好文件夹命名为版本号，因为后面切命令要输入完整名字挺麻烦的。
```bash
fvm install 3.7.0
```
#### 2、对项目进行配置：
完成上述配置后，进入对应的项目，如果要切成指定版本：
```bash
# 查看对应的当前电脑所有版本
fvm list
# 切换到对应的版本
fvm use 3.7.0
```
此时会创建`.fvm`文件夹，记得将改文件添加到`.gitignore`文件中，因为此文件夹会存在对应的`sdk`。
#### 3、其他注意点
至此配置相关已经完毕，可以运行以下命令
```
fvm flutter {command}
# 比如
fvm flutter doctor
```
- ### 三、其他说明
用了`Android Studio`感觉这个可有可无。因为这个编辑器可以对不同的项目设置不同的环境，需要说明的是，创建新项目的时候需要去指定sdk目录下运行`flutter.bat`文件来进行创建
```bash
# 进入指定sdk目录下，运行命令创建项目
C:\\dev\\fvm\\sdk\\3.7.0\\bin> ./flutter.bat create demo
```
因为不同的`sdk`创建的项目对应的`jdk`版本、`gradle`版本、以及`flutter`的`pubspec`文件都不一样。这里拿`3.7.0`和`2.8.0`比较，`2.8.0`创建的项目如果用`3.7.0`的启动会失败，因为`3.7.0`用的是`jdk11`，而`2.8.0`用`jdk8`。<br/>
此时需要修改编辑器的一些东西：<br/>
`Project Structure > Project Settings > Project > SDK` 选择11；<br/>
`Project Structure > Platform Settings > SDKs` 选择11；<br/>
还要修改gradle的配置文件<br/>
```bash
# 项目根目录/android/gradle.properties 新增jdk对应的物理地址
org.gradle.java.home=C:\\tool\\jdk11
```
此时才可以正常启动。否则会报找不到jdk对应的版本地址。<br/>
个人建议采用自己配置，`fvm`适合那些用编辑器的，项目多的用户。
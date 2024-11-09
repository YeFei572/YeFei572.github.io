---
layout: post
tags: Flutter
categories: 技术分享
title:  "flutter使用iconfont.cn上的图标"
---

> 自定义`Icon`图标教程，记录一下。

### 1、`iconfont.cn`网站处理一下：

下载文件：

![image.png](https://pic.v2ss.cn/1kd_image.png)

**两个文件需要试用一下**， 一是ttf文件，在第2步中直接引入即可。另外一个`css`文件转化成`dart`文件，用一个网站转就可以了 [https://xwrite.gitee.io/blog/](https://xwrite.gitee.io/blog/)

![image.png](https://pic.v2ss.cn/byB_image.png)

### 2、`Flutter`项目引入一下。

首先`pubspec.yaml`也要引入一些依赖。

```yml
fonts:
    - family: IconFont
      fonts:
        - asset: assets/fonts/iconfont.ttf
```

引入`ttf`文件：

![image.png](https://pic.v2ss.cn/yW5_image.png)

### 3、使用一下：

![image.png](https://pic.v2ss.cn/jkk_image.png)

使用如下：

```dart
BottomNavigationBarItem(
  icon: Icon(IconFont.icon_home),
  label: "主页",
),
```

### 4、还有一种转换方法，如[链接](https://github.com/iconfont-cli/flutter-iconfont-cli)所示


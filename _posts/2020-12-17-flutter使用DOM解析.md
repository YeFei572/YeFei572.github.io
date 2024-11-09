---
layout: post
tags: Flutter
categories: 技术分享
title:  "flutter使用DOM解析"
---

> 本文主要是记录在写 `flutter` 项目的时候如果碰到比较好的、比较适用的网页，而该网页又没有提供专门的 `RESTful` 风格的接口的时候，此时使用本教程解析 `html` 可以很好地获取自己想要的数据，跟爬虫的解析有异曲同工之妙！话不多说，开整!

### 一、技术选型和初始准备：

首先引入本次教程所需要的依赖包：

````dart
dependencies:
  html: ^0.14.0+4
  dio: ^3.0.10
````

此处使用了两个包， 一个是 `html` 包，该包主要是用来解析 `html标签` 的。另外一个包 `dio`，该包用来发起请求的，这个包很常见，就不细说了。

### 二、发起请求获取所有的 `html源码`：

首先通过 `dio` 发起请求获取要爬取的网页页面，此处以掘金首页的导航为准做个小例子：

````dart
/// 存放标签
  List<String> tags = [];
  /// 存放标签跳转路径
  List<String> tagUrls = [];
  
  @override
  void initState() {
    /// 初始化的时候将标签、标签路径清空
    this.tagUrls = [];
    this.tags = [];
    getPost();
    super.initState();
  }

  getPost() async {
    Dio dio = new Dio();
    // 发起请求获取首页的html数据
    Response res = await dio.get("https://juejin.cn/android");
    // 解析标签的值
    List titles = parse(res.data).querySelectorAll("li.nav-item > a ");
    // 添加标签到集合
    titles.forEach((element) {
      this.setState(() {
        this.tags.add(element.text.trim());
        this.tagUrls.add(element.attributes['href']);
      });
      print(element.text.trim() + ">>>>" + element.attributes['href']);
    });
  }


 @override
  Widget build(BuildContext context) {
    // This method is rerun every time setState is cal
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Wrap(
              runSpacing: 5,
              spacing: 4,
              children: List.generate(
                  tags.length,
                  (index) => Container(
                        padding:
                            EdgeInsets.symmetric(horizontal: 10, vertical: 5),
                        decoration: BoxDecoration(
                            borderRadius: BorderRadius.circular(6),
                            color: Colors.pinkAccent),
                        child: Text(
                          tags[index],
                          style: TextStyle(color: Colors.white),
                        ),
                      )),
            ),
          ],
        ),
      ),
    );
  }
````

### 三、说明一下 `html` 的使用方法：

- 要保证`parse`的是一个标准的 `Document`文件，也就是说你 `parse`的肯定是在浏览器中可以打开且正确的html源码，否则会 `parse`失败！
- 引包着重说一下，这里的包只需要引入如下即可：

````dart
import 'package:html/parser.dart' show parse;
import 'package:dio/dio.dart';
````

- 再一个就是解析的标签有如下说明：
  - `document.querySelectorAll(".xxx")`: 获取所有样式为 `xxx` 的标签列表；
  - `document.querySelectorAll(".xxx>div>p.example")`：获取样式为 `xxx`的标签下面含有`div`，`div`下面包含的样式为 `example`的 `p标签`的列表，（就一句话前面两个大于号是一味最后的 `p标签` 做规则限制的；
  - `document.querySelectorAll(".xxx")[0].text.strim()`：获取标签下面的值，比如：`<div class=xxx>hello</div>`；
  - `document.querySelectorAll(a)[0].attributes['href']`: 获取 `a标签`的跳转地址；




---
layout: post
tags: Flutter
categories: 技术分享
title:  "Flutter使用插件flutter_staggered_grid_view实现分页瀑布流效果"
---

> 最近研究一下flutter, flutter是跨平台开发, 一处编写代码多处使用, 也就是说, 写个app以后再也不需要安卓和ios两个开发人员了, 一个人就够了, 本文是记录一下使用插件来实现响应式瀑布流布局.

### 一、实现效果
如下图所示, 要实现下图效果, 主要分为两点:

- 一点是瀑布流响应式布局, 该功能已经在市面上有插件. 只是是英文版的, 看起来有些吃力, 还好能看懂代码, 搞出来了;
- 第二点是分页, 因为该插件没有带分页, 所以需要自己写分页逻辑

![20190426gif](https://pic.v2ss.cn/qiniuyun/20190426.gif)

### 二、实现步骤
- 引入插件依赖

在`pubspec.yaml`文件中引入所需依赖:

```
dependencies:
  flutter:
    sdk: flutter
  dio: ^2.1.2
  flutter_staggered_grid_view: ^0.2.7   // 瀑布流插件
  shared_preferences: ^0.5.2            // 本地存储插件
  cached_network_image : ^0.6.2         // 网络缓存图片
  flutter_screenutil: ^0.5.2            // 屏幕尺寸工具插件
  cupertino_icons: ^0.1.2               // 自带的图标插件
```

- 编写代码

flutter也是刚入门, 所以就直接放代码了:

```dart
import 'package:demo03_movie/widgets/tile_card.dart';
import 'package:dio/dio.dart';
import 'package:flutter/material.dart';
import 'package:flutter_staggered_grid_view/flutter_staggered_grid_view.dart';
import 'package:flutter_screenutil/flutter_screenutil.dart';

Dio dio = new Dio();

class GridPage extends StatefulWidget {
  @override
  GridPageState createState() => new GridPageState();
}

class GridPageState extends State<GridPage> with AutomaticKeepAliveClientMixin {
  ScrollController _scrollController = new ScrollController();

  int _page = 0;
  int _size = 10;
  int _beLoad = 0; // 0表示不显示, 1表示正在请求, 2表示没有更多数据

  var posts = [];

  @override
  Widget build(BuildContext context) {
    ScreenUtil.instance = ScreenUtil.getInstance()..init(context);
    return RefreshIndicator(
      onRefresh: _refreshData,
      child: Container(
        color: Colors.grey[100],
        child: StaggeredGridView.countBuilder(
          controller: _scrollController,
          itemCount: posts.length,
          primary: false,
          crossAxisCount: 4,
          mainAxisSpacing: 4.0,
          crossAxisSpacing: 4.0,
          itemBuilder: (context, index) => TileCard(
                img: '${posts[index]['images'][0]}',
                title: '${posts[index]['title']}',
                author: '${posts[index]['userName']}',
                authorUrl: '${posts[index]['iconUrl']}',
                type: '${posts[index]['type']}',
                worksAspectRatio: posts[index]['worksAspectRatio'],
              ),
          staggeredTileBuilder: (index) => StaggeredTile.fit(2),
        ),
      ),
    );
  }

  @override
  bool get wantKeepAlive => true;

  // 下拉刷新数据
  Future<Null> _refreshData() async {
    _page = 0;
    _getPostData(false);
  }

  // 上拉加载数据
  Future<Null> _addMoreData() async {
    _page++;
    _getPostData(true);
  }

  void _getPostData(bool _beAdd) async {
    var response = await dio.get(
        'https://xxxxx.com/getdata?page=$_page&size=$_size');
    var result = response.data;
    print('result: ${result['data']}');
    setState(() {
      if (!_beAdd) {
        posts.clear();
        posts = result['data'];
      } else {
        posts.addAll(result['data']);
      }
    });
  }

  @override
  void initState() {
    super.initState();
    // 首次拉取数据
    _getPostData(true);
    _scrollController.addListener(() {
      if (_scrollController.position.pixels ==
          _scrollController.position.maxScrollExtent) {
        _addMoreData();
        print('我监听到底部了!');
      }
    });
  }
}

```

再放上每个小部件widget的代码:

```dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';
import 'package:flutter_screenutil/flutter_screenutil.dart';

class TileCard extends StatelessWidget {
  final String img;
  final String title;
  final String author;
  final String authorUrl;
  final String type;
  final double worksAspectRatio;
  TileCard(
      {this.img,
      this.title,
      this.author,
      this.authorUrl,
      this.type,
      this.worksAspectRatio});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        mainAxisAlignment: MainAxisAlignment.start,
        children: <Widget>[
          Container(
            color: Colors.deepOrange,
            child: CachedNetworkImage(
              imageUrl: '$img'
            ),
          ),
          Container(
            padding:
                EdgeInsets.symmetric(horizontal: ScreenUtil().setWidth(20)),
            margin: EdgeInsets.symmetric(vertical: ScreenUtil().setWidth(10)),
            child: Text(
              '$title',
              style: TextStyle(
                  fontSize: ScreenUtil().setSp(30),
                  fontWeight: FontWeight.bold),
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
            ),
          ),
          Container(
            padding: EdgeInsets.only(
                left: ScreenUtil().setWidth(20),
                bottom: ScreenUtil().setWidth(20)),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.start,
              children: <Widget>[
                CircleAvatar(
                  backgroundImage: NetworkImage('$authorUrl'),
                  radius: ScreenUtil().setWidth(30),
                  // maxRadius: 40.0,
                ),
                Container(
                  margin: EdgeInsets.only(left: ScreenUtil().setWidth(20)),
                  width: ScreenUtil().setWidth(250),
                  child: Text(
                    '$author',
                    style: TextStyle(
                      fontSize: ScreenUtil().setSp(25),
                    ),
                  ),
                ),
                Container(
                  margin: EdgeInsets.only(left: ScreenUtil().setWidth(80)),
                  child: Text(
                    '${type == 'EXISE' ? '练习' : '其他'}',
                    style: TextStyle(
                      fontSize: ScreenUtil().setSp(25),
                    ),
                  ),
                )
              ],
            ),
          )
        ],
      ),
    );
  }
}

```

### 三、总结
刚开始看的插件自带demo没有分页功能, 然后自己做了个`ListView`分页效果, 从中得到了启发, 创建了一个 `ScrollCtroller`, 但是这个插件好像在这里使用该controller不生效, 于是又到处找解决办法, 但是还是找不到. 最后自己看插件源码, 发现`StaggeredGridView.countBuilder`有个参数是`itemCount`, 我的代码里没有, 于是添加上去, controller就生效了..
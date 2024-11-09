---
layout: post
tags: Flutter
categories: 经验心得
title:  "flutter中使用RSA进行公钥加密"
---


> 新公司使用`flutter`进行密码加密, 特写文章记录一下方便以后使用

### 一、准备工作

首先要知道, `RSA`为非对称加密, 此处记录的是前端使用公钥加密, 后端使用私钥解密, 此处的公钥是网络静态文件. 举个栗子:
- 公钥地址: `https://keppel.fun/publicKey.pem`
- 公钥内容:

```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDAJz2dB47fquw+Y6WVLcpJojMw
z9Qy2rs0FVdSE4fffDIoryxvLvmUe4ZKQlLHRZgcFESVvo4TaLcAcuni60gNCEem
y1w+P93GKcjEFyt705PibOSQFYcN07vC77brJ6SbP9hd8g8RslSK7CZ9VQ9uQHge
q8MN5q84Q4rwOZZSCQIDAQAB
-----END PUBLIC KEY-----
```
以上是公钥的内容, 不是单纯的秘钥, 而是包含了一个头和一个尾,中间才是正文, 但是解析的时候会自动跳过去头尾!

### 二、show me your code

- 首先需要解决依赖问题:

```yml
encrypt: ^3.3.1
```

- 编写工具类:

```dart
import 'package:encrypt/encrypt.dart';
import 'package:encrypt/encrypt_io.dart';
import 'package:path_provider/path_provider.dart';

/// 公钥加密工具
Future<String> encodePassword(String src) async {
  var directory = await getApplicationDocumentsDirectory();
  var publicKey = await parseKeyFromFile('${directory.path}/dir/pub.pem');
  final encrypter = Encrypter(RSA(publicKey: publicKey));
  final res = encrypter.encrypt(src).base16.toUpperCase();
  return res;
}
```

- 登录页开始缓存公钥文件到本地:

```dart
/// 在登录页面的init()方法内执行获取公钥
/// 先获取公钥
  Future getPublicSecrect() async {
    try {
      String path = '';
      Dio _dio = new Dio();
      // 获取应用文档目录并创建文件
      Directory documentsDir = await getApplicationDocumentsDirectory();
      String documentsPath = documentsDir.path;
      await new Directory(documentsPath + '/dir')
          .create(recursive: true)
          .then((Directory directory) {
        path = directory.path;
      });
      await _dio.download(
        UrlPath.PUB_SECRECT,
        path + '/pub.pem',
      );
    } catch (e) {
      print('下载有异常！$e');
      return null;
    }
  }
```

- 最后调用加密方法进行加密

```dart
encodePassword(password).then((val) {
  HttpManager().request('get', UrlPath.LOGIN_PATH, data: {
    'deviceBrand': 'HONOR%20DUA-AL00',
    'deviceToken': '3d8a27b89a6d4d66b164e2108d0fab37',
    'deviceVersion': 'Exhibition-1.0.0',
    'phone': userName,
    'deviceType': '1',
    'password': val,
    'appVersion': '1.0.0'
  }).then((val) {
    print('登录结果》》$val');
    if (val.resultCode == 0) {
      Fluttertoast.showToast(msg: '登录成功！');
      NavigatorUtils.goHome(context);
    } else {
      Fluttertoast.showToast(msg: '账号密码错误！');
    }
  });
});
```

如此这般就解决了加密问题!

### 三、最后说一下该项工作的相关注意点:
- 加密工具类中使用的是`base16`进行转码, 普通的语言(java之类)都是使用`BASE64`,这是个坑, 需要注意一下.
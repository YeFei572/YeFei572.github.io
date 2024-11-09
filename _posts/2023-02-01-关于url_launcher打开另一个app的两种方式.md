---
layout: post
tags: Flutter
categories: 技术分享
title:  "关于url_launcher打开另一个app的两种方式"
---

> 正常来说，url_launcher使用android自带的deep_link可以唤醒本机上任何app。但是有两种方式打开，一种是应用内，一种是应用外，两种方式的区别是，前者后退可以直接回原应用，后者是返回到主页面。记录一下配置两种实现方式。

### 一、修改被调用方的AndroidMainfest.xml
这里的修改的时候注意一个属性即可`launchMode`：
```gradle
 <activity
    android:name=".AdMainActivity"
    android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
    android:hardwareAccelerated="true"
    android:launchMode="singleTask"
    android:theme="@style/LaunchTheme"
    android:windowSoftInputMode="adjustResize">
    <meta-data
        android:name="io.flutter.embedding.android.NormalTheme"
        android:resource="@style/NormalTheme"
        tools:replace="android:resource" />
    <meta-data
        android:name="io.flutter.embedding.android.SplashScreenDrawable"
        android:resource="@drawable/launch_background" />

    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="demo" />
    </intent-filter>
</activity>
```
上面的`launchMode`有四个属性，如果是想在应用内打开的话就选`singleTop`，如果在应用外打开的话就设置为`singleTask`。

### 二、url_launcher方法修改源码
`url_launcher`默认打开依赖方法一的设置，如果想强制应用外打开的话，就需要修改`url_launcher`的源码，`url_launcher`没有对外暴露应用外打开的参数。使用`AndroidStudio`打开项目，按照下图修改：
![]({{ site.url }}/assets/19.png)

### 三、总结
个人建议能不修改源码就不修改，如果非要修改就可以`fork`一分代码，自己依赖。
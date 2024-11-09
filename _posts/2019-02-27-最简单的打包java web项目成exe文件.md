---
layout: post
tags: IDEA
categories: 技术分享
title:  "最简单的打包：javaWeb项目打包成exe文件的方法！"
---

> 最近朋友公司有个需求说是因为开发部署到现场比较麻烦, 因为浏览器设置等等一些东西, 所以需要打包成普通的安装软件, 然后就有了这篇文章...
  网上搜了一下解决办法, 都太麻烦, 习惯性看了一下github, 还真发现了一个 [牛逼的项目](https://github.com/jiahaog/nativefier), 于是写个博客记录一下吧..



- 本地安装node: 这个就不多说了, 直接安装node就可以了,如果不懂直接百度去吧,教程很多.

![imagepng](https://pic.v2ss.cn/qiniuyun/13926d3dc1b541e08b43497e3c6dee0a_image.png) 


- 安装软件执行以下命令进行全局安装压缩软件, 该软件应该是基于node编写:
```javascript
  npm install nativefier -g
```
![imagepng](https://pic.v2ss.cn/qiniuyun/7e7a8a5a08d944868ca61c591fa78700_image.png) 


- 执行上面的命令以后就可以进行打包了, 找一个空文件夹,使用管理员打开cmd, 执行以下命令即可:
```javascript
  nativefier "https://keppel.fun"
```
![imagepng](https://pic.v2ss.cn/qiniuyun/ec0c13e295c34d92824d0622702bdf0b_image.png) 


然后静静等待就行了, 完事儿了再打开刚才那个目录,就可以发现一堆的动态库乱遭的东西中间还夹杂一个exe的文件, 双击以下就可以看到打包后的效果了

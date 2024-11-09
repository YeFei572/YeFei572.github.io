---
layout: post
tags: Docker Gradle SpringBoot
categories: 技术分享
title:  "使用docker+gradle打包部署多模块springboot项目到Ubuntu服务器"
---

> 有一段时间没写博客了, 最近比较忙. 前些日子朋友说想学docker, 本博客就是通过docker部署到Ubuntu服务器上的, docker可以看成一个虚拟机, 对于低配置服务器效果更加明显, 资源合理运用了. 至于更深层的好处可以去[docker中文文档](https://docs.docker-cn.com/) 看看, 这个大家应该很开心, 因为是中文, 讲的很细.

废话不多说了,开整:

### 1. 该流程接上次博客中创建的 [多模块项目](https://www.keppel.fun/articles/2018/11/23/1542977710422.html) 接着搞. 多模块项目如果要加入docker必须要在主项目中引入docker插件依赖:

```java
buildscript {
  ext {
  springBootVersion = '1.5.9.RELEASE'
  springRepo = 'https://plugins.gradle.org/m2/'
}
repositories {
  maven { url springRepo }
  jcenter()
  mavenLocal()
  mavenCentral()
}  
dependencies {
  classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
  classpath("se.transmode.gradle:gradle-docker:1.2")
}}

tasks.withType(JavaCompile) {
  options.encoding = 'UTF-8'
}
```

上面有个坑要注意下, 仓库地址由原来的那啥地址换成了

```java
https://plugins.gradle.org/m2/
```

因为原来的地中没有docker插件的依赖, 所以这里要注意下！

### 2. 开始第二步, 在入口处的依赖(gradle.build)中引入对docker插件的设置, 什么叫入口处, 就是整个多模块项目主项目的入口, 我们目前的入口是 **api** 模块, 所以在 api 模块下的 `build.gradle` 写下配置文件

```java
apply plugin: 'application'
apply plugin: 'docker'

jar {
  baseName 'spring-boot-gradle-for-docker'
  version '1.0'
}

distDocker {
  baseImage 'openjdk'
  maintainer 'harrison'
}

task dockerBuilder(type: Docker) {
  applicationName = jar.baseName
  tagVersion = jar.version
  volume('/tmp')
  addFile("${jar.baseName}-${jar.version}.jar", "app.jar")
  entryPoint(["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", 'app.jar'])
  exposePort(8081)
  doFirst {
    copy {
  	from jar
  	into stageDir
    }
  }
}
```

什么意思? 这里的步骤少了像市面上的Dockerfile, 我是很烦那个破文件, 放哪儿容易搞错, 这种就避免了这种尴尬问题

### 3. 至此配置就ojbk了! 我再上传修改过的两个文件一次, 先上主项目的 `build.gradle` **完整文件**

```java
def vJavaLang = '1.8'
def bootProjects = [project(':api')]
def gradleDir = "${rootProject.rootDir}/gradle"

group = 'com.keppel'
version = '0.0.1-SNAPSHOT'

buildscript {
  ext {
    springBootVersion = '1.5.9.RELEASE'
    springRepo = 'https://plugins.gradle.org/m2/'
  }
  repositories {
    maven { url springRepo }
    jcenter()
    mavenLocal()
    mavenCentral()
  }  
  dependencies {
    classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    classpath("se.transmode.gradle:gradle-docker:1.2")
  }
}

tasks.withType(JavaCompile) {
options.encoding = 'UTF-8'
}

subprojects {
  apply plugin: "eclipse"
  apply plugin: "idea"
  apply plugin: 'java'

  targetCompatibility = vJavaLang
  sourceCompatibility = vJavaLang

  repositories {
    mavenLocal()
    jcenter()
    mavenCentral()
  }
}

configure(bootProjects) {
  apply plugin: 'eclipse'
  apply plugin: 'idea'
  apply plugin: 'java'
  apply plugin: 'org.springframework.boot'

  targetCompatibility = vJavaLang
  sourceCompatibility = vJavaLang

  repositories {
    mavenLocal()
    jcenter()
    mavenCentral()
  }
}
```

再上入口子模块的`build.gradle`文件完整版

```java
jar {
  baseName = 'keppel-api'
  version = '0.0.1-SNAPSHOT'
}

springBoot {
  mainClass = 'com.keppel.ApiApplication'
}

dependencies {
  compile project(':common')
  compile project(':user')

  compile('io.jsonwebtoken:jjwt:0.9.0')
  compile('org.springframework.boot:spring-boot-starter-mobile')

  compile('org.springframework.boot:spring-boot-starter-security')
  compile("org.springframework.boot:spring-boot-starter-web")
  testCompile('org.springframework.boot:spring-boot-starter-test')
}

apply plugin: 'application'
apply plugin: 'docker'

jar {
  baseName 'spring-boot-gradle-for-docker'
  version '1.0'
}

distDocker {
  baseImage 'openjdk'
  maintainer 'harrison'
}

task dockerBuilder(type: Docker) {
  applicationName = jar.baseName
  tagVersion = jar.version
  volume('/tmp')
  addFile("${jar.baseName}-${jar.version}.jar", "app.jar")
  entryPoint(["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", 'app.jar'])
  exposePort(8081)
  doFirst {
    copy {
  	from jar
  	into stageDir
    }
  }
}
```

如果你开发主机为win系列的你可以直接通过idea来进行build
![imagepng](https://pic.v2ss.cn/qiniuyun/db89288eb7034a298d364d4627f3db78_image.png)
双击命令就可以了
![imagepng](https://pic.v2ss.cn/qiniuyun/0ff38e838d7843b6a0aa207a8802aea1_image.png)
镜像文件找不到在哪儿, 反正就是api.tar里面的吧
真正的是通过Ubuntu来进行打包, 贴个自己写的脚本, 亲测已经打包docker成功

```bash
// 进入项目目录
 cd keppelfei
 
 // 拉取最新代码
 git pull
 
 // 切换分支到主分支
 git checkout master
 
 // 开始打包啦
 ./gradlew clean build dockerBuilder --info
 
 // 停止正在运行的docker镜像进行替换
 docker stop gradle-boot
 
 // 删除所有已停止的镜像
 sudo docker rm $(sudo docker ps -a -q)
 
 // 导入打包好的镜像
 docker images
 
 // 开始以gradle-boot为命名启动刚打好的镜像
 docker run -d --name gradle-boot -p 8081:8081 keppel/spring-boot-gradle-for-docker:1.0
```

至此结束了, 有什么问题欢迎留言!


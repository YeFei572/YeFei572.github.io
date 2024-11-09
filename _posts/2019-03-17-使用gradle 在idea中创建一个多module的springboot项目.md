---
layout: post
tags: Gradle SpringBoot
categories: 技术分享
title:  "使用gradle 在idea中创建一个多module的SpringBoot项目"
---

> 网上搜索了很久没找到这类的详细说明,都是你抄袭我的我抄袭他的，完事儿了还不能打包,很难受,所以自己摸索了一套流程来创建多module项目，其中参考了官方文档（gradle）

### 一、版本说明

- 1 认使用`springboot`使用`1.5.9`, 目前`2.1.0`已经发布, 但是该版本的`gradle`构建时相对于`1.5.9` 版本有些改动， 目前还没找到哪儿出了问题， 不过`maven`是可以正常搭建；


- 2 默认`gradle`使用的是`4.6`;
  
  ----
  
### 二、搭建步骤

#### 一. 新建文件夹keppel作为项目的根目录， 打开cmd进入该目录， 执行如下命令来初始化根项目：

  ```
  gradle init
  ```
#### 二. 以上命令已经初始化完了项目的根目录，然后就直接用intellij idea打开该目录
<br/> ![png](https://pic.v2ss.cn/qiniuyun/39d7c5db807e4f86bd40c475ac8db288_1.png)
	---
	![2jpg](https://pic.v2ss.cn/qiniuyun/1d1de6d887e24998aeb40910442e02c9_2.jpg) 
<br>

#### 三. 此时项目结构如上图所示， 然后开始新增通用模块 **common**, 右键keppel根项目new-module
<br/>![3jpg](https://pic.v2ss.cn/qiniuyun/21e11324a06a4572b588cb6de2cb2ab2_3.jpg) 



#### 四. 修改common模块下的build.gradle, 如下所示:
```java
  buildscript {
	repositories {
	  mavenLocal()
	  jcenter()
	  mavenCentral()
	  maven { url springRepo }
	  maven {
		url "https://plugins.gradle.org/m2/"
	  }
	}
  }

  apply plugin: 'io.spring.dependency-management'

  jar {
	baseName = 'keppel-common'
	version = '0.0.1-SNAPSHOT'
  }

  dependencies {
	compile('org.springframework:spring-context')
	compile("org.hibernate:hibernate-validator")
	compile('com.google.code.gson:gson:2.2.4')
	compile 'com.fasterxml.jackson.core:jackson-databind:2.6.4'
	compile 'com.fasterxml.jackson.core:jackson-core:2.6.4'
	compile 'com.fasterxml.jackson.core:jackson-annotations:2.6.4'
	compile 'org.jdom:jdom2:2.0.5'
	compileOnly('org.projectlombok:lombok')
	testCompile('org.springframework.boot:spring-boot-starter-test')
  }

  dependencyManagement {
	imports { mavenBom("org.springframework.boot:spring-boot-dependencies:${springBootVersion}") }
  }
  ```
#### 五. 接着修改根目录下的build.gradle文件：
  ```js
  def vJavaLang = '1.8'
  def bootProjects = [project(':api')]
  def gradleDir = "${rootProject.rootDir}/gradle"

  group = 'com.keppel'
  version = '0.0.1-SNAPSHOT'

  buildscript {
	ext {
	  springBootVersion = '1.5.9.RELEASE'
	  springRepo = 'http://repo.spring.io/libs-release'
	}
	repositories {
	  mavenLocal()
	  jcenter()
	  mavenCentral()
	  maven { url springRepo }
	}
	dependencies {
	  classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	}
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

#### 六. 新增项目入口模块 **api**, 方法同上，生成的子模块下面都是只有一个build.gradle文件，修改该文件：
```gradle
jar {
    baseName = 'keppel-api'
    version = '0.0.1-SNAPSHOT'
}

springBoot {
    mainClass = 'com.keppel.ApiApplication'
}

dependencies {
    compile project(':core')
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```
#### 七. 修改完成后再在api模块下面添加src-main-java文件夹，java文件夹下面并排添加resources文件夹，java文件夹下面创建主类ApiApplication:
```java
package com.keppel;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
* Created by keppel on 2018/11/23.
*/
@SpringBootApplication
public class ApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiApplication.class, args);
    }
}
```
#### 八. 此时此刻，项目已经搭建完成大半了。项目的结构图如下图所示：
****
  ![4jpg](https://pic.v2ss.cn/qiniuyun/b26518a6cd7b496eaec58a0cf14df931_4.jpg) 



#### 九. 开始测试项目是否搭建成功
  创建一个新的module： **user** module， build.gradle和**api**的一样，引入common
  然后编写一个类进行测试

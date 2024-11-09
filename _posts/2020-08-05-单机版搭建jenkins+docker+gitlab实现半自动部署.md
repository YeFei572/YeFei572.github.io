---
layout: post
tags: Jenkins Docker Git
categories: 经验心得
title:  "单机版搭建jenkins+docker+gitlab实现半自动部署"
---

> 公司搭建jenkins+docker+gitlab实现半自动（非gitlab的钩子）持续集成，因为有时连续推两次代码会连续触发两次钩子造成不必要的更新，浪费时间和资源。如果后面需要持续集成加上钩子可以加上。另外还有版本号这个目前没有做定义，都是写死的latest，镜像都是死的，如果正式服，那么就要定义动态版本号。测试服不影响就没管！

###  1、环境说明

- 服务器目前涉及3台，分别是：
	- `192.168.88.246`  `jenkins`部署机器，`docker`仓库部署机器
	- `192.168.88.100` `gitlab`部署机器
	- `192.168.88.250` 各种微服务部署机器 
- 版本介绍：
	- 系统环境：`centos 8`
	- `jenkins`：`2.235.3`
	- `gitlab`：`12.10.1`
	- `docker`：`19.03.12`
	- `git`:  `2.18.4`
- 说明：
这里对系统安装、`gitlab`安装、`docker`安装`不做说明，亦可以自行谷歌百度。

### 2、安装`jenkins`

#### 1）安装JDK

Jenkins需要依赖JDK，所以先安装JDK1.8

```bash
# 安装jdk8
yum install java-1.8.0-openjdk* -y
# 配置环境变量，底部添加java的环境变量
vim /etc/profile

# jdk8
export JAVA_HOME=/usr/lib/jvm
export JRE_HOME=$JAVA_HOME/jre
export CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export JAVA_HOME JRE_HOME CLASS_PATH PATH
```

安装目录为：`/usr/lib/jvm`

#### 2）获取jenkins安装包

下载页面：https://jenkins.io/zh/download

#### 3）把安装包上传到192.168.88.246服务器，进行安装

```bash
rpm -ivh jenkins-2.235.3-1.1.noarch.rpm
```

#### 4）修改Jenkins配置

因为安装后默认是jenkins用户，为了避免麻烦，直接将启动用户改成`root`

```bash
vi /etc/syscofig/jenkins
# 修改内容如下：
JENKINS_USER="root"
# 如若要修改端口可以在此处修改，不修改的话默认是8080端口
# JENKINS_PORT="8888"
```

#### 5）启动jenkins

此处启动需要放行端口，我这里把防火墙关掉了，如果不关防火墙记得释放端口，否则无法访问

```bash
systemctl start jenkins
```

访问url: http://192.168.88.246:8080

#### 6）获取并输入admin账户密码

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```
#### 7）跳过插件安装

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

将初始密码粘贴到对话框点击完成即可，跳过安装插件，由于插件源是国外网站，所以跳过改国内的地址
![微信截图20200805134440.png](https://pic.v2ss.cn/L3dvmiHyOv)

#### 8）添加一个管理员账户，并进入Jenkins后台

配置账号和密码，这里是管理员，配置完成一直点下一步即可！

#### 9）修改jenkins插件源地址

Jenkins->Manage Jenkins->Manage Plugins，点击Available
![微信截图20200805134440.png](https://pic.v2ss.cn/4GfAKogfKM)
要让这个列表完全加载完，这样做是为了把Jenkins官方的插件列表下载到本地，接着修改地址文件，替换为国内插件地址

```bash
cd /var/lib/jenkins/updates
sed -i 's/http:\/\/updates.jenkinsci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

再点Advanced，滑到底部，修改更新源地址为清华大学的镜像源地址
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
![image.png](https://pic.v2ss.cn/RxLtRiiYYr)
Sumbit后，在浏览器输入： http://192.168.66.101:8888/restart ，重启Jenkins

#### 10）安装插件
安装完以下插件后再次重启
- 汉化插件 `Localization: Chinese (Simplified)`
- 授权方面 `Role-based Authorization Strategy`
- 凭证管理 `Credentials Binding`
- Git插件 `git`
- Maven相关 `Maven Integration`
- Pipeline插件 `Pipeline`
- Gitlab Hook钩子插件 `Gitlab Hook`
- GitLab插件 `GitLab`
- 远程脚本调用 `Publish Over SSH`

### 3、配置`Jenkins`

#### 1）全局工具配置

主要配置`Maven`、`JDK`、`Git`等相关信息
这里说明一下，`Maven`去官网下载linux的安装包解压，放到`/usr/develop/apache-maven-3.6.3`
![image.png](https://pic.v2ss.cn/gkMbI46ZTA)
![image.png](https://pic.v2ss.cn/ykFsF6Kbul)

#### 2）系统配置
这里主要配置一些系统的环境参数：

![image.png](https://pic.v2ss.cn/m6yKmsuzYL)

还有最底下的ssh远程配置， 这里或许有个[坑儿](https://www.keppel.fun/articles/2020/08/05/1596593717959.html)，如果没有那就棒棒哒！

![image.png](https://pic.v2ss.cn/b1ThIsnbWH)

#### 3） gitlab拉代码的权限配置和docker仓库的权限配置

依次添加gitlab的和nexus3（docker仓库）的密码凭证

![image.png](https://pic.v2ss.cn/ztEra5rkrk)
![image.png](https://pic.v2ss.cn/yIOfTmoZyq)
![image.png](https://pic.v2ss.cn/1rFl6n1DjN)

至此，系统配置完成

### 4、创建流水线相关

#### 1）新建item

![image.png](https://pic.v2ss.cn/PYvkpSAcTN)
![image.png](https://pic.v2ss.cn/oDvDffTZzP)

#### 2）流水线定义模板
```js
node {
    // docker仓库地址
    def harbor_url = "192.168.88.246:8083"
    // 冗余字段（当为harbor的时候可能有一个父目录，此处用的是nexus3，所以不存在）
    def harbor_project_name = "123"
    //构建版本的名称
    def tag = "latest"
    //Harbor的凭证, 上面配的docker仓库凭证的id
    def harbor_auth = "xxxxxxxxx"
    
    // 项目名字 
    def project_name = "dikar-codegen"
    // docker容器暴露的端口
    def port = "5003"
    // git仓库地址
    def git_url = "http://192.168.88.100/dikar-oa-platform/dikar-visual.git"
    // 拉代码，没啥好说的
    stage('pull code') { // for display purposes
       checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'd4ce27b6-8a28-4390-8360-4362fec3c16d', url: "${git_url}"]]])
    }
    // 打jar包，构建docker镜像， 这里要注意，如果是单层pom文件无需指定，如果是多层，需要强制指定pom文件
    stage('Build docker') {
	// 单层，无需指定pom
        // sh "mvn clean package dockerfile:build" 
	// 多层，强制指定pom
	sh "mvn -f dikar-codegen clean package dockerfile:build"
    }
    // 开始推送镜像
    stage('push docker') {
        def imageName = "${project_name}:${tag}"
        sh "docker tag ${imageName} ${harbor_url}/${imageName}"       
        
         //登录docker私服，并上传镜像
        withCredentials([usernamePassword(credentialsId: "${harbor_auth}",
            passwordVariable: 'password', usernameVariable: 'username')]) {
            //登录
            sh "docker login -u ${username} -p ${password} ${harbor_url}"
            //上传镜像
            sh "docker push ${harbor_url}/${imageName}"
        }
        // 删除本地镜像
        sh "docker rmi -f ${imageName}"
        sh "docker rmi -f ${harbor_url}/${imageName}"
    }
    // 开始触发远程脚本
    stage ('remote deploy') {
        // 触发远程脚本
        sshPublisher(publishers: [sshPublisherDesc(configName: '192.168.88.250',
        transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand:
        "/data/jenkins_shell/deploy.sh $harbor_url $harbor_project_name $project_name $tag $port", execTimeout: 120000, flatten: false, makeEmptyDirs: false,
        noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '',
        remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')],
        usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
    }
}
```

#### 3）远程脚本

以上是一个完整的流水线脚本，这里还放一下远程调用192.168.88.250的远程脚本

```bash
#! /bin/sh
#接收外部参数
harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4
port=$5

imageName=$harbor_url/$project_name:$tag
#imageName=$harbor_url/$harbor_project_name/$project_name:$tag

echo "$imageName"

#查询容器是否存在，存在则删除
containerId=`docker ps -a | grep -w ${project_name}:${tag}  | awk '{print $1}'`
if [ "$containerId" !=  "" ] ; then
    #停掉容器
    docker stop $containerId
    #删除容器
    docker rm $containerId
        echo "成功删除容器"
fi

#查询镜像是否存在，存在则删除
imageId=`docker images | grep -w $project_name  | awk '{print $3}'`
if [ "$imageId" !=  "" ] ; then
    #删除镜像
    docker rmi -f $imageId
        echo "成功删除镜像"
fi
# 登录Harbor
docker login -u admin -p dikar2020 $harbor_url
# 下载镜像
docker pull $imageName
# 启动容器
docker run -di --add-host=dikar-mysql:192.168.88.223 --add-host=dikar-redis:192.168.88.223 --add-host=dikar-rabbitmq:192.168.88.223 --add-host=dikar-register:192.168.88.250 -p $port:$port --name $project_name $imageName

echo "容器启动成功"

```

![image.png](https://pic.v2ss.cn/etG7qb42t4)




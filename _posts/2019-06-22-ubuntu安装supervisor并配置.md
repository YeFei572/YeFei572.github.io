
---
layout: post
tags: Ubuntu
categories: 经验心得
title:  "Ubuntu安装supervisor并配置"
---

> Supervisor是用Python开发的一套通用的进程管理程序，能将一个普通的命令行进程变为后台daemon，并监控进程状态，异常退出时能自动重启。

### 1. 安装`Supervisor`

 ```
 sudo apt-get install supervisor
 ```

 安装完毕后自动设置为开机启动

### 2. 配置`Supervisor`文件

开始配置`supervisor`, 他的配置类型和`nginx`很像, 有一个"父配置", 其余的每个进程单独的成为"子配置", 先看一眼"父配置":

```js
vi /etc/supervisor/supervisord.conf
```

```js
[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file)
chmod=0700                       ; sockef file mode (default 0700)

[supervisord]
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisor/conf.d/*.conf
```

由上面的配置(最后一行)可以看出来, 子配置所在地方,所以我们接下来的配置都要在子配置中进行了:

```js
vi /etc/supervisor/conf.d/demo01.conf   # 新建配置文件
```
```js
[program:api-service]
command=/usr/bin/java -jar -Dspring.profiles.active=prod /root/apps/demo-api-0.0.1-SNAPSHOT.jar
user=root
autostart=true
autorestart=true
startsecs=10
startretries=5
stdout_logfile=/root/logs/api-service.log
stderr_logfile=/root/logs/api-service-error.log
```

做一下配置文件的解释:

```js
[program:xxxxx]                             #程序名称，在supervisorctl中通过这个值来对程序进行一系列的操作
command=/usr/bin/java -jar -Dspring.profiles.active=prod /root/apps/demo-api-0.0.1-SNAPSHOT.jar     #启动命令，与手动在命令行启动的命令是一样的
autorestart=True                            # 程序异常退出后自动重启
autostart=True                              # 在supervisord启动的时候也自动启动
stderr_logfile=/home/app/logs/err.log       # 错误日志
stdout_logfile=/home/app/logs/run.log       # 运行日志
user=ubuntu                                 # 用哪个用户启动
startsecs=1                                 # 启动间隔
```
### 3. 命令方面说一下
```js
# 更新配置文件sudo
sudo supervisorctl update 
# 查看当前子进程状态
supervisorctl status
# 启动子进程 [program:xxxxxx]
sudo supervisorctl start xxxxx 
# 停止子进程
sudo supervisorctl stop xxxxx 
# 重启
sudo supervisorctl restart xxxxx 
```
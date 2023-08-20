---
title: jenkins的部署问题记录
date: 2023-08-20 15:54:23
tags: jenkins CD
---
因为工作的原因，需要给外包同学部署一个jenkins来帮助他们部署一些东西。
## docker方式
一开始使用docker方式部署jenkins，简单快速又方便。很快就部署好了。
但是docker是一个容器，而要进行CI、CD的项目是在宿主机上，查了下，在jenkins上安装了一个pushover ssh插件，等于是用ssh登录的方式来执行宿主机上面的脚本。
然而，这种是获取不到脚本的输出的，也无法真正判断脚本是否执行成功（可能也有办法判断，我没有深入研究）。
再加上服务器宕机了一次，脚本怎么也跑不起来了。于是还是决定直接装一个非docker的jenkins服务。

## 非docker方式
按照[官网说明](https://pkg.jenkins.io/redhat-stable/)，很快就装好了。
启动后遇到两个问题。
### 修改默认端口
一般是修改`/etc/sysconfig/jenkins`里面的`JENKINS_PORT`来修改，但是一直不生效。原来新版的jenkins必须修改这里：
```shell
vim /usr/lib/systemd/system/jenkins.service
找到Environment="JENKINS_PORT=8080",把端口修改掉就行了
```
### 修改启动用户
默认的启动用户是jenkins/jenkins。 但是很多脚本是在其他用户目录下，jenkins用户是没有权限的。
我就把他修改为dev用户。
一开始也是修改`/etc/sysconfig/jenkins`里面的`JENKINS_USER`，还是不生效，后来还是修改这里
```
vim /usr/lib/systemd/system/jenkins.service
文件最开头的user、group修改成你需要的用户和用户组就行了
```
修改过后，记得重载配置文件
```
systemctl daemon-reload
```
然后重启服务就行了
```shell
service jenkins restart
```

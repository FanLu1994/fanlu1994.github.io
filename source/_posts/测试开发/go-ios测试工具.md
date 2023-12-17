---
title: go-ios测试工具
date: 2023-12-17 20:24:47
tags:
  - golang
  - 工具
categories: 测试开发
---
> 这是一个go实现的ios自动化测试工具,用于实现在linux window mac 上实现ios自动化
> https://github.com/danielpaulus/go-ios
！！ ios17已经无法挂载开发者镜像

## 功能简介
- 安装ipa到iphone
- 在多平台运行XCTest，包括webdriverAgent
- 启动和停止应用
- 无需手动点击弹窗就可以配对设备
- 自动安装开发者镜像
- 在设备上执行热状态和网络仿真（弱网测试）

## 安装
两种方案
-  npm install go-ios
-  下载代码，编译go项目

>所有命令的默认输出都是 JSON 格式。如果你更喜欢人类可读的输出结果，请在命令中指定 --nojson 选项。
默认情况下，除非指定 --udid=some_udid 开关，否则命令将使用找到的第一个设备。
指定 -v 用于调试日志，指定 -t 用于转储每条信息。
## 使用
！！windows要安装并启动itunes；
！！ios17目前很多功能不支持，手上也没有其他设备，所以很多功能试不了
！！若命令未指定--udid，则命令默认在列表中第一个设备执行

## 查看设备列表
> 以下命令类似，windows如果是自己变异的就用exe执行

```shell
# mac
ios  list
# windows
.\go-ios.exe list
```

### 安装开发者镜像
```
ios image auto
```
这里有点问题 我的iphone设备使用mac或者widnows都无法成功安装镜像
- 从github下载镜像出错，众所周知的原因
- 手动下载后报错，原因暂时位置，并且我的设备时17.1了，为什么要下载16.6呢？
```
{"err":"open devimages\\16.6\\DeveloperDiskImage.dmg.signature: The system cannot find the file specified.","image":"devimages\\16.6\\DeveloperDiskImage.dmg","level":"error","msg":"error mounting image","time":"2023-12-06T08:53:56+08:00","udid":"00008101-001E38100242001E"}
```

经过排查，iphone需要开启开发者模式：
设置-》通用-》隐私与安全性-》开发者模式
打开之后重启设备

### 设备信息
```
ios info
```

### 系统日志
```
ios syslog
```

### 应用相关
#### 应用列表
```
ios apps --list  安装应用  不是json输出
ios apps --all  全部应用
ios apps --system  系统应用
```

#### 安装ipa


#### 启动停止应用程序
需要开发者镜像

### 状态模拟
需要开发者镜像

### 崩溃日志
```
 .\go-ios.exe crash ls  查看崩溃日志
 .\go-ios.exe crash cp "*" "./"  拷贝日志到指定目录
```

### 截图
ios17 不行
### 抓包
需要进程id，进程id需要开发者镜像
### 重启

### 磁盘相关
查看磁盘大小
```
(base) PS D:\git\go-ios> .\go-ios.exe diskspace
{"level":"info","msg":"no udid specified using first device in list","time":"2023-12-12T08:28:29+08:00","udid":"00008101-001E38100242001E"}
      Model: iPhone13,2
  BlockSize: 512
  FreeSpace: 34.5GB
  UsedSpace: 93.3GB
 TotalSpace: 127.9GB
```

## 配对密钥
```
.\go-ios.exe readpair
```

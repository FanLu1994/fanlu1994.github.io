---
title: Fastbot_Android
date: 2023-10-24 08:50:37
tags:
  - 工具
  - 自动化测试
categories: 测试开发
---
参考资料：
https://www.cnblogs.com/fnng/p/15738284.html
>字节开发的一个基于强化学习的app遍历测试或者monkey测试工具。

- 只能在macos执行
## 获取应用安装的列表
可以用tidevice来获取https://github.com/alibaba/taobao-iphone-device
执行命令：
```shell
tidevice applist
```
## 环境配置
- 安装xcode
- 打开fastbot_ios项目
- 允许fastbotrunner
    - 这里会报错，因为xcode需要添加一个apple账号
    - 此外需要修改应用名字
      具体操作可以参考： https://github.com/bytedance/Fastbot_iOS/blob/main/Doc/handbook-cn.md

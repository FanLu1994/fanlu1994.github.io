---
title: SoloX之应用启动时间
date: 2023-07-14 08:37:42
tags: 测试开发 性能 工具
categories: 测试开发
---
## 安卓

adb命令获取到activity启动时间

```python
def getStartupTimeByAndroid(self, activity, deviceId):
    result = adb.shell(cmd='am start -W {}'.format(activity), deviceId=deviceId)
    return result
```





## IOS

基于pydevice获取

```python
def getStartupTimeByiOS(self, pkgname):
    try:
        import ios_device
    except ImportError:
        logger.error('py-ios-devices not found, please run [pip install py-ios-devices]') 
    result = self.execCmd('pyidevice instruments app_lifecycle -b {}'.format(pkgname))       
    return result
```

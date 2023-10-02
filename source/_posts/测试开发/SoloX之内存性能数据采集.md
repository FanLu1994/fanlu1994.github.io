---
title: SoloX之内存性能数据采集
date: 2023-07-06 20:57:47
tags: 测试开发 性能 工具
categories: 测试开发
---
## 安卓
### 获取内存信息

给定进程的PID，返回总内存、本地堆内存、dalvik堆内存  并转换为MB单位

```python
def getAndroidMem(self):
    """Get the Android memory ,unit:MB"""
    cmd = f'dumpsys meminfo {self.pid}'
    output = adb.shell(cmd=cmd, deviceId=self.deviceId)
    m_total = re.search(r'TOTAL\s*(\d+)', output)
    m_native = re.search(r'Native Heap\s*(\d+)', output)
    m_dalvik = re.search(r'Dalvik Heap\s*(\d+)', output)
    totalPass = round(float(float(m_total.group(1))) / 1024, 2)
    nativePass = round(float(float(m_native.group(1))) / 1024, 2)
    dalvikPass = round(float(float(m_dalvik.group(1))) / 1024, 2)
    return totalPass, nativePass, dalvikPass
```



## IOS

同样通过instruments协议获取；不过只能获取到总内存

```python
def getiOSMem(self):
    """Get the iOS memory"""
    apm = iosAPM(self.pkgName)
    totalPass = round(float(apm.getPerformance(apm.memory)), 2)
    nativePass = 0
    dalvikPass = 0
    return totalPass, nativePass, dalvikPass
```

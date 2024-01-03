---
title: pymobiledevice3
date: 2024-01-03 21:46:09
tags:
  - ios
  - 工具
categories: 测试开发
---
地址： https://github.com/doronz88/pymobiledevice3
> 简介： 完全由python3实现，用于和iphone通信的工具，类tidevice，支持ios17

！！windows使用需要启动itunes

## 检测连接设备
` python -m pymobiledevice3 bonjour browse`
```shell
(base) PS C:\Users\fanlu> python -m pymobiledevice3 bonjour browse
[
    {
        "ipv4": [
            "192.168.31.243"
        ],
        "ipv6": [
            "fe80::1cb4:55c1:f952:1ca6"
        ],
        "lockdown_info": {
            "CPUArchitecture": "arm64e",
            "DeviceName": "\u590f\u76ee\u7684iphone12",
            "HardwareModel": "D53gAP",
            "HumanReadableProductVersionString": "17.1.2",
            "ProductName": "iPhone OS",
            "ProductType": "iPhone13,2",
            "ProductVersion": "17.1.2",
            "SupportedDeviceFamilies": [
                1
            ]
        },
        "mac_address": "8c:ec:7b:42:5a:90",
        "name": "8c:ec:7b:42:5a:90@fe80::8eec:7bff:fe42:5a90-supportsRP-19._apple-mobdev2._tcp.local."
    }
]
```

命令行不如 tidevice 好用

可能更适合代码来使用，使用代码来试一下。

## 使用代码方式
```python
from pymobiledevice3.remote.remote_service_discovery import RemoteServiceDiscoveryService  
from pymobiledevice3.lockdown import create_using_usbmux  
from pymobiledevice3.services.syslog import SyslogService  
  
# Connecting via usbmuxd  
lockdown = create_using_usbmux()  
for line in SyslogService(service_provider=lockdown).watch():  
    # just print all syslog lines as is  
    print(line)  
  
# Or via remoted (iOS>=17)  
# First, create a tunnel using:  
#     $ sudo pymobiledevice3 remote start-tunnel  
# You can of course implement it yourself by copying the same pieces of code from:  
#     https://github.com/doronz88/pymobiledevice3/blob/master/pymobiledevice3/cli/remote.py#L68  
# Now you can simply connect to the created tunnel's host and port  
host = 'fded:c26b:3d2f::1'  # randomized  
port = 65177  # randomized  
with RemoteServiceDiscoveryService((host, port)) as rsd:  
    for line in SyslogService(service_provider=rsd).watch():  
        # just print all syslog lines as is  
        print(line)
```

- windows不支持开启通道


## iOS17 mac 测试
![[Pasted image 20231228083943.png]]
1. 首先开启 remote 通道
```
sudo python3 -m pymobiledevice3 remote start-tunnel
```
2. 然后在命令执行中，带上参数
```
python -m pymobiledevice3 developer dvt screenshot screen.png --rsd fdf2:3d44:d0::1 51528
```

文档中的部分命令使用过都可以正常使用。

并且每个模块都被封装成了 python 类，因此可以在代码中很好的调用,例如


## 截图功能分析
代码部分：
- pymobiledevice3/cli/developer.py
    - line 256  dvt执行的命令
    - line 865
- pymobiledevice3/services/screenshot.py
- pymobiledevice3/dvt/instruments/screenshot.py

```
with RemoteServiceDiscoveryService((hostname, RSD_PORT)) as rsd:
```

## 使用python代码进行截图操作
```python
from pymobiledevice3.remote.remote_service_discovery import RemoteServiceDiscoveryService  
from pymobiledevice3.lockdown import create_using_usbmux  
from pymobiledevice3.services.syslog import SyslogService  
from pymobiledevice3.services.dvt.instruments.screenshot import Screenshot  
from pymobiledevice3.services.dvt.dvt_secure_socket_proxy import DvtSecureSocketProxyService  
from pymobiledevice3.services.remote_server import RemoteServer  
from pymobiledevice3.lockdown_service_provider import LockdownServiceProvider  
  
# Connecting via usbmuxd  
# lockdown = create_using_usbmux()  
# for line in SyslogService(service_provider=lockdown).watch():  
#     # just print all syslog lines as is  
#     print(line)  
  
# screenShotService = ScreenshotService(lockdown=lockdown)  
# screenShotService.take_screenshot()  
  
# Or via remoted (iOS>=17)  
# First, create a tunnel using:  
#     $ sudo pymobiledevice3 remote start-tunnel  
# You can of course implement it yourself by copying the same pieces of code from:  
#     https://github.com/doronz88/pymobiledevice3/blob/master/pymobiledevice3/cli/remote.py#L68  
# Now you can simply connect to the created tunnel's host and port  
host = 'fdd3:399:962c::1'  # randomized  
port = 51572  # randomized  
with RemoteServiceDiscoveryService((host, port)) as rsd:  
    out = open("test.png","wb")  
    with DvtSecureSocketProxyService(lockdown=rsd) as dvt:  
        out.write(Screenshot(dvt).get_screenshot())  
    out.close()
```
一张图大概截图时间0.7s（mac m2）

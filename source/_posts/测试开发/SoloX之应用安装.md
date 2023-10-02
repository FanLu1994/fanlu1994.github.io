---
title: SoloX之应用安装
date: 2023-07-14 08:39:43
tags: 测试开发 性能 工具
---
## 安卓
1. 首先保存上传的apk到临时目录

```python
file_path = os.path.join(currentPath, '{}.apk'.format(unixtime))
if install.uploadFile(file_path, file):
    install_status = install.installAPK(file_path)
```

2. 然后安装

```python
def installAPK(self, path):
    result = adb.shell_noDevice(cmd='install -r {}'.format(path))
    if result == 0:
        os.remove(path)
        return True, result
    else:
        return False, result
```


## IOS

1. 首先保存ipa包到临时目录
2. 然后安装 （基于tidevice）

```python
def installIPA(self, path):
    result = Devices.execCmd('tidevice install {}'.format(path))
    if result == 0:
        os.remove(path)
        return True, result
    else:
        return False, result

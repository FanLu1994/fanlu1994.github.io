---
title: SoloX之GPU性能采集
date: 2023-07-14 08:36:15
tags: 测试开发 性能 工具
---
## 安卓
目前不支持，其实是可以拿到的。


## ios

```python
class GPU(object):
    def __init__(self, pkgName):
        self.pkgName = pkgName

    def getGPU(self, noLog=False):
        apm = iosAPM(self.pkgName)
        gpu = apm.getPerformance(apm.gpu)
        if noLog is False:
            apm_time = datetime.datetime.now().strftime('%H:%M:%S.%f')
            f.add_log(os.path.join(f.report_dir,'gpu.log'), apm_time, gpu)
        return gpu 

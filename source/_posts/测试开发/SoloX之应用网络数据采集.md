---
title: SoloX之应用网络数据采集
date: 2023-07-14 08:39:07
tags: 
- 测试开发 
- 性能 
- 工具
categories: 测试开发
---
## 安卓
基于linux的原理，通过`cat /proc/进程id/net/dev` 拿到网卡数据；

间隔一秒 分别拿到两次发送数据量、接受数据量 （`sendNum_pre` ` recNum_pre`）

然后计算数据差，得到网速

```python
def getAndroidNet(self, wifi=True):
    """Get Android send/recv data, unit:KB wlan0/rmnet0"""
    net = 'wlan0' if wifi else 'rmnet0'
    cmd = f'cat /proc/{self.pid}/net/dev |{d.filterType()} {net}'
    output_pre = adb.shell(cmd=cmd, deviceId=self.deviceId)
    m_pre = re.search(r'{}:\s*(\d+)\s*\d+\s*\d+\s*\d+\s*\d+\s*\d+\s*\d+\s*\d+\s*(\d+)'.format(net), output_pre)
    sendNum_pre = round(float(float(m_pre.group(2)) / 1024), 2)
    recNum_pre = round(float(float(m_pre.group(1)) / 1024), 2)
    time.sleep(1)
    output_final = adb.shell(cmd=cmd, deviceId=self.deviceId)
    m_final = re.search(r'{}:\s*(\d+)\s*\d+\s*\d+\s*\d+\s*\d+\s*\d+\s*\d+\s*\d+\s*(\d+)'.format(net), output_final)
    sendNum_final = round(float(float(m_final.group(2)) / 1024), 2)
    recNum_final = round(float(float(m_final.group(1)) / 1024), 2)
    sendNum = round(float(sendNum_final - sendNum_pre), 2)
    recNum = round(float(recNum_final - recNum_pre), 2)
    return sendNum, recNum
```





## IOS

间隔3秒拿到数据，基于instrument

```python
def getPerformance(self, perfTpe: DataType):
    if perfTpe == DataType.NETWORK:
        perf = Performance(self.deviceId, [perfTpe])
        perf.start(self.pkgName, callback=self.callback)
        time.sleep(3)
        perf.stop()
        perf_value = self.downflow, self.upflow
    else:
        perf = iosP.Performance(self.deviceId, [perfTpe])
        perf_value = perf.start(self.pkgName, callback=self.callback)
    return perf_value
```

```python
def iter_network_flow(d: BaseDevice, rp: RunningProcess) -> Iterator[Any]:
    n = 0
    with d.connect_instruments() as ts:
        for nstat in ts.iter_network():
            # if n < 2:
            #     n += 1
            #     continue
            yield DataType.NETWORK, {
                "timestamp": gen_stimestamp(),
                "downFlow": (nstat['rx.bytes'] or 0) / 1024,
                "upFlow": (nstat['tx.bytes'] or 0) / 1024
            }
```



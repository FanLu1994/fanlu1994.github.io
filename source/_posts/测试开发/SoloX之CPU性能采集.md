---
title: SoloX之CPU性能采集
date: 2023-07-06 20:57:07
tags: 测试开发 性能 工具
---
## 安卓
### 获取进程cpu使用时间

```python
def getprocessCpuStat(self):
    """get the cpu usage of a process at a certain time"""
    cmd = f'cat /proc/{self.pid}/stat'
    result = adb.shell(cmd=cmd, deviceId=self.deviceId)
    r = re.compile("\\s+")
    toks = r.split(result)
    processCpu = float(int(toks[13]) + int(toks[14]) + int(toks[15]) + int(toks[16]))
    return processCpu
```

### 获取总cpu使用时间

```python
def getTotalCpuStat(self):
    """get the total cpu usage at a certain time"""
    cmd = f'cat /proc/stat |{d.filterType()} ^cpu'
    result = adb.shell(cmd=cmd, deviceId=self.deviceId)
    r = re.compile(r'(?<!cpu)\d+')
    toks = r.findall(result)
    totalCpu = 0
    for i in range(1, 9):
        totalCpu += float(toks[i])
    return float(totalCpu)
```

### 获取CPU核数

```py
def getCpuCores(self):
    """get Android cpu cores"""
    cmd = 'cat /sys/devices/system/cpu/online'
    result = adb.shell(cmd=cmd, deviceId=self.deviceId)
    try:
        nums = int(result.split('-')[1]) + 1
    except:
        nums = 1
    return nums
```

### 获取系统CPU使用时间

```py
def getSysCpuStat(self):
    """get the total cpu usage at a certain time"""
    cmd = f'cat /proc/stat |{d.filterType()} ^cpu'
    result = adb.shell(cmd=cmd, deviceId=self.deviceId)
    r = re.compile(r'(?<!cpu)\d+')
    toks = r.findall(result)
    ileCpu = int(toks[4])
    sysCpu = self.getTotalCpuStat() - ileCpu
    return sysCpu
```

### 采集结果

> 间隔0.5s连续获取两次采集结果，然后计算得到cpu使用率  https://yestermorrow.github.io/2021/03/17/CPU%E4%BD%BF%E7%94%A8%E7%8E%87/

```py
def getAndroidCpuRate(self, noLog=False):
    """get the Android cpu rate of a process"""
    processCpuTime_1 = self.getprocessCpuStat()
    totalCpuTime_1 = self.getTotalCpuStat()
    sysCpuTime_1 = self.getSysCpuStat()
    time.sleep(0.5)
    processCpuTime_2 = self.getprocessCpuStat()
    totalCpuTime_2 = self.getTotalCpuStat()
    sysCpuTime_2 = self.getSysCpuStat()
    appCpuRate = round(float((processCpuTime_2 - processCpuTime_1) / (totalCpuTime_2 - totalCpuTime_1) * 100), 2)
    sysCpuRate = round(float((sysCpuTime_2 - sysCpuTime_1) / (totalCpuTime_2 - totalCpuTime_1) * 100), 2)
    if noLog is False:
        apm_time = datetime.datetime.now().strftime('%H:%M:%S.%f')
        f.add_log(os.path.join(f.report_dir,'cpu_app.log'), apm_time, appCpuRate)
        f.add_log(os.path.join(f.report_dir,'cpu_sys.log'), apm_time, sysCpuRate)

    return appCpuRate, sysCpuRate
```

## IOS

> ios基于pydevice和instrument协议解析

采集数据来自于 tidevice\_instruments.py  iter_cpu_memory方法

```py
def iter_cpu_memory(self) -> Iterator[dict]:
    """
    Close connection after iterator stop

    Iterator content eg:
        [{'CPUCount': 2,
        'EnabledCPUs': 2,
        'EndMachAbsTime': 2158497307470,
        'PerCPUUsage': [{'CPU_NiceLoad': 0.0,
                        'CPU_SystemLoad': -1.0,
                        'CPU_TotalLoad': 13.0,
                        'CPU_UserLoad': -1.0},
                        {'CPU_NiceLoad': 0.0,
                        'CPU_SystemLoad': -1.0,
                        'CPU_TotalLoad': 31.0,
                        'CPU_UserLoad': -1.0}],
        'StartMachAbsTime': 2158473307786,
        'SystemCPUUsage': {'CPU_NiceLoad': 0.0,
                            'CPU_SystemLoad': -1.0,
                            'CPU_TotalLoad': 44.0,
                            'CPU_UserLoad': -1.0},
        'Type': 33},
        {'EndMachAbsTime': 2158515468993,
        "cpuUsage", "ctxSwitch", "intWakeups", "physFootprint",
            "memResidentSize", "memAnon", "pid"
        'Processes': {0: [0.20891292720792148, # cpuUsage
                            335770139, # contextSwitch
                            120505483, # interruptWakeups
                            7913472,   # physical Footprint
                            869646336, # memory RSS
                            232210432, # memory Anon?
                            0],        # pid
                        1: [0.0005502246751775457,
                            691065,
                            6038,
                            12353840,
                            4177920,
                            12255232,
                            1]
                        }
        }]
    """
    config = {
        "bm": 0,
        "cpuUsage": True,
        "procAttrs": [
            "memVirtualSize", "cpuUsage", "ctxSwitch", "intWakeups",
            "physFootprint", "memResidentSize", "memAnon", "pid"
        ],
        "sampleInterval": 1000000000, # 1e9 ns == 1s
        "sysAttrs": [
            "vmExtPageCount", "vmFreeCount", "vmPurgeableCount",
            "vmSpeculativeCount", "physMemSize"
        ],
        "ur": 1000
    }

    channel_id = self.make_channel(InstrumentsService.Sysmontap)
    self.call_message(channel_id, "setConfig:", [config])
    self.call_message(channel_id, "start", [])

    # channel = self.make_channel(
    #     "com.apple.instruments.server.services.processcontrol")
    # aux = AUXMessageBuffer()
    # aux.append_obj(1)  # TODO: pid
    # payload = DTXPayload.build("startObservingPid:", aux)
    # self.send_dtx_message(channel, payload)
    notification_channel_id = (1<<32) - channel_id
    try:
        for m in self.iter_message(Event.NOTIFICATION):
            if m.flags == 0x01 and m.channel_id == notification_channel_id:
                yield m.result
    except GeneratorExit:
        self.close() # 停止connection，防止消息不停的发过来，暂时不会别的方法
        # print("Stop channel")
        ## The following code is not working
        # self.call_message(channel_id, "stopSampling")
        # aux = AUXMessageBuffer()
        # aux.append_obj(channel_id)
        # self.send_dtx_message(channel_id, DTXPayload.build('_channelCanceled:', aux))
```

- 该方法与instrument建立通信 发送协议请求，获取数据
  其数据结果包含了
- cpu数量
- 启动的cpu
- 单个cpu使用率
- ...

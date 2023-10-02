---
title: SoloX之电池性能采集
date: 2023-07-14 08:35:36
tags: 
- 测试开发 
- 性能 
- 工具
categories: 测试开发
---
## 安卓

### 获取安卓电量以及温度

- 首先切换到非充电模式  `dumpsys battery set status 1`
- 然后获取电量信息以及温度 `dumpsys battery`

```python
def getAndroidBattery(self, noLog=False):
    """Get android battery info, unit:%"""
    # Switch mobile phone battery to non-charging state
    cmd = 'dumpsys battery set status 1'
    adb.shell(cmd=cmd, deviceId=self.deviceId)
    # Get phone battery info
    cmd = 'dumpsys battery'
    output = adb.shell(cmd=cmd, deviceId=self.deviceId)
    level = int(re.findall(u'level:\s?(\d+)', output)[0])
    temperature = int(re.findall(u'temperature:\s?(\d+)', output)[0]) / 10
    if noLog is False:
         apm_time = datetime.datetime.now().strftime('%H:%M:%S.%f')
         f.add_log(os.path.join(f.report_dir,'battery_level.log'), apm_time, level)
         f.add_log(os.path.join(f.report_dir,'battery_tem.log'), apm_time, temperature)
    return level, temperature
```

命令行会输出：

```shell
(base) PS C:\Users\fanlu> adb shell dumpsys battery
* daemon not running; starting now at tcp:5037
* daemon started successfully
Current Battery Service state:
  AC powered: false
  USB powered: true
  Wireless powered: false
  Max charging current: 150000
  Max charging voltage: 5000000
  Charge counter: 3904000
  status: 2
  health: 2
  present: true
  level: 96
  scale: 100
  voltage: 4380
  temperature: 277
  technology: Li-poly
```

## IOS

ios直接使用tidevice的接口获取数据

```python
def getiOSBattery(self, noLog=False):
    """Get ios battery info, unit:%"""
    d  = tidevice.Device()
    ioDict =  d.get_io_power()
    tem = m._setValue(ioDict['Diagnostics']['IORegistry']['Temperature'])
    current = m._setValue(abs(ioDict['Diagnostics']['IORegistry']['InstantAmperage']))
    voltage = m._setValue(ioDict['Diagnostics']['IORegistry']['Voltage'])
    power = current * voltage / 1000
    if noLog is False:
        apm_time = datetime.datetime.now().strftime('%H:%M:%S.%f')
        f.add_log(os.path.join(f.report_dir,'battery_tem.log'), apm_time, tem) # unknown
        f.add_log(os.path.join(f.report_dir,'battery_current.log'), apm_time, current) #mA
        f.add_log(os.path.join(f.report_dir,'battery_voltage.log'), apm_time, voltage) #mV
        f.add_log(os.path.join(f.report_dir,'battery_power.log'), apm_time, power)
    return tem, current, voltage, power

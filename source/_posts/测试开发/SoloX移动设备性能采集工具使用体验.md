---
title: SoloX移动设备性能采集工具使用体验
date: 2023-06-25 21:46:33
tags: 测试开发 性能 工具
---
> https://github.com/smart-test-ti/SoloX

基于python的移动设备性能测试工具，支持android，通过itunes（tidevice的技术）支持了ios设备。

## 功能介绍

使用python -m solox 启动服务，会自动打开web客户端：默认端口为50003。

### 性能测试

- 设备类型支持安卓和苹果
- 支持选择设备列表
- 支持选择目标app 
- 支持选择子进程 (仅安卓)
- 支持性能指标的开关 （只控制显示，不控制是否采集）
  - CPU 
  - 内存
  - 流量
  - FPS
  - 电池
  - surfaceview (仅安卓)
  - wifi (仅安卓)
  - GPU （仅苹果）
- 支持显示设备信息
  - 品牌
  - 设备名
  - 系统版本
  - 序列号
  - wifi地址
- 支持启动数据采集、停止设备采集（此时生成报告）
- 支持显示logcat 日志 *重要*   (仅安卓)
- 支持安装应用
- 支持查看当前应用启动时间 （安卓支持较好）

- 支持对比模式 （仅安卓）
  多台设备同时连接，同时测试
  支持 
  	- CPU
  	- Memory
  	- NetWorkData
  	- FPS

### 报告管理

- 报告展示
- 报告管理
  - 编辑 （场景名）
  - 导出为excel
  - 删除
- 报告详情
  - 结果对比
  - 保存html
  - 保存图片
  - 单个图片导出

## 功能剖析

### web功能

#### flask服务

web功能基于flask flask-template flask-soketio实现。

```python
app = Flask(__name__, template_folder='templates', static_folder='static')
socketio = SocketIO(app, cors_allowed_origins="*")


def startServer(host: str, port: int):  
    socketio.run(app, host=host, debug=False, port=port)

```

#### 自动打开页面

服务启动时会自动打开浏览器，打开solox前端客户端

```python
def openUrl(host: str, port: int):  
    flag = True  
    while flag:  
        logger.info('Start solox server ...')  
        f = Figlet(font="slant", width=300)  
        print(f.renderText("SOLOX 2. 6. 7"))  
        flag = getServerStatus(host, port)  
    webbrowser.open('http://{}:{}/?platform=Android&lan=en'.format(host, port), new=2)  
    logger.info('Running on http://{}:{}/?platform=Android&lan=en (Press CTRL+C to quit)'.format(host, port))
```

其中，使用了

- Figlet 打印艺术字
- webbrowser  是python内置的一个库，可以打开url

#### 端口检查

服务启动时，还调用了检查端口的函数

- 对于非windows系统，直接干掉进程

```
os.system("lsof -i:%s| grep LISTEN| awk '{print $2}'|xargs kill -9" % port)
```

- 对于windows系统
  - 使用 `netstat` 命令结合 `findstr` 过滤器，在指定端口上查找监听的进程。
  - 读取命令输出的数据，并逐行处理。
  - 提取每行中的进程 PID，并将其添加到 `pid_list` 列表中。
  - 将 `pid_list` 转换为集合，再转换回列表，以去除重复的 PID。
  - 选取列表中的第一个 PID，并使用 `taskkill` 命令强制终止该进程。

#### 启动

- 检查python版本：必选3.10以上
- 然后使用Fire启动

> fire库可以自己的库封装成一个命令 通过python -m 命令调用

其他所有的功能都是由http请求触发，logcat是一个websocket
前端使用jquery处理事件和操作dom；

### 设备监听与信息抓取

1. 前端每次打开页面（`$(document).ready`）后，会自动触发连接按钮的点击事件，从而调用`initializeEnv`方法；
2. `initializeEnv`请求接口 /device/ids?platform=当前选择设备类型
3. 后端接口：

#### 安卓

- 获取到当前的设备id列表 （解析命令行`adb devices`的输出）
  - 获取第一个设备的pkg列表 （adb -s deviceID shell pm list package）
  - 获取设备信息 (安卓基于`adb`,ios基于`tidevice`)

```python
def getDdeviceDetail(self, deviceId, platform):  
    result = {}  
    match(platform):  
        case Platform.Android:  
            result['brand'] = adb.shell(cmd='getprop ro.product.brand', deviceId=deviceId)  
            result['name'] = adb.shell(cmd='getprop ro.product.model', deviceId=deviceId)  
            result['version'] = adb.shell(cmd='getprop ro.build.version.release', deviceId=deviceId)  
            result['serialno'] = adb.shell(cmd='getprop ro.serialno', deviceId=deviceId)  
            cmd = f'ip addr show wlan0 | {self.filterType()} link/ether'            result['wifiadr'] = adb.shell(cmd=cmd, deviceId=deviceId).split(' ')[1]  
        case Platform.iOS:  
            iosInfo = json.loads(self.execCmd('tidevice info --json'))  
            result['brand'] = iosInfo['DeviceClass']  
            result['name'] = iosInfo['DeviceName']  
            result['version'] = iosInfo['ProductVersion']  
            result['serialno'] = iosInfo['SerialNumber']  
            result['wifiadr'] = iosInfo['WiFiAddress']  
        case _:  
            raise Exception('{} is undefined'.format(platform))   
    return result
```

#### ios

- 获取设备信息 （解析命令行 tidevice list --json）
- 获取第一个设备pkg列表： `tidevice --udid 设备uid  applist`
- 获取设备详情，同上

### 性能采集

点击开始按钮，触发启动测试。

- 必须先选中设备和应用
- 分别调用collectCpu、collectMem、collectNetwork、collectFps、collectBattery  如果是ios系统，还要采集GPU collectGpu
- 这些方法再调用collectPers方法（参数包括一个获取采集数据的方法），然后使用[highcharts](https://www.highcharts.com/)绘制图表；其中采集数据的方法作为highcharts实例的数据load方法，并且图表支持切换时间范围；  这个请求函数成功后会设置一个settimeout定时器，一秒钟后再次请求这个接口更新数据，并将数据塞入到图表数据的最新；

性能采集需要比较复杂，打算分成系列的博客慢慢写一下。




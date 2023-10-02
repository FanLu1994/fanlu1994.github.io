---
title: solox之fps性能采集
date: 2023-07-01 22:14:39
tags: 
- 测试开发 
- 性能 
- 工具
categories: 测试开发
---
## 安卓

通过fpsMonitor类获取到fps计算器；

```python
def getAndroidFps(self, noLog=False):
    """get Android Fps, unit:HZ"""
    monitors = FPSMonitor(device_id=self.deviceId, package_name=self.pkgName, frequency=1,
                          surfaceview=self.surfaceview, start_time=TimeUtils.getCurrentTimeUnderline())
    monitors.start()
    fps, jank = monitors.stop()
    if noLog is False:
        apm_time = datetime.datetime.now().strftime('%H:%M:%S.%f')
        f.add_log(os.path.join(f.report_dir,'fps.log'), apm_time, fps)
        f.add_log(os.path.join(f.report_dir,'jank.log'), apm_time, jank)
    return fps, jank
```

在FPSMonitor中使用SurfaceStatsCollector来采集FPS数据

```python
class FPSMonitor(Monitor):
    def __init__(self, device_id, package_name=None, frequency=1.0, timeout=24 * 60 * 60, fps_queue=None,
                 jank_threshold=166, use_legacy=False, surfaceview=True, start_time=None, **kwargs):
        super().__init__(**kwargs)
        self.start_time = start_time
        self.use_legacy = use_legacy
        self.frequency = frequency  # 取样频率
        self.jank_threshold = jank_threshold
        self.device = device_id
        self.timeout = timeout
        self.surfaceview = surfaceview
        if not package_name:
            package_name = self.device.adb.get_foreground_process()
        self.package = package_name
        self.fpscollector = SurfaceStatsCollector(self.device, self.frequency, package_name, fps_queue,
                                                  self.jank_threshold, self.surfaceview, self.use_legacy)
```

SurfaceStatsCollector涵盖的功能有：

#### 获取surface的activity名字

```python
def get_surfaceview_activity(self):
    activity_name = ''
    activity_line = ''
    try:
        dumpsys_result = adb.shell(cmd='dumpsys SurfaceFlinger --list | {} {}'.format(d.filterType(), self.package_name), deviceId=self.device)
        dumpsys_result_list = dumpsys_result.split('\n')    
        for line in dumpsys_result_list:
            if line.startswith('SurfaceView') and line.find(self.package_name) != -1:
                activity_line = line.strip()
                break
        if activity_line:
            if activity_line.find(' ')  != -1:      
                activity_name = activity_line.split(' ')[2]
            else:
                activity_name = activity_line.replace('SurfaceView','').replace('[','').replace(']','')    
        else:
            activity_name = dumpsys_result_list[len(dumpsys_result_list) - 1]
            if not activity_name.__contains__(self.package_name):
                logger.error('get activity name failed, Please provide SurfaceFlinger --list information to the author')
                logger.info('dumpsys SurfaceFlinger --list info: {}'.format(dumpsys_result))
    except Exception:
        traceback.print_exc()
        logger.error('get activity name failed, Please provide SurfaceFlinger --list information to the author')
        logger.info('dumpsys SurfaceFlinger --list info: {}'.format(dumpsys_result))
    return activity_name
```



#### 获取聚焦的activity名字

```python
def get_focus_activity(self):
    activity_name = ''
    activity_line = ''
    dumpsys_result = adb.shell(cmd='dumpsys window windows', deviceId=self.device)
    dumpsys_result_list = dumpsys_result.split('\n')
    for line in dumpsys_result_list:
        if line.find('mCurrentFocus') != -1:
            activity_line = line.strip()
    if activity_line:
        activity_line_split = activity_line.split(' ')
    else:
        return activity_name
    if len(activity_line_split) > 1:
        if activity_line_split[1] == 'u0':
            activity_name = activity_line_split[2].rstrip('}')
        else:
            activity_name = activity_line_split[1]
    if not activity_name:
        activity_name = self.get_surfaceview_activity()        
    return activity_name
```

#### 获取前台进程名字

```python
def get_foreground_process(self):
    focus_activity = self.get_focus_activity()
    if focus_activity:
        return focus_activity.split("/")[0]
    else:
        return ""
```

#### 获取sdk版本

```python
def get_sdk_version(self):
    sdk_version = int(adb.shell(cmd='getprop ro.build.version.sdk', deviceId=self.device))
    return sdk_version
```

#### 版本兼容

为了兼容不同版本的安卓系统，做了两套方法，在启动时进行区分：

```python
def start(self, start_time):
    if not self.use_legacy_method:
        try:
            self.focus_window = self.get_focus_activity()
            if self.focus_window.find('$') != -1:
                self.focus_window = self.focus_window.replace('$', '\$')
        except Exception:
            logger.warning(u'Unable to dynamically obtain the current activity name, using page_ Flip statistics full screen frame rate')
            self.use_legacy_method = True
            self.surface_before = self._get_surface_stats_legacy()
    else:
        logger.debug("dumpsys SurfaceFlinger --latency-clear is none")
        self.use_legacy_method = True
        self.surface_before = self._get_surface_stats_legacy()
    self.collector_thread = threading.Thread(target=self._collector_thread)
    self.collector_thread.start()
    self.calculator_thread = threading.Thread(target=self._calculator_thread, args=(start_time,))
    self.calculator_thread.start()
```

这里分别启动了采集线程和计算线程。

#### 旧版本方法

> 获取当前的surface索引和时间戳

```python
def _get_surface_stats_legacy(self):
    """Legacy method (before JellyBean), returns the current Surface index
         and timestamp.

    Calculate FPS by measuring the difference of Surface index returned by
    SurfaceFlinger in a period of time.

    Returns:
        Dict of {page_flip_count (or 0 if there was an error), timestamp}.
    """
    cur_surface = None
    timestamp = datetime.datetime.now()
    ret = adb.shell(cmd="service call SurfaceFlinger 1013", deviceId=self.device)
    if not ret:
        return None
    match = re.search('^Result: Parcel\((\w+)', ret)
    if match:
        cur_surface = int(match.group(1), 16)
        return {'page_flip_count': cur_surface, 'timestamp': timestamp}
    return None
```

#### 新版本方法

##### 首先获取到帧数据列表

```python
def _get_surfaceflinger_frame_data(self):
    """Returns collected SurfaceFlinger frame timing data.
    return:(16.6,[[t1,t2,t3],[t4,t5,t6]])
    Returns:
        A tuple containing:
        - The display's nominal refresh period in seconds.
        - A list of timestamps signifying frame presentation times in seconds.
        The return value may be (None, None) if there was no data collected (for
        example, if the app was closed before the collector thread has finished).
    """
    # shell dumpsys SurfaceFlinger --latency <window name>
    # prints some information about the last 128 frames displayed in
    # that window.
    # The data returned looks like this:
    # 16954612
    # 7657467895508     7657482691352     7657493499756
    # 7657484466553     7657499645964     7657511077881
    # 7657500793457     7657516600576     7657527404785
    # (...)
    #
    # The first line is the refresh period (here 16.95 ms), it is followed
    # by 128 lines w/ 3 timestamps in nanosecond each:
    # A) when the app started to draw
    # B) the vsync immediately preceding SF submitting the frame to the h/w
    # C) timestamp immediately after SF submitted that frame to the h/w
    #
    # The difference between the 1st and 3rd timestamp is the frame-latency.
    # An interesting data is when the frame latency crosses a refresh period
    # boundary, this can be calculated this way:
    #
    # ceil((C - A) / refresh-period)
    #
    # (each time the number above changes, we have a "jank").
    # If this happens a lot during an animation, the animation appears
    # janky, even if it runs at 60 fps in average.
    #

    # Google Pixel 2 android8.0 dumpsys SurfaceFlinger --latency结果
    # 16666666
    # 0       0       0
    # 0       0       0
    # 0       0       0
    # 0       0       0
    # 但华为 荣耀9 android8.0 dumpsys SurfaceFlinger --latency结果是正常的 但数据更新很慢  也不能用来计算fps
    # 16666666
    # 9223372036854775807     3618832932780   9223372036854775807
    # 9223372036854775807     3618849592155   9223372036854775807
    # 9223372036854775807     3618866251530   9223372036854775807

    refresh_period = None
    timestamps = []
    nanoseconds_per_second = 1e9
    pending_fence_timestamp = (1 << 63) - 1
    if self.surfaceview is not True:
        results = adb.shell(
            cmd='dumpsys SurfaceFlinger --latency %s' % self.focus_window, deviceId=self.device)
        results = results.replace("\r\n", "\n").splitlines()
        refresh_period = int(results[0]) / nanoseconds_per_second
        results = adb.shell(cmd='dumpsys gfxinfo %s framestats' % self.package_name, deviceId=self.device)
        results = results.replace("\r\n", "\n").splitlines()
        if not len(results):
            return (None, None)
        isHaveFoundWindow = False
        PROFILEDATA_line = 0
        activity = self.focus_window
        if self.focus_window.__contains__('#'):
            activity = activity.split('#')[0]
        for line in results:
            if not isHaveFoundWindow:
                if "Window" in line and activity in line:
                    isHaveFoundWindow = True
            if not isHaveFoundWindow:
                continue
            if "PROFILEDATA" in line:
                PROFILEDATA_line += 1
            fields = []
            fields = line.split(",")
            if fields and '0' == fields[0]:
                timestamp = [int(fields[1]), int(fields[2]), int(fields[13])]
                if timestamp[1] == pending_fence_timestamp:
                    continue
                timestamp = [_timestamp / nanoseconds_per_second for _timestamp in timestamp]
                timestamps.append(timestamp)
            if 2 == PROFILEDATA_line:
                break
    else:
        self.focus_window = self.get_surfaceview_activity()
        results = adb.shell(
            cmd='dumpsys SurfaceFlinger --latency %s' % self.focus_window, deviceId=self.device)
        results = results.replace("\r\n", "\n").splitlines()
        if not len(results):
            return (None, None)
        if not results[0].isdigit():
            return (None, None)
        try:
            refresh_period = int(results[0]) / nanoseconds_per_second
        except Exception as e:
            logger.exception(e)
            return (None, None)
        # If a fence associated with a frame is still pending when we query the
        # latency data, SurfaceFlinger gives the frame a timestamp of INT64_MAX.
        # Since we only care about completed frames, we will ignore any timestamps
        # with this value.

        for line in results[1:]:
            fields = line.split()
            if len(fields) != 3:
                continue
            timestamp = [int(fields[0]), int(fields[1]), int(fields[2])]
            if timestamp[1] == pending_fence_timestamp:
                continue
            timestamp = [_timestamp / nanoseconds_per_second for _timestamp in timestamp]
            timestamps.append(timestamp)
    return (refresh_period, timestamps)
```

然后将数据插入到队列中

```python
refresh_period, new_timestamps = self._get_surfaceflinger_frame_data()
if refresh_period is None or new_timestamps is None:
    self.focus_window = self.get_focus_activity()
    logger.warning("refresh_period is None or timestamps is None")
    continue
timestamps += [timestamp for timestamp in new_timestamps
               if timestamp[1] > self.last_timestamp]
if len(timestamps):
    first_timestamp = [[0, self.last_timestamp, 0]]
    if not is_first:
        timestamps = first_timestamp + timestamps
    self.last_timestamp = timestamps[-1][1]
    is_first = False
else:
    is_first = True
    cur_focus_window = self.get_focus_activity()
    if self.focus_window != cur_focus_window:
        self.focus_window = cur_focus_window
        continue
self.data_queue.put((refresh_period, timestamps, time.time()))
time_consume = time.time() - before
delta_inter = self.frequency - time_consume
if delta_inter > 0:
    time.sleep(delta_inter)
```

#### 计算线程计算fps和jank

##### 旧版本方法

不知道为什么fps设置最高就是60

通过一段时间内的帧总数除以时间差获取到fps

```python
td = data['timestamp'] - self.surface_before['timestamp']
seconds = td.seconds + td.microseconds / 1e6
frame_count = (data['page_flip_count'] -
               self.surface_before['page_flip_count'])
fps = int(round(frame_count / seconds))
if fps > 60:
    fps = 60
self.surface_before = data
# logger.debug('FPS:%2s'%fps)
collect_fps = fps
```

#### 新版本方法

1. 首先拿到帧数
2. 对异常数据的处理
3. 2、3、4帧数为啥这样处理？

```python
def _calculate_results_new(self, refresh_period, timestamps):
    frame_count = len(timestamps)
    if frame_count == 0:
        fps = 0
        jank = 0
    elif frame_count == 1:
        fps = 1
        jank = 0
    elif frame_count == 2 or frame_count == 3 or frame_count == 4:
        seconds = timestamps[-1][1] - timestamps[0][1]
        if seconds > 0:
            fps = int(round((frame_count - 1) / seconds))
            jank = self._calculate_janky(timestamps)
        else:
            fps = 1
            jank = 0
    else:
        seconds = timestamps[-1][1] - timestamps[0][1]
        if seconds > 0:
            fps = int(round((frame_count - 1) / seconds))
            jank = self._calculate_jankey_new(timestamps)
        else:
            fps = 1
            jank = 0
    return fps, jank
```

##### 计算jank

###### 旧版本

```python
def _calculate_janky(self, timestamps):
    tempstamp = 0
    jank = 0
    for timestamp in timestamps:
        if tempstamp == 0:
            tempstamp = timestamp[1]
            continue
        costtime = timestamp[1] - tempstamp
        if costtime > self.jank_threshold:
            jank = jank + 1
        tempstamp = timestamp[1]
    return jank
```

##### 新版本

通过预设的jank_threshold阈值判断是否有jank（默认166ms的渲染时间）

通过过去4帧的时间，也即过去三帧的平均渲染时间 *2，如果当前帧渲染时间超过这个值，并且当前帧渲染时间大于上一帧渲染时间，判断为一次jank

```python
def _calculate_jankey_new(self, timestamps):
    twofilmstamp = 83.3 / 1000.0
    tempstamp = 0
    jank = 0
    for index, timestamp in enumerate(timestamps):
        if (index == 0) or (index == 1) or (index == 2) or (index == 3):
            if tempstamp == 0:
                tempstamp = timestamp[1]
                continue
            costtime = timestamp[1] - tempstamp
            if costtime > self.jank_threshold:
                jank = jank + 1
            tempstamp = timestamp[1]
        elif index > 3:
            currentstamp = timestamps[index][1]
            lastonestamp = timestamps[index - 1][1]
            lasttwostamp = timestamps[index - 2][1]
            lastthreestamp = timestamps[index - 3][1]
            lastfourstamp = timestamps[index - 4][1]
            tempframetime = ((lastthreestamp - lastfourstamp) + (lasttwostamp - lastthreestamp) + (
                    lastonestamp - lasttwostamp)) / 3 * 2
            currentframetime = currentstamp - lastonestamp
            if (currentframetime > tempframetime) and (currentframetime > twofilmstamp):
                jank = jank + 1
    return jank
```

## IOS

同样使用instrument协议

```python
def iter_fps(d: BaseDevice) -> Iterator[Any]:
    with d.connect_instruments() as ts:
        for data in ts.iter_opengl_data():
            fps = data['CoreAnimationFramesPerSecond'] # fps from GPU
            # print("FPS:", fps)
            yield DataType.FPS, {"fps": fps, "time": time.time(), "value": fps}
```


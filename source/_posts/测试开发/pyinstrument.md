---
title: pyinstrument
date: 2023-11-21 22:41:31
tags:
  - DEBUG
  - python
  - 工具
categories: 测试开发
---
>https://pyinstrument.readthedocs.io/en/latest/guide.html

这是一个python性能分析工具.

大致有两种使用方式：
1. 命令行直接运行脚本
```
pyinstrument script.py
```
2. 嵌入到项目中
-  简单包裹代码块
```python
from pyinstrument import Profiler

profiler = Profiler()
profiler.start()

# code you want to profile

profiler.stop()

profiler.print()

```
- 作为中间件嵌入到`django`、`flask`、`fastapi`、`falcon`中使用

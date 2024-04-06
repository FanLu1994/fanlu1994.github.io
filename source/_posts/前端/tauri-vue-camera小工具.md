---
title: tauri-vue-camera小工具
date: 2024-04-06 11:48:04
tags:
  - 前端
  - 工具
categories: 前端
---
仿照https://www.bilibili.com/video/BV1AQ4y1c7Le?p=2
> 原版使用electron实现
> 打算使用tauri来实现一下

项目地址：https://github.com/FanLu1994/tauri-vue-camera

## 调用摄像头
1. web中调用摄像头需要申请权限
```js
await navigator.mediaDevices.getUserMedia({  
  audio: true,  
  video: true,  
});
```

2. 然后获取摄像头列表
```js
// 这里使用了vueuse的hook https://vueuse.org/core/useDevicesList/
const { videoInputs: cameras } = useDevicesList({  
  requestPermissions: true,  
  onUpdated() {  
    if (!cameras.value.find(i => i.deviceId === currentCamera.value)) {  
      currentCamera.value = cameras.value[0].deviceId  
      onCameraChange(currentCamera.value)  
      }  
    },  
})
```
3. 然后获取视频流
```html
// template
<video ref="videoRef" muted autoplay class="h-100 w-auto video"  
       :class="{'circle':shape==='circle','rectangle':shape==='rectangle','mirror':mirror?'mirror':''}"  
       data-tauri-drag-region/>
```
```js

//js
function onCameraChange(cameraId) {  
  currentCamera.value = cameraId  
  if (currentStream){  
    currentStream.getTracks().forEach(track => track.stop());  
  }  
  if(currentCamera.value){  
    let constraints = { deviceId: { exact: currentCamera.value } };  
    navigator.mediaDevices  
        .getUserMedia({ video: constraints, audio: false })  
        .then(function (stream) {  
          videoRef.value.srcObject = stream;  
          currentStream = stream;  
        });  
  }  
}
```
## 无边框和拖动
都可以在/src-tauri/tauri.conf.json中配置

### 无边框配置
```json
{
	...
	"tauri": {  
		... 
	  "windows": [  
	    {  
	      "title": "tauri-vue-camera",  
	      "label": "tauri-vue-camera",  
	      "width": 300,  
	      "height": 200,  
	      "center": true,  // 是否在屏幕中央
	      "resizable": true,  // 是否可修改尺寸
	      "transparent": true,  // 背景是否透明
	      "hiddenTitle": true,  // 是否隐藏标题栏
	      "alwaysOnTop":true,  // 是否保持在最前
	      "decorations": false  // 是否无边框
	    }  
	  ],  
		...  
	}
	...
}
```
### 拖动配置
```json
{
	...
	"tauri": {  
		... 
		"window": {  
		  "startDragging": true  
		}
		...  
	}
	...
}
```
在html中指定可以拖动的区域，加上`data-tauri-drag-region`
```html
<div class="container" data-tauri-drag-region>
```


## 摄像头镜像
镜像使用css来实现即可
html  mirror变量来控制
```html
<video ref="videoRef" muted autoplay class="h-100 w-auto video"  
       :class="{'circle':shape==='circle','rectangle':shape==='rectangle','mirror':mirror?'mirror':''}"  
       data-tauri-drag-region/>
```
css:
```css
.video.mirror{  
  transform: rotateY(180deg);  
}
```


## 退出窗口
因为隐藏了标题栏，没有了退出窗口，所以自定义一个点击事件来退出程序
### 首先注册一个tauri事件
```rust
#[tauri::command]  
fn close(){  
    std::process::exit(0);  
}

fn main() {  

    tauri::Builder::default()  
        .invoke_handler(tauri::generate_handler![greet])  
        .invoke_handler(tauri::generate_handler![close])  // 在这里注册
        .run(tauri::generate_context!())  
        .expect("error while running tauri application");  
}
```
### 然后自定义点击回调事件
```js
const onClose = async () => {  
  await invoke('close', {})  
}
```

---
title: rrweb使用
date: 2023-08-23 23:09:10
tags: 
- 前端 
- 工具
categories: 前端
---
## 介绍
rrweb是一种通过记录页面dom元素的变化，以及鼠标或者键盘输入的变化，来实现web操作的录制回放的功能。
官网在这里：[rrweb](https://www.rrweb.io/)
源码在这里：https://github.com/rrweb-io/rrweb
## 在vue中使用

### 新建vue3项目
```shell
npm install -g @vue/cli
vue create rrweb-vue-demo
```

### 创建两个路由页面
分别用来录制操作和回放操作
```ts
import * as Router from "vue-router";  
  
const routes = [  
    {path:'/',component: import("../Record.vue")},  
    { path: '/play', component: import("../Play.vue") },  
]  
  
export const router = Router.createRouter({  
    // 4. 内部提供了 history 模式的实现。为了简单起见，我们在这里使用 hash 模式。  
    history: Router.createWebHashHistory(),  
    routes, // `routes: routes` 的缩写  
})
```

### 录制和播放
### 安装rrweb
```
npm install rrweb rrweb-player
```
### 录制
这里录制20s自动停止，你也可以改成手动停止
```vue
<template>  
  <button @click="startRecord">开启记录</button>  
  <HelloWorld msg="Vite + Vue" />  
  <button @click="gotoPlay">前往回放</button>  
</template>  
  
<script setup  lang="ts">  
import HelloWorld from './components/HelloWorld.vue'  
import * as rrweb from "rrweb";  
import {useRouter} from "vue-router";  
const startRecord = () => {  
  //record() 方法启动录制  
  //stopFn为暂停录制的方法  
  let stopFn = rrweb.record({  
    //12秒后停止页面的录制，如果想一直录得话可以去掉。  
    emit(event) {  
      setTimeout(() => {  
        stopFn();  
      }, 20000);  
      // 用任意方式存  储 event      // store.commit("updateEvents", { event: event });      let eventList: any[] = JSON.parse(localStorage.getItem("events"))  
      if (eventList == null){  
        eventList = [event]  
      }else{  
        eventList.push(event)  
      }  
      localStorage.setItem("events",JSON.stringify(eventList))  
  
      // lo  
      console.log(event)  
    },  
  });  
  console.log("开启记录")  
};  
  
const route = useRouter()  
const gotoPlay = ()=>{  
  
  console.log(route)  
  route.push("/play")  
}  
  
</script>  
  
<style scoped>  
  
</style>
```

### 回放
```vue
<template>  
  <button @click="startPlay">回放</button>  
  <div class="counte">  
    <div id="playback"></div>  
  </div></template>  
  
<script lang="ts" setup>  
import rrwebPlayer from "rrweb-player";  
import "rrweb-player/dist/style.css"  
import {ref} from "vue";  
  
const player = ref(null)  
const startPlay = ()=>{  
  const events = JSON.parse(localStorage.getItem("events"))  
  player.value = new rrwebPlayer({  
    target: document.getElementById("playback"),  
    props:{  
      events:events,  
      speedOption:[1,2,5,10]  
    }  
  })  
}  
  
</script>  
  
<style scoped>  
  
</style>
```
## 用途
### 自动化测试
对于web的项目，可以预先录制一份操作数据。
然后测试时通过回放操作来进行回放。
不过如何发现测试中的错误，我还没有深入研究。
当然，如果要进行自动化测试，前端还有很多自动化的工具可以使用，这个未必好用。
### bug复现
这个应该是rrweb最重要的功能。
线上如果出现bug，可以利用rrweb录制的结果去回放bug现场。什么时机判断出现了bug呢？可以让用户一键上报，也可以进行埋点，发现问题就自动上报。
### 操作说明
一些需要教育用户的复杂操作，可以通过rrweb录制一份操作指南，让用户去观看学习。
### 其他
其他

## ！！ 注意事项
因为回放的原理是通过iframe加载网页，来回放操作，如果录制时和回放时的内容不一样，那就game over了

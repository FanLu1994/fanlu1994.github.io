---
title: 微信公众号模板工具开发-02（end）
date: 2023-06-17 21:41:45
tags: nuxt axiss 前端
---
## 代码高亮
使用prism.js来实现代码高亮。
每次渲染完md文档过后，调用Prism.hightAll()
#### 安装
```shell
npm install prismjs@1.29.0 @types/prismjs@1.26.0
```
#### 配置
```ts
// nuxt.config.ts
vite: {  
  plugins: [  
    prismjs({  
      // 添加支持的高亮的语言, 如果需要支持全部的话改成这样:  languages: "all"  
      languages: ['cpp',  
        'javascript',  
        'bash',  
        'dart',  
        'sql',  
        'css',  
        'html',  
        'java',  
        'json',  
        'sass',  
        'scss',  
        'c',  
        'log',  
        'swift',  
        'md',  
        'nginx',  
        'yaml',  
        'xml',  
        'shell',  
        'ts'  
      ],  
      // 配置prism插件 toolbar是后面两个插件依赖的插件.  
      // show-language: 显示语言名。  
      // copy-to-clipboard: 添加复制代码功能。  
      plugins: ['line-numbers'],  
      // 主题名称,支持的主题可以在 node_moduels中安装此库的目录下寻找。  
      theme: "okaidia",  
      css: true  
    })  
  ]  
},

```
#### 使用
```ts
import Prism from 'prismjs'
...

// 每次渲染完之后调用这个函数，便会自动构建代码样式
Prism.highlightAll()
...
```

## 引用归纳
将文章中插入的链接，以引用文献的样子放在文章最后。 这个想法来自于阮一峰的周刊。
前端由于浏览器跨域的限制，是无法根据url去获取到网站标题的，（不知道wasm是否可以）。所以需要一个后端的简单接口，使用go或者python都可以，我用的gin写了个接口
```go
func GetWebsiteTitle(c *gin.Context) {  
   url := c.Query("url")  
  
   if url == "" {  
      c.JSON(http.StatusBadRequest, APIResponse{  
         ErrorCode:    http.StatusBadRequest,  
         ErrorMessage: "URL parameter is missing",  
         Result:       "",  
      })  
      return  
   }  
  
   // 使用Colly爬取网页标题  
   collector := colly.NewCollector()  
   collector.SetRequestTimeout(1 * time.Second)  
   var title string  
  
   collector.OnHTML("title", func(e *colly.HTMLElement) {  
      title = e.Text  
   })  
  
   err := collector.Visit(url)  
   if err != nil {  
      c.JSON(http.StatusOK, APIResponse{  
         ErrorCode:    http.StatusInternalServerError,  
         ErrorMessage: "Failed to fetch website data",  
         Result:       "",  
      })  
      return  
   }  
  
   c.JSON(http.StatusOK, APIResponse{  
      ErrorCode:    http.StatusOK,  
      ErrorMessage: "",  
      Result:       title,  
   })  
}
```
然后在nuxt中请求数据方法是 useFetch ，获取所有url的标题
```ts
import {useFetch} from "#app";

const { data, pending, error, refresh } = await useFetch("http://localhost:7777/md/get_website_title",{  
   query:{  
     url:links[i].href  
   }  
 })
 console.log(data.value.result)
```



## 问题

1. 微信样式问题
	发现显示的内容粘贴到微信之后，还能保持该有的样式，发表出去之后，样式反而丢了，很蛋疼
2. 引用列表显示的还不是很完美，需要再搞搞

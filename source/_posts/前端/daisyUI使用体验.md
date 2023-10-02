---
title: daisyUI使用体验
date: 2023-06-20 23:28:33
tags: 
- 前端 
- UI
categories: 前端
---
## tailwind介绍
> https://www.tailwindcss.cn/docs/customizing-colors#-8

*以下介绍来自chatGPT:*
Tailwind CSS 是一个现代化的 CSS 框架，它提供了一套可复用的构建块和实用工具，用于快速开发现代化的、自定义的用户界面。

与其他 CSS 框架相比，Tailwind CSS 的一个主要区别是它并没有预定义的样式类。相反，它提供了一组原子级的 CSS 类，每个类都对应一个特定的样式属性。这些类的命名是基于其功能而不是视觉效果，例如 "text-red-500" 表示红色文本颜色，"bg-blue-200" 表示蓝色背景颜色，"p-4" 表示边距为 4 的元素等等。通过将这些类组合在一起，开发者可以轻松地构建自定义的样式。

使用 Tailwind CSS，开发者可以通过组合这些原子类来创建出复杂的布局和设计，而不需要手动编写大量的 CSS。这使得开发过程更加快速和高效，并且可以在不同的项目中实现一致的设计风格。

Tailwind CSS 还提供了一些有用的实用工具，如网格系统、响应式设计类、颜色调色板、阴影效果等。它还支持定制化配置，可以根据项目需求进行个性化设置，并且可以通过插件系统进行扩展。

总的来说，Tailwind CSS 是一个强大的 CSS 框架，它通过提供原子级的类和实用工具，使得开发者可以更轻松、高效地构建自定义的用户界面。它的灵活性和可定制化的特点使其在现代 Web 开发中广受欢迎。

*个人使用体验*：
优点：
- 节省时间，提升了开发速度
- 丰富的预制样式，可以提供样式设计的思路
缺点：
- 代码会比较混乱，一个元素的class可能会定义的特别长
- 后期不好维护，每个元素都自定义了，后面要修改就比较麻烦 （当然也可以封装）
- 可能与其他组件样式冲突
- 导致元素的基础样式丢失，比如h1 h2等，不过可以使用prose插件解决 （具体见：https://fanlu.top/2023/06/12/%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%8F%B7%E6%A8%A1%E6%9D%BF%E5%B7%A5%E5%85%B7%E5%BC%80%E5%8F%91-01/）

结论：小项目建议使用，大项目不建议

## daisyUI介绍
> https://daisyui.com/docs/install/

这是一个基于tailwindcss的组件库，现在tailwindcss也有自己的组件库（https://tailwindui.com/components），不过是收费的。

daisyui是一个简单的组件库，这里的简单是指它的组件真的很简单，只有预设的样式，没有提供任何原生以外的组件功能。
想到用这个，主要是因为element的原生样式太不能满足需求了，修改也比较麻烦（其实现在看还好）。

## 简单使用
1. 安装 daisyUI:

```undefined
npm i -D daisyui
```

2. 然后，在你的`tailwind.config.js`文件里追加 daisyUI 的设置:

```js
module.exports = {
  //...
  plugins: [require("daisyui")],
}
```

在组件中这样使用
```html
<button class="btn btn-success btn-sm py-0 text-sm leading-3" @click="copy">复制</button>
```

## 与elementui的比较
### 优势：
- 样式美观，自定义很方便
- 功能好修改，更接近原生
劣势：
- 功能不够丰富，大部分功能需要自己单独写
- 组件目前还不如element丰富


## 体验总结
建议作为兴趣使用，如果想要快速出活，还是用其他的组件库比较好。
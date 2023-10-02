---
title: vben-admin换肤实现
date: 2023-05-16 08:19:08
tags: 前端搬砖指南
categories: 前端
---
最近在github上看到了一个后台管理的前端项目，使用了vue3+ts+vite+ant-vue的技术，看起来很不错，功能特别丰富，clone下来发现代码也写的特别好，比我现在的小白代码根本不在同一个等级，因此想要学习一下。 个人觉得从一个功能抽丝剥茧来学习一个功能的写法可能会对自己的技术提高有帮助。

项目中的侧边栏提供了超多的主题选项，可以丰富的变换主题。因此本文想分析一下这个换肤是如何实现的。

![](https://secure2.wostatic.cn/static/wGUf6UvBJUZVXfAPhrzaF/image.png?auth_key=1684196309-2jkBMzvREcZ21xSJjxnEvU-0-4173f6dd779469091bae7835b4c93aa7)

## 黑色/亮色主题切换

主题切换组件AppDarkModeToggle.vue

- 定义点击事件toggleDarkMode
    1. 调用设置黑色主题函数setDarkMode

        修改pinia状态中的dark模式，并将变量存储到localStorage中
    2. 调用updateDarkTheme
        - 获取htmlRoot dom节点，即本项目应用的根节点
        - 判断根节点是否包含dark class定义
        - 如果是dark
            - 判断是否为生产模式，并加载dark主题css（由vite-plugin-theme支持）
            - 将根节点的data-teme设置为dark
            - 并添加class为dark
        - 如果不是dark
            - 将根节点data-theme设置为light
            - 并且移除dark class

```Python
这里修改data-theme为dark，利用了less中条件判断语句
例如：

  html[data-theme='dark'] {
    .@{prefix-cls} {
      border: 1px solid rgb(196 188 188);
    }
  }

ps：less还支持动态变量名，6666
```
    3. 调用updateHeaderBgColor修改header的背景色
        - 判断是否为dark模式，获取到颜色，如果不是暗色，那就获取当前设置的颜色
 color = appStore.getHeaderSetting.bgColor;
        - 将获取到的颜色设置css变量  setCssVar

```JavaScript
export function setCssVar(prop: string, val: any, dom = docEle) {
  console.log(prop,val)
  dom.style.setProperty(prop, val);
}
```
        - 计算得到hover颜色（亮度提高6），同样设置css变量

            这里用到了自定义的颜色函数，我觉得很有用

```TypeScript
/**
 * 判断是否 十六进制颜色值.
 * 输入形式可为 #fff000 #f00
 *
 * @param   String  color   十六进制颜色值
 * @return  Boolean
 */
export function isHexColor(color: string) {
  const reg = /^#([0-9a-fA-F]{3}|[0-9a-fA-f]{6})$/;
  return reg.test(color);
}

/**
 * RGB 颜色值转换为 十六进制颜色值.
 * r, g, 和 b 需要在 [0, 255] 范围内
 *
 * @return  String          类似#ff00ff
 * @param r
 * @param g
 * @param b
 */
export function rgbToHex(r: number, g: number, b: number) {
  // tslint:disable-next-line:no-bitwise
  const hex = ((r << 16) | (g << 8) | b).toString(16);
  return '#' + new Array(Math.abs(hex.length - 7)).join('0') + hex;
}

/**
 * Transform a HEX color to its RGB representation
 * @param {string} hex The color to transform
 * @returns The RGB representation of the passed color
 */
export function hexToRGB(hex: string) {
  let sHex = hex.toLowerCase();
  if (isHexColor(hex)) {
    if (sHex.length === 4) {
      let sColorNew = '#';
      for (let i = 1; i < 4; i += 1) {
        sColorNew += sHex.slice(i, i + 1).concat(sHex.slice(i, i + 1));
      }
      sHex = sColorNew;
    }
    const sColorChange: number[] = [];
    for (let i = 1; i < 7; i += 2) {
      sColorChange.push(parseInt('0x' + sHex.slice(i, i + 2)));
    }
    return 'RGB(' + sColorChange.join(',') + ')';
  }
  return sHex;
}

export function colorIsDark(color: string) {
  if (!isHexColor(color)) return;
  const [r, g, b] = hexToRGB(color)
    .replace(/(?:\(|\)|rgb|RGB)*/g, '')
    .split(',')
    .map((item) => Number(item));
  return r * 0.299 + g * 0.578 + b * 0.114 < 192;
}

/**
 * Darkens a HEX color given the passed percentage
 * @param {string} color The color to process
 * @param {number} amount The amount to change the color by
 * @returns {string} The HEX representation of the processed color
 */
export function darken(color: string, amount: number) {
  color = color.indexOf('#') >= 0 ? color.substring(1, color.length) : color;
  amount = Math.trunc((255 * amount) / 100);
  return `#${subtractLight(color.substring(0, 2), amount)}${subtractLight(
    color.substring(2, 4),
    amount,
  )}${subtractLight(color.substring(4, 6), amount)}`;
}

/**
 * Lightens a 6 char HEX color according to the passed percentage
 * @param {string} color The color to change
 * @param {number} amount The amount to change the color by
 * @returns {string} The processed color represented as HEX
 */
export function lighten(color: string, amount: number) {
  color = color.indexOf('#') >= 0 ? color.substring(1, color.length) : color;
  amount = Math.trunc((255 * amount) / 100);
  return `#${addLight(color.substring(0, 2), amount)}${addLight(
    color.substring(2, 4),
    amount,
  )}${addLight(color.substring(4, 6), amount)}`;
}

/* Suma el porcentaje indicado a un color (RR, GG o BB) hexadecimal para aclararlo */
/**
 * Sums the passed percentage to the R, G or B of a HEX color
 * @param {string} color The color to change
 * @param {number} amount The amount to change the color by
 * @returns {string} The processed part of the color
 */
function addLight(color: string, amount: number) {
  const cc = parseInt(color, 16) + amount;
  const c = cc > 255 ? 255 : cc;
  return c.toString(16).length > 1 ? c.toString(16) : `0${c.toString(16)}`;
}

/**
 * Calculates luminance of an rgb color
 * @param {number} r red
 * @param {number} g green
 * @param {number} b blue
 */
function luminanace(r: number, g: number, b: number) {
  const a = [r, g, b].map((v) => {
    v /= 255;
    return v <= 0.03928 ? v / 12.92 : Math.pow((v + 0.055) / 1.055, 2.4);
  });
  return a[0] * 0.2126 + a[1] * 0.7152 + a[2] * 0.0722;
}

/**
 * Calculates contrast between two rgb colors
 * @param {string} rgb1 rgb color 1
 * @param {string} rgb2 rgb color 2
 */
function contrast(rgb1: string[], rgb2: number[]) {
  return (
    (luminanace(~~rgb1[0], ~~rgb1[1], ~~rgb1[2]) + 0.05) /
    (luminanace(rgb2[0], rgb2[1], rgb2[2]) + 0.05)
  );
}

/**
 * Determines what the best text color is (black or white) based con the contrast with the background
 * @param hexColor - Last selected color by the user
 */
export function calculateBestTextColor(hexColor: string) {
  const rgbColor = hexToRGB(hexColor.substring(1));
  const contrastWithBlack = contrast(rgbColor.split(','), [0, 0, 0]);

  return contrastWithBlack >= 12 ? '#000000' : '#FFFFFF';
}

/**
 * Subtracts the indicated percentage to the R, G or B of a HEX color
 * @param {string} color The color to change
 * @param {number} amount The amount to change the color by
 * @returns {string} The processed part of the color
 */
function subtractLight(color: string, amount: number) {
  const cc = parseInt(color, 16) - amount;
  const c = cc < 0 ? 0 : cc;
  return c.toString(16).length > 1 ? c.toString(16) : `0${c.toString(16)}`;
}

```
        - updateSidebarBgColor  修改侧边栏颜色 原理同上

以上大概有几个关键点：

1. 充分利用less的用法
    - 条件语句
    - 动态前缀变量名
2. 利用js来修改原生css变量的颜色，同时计算悬浮颜色
3. 项目中大部分样式类名以前缀方式定义，主less中定义了一个vben为namespace，在less中作为全局变量；而designSetting中定义了prefixCls在ts中作为全局变量。 他们存在这对应关系，因此需要同时修改才能起作用。

## 导航栏模式切换

> 导航栏模式分为了四种：

![](https://secure2.wostatic.cn/static/sPjnd9xQ7suquNPauh5Qqw/image.png?auth_key=1684196309-2o96KrtYmowc5t254R8edZ-0-bbe91f85e3201ce1060d94391f94427b)

1. 左边可折叠菜单，右边上部面包屑，下部内容
2. 上下布局，上部面包屑，下面左边菜单右边内容
3. 上下布局，上面菜单，下面内容
4. 左右布局，左边菜单点击展开子目录，右上方面包屑，下方内容



右边的样式选项都是通过自定义的Picker组件来实现的，导航栏模式选择的是TypePicker组件，传入的方法是baseHandler:

```Vue
 <TypePicker
    menuTypeList={menuTypeList}
    handler={(item: typeof menuTypeList[0]) => {
      baseHandler(HandlerEnum.CHANGE_LAYOUT, {
        mode: item.mode,
        type: item.type,
        split: unref(getIsHorizontal) ? false : undefined,
      });
    }}
    def={unref(getMenuType)}
  />
```

其中menuTypeList表示上方提到的四种模式，其定义如下：

```TypeScript
export const menuTypeList = [
  {
    title: t('layout.setting.menuTypeSidebar'),
    mode: MenuModeEnum.INLINE,
    type: MenuTypeEnum.SIDEBAR,
  },
  {
    title: t('layout.setting.menuTypeMix'),
    mode: MenuModeEnum.INLINE,
    type: MenuTypeEnum.MIX,
  },

  {
    title: t('layout.setting.menuTypeTopMenu'),
    mode: MenuModeEnum.HORIZONTAL,
    type: MenuTypeEnum.TOP_MENU,
  },
  {
    title: t('layout.setting.menuTypeMixSidebar'),
    mode: MenuModeEnum.INLINE,
    type: MenuTypeEnum.MIX_SIDEBAR,
  },
];
```

> ps: 由样式定义来看，less支持不同状态下，class后面拼接字符串的样式，比如&--active

调用handler函数：

1. 获取appStore配置信息
2. 根据传来的mode和type生成新的menuSetting
3. 将新的配置更新到pinia全局配置中
4. 更新来的配置几乎每一个属性都封装为一个computed

```TypeScript
export interface MenuSetting {
  bgColor: string;
  fixed: boolean;
  collapsed: boolean;
  siderHidden: boolean;
  canDrag: boolean;
  show: boolean;
  hidden: boolean;
  split: boolean;
  menuWidth: number;
  mode: MenuModeEnum;
  type: MenuTypeEnum;
  theme: ThemeEnum;
  topMenuAlign: 'start' | 'center' | 'end';
  trigger: TriggerEnum;
  accordion: boolean;
  closeMixSidebarOnChange: boolean;
  collapsedShowTitle: boolean;
  mixSideTrigger: MixSidebarTriggerEnum;
  mixSideFixed: boolean;
}

```

全都定义在useMenuSetting.ts中，这是一个**自定义hook**



## 系统主题切换

自定义组件ThemeColorPicker实现，包含三个prop

1. 颜色列表
2. 默认颜色 通过getThemeColor计算属性获取（真实来源自pinia中存储的themeColor）** 默认值都配置在src/projectSetting.ts下面**
3. event，表示事件ID



通过点击事件，调用baseHandle修改全局配置；

调用generateColors方法生成一组颜色，这组颜色的计算可以参考：

```TypeScript
export function generateColors({
  color = primaryColor,
  mixLighten,
  mixDarken,
  tinycolor,
}: GenerateColorsParams) {
  const arr = new Array(19).fill(0);
  const lightens = arr.map((_t, i) => {
    return mixLighten(color, i / 5);
  });

  const darkens = arr.map((_t, i) => {
    return mixDarken(color, i / 5);
  });

  const alphaColors = arr.map((_t, i) => {
    return tinycolor(color)
      .setAlpha(i / 20)
      .toRgbString();
  });

  const shortAlphaColors = alphaColors.map((item) => item.replace(/\s/g, '').replace(/0\./g, '.'));

  const tinycolorLightens = arr
    .map((_t, i) => {
      return tinycolor(color)
        .lighten(i * 5)
        .toHexString();
    })
    .filter((item) => item !== '#ffffff');

  const tinycolorDarkens = arr
    .map((_t, i) => {
      return tinycolor(color)
        .darken(i * 5)
        .toHexString();
    })
    .filter((item) => item !== '#000000');
  return [
    ...lightens,
    ...darkens,
    ...alphaColors,
    ...shortAlphaColors,
    ...tinycolorDarkens,
    ...tinycolorLightens,
  ].filter((item) => !item.includes('-'));
}
```

然后利用vite-plugin-theme方法替换样式变量

## 顶栏主题

- 自定义组件ThemeColorPicker
- 调用updateHeaderBgColor方法

    首先判断是否为夜间模式，夜间模式不生效；

    然后修改css变量--header-bg-color

    修改悬浮颜色： const hoverColor = lighten(color!, 6); 修改css变量

    修改headerSetting： 判断选择的颜色是否属于暗色，然后结合当前是否为暗色模式，判断设置是否生效



## 菜单主题

同顶栏主题



## 最后

> vben这个项目比较大，功能可以说是非常丰富，也可以说时非常冗杂，想要啃下来非常困难。 看到一个博客专门分析vben的可以参考：


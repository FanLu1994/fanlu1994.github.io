---
title: Fastbot_Android
date: 2023-10-24 08:50:37
tags:
  - 工具
  - 自动化测试
categories: 测试开发
---
字节的一个安卓monkey测试工具

参考 https://blog.csdn.net/u010698107/article/details/127347704 进行操作
参考： https://zhuanlan.zhihu.com/p/517792117

## 首先提取apk中的文本
这个可以改善模型的运作
```shell
aapt2 dump strings dfcf_0005564.apk > max.valid.strings
adb push max.valid.strings /sdcard 

```
### 测试一下豌豆荚app
```shell
adb -s 设备号 shell CLASSPATH=/sdcard/monkeyq.jar:/sdcard/framework.jar:/sdcard/fastbot-thirdpart.jar exec app_process /system/bin com.android.commands.monkey.Monkey -p com.wandoujia.phoenix2 --agent reuseq --running-minutes 2 --throttle 10 -v -v
```
覆盖率：
```shell
[Fastbot][2023-10-23 08:44:19.113] Explored app activities:
[Fastbot][2023-10-23 08:44:19.113]    1 com.pp.assistant.activity.AppDetailActivity
[Fastbot][2023-10-23 08:44:19.114]    2 com.pp.assistant.activity.BrowserActivity
[Fastbot][2023-10-23 08:44:19.114]    3 com.pp.assistant.activity.CommentReplyListActivity
[Fastbot][2023-10-23 08:44:19.114]    4 com.pp.assistant.activity.CommonWebActivity
[Fastbot][2023-10-23 08:44:19.114]    5 com.pp.assistant.activity.FullScreenImageActivity
[Fastbot][2023-10-23 08:44:19.114]    6 com.pp.assistant.activity.PPDefaultFragmentActivity
[Fastbot][2023-10-23 08:44:19.114]    7 com.pp.assistant.activity.PPMainActivity
[Fastbot][2023-10-23 08:44:19.114]    8 com.pp.assistant.activity.PPSearchActivity
[Fastbot][2023-10-23 08:44:19.115]    9 com.pp.assistant.activity.ReportWebActivity
[Fastbot][2023-10-23 08:44:19.115]   10 com.pp.assistant.activity.SearchResultActivity
[Fastbot][2023-10-23 08:44:19.115]   11 com.pp.assistant.activity.SettingActivity
[Fastbot][2023-10-23 08:44:19.115]   12 com.sina.weibo.sdk.web.WebActivity
[Fastbot][2023-10-23 08:44:19.115]   13 com.wandoujia.account.activities.PhoenixAccountActivity
[Fastbot][2023-10-23 08:44:19.115] Activity of Coverage: 11.403508%
```
## 测试一下无期迷途
```shell
adb -s 设备号 shell CLASSPATH=/sdcard/monkeyq.jar:/sdcard/framework.jar:/sdcard/fastbot-thirdpart.jar exec app_process /system/bin com.android.commands.monkey.Monkey -p com.zy.wqmt.cn --agent reuseq --running-minutes 2 --throttle 10 -v -v
```
测试结果：
```shell
[Fastbot][2023-10-23 08:49:52.642] Total app activities:
[Fastbot][2023-10-23 08:49:52.643]    1 com.papegames.pcsdktp.RegisterAccountActivity
[Fastbot][2023-10-23 08:49:52.643]    2 com.tencent.midas.wx.APMidasWXPayActivity
[Fastbot][2023-10-23 08:49:52.643]    3 com.tencent.connect.common.AssistActivity
[Fastbot][2023-10-23 08:49:52.643]    4 com.tencent.tauth.AuthActivity
[Fastbot][2023-10-23 08:49:52.644]    5 com.mobile.auth.gatewayauth.activity.AuthWebVeiwActivity
[Fastbot][2023-10-23 08:49:52.644]    6 com.xiaomi.mipush.sdk.NotificationClickedActivity
[Fastbot][2023-10-23 08:49:52.645]    7 com.papegames.gamelib.ui.webview.NativeWebviewActivity
[Fastbot][2023-10-23 08:49:52.645]    8 com.sina.weibo.sdk.web.WebActivity
[Fastbot][2023-10-23 08:49:52.645]    9 com.alipay.sdk.app.H5PayActivity
[Fastbot][2023-10-23 08:49:52.645]   10 com.papegames.gamelib.ui.webview.WebviewActivity
[Fastbot][2023-10-23 08:49:52.646]   11 com.papegames.log.widget.LogcatActivity
[Fastbot][2023-10-23 08:49:52.646]   12 com.tencent.midas.jsbridge.APWebJSBridgeActivity
[Fastbot][2023-10-23 08:49:52.646]   13 com.papegames.gamelib.utils.logcat.AppInfoActivity
[Fastbot][2023-10-23 08:49:52.646]   14 com.papegames.gamelib.utils.logcat.DeutolActivity
[Fastbot][2023-10-23 08:49:52.646]   15 com.papegames.gamelib.utils.logcat.LogcatViewerActivity
[Fastbot][2023-10-23 08:49:52.647]   16 com.alipay.sdk.app.H5OpenAuthActivity
[Fastbot][2023-10-23 08:49:52.647]   17 com.alipay.sdk.app.AlipayResultActivity
[Fastbot][2023-10-23 08:49:52.647]   18 com.papegames.gamelib_unity.BaseUnityImplActivity
[Fastbot][2023-10-23 08:49:52.647]   19 com.huawei.hms.activity.EnableServiceActivity
[Fastbot][2023-10-23 08:49:52.647]   20 com.cmic.sso.sdk.activity.LoginAuthActivity
[Fastbot][2023-10-23 08:49:52.648]   21 com.sina.weibo.sdk.share.ShareTransActivity
[Fastbot][2023-10-23 08:49:52.648]   22 com.tencent.midas.proxyactivity.APMidasPayProxyActivity
[Fastbot][2023-10-23 08:49:52.648]   23 com.tencent.connect.webview.JumpShareActivity
[Fastbot][2023-10-23 08:49:52.648]   24 com.tencent.midas.qq.APMidasQQWalletActivity
[Fastbot][2023-10-23 08:49:52.648]   25 com.zy.wqmt.cn.wxapi.WXPayEntryActivity
[Fastbot][2023-10-23 08:49:52.649]   26 com.huawei.hms.support.api.push.TransActivity
[Fastbot][2023-10-23 08:49:52.649]   27 com.tencent.connect.webview.BrowserActivity
[Fastbot][2023-10-23 08:49:52.649]   28 com.alipay.sdk.app.APayEntranceActivity
[Fastbot][2023-10-23 08:49:52.649]   29 com.onevcat.uniwebview.UniWebViewFileChooserActivity
[Fastbot][2023-10-23 08:49:52.649]   30 com.tencent.connect.avatar.ImageActivity
[Fastbot][2023-10-23 08:49:52.649]   31 com.aliyun.ams.emas.push.NotificationActivity
[Fastbot][2023-10-23 08:49:52.649]   32 com.huawei.hms.activity.BridgeActivity
[Fastbot][2023-10-23 08:49:52.649]   33 com.papegames.pcsdktp.ForgetPasswordActivity
[Fastbot][2023-10-23 08:49:52.650]   34 com.alipay.sdk.app.PayResultActivity
[Fastbot][2023-10-23 08:49:52.650]   35 com.alipay.sdk.app.H5AuthActivity
[Fastbot][2023-10-23 08:49:52.651]   36 com.mobile.auth.gatewayauth.LoginAuthActivity
[Fastbot][2023-10-23 08:49:52.651] Explored app activities:
[Fastbot][2023-10-23 08:49:52.651]    1 com.papegames.gamelib_unity.BaseUnityImplActivity
[Fastbot][2023-10-23 08:49:52.651] Activity of Coverage: 2.777778%
```

因为覆盖率是针对activity的，游戏中一般只有一个activity。可以看出对游戏的支持比较差。

## 支持自定义操作序列
例如新建 max.xpath.actions
```xml
```bash
# 谷歌电话示例
[
{
    "prob":1,
    "activity":"com.google.android.dialer.extensions.GoogleDialtactsActivity",
    "times":1,
    "actions":[
        {
            "xpath":"//*[@resource-id='com.google.android.dialer:id/dialpad_fab']",
            "action":"CLICK",
            "throttle":2000
        },
        {
            "xpath":"//*[@resource-id='com.google.android.dialer:id/search_edit_text']",
            "action":"CLICK",
            "text":"搜索联系人和地点",
            "clearTest":false,
            "throttle":2000
        }
    ]
}
]
```

其中
```bash
prob：发生概率，"prob"：1,代表发⽣概率为100%
activity：所属场景（当前页面所属的Activity）
times：重复次数，默认为1即可
actions：具体步骤的执行类型
throttle：action间隔事件（ms）
```

action类型包括
```objectivec
CLICK：点击，如果想要输入内容，则在action下补充text，如果有text属性则执行文本输入
LONG_CLICK：长按
BACK：返回
SCROLL_TOP_DOWN：从上向下滚动
SCROLL_BOTTOM_UP：从下向上滑动
SCROLL_LEFT_RIGHT：从左向右滑动
SCROLL_RIGHT_LEFT：从右向左滑动
```
然后将文件推送到/sdcard里面

## 控制要测试的activity
将要测试的activity写入awl.strings文件，并推送到/sdcard
运行时添加参数`--act-whitelist-file /sdcard/awl.strings`

同样可以写一个黑名单
`--act-blacklist-file /sdcard/abl.strings`

## 其他功能
- 屏蔽控件或者区域
- 树剪枝屏蔽
- 反混淆
- 高速截图并打印xml结构
- 权限自动授予

参考：https://www.jianshu.com/p/1d8da2f78bdd

## 原理
https://juejin.cn/post/7280435142244614202
https://github.com/bytedance/Fastbot_Android/blob/main/fastbot_code_analysis.md

# JSSDK Invalid Signature 错误的解决

## 问题描述

在前端使用 `History` 路由和后端签名机制完全正常的情况下， `wx.config()` 在 Android 表现正常，能正确显示 "ok" 字样，然而在 iOS 显示 `Invalid Signature` 。

## 问题原因

微信在两平台上对 `SPA` 的 `url` 处理机制是截然不同的。

+ iOS：每次切换路由，SPA 的 url 是不会变的，始终为最初进入页面的 url ，故发起签名请求的 url 参数必须为当初进入页面的 url
+ Android：每次切换路由，SPA 的 url 是会动态变化的，故发起签名请求的 url 参数必须是当前页面的 url

## 解决方法

全局存储进入 `SPA` 时的 `url` ，在发送签名请求时，根据平台种类发送对应的 `url` 。

```javascript
// 在进入 SPA 时，全局存储 Entry URL
// 由于我使用的是 React ，我在 index.html 的 scripts 处添加以下代码
if (!window.entryURL) {
    window.entryURL = window.location.href; // 使用 History 路由
    window.entryURL = window.location.href.split("#")[0]; // 使用 Hash 路由
}

// 发送签名请求时，做好平台适配
const url = isAndroid() ? window.location.href : window.entryURL;
```

## 其他易错点

+ 后端进行签名时，使用的`timestamp` ，应为精确到**秒**的时间戳
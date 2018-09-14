# 'cookie'与'session'中的'secret'区别

在使用 `Express` 开发应用时，我们会使用 `cookie-parser` 和 `express-session` 维护用户的状态信息。在使用这两种模块时，我们通常会设置 `secret` 值。那在 `cookie-parser` 设置的 `secret` 和在 `express-session` 设置的 `secret` 的区别在哪里，二者的作用分别是什么？

相关阅读：
+ `cookie-parser` : [expressjs/cookie-parser](https://github.com/expressjs/cookie-parser)
+ `express-session` : [expressjs/session](https://github.com/expressjs/session)

## cookie-parser

该中间件的 `secret` 用于服务端与前端（浏览器等）的 `cookies` 加密与解密。

## express-session

该中间件的 `secret` 用于 `sessionID` 的加密与解密，即用于对 `connect.sid` 的值加密与解密操作。

## 注意点

+ 若使用 `express-session` 的 `secret` 属性，则需先注册 `cookie-parser` 中间件。
+ `session` 是通过特殊的 `cookie` 来表示请求 ID ，其名称为 `connect.sid` ，其保存的值为（用 `express-session` 中的 `secret` 加密后的）会话ID，即 加密后的 `sessionID` 。

## 相关阅读

+ [cookie 和 session](http://wiki.jikexueyuan.com/project/node-lessons/cookie-session.html)
+ [简明理解签名cookie在node+express中的使用](https://www.jianshu.com/p/2615b364ee78)

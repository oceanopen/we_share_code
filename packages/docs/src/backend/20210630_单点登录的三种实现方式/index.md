# 单点登录的三种实现方式

## 1. 前言

### 1.1 术语介绍：域名分级

从专业的角度来说（根据《计算机网络》中的定义），`.com`、`.cn` 为一级域名（也称顶级域名），`.com.cn`、`baidu.com` 为二级域名，`sina.com.cn`、`tieba.baidu.com` 为三级域名。
以此类推，`N` 级域名就是 `N-1` 级域名的直接子域名。

从使用者的角度来说，一般把可支持独立备案的主域名称作一级域名，如 `baidu.com`、`sina.com.cn` 皆可称作一级域名，在主域名下建立的直接子域名称作二级域名，如 `tieba.baidu.com` 为二级域名。

为了避免歧义，本文将使用“主域名”替代“一级域名”的说法。

### 1.2 背景介绍

在 `B/S` 系统中，登录功能通常都是基于 `Cookie` 来实现的。

当用户登录成功后，一般会将登录状态记录到 `Session` 中，或者是给用户签发一个 `Token`, 无论哪一种方式，都需要在客户端保存一些信息（`Session ID` 或 `Token`），并要求客户端在之后的每次请求中携带它们。
在这样的场景下，使用 `Cookie` 无疑是最方便的，因此我们一般都会将 `Session` 的 `ID` 或 `Token` 保存到 `Cookie` 中，当服务端收到请求后，通过验证 `Cookie` 中的信息来判断用户是否登录。

`单点登录 (Single Sign On, SSO)` 是指在同一帐号平台下的多个应用系统中, 用户只需登录一次, 即可访问所有相Q互信任的应用系统。

举例来说，百度贴吧和百度地图是百度公司旗下的两个不同的应用系统，如果用户在百度贴吧登录过之后，当他访问百度地图时无需再次登录，那么就说明百度贴吧和百度地图之间实现了单点登录。

单点登录的本质就是在多个应用系统中共享登录状态。

如果用户的登录状态是记录在 `Session` 中的，要实现共享登录状态，就要先共享 `Session`。
比如可以将 `Session` 序列化到 `Redis` 中，让多个应用系统共享同一个 `Redis`, 直接读取 `Redis` 来获取 `Session`。

当然仅此是不够的，因为不同的应用系统有着不同的域名，尽管 `Session` 共享了，但是由于 `Session ID` 是往往保存在浏览器 `Cookie` 中的，因此存在作用域的限制，无法跨域名传递。
也就是说当用户在 `app1.com` 中登录后, `Session ID` 仅在浏览器访问 `app1.com` 时才会自动在请求头中携带，而当浏览器访问 `app2.com` 时, `Session ID` 是不会被带过去的。

实现单点登录的关键在于，如何让 `Session ID (或 Token)`在多个域中共享。

## 2. 实现方式一：父域 Cookie

在将具体实现之前，我们先来聊一聊 `Cookie` 的作用域。

- `Cookie` 的作用域由 `domain` 属性和 `path` 属性共同决定。
- `domain` 属性的有效值为当前域或其父域的 `域名 / IP 地址`。
- `path` 属性的有效值是以 `/` 开头的路径。

如果将 `Cookie` 的 `domain` 属性设置为当前域的父域，那么就认为它是父域 `Cookie`。
`Cookie` 有一个特点，即父域中的 `Cookie` 被子域所共享，换言之，子域会自动继承父域中的 `Cookie`。

利用 `Cookie` 的这个特点，不难想到，将 `Session ID (或 Token)` 保存到父域中不就行了。
没错，我们只需要将 `Cookie` 的 `domain` 属性设置为父域的域名（主域名），同时将 `Cookie` 的 `path` 属性设置为根路径，这样所有的子域应用就都可以访问到这个 `Cookie` 了。
不过这要求应用系统的域名需建立在一个共同的主域名之下，如 `tieba.baidu.com` 和 `map.baidu.com`, 它们都建立在 `baidu.com` 这个主域名之下，那么它们就可以通过这种方式来实现单点登录。

总结：此种实现方式比较简单，但不支持跨主域名。

## 3. 实现方式二：认证中心

我们可以部署一个认证中心，认证中心就是一个专门负责处理登录请求的独立的 `Web` 服务。

用户统一在认证中心进行登录，登录成功后，认证中心记录用户的登录状态，并将 `Token` 写入` Cookie`。（注意这个 `Cookie` 是认证中心的，应用系统是访问不到的。）

应用系统检查当前请求有没有 `Token`, 如果没有，说明用户在当前系统中尚未登录，那么就将页面跳转至认证中心。
由于这个操作会将认证中心的 `Cookie` 自动带过去，因此，认证中心能够根据 `Cookie` 知道用户是否已经登录过了。

如果认证中心发现用户尚未登录，则返回登录页面，等待用户登录。
如果发现用户已经登录过了，就不会让用户再次登录了，而是会跳转回目标 `URL`, 并在跳转前生成一个 `Token`, 拼接在目标 URL 的后面，回传给目标应用系统。

应用系统拿到 `Token` 之后，还需要向认证中心确认下 `Token` 的合法性，防止用户伪造。
确认无误后，应用系统将 `Token` 写入 `Cookie`, 然后给本次访问放行。（注意这个 `Cookie` 是当前应用系统的，其他应用系统是访问不到的。）
当用户再次访问当前应用系统时，就会自动带上这个 `Token`, 应用系统向认证中心验证 `Token` 发现用户已登录，于是就继续处理当前应用系统业务逻辑了。

总结：此种实现方式相对复杂，支持跨域，扩展性好，是单点登录的标准做法。

## 4. 实现方式三: LocalStorage 跨域

前面，我们说实现单点登录的关键在于，如何让 `Session ID (或 Token)`在多个域中共享。

父域 `Cookie` 确实是一种不错的解决方案，但是不支持跨域。
那么有没有什么奇淫技巧能够让 `Cookie` 跨域传递呢？

很遗憾，浏览器对 `Cookie` 的跨域限制越来越严格。
`Chrome` 浏览器还给 `Cookie` 新增了一个 `SameSite` 属性，此举几乎禁止了一切跨域请求的 `Cookie` 传递（超链接除外），并且只有当使用 `HTTPS` 协议时，才有可能被允许在 `AJAX` 跨域请求中接受服务器传来的 `Cookie`。

不过，在前后端分离的情况下，完全可以不使用 `Cookie`, 我们可以选择将 `Session ID (或 Token)` 保存到浏览器的 `LocalStorage` 中，让前端在每次向后端发送请求时，主动将 `LocalStorage` 的数据传递给服务端。
这些都是由前端来控制的，后端需要做的仅仅是在用户登录成功后，将 `Session ID (或 Token)`放在响应体中传递给前端。

在这样的场景下，单点登录完全可以在前端实现。
前端拿到 `Session ID (或 Token)` 后，除了将它写入自己的 `LocalStorage` 中之外，还可以通过特殊手段将它写入多个其他域下的 `LocalStorage` 中。

关键代码如下：

```js
// http://app2.com

// 获取 token
const token = result.data.token;

// 动态创建一个不可见的iframe，在iframe中加载一个跨域HTML
const iframe = document.createElement('iframe');
iframe.src = 'http://app1.com/localstorage.html';
document.body.append(iframe);

// 使用postMessage()方法将token传递给iframe
setTimeout(() => {
  iframe.contentWindow.postMessage(token, 'http://app1.com');
}, 4000);
setTimeout(() => {
  iframe.remove();
}, 6000);
```

```js
// http://app1.com

// 在这个iframe所加载的HTML中绑定一个事件监听器，当事件被触发时，把接收到的token数据写入localStorage
window.addEventListener(
  'message',
  (event) => {
    localStorage.setItem('token', event.data);
  },
  false
);
```

前端通过 `iframe + postMessage()` 方式，将同一份 `Token` 写入到了多个域下的 `LocalStorage` 中。
前端每次在向后端发送请求之前，都会主动从 `LocalStorage` 中读取 `Token` 并在请求中携带，这样就实现了同一份 `Token` 被多个域所共享。

总结：此种实现方式完全由前端控制，几乎不需要后端参与，同样支持跨域。

## 5. 参考

- [单点登录的三种实现方式](https://www.cnblogs.com/yonghengzh/p/13712729.html)

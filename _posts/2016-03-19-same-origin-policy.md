---
layout: post
title: 同源策略
---

## Info
对同源策略及其相关知识点的整理。

### 源
源的界定元素是协议、域名和端口号，如果两个页面的这三个元素相同，那么它们就是同源的。

### 同源策略
同源策略，Same-origin policy， SOP。该策略约束了一个源中所加载的文档和脚本对另一个源中的资源的访问。

### 约束
同源策略对资源间的访问进行了以下三类约束：

- 允许跨源写操作，包括链接、重定向和表单提交
- 允许跨源嵌入资源，包括如下几种嵌入方式：
	- `<script>`，可以实现JSONP
	- `<link rel="stylesheet">`
	- `<img>`
	- `<video>`、`<audio>`
	- `<object>`、`<embed>`、`<applet>`
	- `@font-face`
	- `<frame>`、`<iframe>`

- 不允许跨源读操作，但是可以通过嵌入资源来读取

### 跨源访问
实际应用中的业务需求有时要满足跨源通信，以下是几种跨源方式：

- [`document.domain`](https://developer.mozilla.org/en-US/docs/Web/API/Document/domain)：这实际是一种源的改变，脚本可以设置该值为当前域的一个后缀，之后该值即变为源的判断依据，利用这种方式可以实现父域与子域间资源的访问。需要注意的是其值不能为.com等顶级域。
- [`window.postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)： [JavaScript中窗口的句柄是可以跨源访问的](https://developer.mozilla.org/en/docs/Web/Security/Same-origin_policy#Window)，只要发送端拥有某个窗口的JavaScript句柄，就可以通过这个机制向该窗口发送任意长度的文本信息。这套机制由发送端决定哪个页面能接受到发送的信息，在发送的过程中，也会为接受端提供发送者的身份信息以供认证，从而确保了传输的正确性。
- [Cross-Origin Resource Sharing（CORS）](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)：即跨源资源共享，这种机制让Web服务器支持访问控制，其基本思想是使用自定义的HTTP头部让浏览器与服务器进行沟通，从而决定请求是否成功。需要注意的是，不管请求是否成功，请求都成功发送到了服务器上。
- [WebSocket](https://en.wikipedia.org/wiki/WebSocket)：WebSocket的目标是创建一个全双工的socket连接，客户端首先向服务端发送一个特殊的HTTP请求以发起连接，收到服务器响应之后，连接由HTTP升级为WebSocket从而开始通信。

### 参考文档

- [https://developer.mozilla.org/en/docs/Web/Security/Same-origin_policy](https://developer.mozilla.org/en/docs/Web/Security/Same-origin_policy)
- 《Web之困》
- 《Web前端黑客技术揭秘》
- 《JavaScript高级程序设计》

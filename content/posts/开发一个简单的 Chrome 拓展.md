---
title: "开发一个简单的 Chrome 拓展"
date: 2018-06-14 00:42:47
tags: [misc]
---

每次复制域名时都会被 Chrome 复制地址时的 `https://` 烦到, 所以干脆自己写个拓展来解决这个问题. Chrome 拓展其实就是一个小网页, 也就是 HTML, 所以我们可以用 JavaScript 来实现获取域名和复制的操作. 具体实现如下.

<!--more-->

```js
function callback(tab) {
    let url = tab.url;
    let parser = document.getElementById("parser");
    parser.href = url;
    let hostname = parser.hostname;
    let text = document.getElementById("text");
    text.setAttribute("value", hostname);
    text.select();
    document.execCommand("copy");
}

function getHostName(url) {
    chrome.tabs.query({
        active: true,
        windowId: chrome.windows.WINDOW_ID_CURRENT
    }, function (tabs) {
        if (tabs && tabs[0]) {
            callback(tabs[0]);
        }
    });
}

window.onload = getHostName
```

通过 `window.onload` 加载函数, 在 popup (地址栏右边的小图标点击后的页面) 弹出时自动将域名复制到粘贴版上. 关于获得当前页面 `url`, 因为 popup 自己就是个单独的页面, 与使用者当前浏览的隔离, 所以不能通过 `chrome.tabs.getCurrent` 来获得, 最后在 [segmentfault](https://segmentfault.com/q/1010000005570850/a-1020000005590533) 上找到了大神的答案, 通过 `query` 函数来得到当前页面的 url, 然后调用 callback 函数. 接下来就是从 url 中解析得到 hostname 也就是域名, 不过 JavaScript 并没有现成的函数, 我们也没必要因为此就直接加载个第三方库, 所以我们可以设置 DOM 的 href 属性后通过 hostname 属性间接得到域名 (不知道为什么 JavaScript 不直接搞个函数算了233), 然后通过 `document.execCommand` 来复制 select 中的域名即可. 接下来就是 `manifest.json`, 不过有完整的文档, 弄起来比较容易  

```json
{
      "name": "Get-hostname",
      "version": "1.0",
      "manifest_version": 2,
      "description": "Get hostname of current website.",
      "icons": {
           "16": "icon16.png",
           "48": "icon48.png",
           "128": "icon128.png" 
        },
      "browser_action": {
          "default_icon": "icon48.png",
          "default_popup": "main.html"
        },
      "content_security_policy": "policyString",
      "incognito": "spanning",
      "key": "publicKey",
      "offline_enabled": true,
      "permissions": ["tabs"]
    }
```

最后加上 `main.html` 和 `CSS` 来装饰下就完美了.  
PS: CSS 来自 [button](http://www.bootcss.com/p/buttons/)

```html
<input id="text" class="text">
<a id="parser" type="hidden"></a>

<script src="main.js"></script>
<link rel="stylesheet" href="main.css">
```
```css
.text {
    color: #666;
    background-color: #EEE;
    border-color: #EEE;
    font-weight: 300;
    font-size: 16px;
    font-family: "Helvetica Neue Light", "Helvetica Neue", Helvetica, Arial, "Lucida Grande", sans-serif;
    text-decoration: none;
    text-align: center;
    line-height: 40px;
    height: 40px;
    width: 240;
    margin: 0;
    display: inline-block;
    appearance: none;
    cursor: pointer;
    border: none;
    margin-left: auto; 
    margin-right: auto;
    -webkit-box-sizing: border-box;
       -moz-box-sizing: border-box;
            box-sizing: border-box;
    -webkit-transition-property: all;
            transition-property: all;
    -webkit-transition-duration: .3s;
            transition-duration: .3s;
}
```

最后效果如下  
![](https://i.loli.net/2019/03/08/5c82677b223c6.jpg#center)  
点开那个图标后当前域名就直接复制到剪贴板了, 终于可以摆脱该死的 `https://` 了.
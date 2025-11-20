---
title: "XcodeGhost in 2025"
date: 2025-11-12T22:09:55+08:00
tags: [misc]
---

## 背景
 
在上网冲浪时, 发现一篇文章提到了 XcodeGhost. 想起之前只是了解了大概, 就又去仔细搜索了下 XcodeGhost 事件的细节, 发现 `icloud-analysis.com` 这个域名目前居然是无主状态, 于是有了接下来的故事.

<!--more-->

## 域名注册

在各种分析文章中, 写的比较好的是[安天实验室的这篇](https://www.antiy.com/response/xcodeghost.html), 分析了整个事件的完整时间线和对攻击载荷的分析.

在这篇文章中, 可以发现除了 `icloud-analysis.com` 这个域名之外, 攻击者还用到了下面这两个域名, 分别投递到了不同版本的 Xcode 里面, 甚至还制作了部分 Unity 的恶意安装包.

* `crash-analytics.com`
* `icloud-diagnostics.com`

其中 `icloud-analysis.com`, `icloud-diagnostics.com` 在当时仍然处于未被注册状态, 于是直接在 cloudflare 上注册拿下, 倒是没有遇到什么特殊的告警 (.

顺便可以通过微步的 Whois 历史功能, 分析下这三个域名的历史

### icloud-analysis.com

* 2015-02-25 被攻击者注册
* 2015-10-04 被 shadowserver takeover, 解析到 sinkhole
* 2021-08-28 疑似域名未续费过期, 微步这个 whois 历史有点抽象, 可能更早就已经过期
* 2021-09-07 被 New Ventures Services, Corp 购买, 根据[搜索](https://www.namepros.com/threads/network-solutions-new-ventures-services-corp-stole-my-domain.1055044/)可以得知这是一个靠捡这种过期域名赚钱的公司
* 2022-09-03 域名第二次到期
* 2022-11-23 被未知所有者通过 key-systems.net 注册
* 2023-11-23 域名第三次到期
* 2024-03-05 被姓名 yunfei lin, 邮箱 linzixin0723@gmail.com 通过 value-domain.com 购买, 通过搜索邮箱可以搜到其名下还有很多域名, 疑似也是炒域名的
* 2025-03-05 域名第四次到期

## crash-analytics.com

* 2015-08-26 被攻击者注册, 迷惑的是注册的时候就没开隐私保护, 姓名 (andy wang wang), 邮箱 (778560441@qq.com), 手机号 (13276422520), 地址 (shiji building jinan,jinan,shandong,CHINA) 直接漏出来了
* 2016-08-26 域名到期
* 2016-11-13 被 virustracker.info sinkhole (virustracker 自己好像都倒闭了...?)
* 2023-11-13 域名第二次到期
* 2024-01-29 被未知所有者通过 realtimeregister 注册
* 2025-03-21 转移到 key-systems.net 注册局, 通过 NS 服务器可以知道被淘阿网所有, 经典炒域名的站

### icloud-diagnostics.com

* 2015-05-06 被攻击者注册
* 2015-09-21 最抽象的一集, 域名的隐私保护被关掉了. 可能攻击者手滑了? 直接把姓名 (Huaxian Tao) + 手机号 (13276422520) + QQ 邮箱 (778560441@qq.com)  + 地址 (No. 21 Haiyang Building,QD,China,QD,Shandong,China) 全暴露出来了, 与 crash-analytics.com 对的上
* 2016-05-07 域名到期
* 2016-07-24 同上, 被 virustracker.info sinkhole
* 2023-07-24 域名第二次到期

看下来比较难绷的点有
* 攻击者自己开盒, 邮箱和电话感觉比较真
* 安全机构自身难保, 并且在一定期限后都停止了 sinkhole
* 域名过期后大概率被炒域名机构买下, 然后需要高价赎回

## 搭建 C2 (伪) 服务器

拿下了域名之后, 最想要知道的莫过于在距离 2015 年已经 10 年的现在, 真的还有这么老的设备在运行这么老的应用程序么?

为此, 我们自然需要搭一个 C2 服务器来看看到底有没有请求. 实际上, 社区在当年就有人根据恶意代码写了[对应的服务器](https://github.com/TimonPeng/XcodeGhost_server). 但是我们并不真的需要进行什么恶意操作, 只是想收集 XCodeGhost 发送的遥测数据. 并且还想要尽量的节省预算, 毕竟如果用国内服务器的话, 流量估计是一笔不小的钱. 这自然让我想到了互联网活菩萨 Cloudflare 的 Serverless 产品 Workers, 可以直接免费使用, 完全可以写一个脚本记录所有 http 请求, 然后存在数据库里面即可 (这真的不是 Cloudflare 的广告).

稍微翻一下文档 + vibe coding 可以得到下面的代码
```js
import { Buffer } from 'node:buffer';

export default {
  async fetch(request, env) {
    try {
      // 获取请求信息
      const url = new URL(request.url);
      const method = request.method;
      const headers = JSON.stringify(Object.fromEntries(request.headers));
      const clientIP = request.headers.get('CF-Connecting-IP') || 'unknown';
      
      // 读取请求 body
      let body = '';
      if (method !== 'GET' && method !== 'HEAD') {
        const arrayBuffer = await request.arrayBuffer();
        const uint8Array = new Uint8Array(arrayBuffer);
        body = Buffer.from(uint8Array).toString('base64');
      }
      
      // 保存到 D1 数据库
      const stmt = env.DB.prepare(`
        INSERT INTO requests (method, url, headers, body, ip_address) 
        VALUES (?, ?, ?, ?, ?)
      `);
      
      await stmt.bind(method, url.toString(), headers, body, clientIP).run();
      
      // 返回成功响应
    } catch (error) {
      console.error('Error saving request:', error);
    }
    return Response.redirect("https://github.com/XcodeGhostSource/XcodeGhost", 302);
  }
}
```

需要注意, 为了用 `node:buffer` 这个 package, 需要在 workers 设置里的 `Compatibility flags` 打开 `nodejs_compat` 才行. 

不过剧透一下, 最后还是氪了一个月的 Paid 套餐. 没想到数据量这么大, 导致送的 D1 数据库额度不够了 = = 

## 数据分析

整个 Worker 从 8 月 21 日到 9 月 21 日, 在这两个域名下都挂了一个月, 排除爬虫请求, 收到了超乎想象的 1847830 条有效请求. 大部分请求来自 `icloud-analysis.com`, `icloud-diagnostics.com` 只共享了 10 条有效请求. 当然这么多请求不代表有这么多设备, XCodeGhost 的遥测数据在每次应用 Launch, Resign, Terminate 等请求时都会发送一次. 导致切到后台这个操作可能就会触发一次请求. 不过话又说回来, 由于 cloudflare 本身在中国大陆访问就是不稳定的, 可能会导致部分数据丢失.

XCodeGhost 发送到 C2 的请求是 DES 对称加密过的, 由于没有用非对称加密, 所以解密起来不是很难, 可以参考下面的脚本

```python
from Crypto.Cipher import DES
from Crypto.Util.Padding import pad, unpad
import re
import base64
import json

data = base64.b64decode('XXXX')

des = DES.new(key=b'stringWithFormat'[:8], mode=DES.MODE_ECB)
data = unpad(des.decrypt(data), block_size=8)[8:].decode()

print(data)
```

解密出来的样例结果如下

```json
{
  "country" : "CN",
  "os" : "10.3.4",
  "type" : "iPhone5,1",
  "app" : "网易云音乐",
  "name" : "iPhone",
  "idfv" : "0017D07D-F091-4ECB-8BEB-8B3DB6E13288",
  "timestamp" : "1756046349",
  "bundle" : "com.netease.cloudmusic",
  "status" : "resignActive",
  "version" : "2.7.2",
  "language" : "zh-Hans"
}
```

对几个字段分析了一下, 统计的数据如下

### country + language + 客户端 IP

有高达 82 个不同的 country, 但是通过对应记录的 language 可以发现实际上语言还是中文, 我没搞过 iOS 的开发, 但是这个 country 估计是来自自己设置的选项.

```
['AC', 'AD', 'AE', 'AF', 'AG', 'AI', 'AL', 'AM', 'AO', 'AQ', 'AR', 'AS', 'AT', 'AU', 'AX', 'BG', 'BR', 'BT', 'BW', 'BY', 'BZ', 'CA', 'CF', 'CH', 'CL', 'CN', 'CO', 'CX', 'DE', 'DG', 'DK', 'DZ', 'ES', 'ET', 'FR', 'GB', 'GM', 'GP', 'GY', 'HK', 'IE', 'IL', 'IN', 'IS', 'IT', 'JP', 'KR', 'KW', 'KZ', 'LA', 'LI', 'MK', 'MM', 'MN', 'MO', 'MX', 'MY', 'NG', 'NL', 'NO', 'NZ', 'OM', 'PH', 'PK', 'QA', 'RU', 'RW', 'SA', 'SG', 'TH', 'TJ', 'TM', 'TR', 'TW', 'TZ', 'UA', 'UG', 'UM', 'US', 'VI', 'VN', 'ZA']
```

相比之下, 语言以及客户端 IP 的可信程度更高, 其中语言共有 113 种, 包括但不限于 🇷🇺, 🇸🇦, 🇬🇧 等

```
['ar', 'ar-AE', 'ar-KW', 'ar-SA', 'da', 'de', 'de-AT', 'de-DE', 'en', 'en-AU', 'en-CA', 'en-CN', 'en-ET', 'en-GB', 'en-HK', 'en-MM', 'en-NG', 'en-PH', 'en-SA', 'en-TH', 'en-TW', 'en-US', 'es', 'es-CL', 'es-CO', 'es-ES', 'es-MX', 'es-US', 'fr', 'fr-CA', 'fr-FR', 'he', 'is-CN', 'it', 'it-IT', 'ja', 'ja-CN', 'ja-JP', 'ja-US', 'ko', 'ko-CN', 'ms', 'nb', 'nb-NO', 'pt', 'pt-PT', 'ru', 'ru-AM', 'ru-CN', 'ru-RU', 'ru-TJ', 'ru-TM', 'ru-UA', 'ru-US', 'th', 'th-CN', 'th-TH', 'ug-CN', 'ug-US', 'vi', 'vi-VN', 'zh-Hans', 'zh-Hans-AC', 'zh-Hans-AD', 'zh-Hans-AF', 'zh-Hans-AG', 'zh-Hans-AI', 'zh-Hans-AL', 'zh-Hans-AO', 'zh-Hans-AQ', 'zh-Hans-AR', 'zh-Hans-AU', 'zh-Hans-AX', 'zh-Hans-BR', 'zh-Hans-CA', 'zh-Hans-CF', 'zh-Hans-CN', 'zh-Hans-CX', 'zh-Hans-DE', 'zh-Hans-DZ', 'zh-Hans-ES', 'zh-Hans-GB', 'zh-Hans-GM', 'zh-Hans-GP', 'zh-Hans-GY', 'zh-Hans-HK', 'zh-Hans-IS', 'zh-Hans-IT', 'zh-Hans-JP', 'zh-Hans-KR', 'zh-Hans-MK', 'zh-Hans-MM', 'zh-Hans-MO', 'zh-Hans-MY', 'zh-Hans-NZ', 'zh-Hans-OM', 'zh-Hans-QA', 'zh-Hans-SG', 'zh-Hans-TM', 'zh-Hans-TR', 'zh-Hans-TW', 'zh-Hans-US', 'zh-Hans-VN', 'zh-Hans-ZA', 'zh-Hant', 'zh-Hant-CN', 'zh-Hant-GB', 'zh-Hant-HK', 'zh-Hant-MO', 'zh-Hant-TW', 'zh-Hant-US', 'zh-HK', 'zh-TW']
```

共有 127229 个独立的 IP, 通过 IP 可以确定确实是有外国人中招的. 通过对应的 bundle 字段, 使用的也是 com.zhiliaoapp.musically, com.mobivio.cutecutpro, com.ilegendsoft.MBrowser 这种国外应用.

### 应用

这个可能是最准确的中招列表了 (

```
['cairot', 'cn.12306.rails12306', 'cn.com.10jqka.IHexin', 'cn.com.10jqka.iHexinFee', 'cn.com.10jqka.ipad', 'cn.com.cdzq.ths', 'cn.com.doudizhu.happy', 'cn.com.qdqdjoys.jl', 'cn.net.hzcaifeng.yb', 'com.17zuoye.homework', 'com.aemobile.games.sudoku', 'com.apple.AppStore', 'com.apple.MobileAddressBook', 'com.apple.Preferences', 'com.apple.springboard', 'com.arcsoft.Perfect365', 'com.arkids.poketzoo', 'com.autonavi.amap', 'com.baidu.BaiduMobile', 'com.baidu.TingIPhone', 'com.barhatgames.lol.erolabs', 'com.brandy.scys', 'com.brightlabs.quicksave', 'com.caixin.magazineapp', 'com.candy.yy', 'com.cherrygame.majiang', 'com.cn.greenbrowser', 'com.csair.MBP', 'com.dandan.DongHuaDuoDuo', 'com.dayup11.LaiDianGuiShuDiFree', 'com.degoo.icemommy', 'com.diary.mood', 'com.DragonGame.fruitworlds', 'com.dtcsolutions.dtc', 'com.dw.fs.PP', 'com.flyhuang.kangkang', 'com.gemd.iting', 'com.haomee.video', 'com.hiddensweet.bubbleshooterhdfree', 'com.hmzl.hmxx', 'com.htzq.ths.iphone', 'com.iflytek.iflyinput', 'com.ilegendsoft.MBrowser', 'com.ilegendsoft.MBrowserPro', 'com.intsig.camcard.lite', 'com.intsig.CamScannerLite', 'com.jailbreak.kk', 'com.jingling.bssj', 'com.kdanmobile.ipad.PDFReader', 'com.kdanmobile.ipad.pdfreaderlite', 'com.kdanmobile.PDFReaderLite', 'com.KKGOO.kk', 'com.konka.MultiscreeninteractionV3.0', 'com.luobocn.carrotfantasy2', 'com.luomao.GamePlayer', 'com.lyw.dmbj5', 'com.mobileventures.LittleFarmers', 'com.mobivio.cutecut', 'com.mobivio.cutecutpro', 'com.my.laugheveryday', 'com.mystylinglounge.eyelashes-makeup', 'com.mystylinglounge.promhairsalon', 'com.netease.cloudmusic', 'com.netease.videoHD', 'com.ng.simulator', 'com.octInn.br', 'com.olimsoft.oplayer', 'com.olimsoft.oplayer.hd', 'com.olimsoft.oplayer.hd.lite', 'com.olimsoft.oplayer.lite', 'com.pingan.PAMobileStockHigh', 'com.rovio.scn.baba', 'com.rovio.tcn.baba', 'com.s5game.fishsaga', 'com.shoujiduoduo.ringtone', 'com.shoujiduouod.ergeduoduoipad', 'com.ss.iphone.ugc.Aweme', 'com.starsea.wblock', 'com.stockradar.radar1', 'com.taofang.iphone', 'com.tapinfinity.PhotoshopHuDongJiaoCheng', 'com.teiron.pphelperns', 'com.telecom-guoling.feiin', 'com.tencent.mt2', 'com.tencent.qjnn', 'com.tencent.xin', 'com.tiger515.lobby', 'com.tinmanarts.JoJoTiezhi', 'com.tinmanarts.zuokezhenmafan', 'com.tongbu.wanji', 'com.toprankapps.secretphotofree', 'com.ttpod.music', 'com.tuyoo.chinesechess', 'com.ucweb.iphone', 'com.umonistudio.tapTile', 'com.uniview.app.smb.phone.ezview', 'com.wangfan.myChevy', 'com.weiying.Wvod', 'com.winzip.ios', 'com.WXTool.WXTool', 'com.wxxm.erpaoshou', 'com.xbzq.iphone.20101224', 'com.xiachufang.recipe', 'com.xianqing.ExcavatorCN', 'com.xianqing.parking3dcn', 'com.xianqing.Parking3DHD', 'com.xianqing.parking3dhdcn', 'com.xiaojukeji.didi', 'com.xiaoxiaov', 'com.xiaoyun.solitaire1124', 'com.xiaoyun.spiderpro', 'com.yeahworld.defendempire', 'com.YgCitylgMac.iPhone', 'com.yoho.buy', 'com.zhiliaoapp.musically', 'com.zhiliaoapp.musically.SQV9Z2C579', 'com.zhiliaoapp.musically.VQR5NX8778', 'com.zyzq.ths.iphone', 'linfengkun.NaviOne', 'net.beautyincorporated.fitnesssalon', 'net.tbet.qx', 'P02.tencent.xin', 'ph.spinnr.Spinnr', 'tongpengfei.piper', 'WiMax.WiMax-4G']
```

对应的请求次数如下, 可以看到大部分都是来自网易云音乐, 也比较合理, 毕竟这么老的设备, 估计就是拿来当离线播放器来用了. 其他的联网应用真的还有能用的么...

```
1651352	com.netease.cloudmusic
110874	cn.com.10jqka.ipad
25326	com.apple.springboard
16339	cn.com.10jqka.IHexin
10035	com.luobocn.carrotfantasy2
9751	cn.com.doudizhu.happy
2743	cairot
2491	com.tencent.xin
2127	com.ilegendsoft.MBrowser
1498	com.htzq.ths.iphone
1424	com.gemd.iting
955	com.tuyoo.chinesechess
871	com.rovio.scn.baba
696	com.ss.iphone.ugc.Aweme
625	com.cn.greenbrowser
487	com.cherrygame.majiang
472	com.ilegendsoft.MBrowserPro
387	com.autonavi.amap
374	com.winzip.ios
270	com.s5game.fishsaga
266	com.hiddensweet.bubbleshooterhdfree
261	com.kdanmobile.ipad.pdfreaderlite
253	com.xiaoyun.solitaire1124
180	com.starsea.wblock
171	com.ttpod.music
146	com.olimsoft.oplayer
92	com.zhiliaoapp.musically
77	com.baidu.TingIPhone
73	ph.spinnr.Spinnr
64	com.toprankapps.secretphotofree
62	com.arcsoft.Perfect365
62	com.iflytek.iflyinput
62	com.xianqing.Parking3DHD
57	com.olimsoft.oplayer.hd.lite
56	cn.net.hzcaifeng.yb
56	net.tbet.qx
55	com.aemobile.games.sudoku
55	com.dandan.DongHuaDuoDuo
54	com.umonistudio.tapTile
53	com.kdanmobile.ipad.PDFReader
52	com.dtcsolutions.dtc
49	cn.12306.rails12306
44	com.xiaoyun.spiderpro
42	com.xiachufang.recipe
39	com.netease.videoHD
... 剩下的请求次数比较少, 不写了
```

比较抽象的是还有 com.apple.springboard, 这个应该是 app 启动器? 不知道是怎么混进去的, 估计是来自的越狱的用户装了魔改版 springboard, 而恰巧这个做越狱应用的人用了被感染的 XCode, 非常神奇了.

### OS

大部分来自经典的 6.1.3, 

```
695007	6.1.3
362126	9.3.6
226928	7.1.2
142910	10.3.4
83410	6.1.6
77440	12.5.7
35197	8.4
34599	8.0
32042	8.4.1
18677	9.3.5
11467	8.2
11281	6.1.4
9226	8.3
7980	10.3.3
7511	12.5.5
5427	7.1.1
5411	7.0.4
5410	7.0.6
4839	8.0.2
4519	8.1.3
4308	8.1.2
4270	6.1
3837	6.0.1
...
```

比较抽象的是在后面还有个请求了 10 次的 26.0, 手机型号是 iPhone 16, IP 归属是加拿大, 大受震撼.jpg, 不知道这是越狱之后改的系统属性, 还是是有人没事用脚本往这个地址发假请求?

### 机型

大部分都是经典 iPhone 4 系列, 真是老当益壮, 距离 14 年停产已经 11 年了

```
1027889	iPhone4,1
153069	iPhone3,1
122860	iPhone5,2
86142	iPod4,1
50750	iPhone3,2
49124	iPhone7,2
39761	iPhone3,3
35278	iPhone5,3
34477	iPad2,4
27171	iPhone7,1
27103	iPad4,7
26750	iPhone6,2
23244	iPad2,1
20315	iPad3,4
16310	iPad2,5
15320	iPad4,1
15097	iPad4,4
13670	iPad3,1
10569	iPhone5,1
9581	iPhone5,4
7362	iPad2,2
7322	iPhone6,1
6297	iPad3,3
3517	iPad5,3
3134	iPod5,1
2238	iPad3,6
1679	iPad2,7
1276	iPhone2,1
700	iPod7,1
504	iPhone8,2
486	iPhone8,1
423	iPad4,3
417	iPad5,4
409	iPad4,9
363	iPhone9,1
220	iPhone8,4
188	iPad14,1
171	iPad4,5
167	iPad4,2
152	iPhone12,8
145	iPad3,5
116	iPhone9,2
92	iPhone13,1
44	iPad3,2
...
```

### 主机名 + 请求时间

以 8 月 29 日一天按小时的请求数量, 可以看出来还是符合中国人的作息的, 可以印证数据的真实性. 如果真的有人闲得没事用脚本往 C2 地址发假请求, 也没有达到可以影响统计结果的程度.

```
2025-08-29 00:00:00.000	3802
2025-08-29 01:00:00.000	2374
2025-08-29 02:00:00.000	1210
2025-08-29 03:00:00.000	848
2025-08-29 04:00:00.000	504
2025-08-29 05:00:00.000	554
2025-08-29 06:00:00.000	730
2025-08-29 07:00:00.000	1121
2025-08-29 08:00:00.000	1846
2025-08-29 09:00:00.000	3585
2025-08-29 10:00:00.000	4322
2025-08-29 11:00:00.000	3632
2025-08-29 12:00:00.000	3428
2025-08-29 13:00:00.000	4007
2025-08-29 14:00:00.000	4809
2025-08-29 15:00:00.000	3284
2025-08-29 16:00:00.000	3492
2025-08-29 17:00:00.000	3506
2025-08-29 18:00:00.000	3047
2025-08-29 19:00:00.000	3633
2025-08-29 20:00:00.000	3959
2025-08-29 21:00:00.000	4617
2025-08-29 22:00:00.000	5350
2025-08-29 23:00:00.000	5233
2025-08-30 00:00:00.000	3796
```

另外, 主机名也是很有活人感, 除了默认的 `iPhone`, 比如有类似 `XXX我们复合吧T^T`, `XXX是sb`, `XXX的萌萌4s`, `苹果16promax` 这类抽象名称, 还是挺真实的. 基本可以判断这些数据大部分不是机器人刷的.

### 存活客户端数据级

由于 idfv 这个字段不同供应商开发的 app 获取的值不一样, 因此只能统计大致数量级, 这里以网易云音乐为例, 独特的 idfv 数量为 29818, 也就是说目前全世界存活的仍然可以受到 XCodeGhost 影响的终端仍然有大约 3w 之多.

# 总结

难以想象, 在 XCodeGhost 风波 10 年过去之后, 仍然有万级别的受害者仍然受到该漏洞影响, 好在这些用户估计只是把这些老设备当作音乐播放器, 就算真的有人劫持 DNS 尝试再利用这个后门, 也再无法掀起更大的波浪. 更有意思的我觉得还是这些废弃 C2 域名的善后措施, 这些购买域名的安全机构有些自身难保, 有些则在没有预告的情况下默默离开. 如果真的还有受影响的受害者, 此时又会暴露在危险之下, 不知道未来是否有更好的善后措施, 比如永久暂停某个域名的注册? 

另外, 这两个域名我未来将会永久 302 到 `https://github.com/XcodeGhostSource/XcodeGhost`, 就让这件事漫漫消逝在历史长河中吧.

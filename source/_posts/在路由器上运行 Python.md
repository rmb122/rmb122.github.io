---
title: "在路由器上运行 Python"
date: 2018-01-07 21:42:56
tags: [misc, linux]
---

首先你需要一台运行LEDE的路由器并且拥有足量的闪存空间（大概10M), SSH连接后输入

```sh
opkg install python3 # python 如果你想装2, 7版本的py
opkg install python3-pip # 可以不装
```

<!-- more -->

然后等待几十秒, BOOM~  搞定. 输入 python3 就可以进入交互界面, enjoy it, 相当于树莓派了. 
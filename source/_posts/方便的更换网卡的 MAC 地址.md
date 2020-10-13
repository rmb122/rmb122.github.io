---
title: "方便的更换网卡的 MAC 地址"
date: 2018-05-18 20:27:49
tags: [linux, misc]
---

因为校园网的关系, 电信会定期封禁长时间在线的 MAC 地址, 这时候只要更换 MAC 地址就行了. 之前一直用的重启就可以随机生成 MAC 地址, 但是我现在用的这个版本的 LEDE 不知道为什么不会自动随机生成 MAC 地址了. 上网也没有找到有用的信息, 自己稍微试了下 ip 指令, 十分强大,

<!-- more -->

```sh
ip link set eth0.2 address xx:xx:xx:xx:xx:xx
```

这样就可以设置任意 MAC 地址了, 十分方便. 也可以写个 python 小脚本一键生成随机 MAC 地址.

```python
from random import randint
from subprocess import run

newAddr = "10:"
for i in range(5):
    i, j = randint(1, 15), randint(1, 15)
    i, j = hex(i)[2:], hex(j)[2:]
    newAddr += (i + j + ":")

newAddr = newAddr[0:-1]
run(["ip", "link", "set", "eth0.2", "address", newAddr])
```
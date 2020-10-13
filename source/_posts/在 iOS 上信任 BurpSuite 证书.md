---
title: "在 iOS 上信任 BurpSuite 证书"
date: 2019-03-19 21:26:56
tags: [misc]
---

今天突然发现在 http://burp 上信任证书后竟然还提示不安全...  
原因是 iOS 更新后又添加一道防护, 得手动在  
通用 -> 关于本机 -> 证书信任设置(最底下) 里面打开对 PortSwiggerCA 的信任.
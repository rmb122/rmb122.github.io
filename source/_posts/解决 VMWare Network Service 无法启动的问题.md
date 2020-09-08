---
title: "解决 VMWare Network Service 无法启动的问题"
date: 2018-09-21 17:21:19
tags: [linux]
---

今天需要在 Linux 下启动虚拟机, 但是不知为何 `vmware-networks.service` 尝试好几次都无法启动, 干脆就跟着提示的 `journalctl -xe` 看看到底出了啥错, 说不定能修复呢 233 日志结果如下

<!-- more -->

```
-- vmware-networks.service 单元已开始启动.
9月 21 16:50:25 rmb122-pc modprobe[16455]: modprobe: FATAL: Module vmnet not found in directory /lib/modules/4.14.69-1-MANJARO
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: Bridge process created.
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: RTM_NEWLINK: name:enp3s0 index:2 flags:0x00011043
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: Adding interface enp3s0 index:2
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: Can't open vmnet device /dev/vmnet0 (No such device or address).
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: RTM_NEWLINK: name:wlp2s0 index:3 flags:0x00001002
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: Can't open vmnet device /dev/vmnet0 (No such device or address).
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: RTM_NEWROUTE: index:2
9月 21 16:50:25 rmb122-pc vmnetBridge[16474]: Can't open vmnet device /dev/vmnet0 (No such device or address).
9月 21 16:50:26 rmb122-pc vmware-networks[16456]: Failed to enable hostonly virtual adapter on vmnet1
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: Internet Software Consortium DHCP Server 2.0
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: Copyright 1995, 1996, 1997, 1998, 1999 The Internet Software Consortium.
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: All rights reserved.
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: 
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: Please contribute if you find this software useful.
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: For info, please visit http://www.isc.org/dhcp-contrib.html
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: 
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: Configured subnet: 192.168.252.0
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: Setting vmnet-dhcp IP address: 192.168.252.254
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: Cannot open /dev/vmnet1: No such device or address
9月 21 16:50:26 rmb122-pc vmnet-dhcpd[16492]: exiting.
9月 21 16:50:26 rmb122-pc vmware-networks[16456]: Failed to start DHCP service on vmnet1
```

去 `/dev` 下看了看, vmnet 0-3 都是有的, 感觉问题出在 `9月 21 16:50:25 rmb122-pc modprobe[16455]: modprobe: FATAL: Module vmnet not found in directory /lib/modules/4.14.69-1-MANJARO` 上, 毕竟前几天刚更新, 可能是版本号的原因.  

到 `/lib/modules` 下一看, 还真是如此, 里面的是 `4.14.70-1-MANJARO`, `sudo ln -s 4.14.70-1-MANJARO 4.14.69-1-MANJARO` 建一个软链接, 再尝试启动服务, 搞定, 看来还真不是软件越新越好啊 233 不过至少能找到错误原因, 如果是 windows 底下出现这种莫名其妙的问题可能就凉凉了 (逃

P.S. 真实原因其实是内核更新了我没重启. 硬盘上当然没有老版本的内核模块, 而且新版本内核模块在 ABI 变动后也加载不进内存中的老版本内核 233, 重启一下就好了
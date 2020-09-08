---
title: "安恒杯三月赛 Writeup"
date: 2018-03-25 21:53:15
tags: [writeup, audit]
---

### 0x01 流量分析(3)

这几题流量分析感觉顺便在考察机器配置啊 233, 渣电脑估计瞬间爆炸. 相当简单, 我这里打开的是 `数据采集 D_eth0_NS_20160809_164452.pcap`, 过滤http协议, 可以看到大量来自 183.129.152.140  的流量, 可以看到是在爆破账号密码, flag{183.129.152.140}. 

<!-- more -->

### 0x02 流量分析(2)

在 `数据采集 H_eth0_NS_20160809_170103.pcap`, 中可以看到.  
`POST /webmail/client/index.php?module=operate&action=login HTTP/1.1`  
猜测action还可以有send之类的, 搜索正则表达式 `action=.*send.*`, 最终在 `数据采集 D_eth0_NS_20160809_170427.pcap` 中可以找到 `action=mail-send-success`.  
接下来追踪http流, 可以发现一个 `welcome to UMAIL`.  将其保存为html文件, 打开可以发现几个与众不同的邮箱.  
尝试可以得到 `flag{xsser@live.cn}`.

### 0x03 流量分析(1)

既然是后门, 自然要有 phpinfo() 233.搜索字符串 phpinfo, 可以找到很多, 但大多是 `404 not found`, 这里可以用 `(ip.src==183.129.152.140 ||ip.dst==183.129.152.140) && (http.response.code!=404 || http.request)` 过滤一下, 然后搜索 phpinfo, 可以在 `数据采集 H_eth0_NS_20160809_172819.pcap` 里面找到后门.  
`flag{admin.bak.php}`.

### 0x03 石头剪刀布

这题就是靠爆破啦, 但是题目有些问题, 主办方也不打算修了, 组里大佬问主办方才拿到flag, 233.  

```python
import socket

sck=socket.socket()
sck.connect(("192.168.5.91",80))
log=open('shiTouLog.txt','a')
key=open('shiTouKey.txt','a')

data=sck.recv(2048)

for i in range(101):    
    sck.send('0\n'.encode())
    data=sck.recv(2048)
    data:str=data.decode()[0:-4]
    if data.find("石头")!=-1:
        key.write('2 ')
    if data.find("剪刀")!=-1:
        key.write('0 ')
    if data.find("布")!=-1:
        key.write('1 ')
    print(data)
    log.write(data)

sck.close()

import socket

class sussces(Exception):
    pass

key=open('shiTouKey.txt','r').read()
keys=key.split()

while True:
    try:
        sck=socket.socket()
        sck.connect(("192.168.5.91",80))
        data=sck.recv(2048) #banner

        for i in range(len(keys)):    
            sck.send((str(keys[i])+'\n').encode())
            data=sck.recv(2048).decode()
            print(data)
            if data.find('LOSE')!=-1:
                break
            if data.find('100 /100')!=-1:
                raise sussces 
    except sussces:
        break

print(data)
sck.close()
```

### 0x04 xor

又是一题简单的逆向, ida F5, 果然是xor加密, 直接放脚本吧.

```c
int main() {
	char a[] = { '\x66','\x0A','\x6B','\x0C','\x77','\x26','\x4F','\x2E','\x40','\x11','\x78','\x0D','\x5A','\x3B','\x55','\x11','\x70','\x19','\x46','\x1f', '\x76', '\x22', '\x4d', '\x23', '\x44','\x0e', '\x67', '\x6', '\x68', '\x0f', '\x47', '\x32', '\x4f','\x0'};
	char last = a[0];
	for (int i = 1; i < 33; i++) {
		char temp = a[i];
		a[i] = last ^ a[i];
		last = temp;
	}
	return 0;
}
```

`flag{QianQiuWanDai_YiTongJiangHu}`  
剩下的题不是没时间做就是不会卡在一半了 233, 像什么半个二维码和谜之蜘蛛侠. 等大佬们的 Writeup 吧~
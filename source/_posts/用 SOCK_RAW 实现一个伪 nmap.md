---
title: "用 SOCK_RAW 实现一个伪 nmap"
date: 2018-05-30 14:42:40
tags: [python, misc]
---

这几天详细研究了下 nmap 的实现原理, 对 TCP 协议还是蛮好奇的. 刚好 python 的 socket 库十分强大, 支持 SOCK_RAW, 意味着我们完全可以自己实现一个 SYN 端口探测器. 首先自然要了解一下 TCP 协议的结构, 可以在 wikipedia 上查到,  

<!-- more -->

![](https://i.loli.net/2019/03/08/5c8274b5c63d5.png)  

比较麻烦的是效验和的计算, 还需要源地址和目标地址才能算出来, 否则会被路由直接丢掉

```python
def makeByteArray(num, byteSize):
    '''
    将数字转化为 bytearray.
    '''
    result = bytearray()
    num = bin(num)[2:].zfill(byteSize * 8)  #补零
    for i in range(0, len(num), 8):
        result.append(int(num[i:i + 8], 2))  #按字节分割
    return result


def calcCheckSum(srcIp, targetIp, tcpHead):
    ''''
    计算效验和.
    '''
    psdHeader = bytearray()
    psdHeader.extend(srcIp)  #来源 Ip
    psdHeader.extend(targetIp)  #目标 Ip
    psdHeader.extend(makeByteArray(0, 1))  #补零
    psdHeader.extend(makeByteArray(socket.IPPROTO_TCP, 1))  #TCP 协议号
    psdHeader.extend(makeByteArray(len(tcpHead), 2))  #TCP 长度
    psdHeader.extend(tcpHead)

    checkSum = 0
    for i in range(0, len(psdHeader), 2):
        checkSum += ((psdHeader[i] << 8) + psdHeader[i + 1])
        checkSum = (checkSum >> 16) + (checkSum & 0xffff)

    checkSum = ~checkSum & 0xffff
    return checkSum


def makeTCPSyn(srcIp, targetIp, srcPort, targetPort):
    tcpPayload = bytearray()
    tcpPayload.extend(makeByteArray(srcPort, 2))  #来源端口
    tcpPayload.extend(makeByteArray(targetPort, 2))  #目标端口
    tcpPayload.extend(makeByteArray(randint(0x10000000, 0xAAAAAAAA), 4))  #SYN 序列号, 随机生成一个
    tcpPayload.extend(makeByteArray(0, 4))  #ACK 序列号
    tcpPayload.extend(makeByteArray(0b0110000000000010, 2))  #数据开始的偏移量、保留字段以及标识符 我们只需要填充 SYN 的标准位为 1 即可
    tcpPayload.extend(makeByteArray(randint(300, 1460), 2))  #窗口大小, 大小随意
    tcpPayload.extend(makeByteArray(0, 2))  #check sum, 先用 0 填充
    tcpPayload.extend(makeByteArray(0, 2))  #紧急指针
    tcpPayload.extend(makeByteArray(2, 1))  #这个以及下面的为 tcp options
    tcpPayload.extend(makeByteArray(4, 1))
    tcpPayload.extend(makeByteArray(1460, 2))

    checkSum = makeByteArray(calcCheckSum(srcIp, targetIp, tcpPayload), 2)
    tcpPayload[16] = checkSum[0]  #将 checksum 替换为计算好的值
    tcpPayload[17] = checkSum[1]
    return tcpPayload
```

最后写出来是这样的, 其实并不困难, 主要是比较复杂以及资料相对比较难找. 接下来我们直接通过 SOCK\_RAW 发出去就可以了, 注意开启 SOCK\_RAW 是需要 root 权限的

```python
def transIp2Bytes(addr):
    '''
    将 Ip 转换为 bytearray.
    '''
    addr = addr.split(".")
    result = bytearray()
    for num in addr:
        result.extend(makeByteArray(int(num), 1))
    return result


def analyzeRes(response):
    port = (response[20] << 8) + response[21]

    if (response[33] >> 2) & 1:  #RST 位
        return (port, PORT_CLOSE)
    elif (response[33] >> 4) & 1:  #SYN 位
        return (port, PORT_OPEN)
    raise Exception("Invaild head.")


def isPortOpen(localAddr, remoteAddr):
    '''
    检测端口是否开放, 端口被过滤返回 `PORT_FILTERED`, 端口关闭返回 `PORT_CLOSE`, 开启返回 `PORT_OPEN`  
    localAddr, remoteAddr 均为 tuple.
    '''
    sck = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_TCP)
    sck.setblocking(False)  #设置为非阻塞式

    payload = makeTCPSyn(transIp2Bytes(localAddr[0]), transIp2Bytes(remoteAddr[0]), localAddr[1], remoteAddr[1])
    sck.sendto(payload, remoteAddr)
    sleep(0.1)

    response = bytearray()
    try:
        response = sck.recv(1024)
    except Exception:
        pass

    retriedTime = 0
    while len(response) == 0 and retriedTime <= RETRY_TIME:
        sleep(RETRY_WAIT_TIME)
        try:
            response = sck.recv(1024)
        except Exception:
            pass

    if len(response) == 0:
        return PORT_FILTERED
    else:
        return analyzeRes(response)
```

如果收到 RST/ACK 包说明端口是关闭的, SYN/ACK 则是开启, 没有收到回复就是被 iptables 之类过滤了. 不过这里其实 SOCK_RAW 可以收到全端口的 TCP 流量, 我比较偷懒就没做过滤了233, 所以可能会有 bug, 要完全版可以在 [github](https://github.com/rmb122/portScanner) 上看看完整的多线程版, 实现的比较完全.
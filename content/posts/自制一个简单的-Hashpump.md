---
title: 自制一个简单的 Hashpump
date: 2019-05-11 00:02:02
tags: [crypto]
katex: true
---

在学完 MD5/SHA1 之后, 很快就能搞懂 `hashpump` 的原理, 本质是因为类 `MD` 哈希函数  
包括 MD5, SHA1, SHA2 都是将信息填充为一个 `Block` 长度的倍数以后分 `Block` 一轮轮计算的,  
最后输出的是寄存器拼接在一起的值, 所以假设用以下公式计算签名  

<!--more-->

$$ hash(key + msg) = signature $$

假如已经知道了 `signature`, `msg` 以及 `key` 的长度(在实战中可以爆破), 那么完全是可以将所有 `Block` 里面除了  
一开始的 key 之外的的内容算出来. 自然而然可以想到, 我们将计算出 `signature` 时  
的状态作为我们的起始状态, 也就是将寄存器的值换成哈希函数的输出, 然后就可以在新的 `Block`  
的里面伪造任意消息, 因为这时候前面 `Block` 的内容实际上只是为了 `padding` 最后的消息长度的正确,  
所以可以在并在不知道 `key` 的情况下得到 `signature`. 这时 `msg` 就变成了  
`原来的 msg + padding + 伪造的 msg`.  

限制就是必须是 `key + msg` 的顺序, 如果是 `msg + key` 的顺序, 伪造的 `msg` 就必须是  
`原来的 msg + key + padding + 伪造的 msg`, 出现了 key, 而如果我们有 key 了, 还伪造什么呢 233  
所以局限还是比较大的, 但漏洞总是存在在哪里, 说不定就有开发者用了呢, 对吧. 所以有专门的 `HMAC` 算法  
来生成签名, 抛开 opad 和 ipad 啥的, 原理很简单, 就是用  

$$ hash(key + hash(key + msg)) = signature $$

这样嵌套的方式来计算签名, 拿到的签名值是外层的 `hash` 的结果, 而内层的 `hash(key + msg)` 的值长度是不可控的,  
显然就不能长度扩展了.  

了解这些思路, 写脚本就非常的 ez 了. 简单的对 `SHA1` 的哈希长度扩展的脚本如下.  

```python
from sha1 import SHA1
from struct import unpack
from binascii import unhexlify


def padding(msg):
    length = len(msg)
    pad = length % 64
    if pad >= 56:
        pad = (64 + 56) - pad
    else:
        pad = 56 - pad
    if pad > 0:
        msg.extend([0x80] + [0] * (pad - 1))

    length *= 8
    length = length % 0x10000000000000000
    for _ in range(8):
        b = (length & 0xff00000000000000) >> 56
        length = length << 8
        msg.append(b)
    return msg


def pump(orghash, orgmsg, keylen, addmsg):
    if type(orgmsg) == str:
        orgmsg = orgmsg.encode()
    if type(addmsg) == str:
        addmsg = addmsg.encode()
    assert len(orghash) == 40  # sha1 20 字节用 16 进制表示为 40 个字符长度
    regs = unpack('>5I', unhexlify(orghash))  # 算出之前轮结束的寄存器值, SHA1 是大端存储

    sha1 = SHA1('')
    sha1.ra, sha1.rb, sha1.rc, sha1.rd, sha1.re = regs

    newmsg = bytearray()
    newmsg += bytearray([0] * keylen)
    newmsg += bytearray(orgmsg)
    newmsg = padding(newmsg) # 还原 orghash 的区块, key 用 0 来填充

    retmsg = newmsg[keylen:] # 返回的 msg
    retmsg += bytearray(addmsg)

    newmsg += bytearray(addmsg)
    newmsg = padding(newmsg) # 因为最后 8 位是长度信息, 所以得将之前的区块一并来 padding
    newmsg = newmsg[-64:] # 但最后计算时只需要最后一块就行, 因为已经将寄存器设为之前的算出来的结果
    sha1.msg = bytearray(newmsg)
    return (retmsg, sha1.hexdigest())
```

`SHA1` 基本来自于[密码学作业][1], 稍微修改一下, 因为这里需要我们自己控制一个 `Block`, 是已经 `padding` 过的了,  
不用重复, 所以把算结果之前的 `padding` 给跳过.  

```diff
138c138
<         # self.__padding()
---
>         self.__padding()
```

最后当然是能实验成功的~  

![][2]  


[1]: https://github.com/rmb122/Cryptography/blob/master/SHA1.py
[2]: https://i.loli.net/2019/05/11/5cd5a7015838e.png

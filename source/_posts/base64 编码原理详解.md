---
title: "base64 编码原理详解"
date: 2018-04-26 08:22:44
tags: [python, misc]
---

之前一直都是直接无脑

```python
from base64 import b64encode as b64e
```

只知道大概 base64 是啥东西, 今天突然心血来潮花了几分钟了解了下 base64, 其实原理并不复杂.  

<!-- more -->

所以今天不做个 api caller, 自己实现一下 base64 编码玩玩. 首先先了解一下 base64 的实际用途, 主要是将一个字节 (1 byte、8 bit) 转换为可以直接显示的字符, 比如一个文件内容是 `\x66\x12\x13\x14` 就可以将其转化为 `ZhITFA==`. 这样就可以通过一些只能输入可见字符的设备、协议或者格式来传递二进制文件, 比如在 json 中就可以将文件 base64 编码后传递. 在 acsii 中除去空格和两个换行符号, 一共有 93 个可显示字符, 而一个字节一共有 2^8=256 种可能, 无法全部一一与 93 个字符对应, 所以我们可以通过另一种取巧的办法, 将 3 个字节拼在一起, 用 4 个 6 bits 来代替.  

这样就可以直接用 2^6=64 个可显示字符一一对应 6 个 bits 的所有可能性. 这也是 base64 里面 64 的来历, 其他 16、32 也是同理, 只是用了更少的字符对应更少的 bits, 但同时对应的存储损耗更大. 比如: base64 转化后每个可见字符保存时占用大小是一个字节, 但却需要 4 个可见字符来表示转化前的 3 个字节, 使得大小是原来的 4/3, 所以一般都使用 base64. 接下来了解了原理后就可以直接实现了, 像图中所示一样, 需要取高低六位、高低四位、高低二位. 我们可以用 python 中的位运算实现.  

```python
def getHighSix(num):
    return (252 & num) >> 2


def getLowSix(num):
    return (63 & num)


def getLowTwo(num):
    return (3 & num) << 4


def getHighTwo(num):
    return (192 & num) >> 6


def getHighFour(num):
    return (240 & num) >> 4


def getLowFour(num):
    return (15 & num) << 2

接下来我们需要一个字符表来一一对应字符与各个 6 bits 的关系,

codeTable = [
    'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O',
    'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', 'a', 'b', 'c', 'd',
    'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's',
    't', 'u', 'v', 'w', 'x', 'y', 'z', '0', '1', '2', '3', '4', '5', '6', '7',
    '8', '9', '+', '/'
]

然后我们就可以写一个简单的函数将 3 个字节转换为 4 个字符,

def encodeThreeBytes(bytecodes):
    if len(bytecodes) != 3:
        raise Exception("The bytecode's length should be three.")
    
    fisrt = getHighSix(bytecodes[0])
    second = getLowTwo(bytecodes[0]) + getHighFour(bytecodes[1])
    third = getLowFour(bytecodes[1]) + getHighTwo(bytecodes[2])
    fourth = getLowSix(bytecodes[2])
    return codeTable[fisrt] + codeTable[second]  + codeTable[third]  + codeTable[fourth] 
    '''
    等价于:
    return codeTable[getHighSix(bytecodes[0])] + codeTable[getLowTwo(
        bytecodes[0]) + getHighFour(bytecodes[1])] + codeTable[getLowFour(
            bytecodes[1]) + getHighTwo(bytecodes[2])] + codeTable[getLowSix(
                bytecodes[2])]
    '''
```

最后需要注意的是当输入的 bytes 长度不是 3 的倍数时, 需要对其补一个或者两个纯零即 `\x00` 字节后再进行编码,  
完成后将结果的倒数一、二个字符替换成 = 号表示这是补零而来即可.

```python
def b64encode(bytecodes):
    if len(bytecodes) == 0:
        return b''
        
    status = 0
    result = ""

    try:
        bytecodes = bytecodes.encode()
    except Exception:
        pass

    if len(bytecodes)% 3 == 2:
        status = 1
        bytecodes += b'\x00'

    if len(bytecodes)% 3 == 1:
        status = 2
        bytecodes += b'\x00\x00'

    for i in range(int(len(bytecodes) / 3)):
        result += encodeThreeBytes(bytecodes[i * 3:i * 3 + 3])
    
    if status == 1:
        return result[0:-1] + "="
    if status == 2:
        return result[0:-2] + "=="   
    return result
```

这样我们就完成了一个简单的 base64 编码, 最后可以写个 tests 来验证.

```python
import base64
import myBase64

bytecodes = b'\x12\x66\x23\xff'
org = myBase64.b64encode(bytecodes)
std = base64.b64encode(bytecodes).decode()

print(org)
print(std)
print(org == std)

'''
输出:
EmYj/w==
EmYj/w==
True
'''
```
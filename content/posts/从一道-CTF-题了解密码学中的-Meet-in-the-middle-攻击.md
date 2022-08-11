---
title: 从一道 CTF 题了解密码学中的 Meet-in-the-middle 攻击
date: 2019-03-25 23:26:21
tags: [writeup, crypto]
---

本文首发于[先知](https://xz.aliyun.com/t/4056)  

在家里无聊打了 nullcon, 其中有一题用到了 `Meet-in-the-middle` 这种攻击方式,  
在这里分享给大家.  

<!--more-->

## 题目
```
24 bit key space is brute forceable so how about 48 bit key space? flag is hackim19{decrypted flag}

16 bit plaintext: b'0467a52afa8f15cfb8f0ea40365a6692' flag: b'04b34e5af4a1f5260f6043b8b9abb4f8'
```

```python
from hashlib import md5
from binascii import hexlify, unhexlify
from secret import key, flag
BLOCK_LENGTH = 16
KEY_LENGTH = 3
ROUND_COUNT = 16

sbox = [210, 213, 115, 178, 122, 4, 94, 164, 199, 230, 237, 248, 54,
        217, 156, 202, 212, 177, 132, 36, 245, 31, 163, 49, 68, 107,
        91, 251, 134, 242, 59, 46, 37, 124, 185, 25, 41, 184, 221,
        63, 10, 42, 28, 104, 56, 155, 43, 250, 161, 22, 92, 81,
        201, 229, 183, 214, 208, 66, 128, 162, 172, 147, 1, 74, 15,
        151, 227, 247, 114, 47, 53, 203, 170, 228, 226, 239, 44, 119,
        123, 67, 11, 175, 240, 13, 52, 255, 143, 88, 219, 188, 99,
        82, 158, 14, 241, 78, 33, 108, 198, 85, 72, 192, 236, 129,
        131, 220, 96, 71, 98, 75, 127, 3, 120, 243, 109, 23, 48,
        97, 234, 187, 244, 12, 139, 18, 101, 126, 38, 216, 90, 125,
        106, 24, 235, 207, 186, 190, 84, 171, 113, 232, 2, 105, 200,
        70, 137, 152, 165, 19, 166, 154, 112, 142, 180, 167, 57, 153,
        174, 8, 146, 194, 26, 150, 206, 141, 39, 60, 102, 9, 65,
        176, 79, 61, 62, 110, 111, 30, 218, 197, 140, 168, 196, 83,
        223, 144, 55, 58, 157, 173, 133, 191, 145, 27, 103, 40, 246,
        169, 73, 179, 160, 253, 225, 51, 32, 224, 29, 34, 77, 117,
        100, 233, 181, 76, 21, 5, 149, 204, 182, 138, 211, 16, 231,
        0, 238, 254, 252, 6, 195, 89, 69, 136, 87, 209, 118, 222,
        20, 249, 64, 130, 35, 86, 116, 193, 7, 121, 135, 189, 215,
        50, 148, 159, 93, 80, 45, 17, 205, 95]
p = [3, 9, 0, 1, 8, 7, 15, 2, 5, 6, 13, 10, 4, 12, 11, 14]


def xor(a, b):
    return bytearray(map(lambda s: s[0] ^ s[1], zip(a, b)))


def fun(key, pt):
    assert len(pt) == BLOCK_LENGTH
    assert len(key) == KEY_LENGTH
    key = bytearray(unhexlify(md5(key).hexdigest()))
    ct = bytearray(pt)
    for _ in range(ROUND_COUNT):
        ct = xor(ct, key)
        for i in range(BLOCK_LENGTH):
            ct[i] = sbox[ct[i]]
        nct = bytearray(BLOCK_LENGTH)
        for i in range(BLOCK_LENGTH):
            nct[i] = ct[p[i]]
        ct = nct
    return hexlify(ct)


def toofun(key, pt):
    assert len(key) == 2 * KEY_LENGTH
    key1 = key[:KEY_LENGTH]
    key2 = key[KEY_LENGTH:]

    ct1 = unhexlify(fun(key1, pt))
    ct2 = fun(key2, ct1)

    return ct2

print("16 bit plaintext: %s" % toofun(key, b"16 bit plaintext"))
print("flag: %s" % toofun(key, flag))
```

## 解题思路

我们的目标就是通过给出的明文 `16 bit plaintext` 和对应的密文 `0467a52afa8f15cfb8f0ea40365a6692`  
算出 key, 从而解密 flag 密文 `04b34e5af4a1f5260f6043b8b9abb4f8`  

根据加密代码和题目描述, 很显然方法只有爆破一种, 但是爆破 6 位 key 的 `2**48`(281,474,976,710,656) 种可能是不现实的.  
但正如题目中所说, 爆破 `2**24`(16,777,216) 还是非常简单的.  

现在我们的问题变成如何简化这个爆破过程, 这里就要用到一开始说的 `Meet-in-the-middle` 攻击,  
加密主要是这两个函数,  

```python
def fun(key, pt):
    assert len(pt) == BLOCK_LENGTH
    assert len(key) == KEY_LENGTH
    key = bytearray(unhexlify(md5(key).hexdigest()))
    ct = bytearray(pt)
    for _ in range(ROUND_COUNT):
        ct = xor(ct, key)
        for i in range(BLOCK_LENGTH):
            ct[i] = sbox[ct[i]]
        nct = bytearray(BLOCK_LENGTH)
        for i in range(BLOCK_LENGTH):
            nct[i] = ct[p[i]]
        ct = nct
    return hexlify(ct)


def toofun(key, pt):
    assert len(key) == 2 * KEY_LENGTH
    key1 = key[:KEY_LENGTH]
    key2 = key[KEY_LENGTH:]

    ct1 = unhexlify(fun(key1, pt))
    ct2 = fun(key2, ct1)

    return ct2
```

发现其实真正的加密函数是 `fun`, 在 `toofun` 中调用了两次  
而且这里并没有将 6 位长的 key 一次性使用, 而是将其从中间分开, 先用前 3 位加密一次, 再用后 3 位加密一次.  
也就是说实际的加密过程是用两个 3 位的 key 连续加密二次.  
这就让我们的攻击有了可能性.  

虽然也是爆破, 但是与一开始的最先能想到的爆破 6 位 key 不同的是  
我们先遍历所有 3 位长的 key, 算出 `16 bit plaintext` 对应的所有第一次加密结果, 将其保存起来,  
然后再遍历一次 key, 将密文解密, 如果能在第一次加密的结果中找到, 那么对应的两个 3 位 key 拼起来就是一开始的 6 位 key.  
然后用这个 key 就可以解密出 flag 了~  
现在能感觉到这个攻击的名字取的还是非常切题的, 一边加密, 一边解密, 在中间碰头得到秘钥.  

与一开始的解法相比, 我们成功的将 `2**48` 降到 `2*2**24`, 不过对应的是需要一个哈希表来保存所有第一次加密的结果,  
相当于用空间换时间了, 好在现在内存都比较大, 应付起来很轻松.  

## 题解

因为 python 不太适合这种大量数据的爆破, 所以就用 C++ 写了, 写的比较烂请见谅  

```cpp
#include "md5.cpp"
#include <iostream>
#include <unordered_map>
#include <string>

const int BLOCK_LENGTH = 16;
const int ROUND_COUNT = 16;
typedef unsigned char uchar;

uchar sbox[]  = {210, 213, 115, 178, 122, 4, 94, 164, 199, 230, 237, 248, 54,
                 217, 156, 202, 212, 177, 132, 36, 245, 31, 163, 49, 68, 107,
                 91, 251, 134, 242, 59, 46, 37, 124, 185, 25, 41, 184, 221,
                 63, 10, 42, 28, 104, 56, 155, 43, 250, 161, 22, 92, 81,
                 201, 229, 183, 214, 208, 66, 128, 162, 172, 147, 1, 74, 15,
                 151, 227, 247, 114, 47, 53, 203, 170, 228, 226, 239, 44, 119,
                 123, 67, 11, 175, 240, 13, 52, 255, 143, 88, 219, 188, 99,
                 82, 158, 14, 241, 78, 33, 108, 198, 85, 72, 192, 236, 129,
                 131, 220, 96, 71, 98, 75, 127, 3, 120, 243, 109, 23, 48,
                 97, 234, 187, 244, 12, 139, 18, 101, 126, 38, 216, 90, 125,
                 106, 24, 235, 207, 186, 190, 84, 171, 113, 232, 2, 105, 200,
                 70, 137, 152, 165, 19, 166, 154, 112, 142, 180, 167, 57, 153,
                 174, 8, 146, 194, 26, 150, 206, 141, 39, 60, 102, 9, 65,
                 176, 79, 61, 62, 110, 111, 30, 218, 197, 140, 168, 196, 83,
                 223, 144, 55, 58, 157, 173, 133, 191, 145, 27, 103, 40, 246,
                 169, 73, 179, 160, 253, 225, 51, 32, 224, 29, 34, 77, 117,
                 100, 233, 181, 76, 21, 5, 149, 204, 182, 138, 211, 16, 231,
                 0, 238, 254, 252, 6, 195, 89, 69, 136, 87, 209, 118, 222,
                 20, 249, 64, 130, 35, 86, 116, 193, 7, 121, 135, 189, 215,
                 50, 148, 159, 93, 80, 45, 17, 205, 95
                };

int p[] = {3, 9, 0, 1, 8, 7, 15, 2, 5, 6, 13, 10, 4, 12, 11, 14};

uchar dbox[] = {221, 62, 140, 111, 5, 213, 225, 242, 157, 167, 40, 80, 121,
                83, 93, 64, 219, 253, 123, 147, 234, 212, 49, 115, 131, 35,
                160, 191, 42, 204, 175, 21, 202, 96, 205, 238, 19, 32, 126,
                164, 193, 36, 41, 46, 76, 252, 31, 69, 116, 23, 247, 201, 84,
                70, 12, 184, 44, 154, 185, 30, 165, 171, 172, 39, 236, 168,
                57, 79, 24, 228, 143, 107, 100, 196, 63, 109, 211, 206, 95,
                170, 251, 51, 91, 181, 136, 99, 239, 230, 87, 227, 128, 26,
                50, 250, 6, 255, 106, 117, 108, 90, 208, 124, 166, 192, 43,
                141, 130, 25, 97, 114, 173, 174, 150, 138, 68, 2, 240, 207,
                232, 77, 112, 243, 4, 78, 33, 129, 125, 110, 58, 103, 237,
                104, 18, 188, 28, 244, 229, 144, 217, 122, 178, 163, 151,
                86, 183, 190, 158, 61, 248, 214, 161, 65, 145, 155, 149, 45,
                14, 186, 92, 249, 198, 48, 59, 22, 7, 146, 148, 153, 179, 195,
                72, 137, 60, 187, 156, 81, 169, 17, 3, 197, 152, 210, 216, 54,
                37, 34, 134, 119, 89, 245, 135, 189, 101, 241, 159, 226,
                180, 177, 98, 8, 142, 52, 15, 71, 215, 254, 162, 133, 56,
                231, 0, 218, 16, 1, 55, 246, 127, 13, 176, 88, 105, 38, 233,
                182, 203, 200, 74, 66, 73, 53, 9, 220, 139, 209, 118, 132,
                102, 10, 222, 75, 82, 94, 29, 113, 120, 20, 194, 67, 11,
                235, 47, 27, 224, 199, 223, 85
               };

int q[] = {2, 3, 7, 0, 12, 8, 9, 5, 4, 1, 11, 14, 13, 10, 15, 6};

inline void mxor(uchar* a, uchar* b, uchar* tmp) {
    for(int i = 0; i < BLOCK_LENGTH; i++) {
        tmp[i] = a[i] ^ b[i];
    }
}

inline void copy(uchar* from, uchar* to) {
    for(int i = 0; i < BLOCK_LENGTH; i++) {
        to[i] = from[i];
    }
}

void fun(uchar* key, uchar* ct, uchar* tmp) {
    for(int i = 0; i < ROUND_COUNT; i++) {
        mxor(ct, key, tmp);
        copy(tmp, ct);
        for(int i = 0; i < BLOCK_LENGTH; i++) {
            ct[i] = sbox[ct[i]];
        }
        uchar nct[BLOCK_LENGTH];
        for(int i = 0; i < BLOCK_LENGTH; i++) {
            nct[i] = ct[p[i]];
        }
        copy(nct, ct);
    }
}

void dec(uchar* key, uchar* ct, uchar* tmp) {
    for(int i = 0; i < ROUND_COUNT; i++) {
        uchar nct[BLOCK_LENGTH];
        for(int i = 0; i < BLOCK_LENGTH; i++) {
            nct[i] = ct[q[i]];
        }
        copy(nct, ct);

        for(int i = 0; i < BLOCK_LENGTH; i++) {
            ct[i] = dbox[ct[i]];
        }
        mxor(ct, key, tmp);
        copy(tmp, ct);
    }
}

int main() {
    std::unordered_map<std::string, std::string> map;
    uchar* md5key = new uchar[BLOCK_LENGTH];
    uchar tmp[BLOCK_LENGTH];
    uchar hexkey[] = {0, 0, 0};

    for(int i = 0; i < 256; i++) { // 爆破明文所有 key 加密的结果
        for(int j = 0; j < 256; j++) {
            for(int k = 0; k < 256; k++) {
                uchar pt[] = "16 bit plaintext";
                copy(MD5((char*)hexkey, 3).digest, md5key);
                fun(md5key, pt, tmp);

                std::string s1((char*)pt, 16);
                std::string s2((char*)hexkey, 3);

                map[s1] = s2;
                hexkey[2] += 1;
            }
            hexkey[1] += 1;
        }
        std::cout << i << std::endl;
        hexkey[0] += 1;
    }

    for(int i = 0; i < 256; i++) { // 爆破密文所有 key 解密的结果, 如果在 map 中找到一样的, 用这两个 key 拿来解密 flag
        for(int j = 0; j < 256; j++) {
            for(int k = 0; k < 256; k++) {
                uchar pt[] = "\x04g\xa5*\xfa\x8f\x15\xcf\xb8\xf0\xea@6Zf\x92"; // "0467a52afa8f15cfb8f0ea40365a6692".decode('hex')
                copy(MD5((char*)hexkey, 3).digest, md5key);
                dec(md5key, pt, tmp);

                std::string s1((char*)pt);
                auto ptr = map.find(s1);

                if(ptr != map.end()) {
                    std::cout << "Key found" << std::endl;
                    uchar key[3];
                    uchar flag[] = "\x04\xb3NZ\xf4\xa1\xf5&\x0f`C\xb8\xb9\xab\xb4\xf8"; // "04b34e5af4a1f5260f6043b8b9abb4f8".decode('hex')
                    key[0] = (*ptr).second.c_str()[0];
                    key[1] = (*ptr).second.c_str()[1];
                    key[2] = (*ptr).second.c_str()[2];
                    copy(MD5((char*)hexkey, 3).digest, md5key);
                    dec(md5key, flag, tmp);
                    copy(MD5((char*)key, 3).digest, md5key);
                    dec(md5key, flag, tmp);
                    std::cout << std::string((char*)flag, 16) << std::endl;
                    return 0;
                }
                hexkey[2] += 1;
            }
            hexkey[1] += 1;
        }
        std::cout << i << std::endl;
        hexkey[0] += 1;
    }
    return 0;
}
```

其中 `dbox` 和 `q` 可以这样生成  
```python
dbox = [0 for i in range(len(sbox))]
q = [0 for i in range(len(p))]
for i in range(len(sbox)):
    dbox[sbox[i]] = i
for i in range(len(p)):
    q[p[i]] = i
```

大约需要 3g 内存和几分钟时间得到 flag  
编译的时候别忘记带上 -O3  
```
Key found
1337_1n_m1ddl38f
```

## 总结

密码学博大精深, `Meet-in-the-middle` 只是其中一种攻击技术, 但其中的思路还是非常值得学习的,  
有时候爆破不可避免, 我们能做的是如何让爆破最简化, 至少不那么暴力 233.  

想要了解更多关于 `Meet-in-the-middle` 的知识可以去看对应的 [WIKI](https://en.wikipedia.org/wiki/Meet-in-the-middle_attack)
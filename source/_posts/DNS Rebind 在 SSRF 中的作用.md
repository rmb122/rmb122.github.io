---
title: "DNS Rebind 在 SSRF 中的作用"
date: 2019-02-21 16:34:34
tags: [web]
---

这几天沉迷 [hgame](https://hgame.vidar.club/), 题目质量还是不错的, 其中有一题用到了 DNS rebind, 这里重新记一下笔记. 因为我估计也没多少人看我博客 233, 就直接写了, 不等比赛结束了.

<!-- more -->

可以先看看 DNS Rebind 的 [wiki](https://en.wikipedia.org/wiki/DNS_rebinding) 页面, 其实就跟这个名字的意思差不多, 重新绑定, 也就是说第一次请求和第二次请求返回的 IP 地址不同. 我这里直接上题目代码, 看看是如何利用的.

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

注意 `String ip = InetAddress.getByName(domain).getHostAddress();` 和 `URLConnection connection = u.openConnection();.` 在拿到 ip 以后, 是直接再用原来的链接打开, 而不是通过 ip 访问. 也就是说, 这里其实解析了两次域名 (如果没有缓存的话, 这个下面说)

这给我们了机会来绕过检测, 我们只要让 DNS 第一次返回一个不是 `127.0.0.1` 的地址, 第二次再返回 `127.0.0.1` 即可. 这样 `u.openConnection` 将会链接 `127.0.0.1`, 实现 SSRF.

可以直接在 Github 上找到已有的[项目](https://github.com/Crypt0s/FakeDns), 试了一下还是不错的. 不过在这个题目下使用有点小问题, 题目这里设置的 DNS 服务器是 `8.8.8.8`, 而 `8.8.8.8` 在递归的时候请求了一个不支持的类型 \x00\xff, 查了一下 [RFC](https://tools.ietf.org/html/rfc1035), 是这个

```
3.2.3. QTYPE values

QTYPE fields appear in the question part of a query.  QTYPES are a
superset of TYPEs, hence all TYPEs are valid QTYPEs.  In addition, the
following QTYPEs are defined:

AXFR            252 A request for a transfer of an entire zone

MAILB           253 A request for mailbox-related records (MB, MG or MR)

MAILA           254 A request for mail agent RRs (Obsolete - see MX)

\*               255 A request for all records
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
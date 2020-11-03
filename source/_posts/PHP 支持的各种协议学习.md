---
title: "PHP 支持的各种协议学习"
date: 2019-01-04 18:27:17
tags: [php, web]
---

PHP 支持了不少协议, 特别是其中的 `php://` 和 `phar://` 经常能在意想不到的地方发挥意想不到的作用 233\. 之前都是花式百度, 这次系统学习一下. 所有的协议都可以在[官网](http://php.net/manual/en/wrappers.php)上找到.

这些协议一般配合 `file_get_contents`, `include` 使用, 部分情况下受到 php.ini 里面的 `allow_url_fopen`(默认开启) 和 `allow_url_include`(默认关闭) 限制. 如果 `allow_url_include` 开启的话, 是非常危险的, 可能存在远程文件包含.

<!-- more -->

### 0x01 file://

不受 `allow_url` 限制, 用来读取文件, 一般不会用到, 毕竟 file\_get\_contents 和 include 本身就是这个功能 233.

而且 file:// 必须跟着绝对路径, 比如 file:///etc/passwd, file://../../../etc/passwd 是不行的

### 0x02 http://

受到 `allow_url` 限制. 这个就是 http 协议啦, 也就是说 file\_get\_contents 可以当 curl 用, 不过比较麻烦就是了, 像这样的

```php
$file = $_GET['file'];
if(file_get_contents($file) === "The quick brown fox jumps over a lazy dog.") {
    echo $flag;
}
```

就可以在自己服务器上写个内容是 `The quick brown fox jumps over a lazy dog.` 的 html, 然后 `file=http://xxxx.xxx/xx.html` 就可以了, 当然后面还有更简单的绕过方法.

### 0x03 ftp://

比较鸡肋, 我试了一下, 开启了 ftp 扩展的情况下, php5, php7 都没用, 似乎已经被废弃了

### 0x04 php://

相对来说内容最多, 也是最重要的一个协议, 协议底下分了许多种流, 各个流分开看看. 需要注意的是这个协议也受到 `allow_url` 限制, 远程读取的流都不给 `include` 233

**@ php://stdin, php://stdout, php://stderr**

这个是拿来给命令行程序用的0.0 比如

```php
<?php
file_put_contents("php://stdout", file_get_contents("php://stdin"));
```

就是将从 stdin 里面的数据输出到 stdout 里面去

**@ php://input**

对任何 HTTP method 都可以用, 读取 HTTP body 里面的内容, 用来读二进制文件的 POST 有奇效, 也可以用来绕过上面的例子

**@ php://output**

官网上解释的很清楚233, 跟 echo, print 一样, 命令行就输出到 stdout, cgi 就输出到 html 里面

**@ php://memory and php://temp**

可以拿来存东西, 但是感觉很鸡肋, 因为在 fd 关闭之后就直接清空了, 233 真的是只能拿来临时存放东西

**@ php://filter**

最骚的东西, 可以用于读取源码, 比如

```php
include $_GET['file'].".php";
```

本来其实没有什么大问题, 但是如果 file 是 `php://filter/read=convert.base64-encode/resource=index` 读取出来的内容将 base64encode, 从而直接显示出来, 可以通过这种方式来读取 `.php` 文件.

注意过滤器是可以叠加的, 可以这样 `php://filter/read=convert.base64-encode/read=convert.base64-encode/resource=index.php` base64两次.

PS. read= 可以省略, 也就是说可以写成 `php://filter/convert.base64-encode/resource=index`

过滤器除了 `convert.base64-encode` 还有很多, 可以看[这里](http://php.net/manual/zh/filters.php)

这些流也可以拿来写入文件, 比如

```php
file_put_contents("php://filter/write=convert.base64-encode/resource=index.php", "test"); // 同样 write= 可以省略
```

index.php 中写入的将是 base64 编码过后的 test, 也就是 dGVzdA==   

最后, resource 后面还可以跟其他协议, 甚至可以 `resource=php://` 这样嵌套, 不过 `allow_url` 限制不会因此改变. 如果是 include 调用的, 依然受到 `allow_url_include` 限制.

### 0x05 compress.zlib://, compress.bzip2://, zip://

不受 `allow_url` 限制, 而且 zip:// 只要符合 zip 格式即可, 不用在意后缀名, 所以在可以上传文件而且可以包含这个文件的情况下, 可以拿来任意代码执行. 用法是 `zip://path_to_zipfile#filename_in_zip`, 注意在地址栏传参的时候记得把 `#` 换成 `%23`, 不然会被当成锚点.

比如 `include $_GET['file'].".php";` 的情况下, 可以上传图片. 就可以将一个含有马的 zip 文件重名成 jpg 后缀上传, 最后 `file=zip://upload/xxx.jpg%23backdoor` 即可.

另外, compress.bzip2:// 和 compress.zlib:// 对于没有压缩的文件是直接读取原文的, 而且不像 file://, 是支持相对路径的, 可以当做强化版来用, 233 在 `parse_url` 之后检测 host 的情况下可能有奇效, 比如 `compress.bzip2://www.baidu.com/../../../../../../etc/passwd`

```php
php > print_r(parse_url('compress.bzip2://www.baidu.com/../../../../../../etc/passwd'));
Array
(
    [scheme] => compress.bzip2
    [host] => www.baidu.com
    [path] => /../../../../../../etc/passwd
)
```

### **0**x06 data://

就和官网上讲的一样 233, [RFC2397](http://www.faqs.org/rfcs/rfc2397.html) 的标准, 格式就是 `data:[<mediatype>][;base64],<data>`, 你可以在浏览器上输入 data://test,data schema test, 也可以看到浏览器其实也是支持这个协议的, 一般用来通过 html 直接传输图片.

比较可惜的是受到 `allow_url` 限制, 不过因为 `allow_url_fopen` 是默认开启的, 拿来构造任意内容还是可以的, 拿来绕上面的例子也是很轻松.

### 0x07 phar://

比较逆天的存在, 不受 `allow_url` 限制

1.  可以这样, phar://test.zip/index.php 代替上面的 zip:// 的作用 (同样不受后缀名影响)
2.  通过 phar pack -f test.phar bd.php 生成的 phar 包来包含 phar 里面的内容 (同样不受后缀名影响)
3.  当然最最最骚的是[这个](https://blog.ripstech.com/2018/new-php-exploitation-technique/), 只要能上传文件到同一个服务器, 就能任意反序列化, 甚至 `file_exist` 这样的函数都会受到影响, 然后通过内部类来各种操作得到 flag. 可以说 2018 年的 CTF 竞赛不知道出现了多少遍这个东西 233, 就如文章所说, 真的是开创了一个新的 `technique`.

其他的协议都不是 php 自带的, 需要安装插件, 就不看了 233
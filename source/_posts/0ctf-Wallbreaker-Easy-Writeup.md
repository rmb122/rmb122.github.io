---
title: 0ctf Wallbreaker Easy Writeup
date: 2019-03-26 16:06:31
tags: [writeup, 0ctf, web]
---

本文首发于[先知](https://xz.aliyun.com/t/4589)  

周末打了两天, 自闭 web 狗就做出来这一题, 另一题不知道调用啥 mbean 能拿 shell. 总之题目质量真的是非常高, 学到了很多.  

## 描述
```
Imagick is a awesome library for hackers to break `disable_functions`.
So I installed php-imagick in the server, opened a `backdoor` for you.
Let's try to execute `/readflag` to get the flag.
Open basedir: /var/www/html:/tmp/949c1400c8390865cb5939a106fec0b6
Hint: eval($_POST["backdoor"]);
```

<!-- more -->

## 提示
```
WALLBREAKER EASY
Ubuntu 18.04 / apt install php php-fpm php-imagick
```

第一眼看上去, 直接给了 webshell, 要求是执行根目录下的 `/readflag`,  
先执行一波 `phpinfo()` 看看信息.
![](https://i.loli.net/2019/03/25/5c98caa8c8e93.png)  
![](https://i.loli.net/2019/03/25/5c98caa8cc2e2.png)  
![](https://i.loli.net/2019/03/25/5c98caa90a2f7.png)  

可以看到是 lnp 64 位环境, disable_function 如下  
```
pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal,pcntl_signal_get_handler,pcntl_signal_dispatch,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,system,exec,shell_exec,popen,proc_open,passthru,symlink,link,syslog,imap_open,ld,mail
```

Imagick 信息如下  
![](https://i.loli.net/2019/03/25/5c98caa92f03a.png)


## 题解

可以看到禁用了全部能直接执行程序的函数, 顺便还禁用了 mail, 不然的话可以直接用 `LD_PRELOAD` 来执行系统命令. 具体可以看这个[项目](https://github.com/yangyangwithgnu/bypass_disablefunc_via_LD_PRELOAD).  

接下来第一反应是通过 ghostscript 的 0day 打一波试试, 但尝试了几次全部都没有反应... 好吧没有报错太蛋疼了, 我们还是照着提示搭一波环境吧.  

然后发现这里其实就真的跟提示一样, 全是最新的环境, 所以肯定是不存在已知的严重 0day 的...  
同时, 可以看到在默认的配置文件中, 已经禁用了 ghostscript 的使用, 不能通过 gs 来命令执行.
```xml
$ cat /etc/ImageMagick-6/policy.xml 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE policymap [
<!ELEMENT policymap (policy)+>
<!ELEMENT policy (#PCDATA)>
<!ATTLIST policy domain (delegate|coder|filter|path|resource) #IMPLIED>
<!ATTLIST policy name CDATA #IMPLIED>
<!ATTLIST policy rights CDATA #IMPLIED>
<!ATTLIST policy pattern CDATA #IMPLIED>
<!ATTLIST policy value CDATA #IMPLIED>
]>
<policymap>
  <!-- <policy domain="resource" name="temporary-path" value="/tmp"/> -->
  <policy domain="resource" name="memory" value="256MiB"/>
  <policy domain="resource" name="map" value="512MiB"/>
  <policy domain="resource" name="width" value="16KP"/>
  <policy domain="resource" name="height" value="16KP"/>
  <policy domain="resource" name="area" value="128MB"/>
  <policy domain="resource" name="disk" value="1GiB"/>
  <policy domain="delegate" rights="none" pattern="URL" />
  <policy domain="delegate" rights="none" pattern="HTTPS" />
  <policy domain="delegate" rights="none" pattern="HTTP" />
  <!-- in order to avoid to get image with password text -->
  <policy domain="path" rights="none" pattern="@*"/>
  <policy domain="cache" name="shared-secret" value="passphrase" stealth="true"/>
  <!-- disable ghostscript format types -->
  <policy domain="coder" rights="none" pattern="PS" />
  <policy domain="coder" rights="none" pattern="EPI" />
  <policy domain="coder" rights="none" pattern="PDF" />
  <policy domain="coder" rights="none" pattern="XPS" />
</policymap>
```

看起来好像一切的都没了希望, 但这时突然想起, 有没有可能通过 `Imagick` 来达到 `mail` 函数类似的效果呢? 上面的 `LD_PRELOAD` 其实是劫持了启动进程这一行为, 也就是说, 如果我们能让 `Imagick` 调用外部进程, 我们完全可以不通过 `mail` 来执行系统命令.  

`Imagick` 的底层是 `ImageMagick`, 这里就要说 `ImageMagick` 的 `delegate` 问题, `ImageMagick` 其实并没有实现所有文件格式的转换, 而是启动外部程序来进行转换, 就像上面的 ghostscript, 如果要将图片转换为 pdf, 就需要调用 ghostscript 来转换, 而这就会启动新的进程, 触发 `LD_PRELAOD`.  

```xml
$ cat /etc/ImageMagick-6/delegates.xml 
<!DOCTYPE delegatemap [
<!ELEMENT delegatemap (delegate)+>
<!ELEMENT delegate (#PCDATA)>
<!ATTLIST delegate decode CDATA #IMPLIED>
<!ATTLIST delegate encode CDATA #IMPLIED>
<!ATTLIST delegate mode CDATA #IMPLIED>
<!ATTLIST delegate spawn CDATA #IMPLIED>
<!ATTLIST delegate stealth CDATA #IMPLIED>
<!ATTLIST delegate thread-support CDATA #IMPLIED>
<!ATTLIST delegate command CDATA #REQUIRED>
]>
<delegatemap>
<!-- 省略很多行 -->
  <delegate decode="bmp" encode="wdp" command="/bin/mv &quot;%i&quot; &quot;%i.bmp&quot;; &quot;JxrEncApp&quot; -i &quot;%i.bmp&quot; -o &quot;%o.jxr&quot;; /bin/mv &quot;%i.bmp&quot; &quot;%i&quot;; /bin/mv &quot;%o.jxr&quot; &quot;%o&quot;"/>
  <delegate decode="ppt" command="&quot;soffice&quot; --convert-to pdf -outdir `dirname &quot;%i&quot;` &quot;%i&quot; 2&gt; &quot;%u&quot;; /bin/mv &quot;%i.pdf&quot; &quot;%o&quot;"/>
  <delegate decode="pptx" command="&quot;soffice&quot; --convert-to pdf -outdir `dirname &quot;%i&quot;` &quot;%i&quot; 2&gt; &quot;%u&quot;; /bin/mv &quot;%i.pdf&quot; &quot;%o&quot;"/>
  <delegate decode="ps:alpha" stealth="True" command="&quot;gs&quot; -sstdout=%%stderr -dQUIET -dSAFER -dBATCH -dNOPAUSE -dNOPROMPT -dMaxBitmap=500000000 -dAlignToPixels=0 -dGridFitTT=2 &quot;-sDEVICE=pngalpha&quot; -dTextAlphaBits=%u -dGraphicsAlphaBits=%u &quot;-r%s&quot; %s &quot;-sOutputFile=%s&quot; &quot;-f%s&quot; &quot;-f%s&quot;"/>
  <delegate decode="ps:cmyk" stealth="True" command="&quot;gs&quot; -sstdout=%%stderr -dQUIET -dSAFER -dBATCH -dNOPAUSE -dNOPROMPT -dMaxBitmap=500000000 -dAlignToPixels=0 -dGridFitTT=2 &quot;-sDEVICE=pamcmyk32&quot; -dTextAlphaBits=%u -dGraphicsAlphaBits=%u &quot;-r%s&quot; %s &quot;-sOutputFile=%s&quot; &quot;-f%s&quot; &quot;-f%s&quot;"/>
<!-- 省略很多行 -->
</delegatemap>
```

可以看到有很多的 delegate, 在进行这些格式的编码以及解码时, `ImageMagick` 会调用这些外部程序. 但是这些外部程序大部分其实都不自带, 得自己安装, 不会起新的进程, 但是可以注意到这个特殊的格式,  
```xml
<delegate decode="bmp" encode="wdp" command="/bin/mv &quot;%i&quot; &quot;%i.bmp&quot;; &quot;JxrEncApp&quot; -i &quot;%i.bmp&quot; -o &quot;%o.jxr&quot;; /bin/mv &quot;%i.bmp&quot; &quot;%i&quot;; /bin/mv &quot;%o.jxr&quot; &quot;%o&quot;"/>
```
其中有系统自带的 mv, 这肯定是能执行成功的, 我们只要通过转换格式就能调用这个 delegate.  
通过 strace 也可以发现确实调用了 mv, 即使因为 JxrEncApp 不存在导致图片转换失败, 但只要因为 mv 调起了新进程, 我们就能执行任意命令.

![](https://i.loli.net/2019/03/25/5c98d6280c9a5.png)  
![](https://i.loli.net/2019/03/25/5c98d627f3530.png)  

这样, 就可以通过 `ImageMagick` 来触发新进程的产生, 并通过修改 `LD_PRELOAD` 的方式来执行任意系统命令.  

最后 exp 如下, 写入so文件  
![](https://i.loli.net/2019/03/25/5c98caa992bb6.png)  
修改 `LD_PRELOAD` 并转换图片格式, 最终执行系统变量 `EVIL_CMDLINE` 里的命令.  
![](https://i.loli.net/2019/03/25/5c98caa98b8aa.png)  

PS. 写入 so 文件的时候记得把 base64 里的 `+` 给编码, 不然会被当成空格.  

还有一种思路是修改 `PATH` 环境变量, 修改为 `/tmp/xxx/`, 然后新建一个名为 `JxrEncApp` 之类的的恶意文件,  
`ImageMagick` 在 `delegete` 的时候将会调用这个, 从而执行命令.  

不得不说题目质量是真的高, 膜 rr.  

## 总结

除了这种 `LD_PRELOAD` 的方式之外, 还有其他一些骚操作, 可以看 [l3m0n](https://github.com/l3m0n/Bypass_Disable_functions_Shell) 师傅总结的.  
在遇到这种困境时, 不防想想有没有什么其他办法能 bypass 掉当前的限制, 开辟新的方法.  
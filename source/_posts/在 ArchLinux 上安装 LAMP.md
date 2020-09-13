---
title: "在 ArchLinux 上安装 LAMP"
date: 2018-11-30 18:58:53
tags: [linux]
---

在平常总得遇到点题目得测试 / 复现一下, 如果本地就有全套 LAMP 环境的话就会方便很多.

### 0x01 首先安装对应的软件包

```sh
sudo pacman -S apache mariadb mariadb-clients php php56 php-apache php56-apache
```

<!-- more -->

`mariadb` 是 `oracle` 收购 `mysql` 后开源社区的 fork, 可以理解为备份... 防止`oracle` 突然修改 `mysql` 的协议. php -> php7, php56 -> php5, `php-apache` 是对应的 `apache` 模块, 因为我们只是本地测试用, 就不搞 `php-fpm` 了

### 0x02 修改配置文件

#### 1. apache

`sudo vim /etc/httpd/conf/httpd.conf` 将 `LoadModule mpm_event_module modules/mod_mpm_event.so` 注释掉, 并将 `LoadModule mpm_prefork_module modules/mod_mpm_prefork.so` 前面的注释给删掉, 因为 `php-apache` 只支持 `pre-fork` 模式. 将 `ServerName www.example.com:80` 修改成 `ServerName 127.0.0.1:80`, 其他地址同理 然后加上

```apache
# Load php7
LoadModule php7_module modules/libphp7.so 
Include conf/extra/php7_module.conf
 
# Load php56
# LoadModule php5_module modules/libphp56.so
# Include conf/extra/php56_module.conf
```

注意只能同时启动一个, 不能同时启动 `php7` 和 `php56`

#### 2. mariadb

初始化数据库 `sudo mysql_install_db --basedir=/usr --user=mysql --datadir=/var/lib/mysql` 启动服务 `systemctl start mysqld` 设置 `root` 密码 `sudo /usr/bin/mysqladmin -u root password '{password}'`

#### 3. php

`sudo vim /etc/php/php.ini` (php5 同理) 启用扩展, 去掉前面的分号注释就可以了 `extension=mysqli` 加上 `unix socket` 的地址 `mysqli.default_socket = /run/mysqld/mysqld.sock` 启用报错 (默认是关闭的) `display_errors = On`

#### 4. phpmyadmin

在[官网](https://www.phpmyadmin.net/)下载后直接解压到 `/srv/http` 就行, 比较方便, 如果不想提示 blowfish 密码的话, 就在 `phpMyAdmin/libraries/config.default.php` 里面的 `$cfg['blowfish_secret']` 后面随便加上点什么就行了.

### 0x03 启动 apache

`sudo apachectl start` 重启就是 `sudo apachectl restart` 注意想要 php.ini 的修改生效也得重启 apache, 然后就可以访问 `http://127.0.0.1` 看到成功启动了~ 可以打个 `phpinfo()` 看看 php 有没有成功启动. 可以看到 `Linux` 下配置还是挺简单 0.0, 不是很麻烦.
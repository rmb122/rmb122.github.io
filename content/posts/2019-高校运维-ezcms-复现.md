---
title: 2019 高校运维 ezcms 复现
date: 2020-01-13 00:04:55
tags: [web, writeup]
---

之前高校运维碰到的题目, 当时就我一个人输出 web + 自己太菜了, 就没做出来 \_(:3」∠)\_, 寒假有时间看看, 复现一下题目.

<!--more-->

题目非常良心的给了 docker-compose.yml, 所以搭建环境很简单. 把 install.lock 删掉重新装一遍就 ok.
经过一段时间审计, 可以发现 `application/collection/controller/collection_content.class.php` 中 `collection_test` 等函数, 实际上未经任何过滤就将 URL 传给了 `collection::get_content`.

```php
/**
* 添加采集节点
*/
public function add() {
    if (isset($_POST['dosubmit'])) {
        if (!$_POST['urlpage']) showmsg('网址配置不能为空！');
        $res = D('collection_node')->insert($_POST);
        if ($res) {
            showmsg(L('operation_success'), U('init'), 1);
        } else {
            showmsg(L('operation_failure'));
        }
    } else {
        include $this->admin_tpl('collection_node_add');
    }
}

public function collection_test() {
    $id = isset($_GET['id']) ? intval($_GET['id']) : 0;
    if (!$id) showmsg(L('lose_parameters'));
    $data = D('collection_node')->where(array('nodeid' => $id))->find();
    if ($data['urlpage'] == '') showmsg('网址配置不能为空！', 'stop');

    //目标网址
    if ($data['sourcetype'] == 1) {
        $url = str_replace('(*)', $data['pagesize_start'], $data['urlpage']);
    } else {
        $url = $data['urlpage'];
    }

    //定义采集列表区间
    $url_start = $data['url_start'];
    $url_end = $data['url_end'];

    if ($url_start == '' || $url_end == '') showmsg('列表区域配置不能为空！', 'stop');

    $content = collection::get_content($url);
```

```php
public static function get_content($url) {
    self::$url = $url;

    $content = '';
    if (extension_loaded('curl')) {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);
        curl_setopt($ch, CURLOPT_HEADER, 0);
        $content = curl_exec($ch);
        curl_close($ch);
    } else {
        $content = @file_get_contents($url);
    }
    return trim($content);
}
```

很明显的 SSRF, 再结合 nginx 配置.

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;

    # With php-fpm (or other unix sockets):
#	fastcgi_pass unix:/run/php/php7.3-fpm.sock;
    # With php-cgi (or other tcp sockets):
    fastcgi_pass 127.0.0.1:9000;
}
```

明显就是 gopher 打 fastcgi 了.

根据 view 里的文字, 可以猜出是采集功能. 用默认账号密码 yzmcms 登录进后台之后在模块管理里面就能找到, 拿 gopherus 一把梭就可以.

然后我去看了眼最新代码 `https://github.com/yzmcms/yzmcms`. 仍然没有修复, 怪不得 wp 还没出来 (逃


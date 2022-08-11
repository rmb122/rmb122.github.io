---
title: "Thinkphp >= 5.0.23 RCE 分析"
date: 2019-01-16 19:03:53
tags: [web, audit, php]
---

起因是刚出来那天刚好有个站是 5.0.23 但是分析文章没给完整的截图, 打了码. 干脆自己分析一波写写 poc.

![](https://i.loli.net/2019/03/08/5c827b967b058.jpg)

<!--more-->

先切换到 v5.0.23

```sh
git clone git@github.com:top-think/framework.git
git checkout v5.0.23
```

主要问题出在 `library/think/Request.php` 中的 `method`, 其中的 Config::get('var\_method') 默认值是 \_method, 而其没有任何过滤就被传进 $this->{$this->method}($_POST) 中, 导致可以调用本类的任何单参数函数.

![](https://i.loli.net/2019/03/08/5c827b9b6fbef.jpg)

官方的补丁也是增加过滤措施

![](https://i.loli.net/2019/03/08/5c827ba0aa16d.jpg)

这里我反着来, 从利用链末端的危险函数 `filterValue` 开始分析,

![](https://i.loli.net/2019/03/08/5c827ba5b5cb0.jpg)

可以看到其中调用了 call\_user\_func, 只要我们能控制 filters 和 value, 就能调用任意函数

这里查找这个方法的引用, 找到 cookie 和 input, 这里我偷懒, 直接加上 echo, 找找看是否有启用

![](https://i.loli.net/2019/03/08/5c827baa8eac6.jpg)

![](https://i.loli.net/2019/03/08/5c827bb04e7d4.jpg)

![](https://i.loli.net/2019/03/08/5c827bb3c3fe5.jpg)

这里可以看到 cookie 方法并未被在框架中直接使用, 所以我们从 input 上下手

![](https://i.loli.net/2019/03/08/5c827bb7efec3.jpg)

可以看到找到一个调用了 input 且 `name = ''` 的方法就行了, data 将会被作为 filterValue 中的 value.

继续查找引用, 这里我随便找了一个 get, var_dump 了一下确认了 `name =''`. $_GET 数组将会被当做 value 传进 input.

![](https://i.loli.net/2019/03/08/5c827bbc2120b.jpg)

这样 filterValue 中的 value 已经构造完成, 接下来只要构造 filter 即可, 这个比较简单, 在构造函数中可以看到

![](https://i.loli.net/2019/03/08/5c827bbf258ed.jpg)

只要抢先在底下对 $this->filter 赋值之前给他赋值就行了.

最后需要添加一个 method=GET, 不然没有路由信息

![](https://i.loli.net/2019/03/08/5c827bc64c589.jpg)

比较遗憾的是, 在比较后面的版本中必须打开 App::debug 或者有 captcha 模块才能利用.

因为有很多地方都调用了 input 方法, 所以 payload 有很多种, 不过本质都是通过类的构造函数来覆盖变量导致 filterValue 被错误的执行.
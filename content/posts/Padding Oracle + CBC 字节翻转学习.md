---
title: "Padding Oracle + CBC 字节翻转学习"
date: 2019-02-21 17:04:38
tags: [crypto]
---

之前一直就很想了解的技术, 这次同样是在 [hgame](https://hgame.vidar.club/) 上碰到, 刚好从[师傅](http://xnianq.cn/2017/07/26/padding-oracle)那里学习一下, 更详细的可以去看 [wiki](https://en.wikipedia.org/wiki/Padding_oracle_attack).

简单来讲就是通过服务器对 Padding 正确和错误返回的不同状态, 来猜解所谓的`中间值`, 从而达成解密数据, 伪造数据.

<!--more-->

以这张图为例子

![](https://i.loli.net/2019/03/08/5c8279f81fce7.png#center)

我们可以暴力枚举倒数第二块密文的最后一个字节 (如果只有一块密文的话就修改 IV), 直到最后一块的最后一个字节刚好为 \\x01, 这样服务器将会返回`权限验证失败`而不是`解密失败`. 这样将 \\x01 与此时爆破出来的倒数第二块密文的字节 xor 一下, 得到的就是`中间值`, 再将中间值与原来的倒数第二块密文的最后一个字节 xor, 就可以得到明文. 此时再将 \\x02 与中间值 xor, 作为密文的最后一个字节, 继续爆破倒数第二个字节, 使得最后两个字节都为 \\x02, 就跟上面一样, 可以得到倒数第二位的中间值.

以此类推, 可以解出全部的明文, 同时, 有了中间值, 我们也可以通过修改前一块的数据来伪造数据, 比如要伪造 'a', 只要将 `中间值 ^ ord('a')` 的结果作为前一块对应位置的数据即可. 然后将前一块作为最后一块, 再去爆破中间值. 最后可以伪造任意数据.

最后再来说说 CBC 字节翻转, 其实原理也是一样的, 通过修改前一块的数据, 不过这次是取 `前一块对应的密文 ^ 后一块对应的明文 ^ 你想要伪造的数据` 的结果. 不过需要注意的是这样将会使得`前一块密文`的解密结果异常, 具体解密出来是啥得看运气 233, 不过大概率是乱码. 当然, 如果只有一块密文或者我们修改的是第一块密文, 我们修改的将会是 IV, 这样可以使结果完全可控.
---
title: "利用 eBPF 技术配合 http 服务实现隐藏后门程序"
date: 2024-11-01T21:38:06+08:00
tags: [misc]
---

## 背景

eBPF 作为 Linux 内核的一个功能, 目前已经被广泛运用在各个发行版中. 比如 systemd 就用了 eBPF 来[限制服务的权限][1], 可以运行 `bpftool prog list` 来查看当前加载的 eBPF 程序, 大概率会出现一堆 systemd 加载的 eBPF 程序. 

eBPF 程序运行在内核的虚拟机中, 很难不让人想到可以拿来作为 rootkit 用, 事实上目前已经有了很多开源 eBPF rootkit, 比如[TripleCross][2]和[ebpfkit][3], 可以实现大部分 rootkit 的功能. 但是目前的 eBPF rootkit 都需要额外启动一个用户态程序来实现命令执行功能. 这样 eBPF 程序只用于隐藏自身和下发命令, 执行命令则完全依靠长期运行的用户态程序. 

<!--more-->

最接近纯 eBPF 实现命令执行的 rootkit 是前面举例的 [TripleCross][2]. 它存在一个 `Library injection` 功能, 这个功能(理论上, 实际上我看实现也是一直在后台跑着的)只需要短暂启动一个用户态程序注入 shellcode 到目标进程, 之后 eBPF 就能利用该 shellcode 劫持控制流并 fork 出新进程完成后续恶意代码的执行, 官方画的流程图如下:

![](https://raw.githubusercontent.com/h3xduck/TripleCross/refs/tags/v0.1.0/docs/images/lib_injection-s5.png)

可以看到非常复杂, 整整有 13 步. 弄这么复杂的原因有两个, 第一个是 TripleCross 想要在恶意代码执行完成后交还控制流回到正常程序, 第二个是 eBPF 的功能非常受限, 毕竟一开始目的就是拿来做 observability 的, 实际上能用于恶意目的的 api 只有 `bpf_probe_write_user` 和 `bpf_override_return`, 而 `bpf_override_return` 又只能在特定条件下用, 使得真正有破坏性功能的函数只有一个可以写入任意内容到用户空间内存中的 `bpf_probe_write_user`.

也就是说 eBPF 本身是没有在内核态 fork / execve 以及分配内存的功能的, 在这种情况下, TripleCross 为了能够不破坏当前程序的执行又要执行自己的恶意代码, 只能弄的这么复杂了. 但是如果我们的目标上本身就运行了类似 apache / nginx 这类 worker 结束后可以无限复活的程序, 事情就不太一样了.

## 方法

首先, apache 这类 http 服务本身监听端口且接受数据. 而且, 整个流程从 accept 开始就已经交由 worker 来进行处理, 通过 worker 很难把 master 干爆. 最后, master 进程会在 worker 退出后自动新建 worker. 这使得我们可以完全不用管 worker 死活, 直接通过覆盖栈上返回地址的方式劫持它的控制流然后退出即可. 可以说 apache / nginx 这类服务是天生 eBPF 后门圣体.

不过还存在一个问题, 虽然我们可以轻松的劫持控制流, 但是 shellcode 在不运行用户态程序的情况下, 单靠 eBPF 是没有直接的方法写入的, 因为 `bpf_probe_write_user` 会检测目标页是否可写, 也就是说只有 rwx 的页才能直接写入 shellcode. 但很显然 nginx / apache 这类服务是不会存在 rwx 页的 (. 一开始还想尝试通过 PHP 8 新加入的 JIT 功能来找找是否存在 rwx 页, 但是 PHP 的[实现][4]相当安全, 会在写入机器码后 mprotect 到 r-x. 因此, 最后还是得用一些简单的 ROP 技巧, 利用 libc 的 gadget 来实现命令执行.

### 整体流程

到这里, 攻击流程也不难想象出来, eBPF 程序内的攻击流程可以分为下面这几步

1. 解析指令, 拿到要执行的命令
2. 遍历虚拟内存, 找到 libc 的基址
3. 获取寄存器, 拿到栈地址
4. 往栈上写入构造好的 ROP payload
5. pwn!

另外, 还需要为 eBPF 找到一个合适的 attach 点, 从而可以方便的获取指令. 在这方面, http 服务显然是需要读 socket 数据的, 所以非常简单, hook 住相关函数就行, 选择非常多, 这里我选择的是 `fexit/ksys_read`.  

#### 解析指令

在第一步其实就有非常大的坑, 一开始我尝试复用 HTTP 协议, 直接从 HTTP 的 body 里面获取要执行的命令. 但是结果发现虽然能够成功获取命令并打印, 但是最后在写入命令到用户栈上时, 很难通过 eBPF 的验证器, 会显示 `Program too large`, 实际上就是由于经过了多重循环, 又从其中取字符串切片, 程序复杂度太高, 验证器无法保证要写入的字符串长度是否超过实际的长度, 就直接摆烂报错了, 可以参考[这个回答][5].

最后, 命令格式直接设计成了简单的 `EXEC{command}\x00`, 可以直接判断前四字节来决定是否走后门流程, 且直接通过 \x00 结尾, 可以省去命令的长度的计算. 同时在 eBPF 中 buffer 长度也设置为 32, 从而保证不会复杂度太高, 在写入栈时直接写去掉 EXEC 前缀的 28 字节即可.

#### 获取 libc 基址

获取 libc 基址如果没有相关接口的话基本上是不可能的事情, 好在 eBPF 存在 bpf_iter_task_vma_new / bpf_iter_task_vma_next / bpf_iter_task_vma_destroy 这一套 api, 可以直接遍历分配的内存, 效果类似 `cat /proc/self/maps`, 通过判断内存映射的文件名就可以知道当前内存区域是否映射至 libc, 从而得到 libc 基址.  

这一步最大的坑是如何匹配 libc, 路径在内核中是以树结构保存的, 文件路径对应的数据结构是个树中的叶子节点, 在常规的内核开发中都可以直接调用 `d_path` 来获取完整路径, 不幸的是虽然 eBPF 中也存在相关 [api][6], 但是却无法调用. 我试了下即使是使用了在文档上标注的 `BPF_PROG_TYPE_TRACING` 类型, 也无法调用, 比较迷. 

还有种思路是直接模拟 d_path 的流程, 靠读内核的内存来慢慢解析, 比如[这个][7]. 然而在循环内套循环的结果相信你也可以猜到了, 依然是复杂度爆炸. 导致最后我的解决方案是简单粗暴的只匹配文件名, 好在一般也不会存在与 libc 同名的被 mmap 的文件.

#### 获取寄存器并写入 gadget

通过 PT_REGS_FP 宏和 bpf_task_pt_regs 可以直接获取到 rbp 的值, 然后 +4 直接写 gadget 就行了...么? 实际上不太行, 需要注意的的一点是我测试所用的 php docker 里面的 read 是 libpthread 的 read 所调用的, 其中的 ebp 相关操作直接被编译器优化掉了. 实际上的返回地址用 ida 看了下, 是直接被 rsp 所指的, 可以用 PT_REGS_SP 宏获取.

生成 gadget 就不细说了, 直接用 pwntools 就能生成, 然后实际写入的时候把 `touch /tmp/pwned` 替换成从指令中获取的命令即可.

```python
from pwn import *
import sys

context.clear(arch='amd64')

binary = ELF(sys.argv[1])
rop = ROP(binary, base=0)

rop.call('execve', [b'/bin/sh', [[b'/bin/sh'], [b'-p'], [b'-c'], [b'touch /tmp/pwned'], 0], 0])

result = rop.build()
print(result)
print(result.dump())
```

#### 效果

![](/images/ebpf-backdoor-demo.gif)

## 总结

相关代码开源在 https://github.com/rmb122/ebpf-backdoor-demo, 可以自行体验, 效果感觉还是不错的. 在代码里还额外实现了如果存在 rwx 页的情况, 直接写 shellcode 就行了, 比较简单.  

目前来看, 虽然 eBPF 不管是拿来做可观测性, 还是 debug, 或者网络栈协议的修改等等都是很好用的, 但是如果拿来做后门还是存在不少问题. 比较小的问题是 eBPF 验证器导致程序复杂度不能太高, 导致很多看似简单的操作都无法完成, 这个其实并不致命, 大不了逻辑做简单点. 最大的问题还是 eBPF 本身的 api 非常受限, 毕竟就不是拿来做后门用的 (. 目前已有的 api 能用的, 具有破坏性的实际上只有 bpf_probe_write_user, 而且如果你查看内核日志的话, 会发现这 api 也被加了限制, 每用一次都会 printk 一次, 非常的不后门 233

```
[88867.273897] bpftool[27084] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88867.273971] bpftool[27084] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88867.299636] bpftool[27084] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88867.299689] bpftool[27084] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88902.122774] bpftool[27201] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88902.122854] bpftool[27201] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88902.149253] bpftool[27201] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
[88902.149304] bpftool[27201] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
```

[1]: https://github.com/systemd/systemd/blob/main/src/core/bpf/restrict_fs/restrict-fs.bpf.c
[2]: https://github.com/Gui774ume/ebpfkit
[3]: https://github.com/h3xduck/TripleCross
[4]: https://github.com/php/php-src/blob/f2602f32009b65e44a33792ad8af992ca0932fd9/ext/opcache/jit/zend_jit.c#L3664-L3667
[5]: https://stackoverflow.com/questions/70147464/program-too-large-threshold-greater-than-actual-instruction-count
[6]: https://docs.ebpf.io/linux/helper-function/bpf_d_path/
[7]: https://github.com/stackrox/falcosecurity-libs/blob/1fb31e0483d8eae3c83df3bcf83e9834158d29aa/driver/bpf/filler_helpers.h#L141
[8]: https://github.com/bminor/glibc/blob/7f045c0b48633b198b42bebdff0024d7cfab3901/posix/pread.c#L24

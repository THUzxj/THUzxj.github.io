---
layout: post
title: Deterministic Compilation by eBPF (基于eBPF的确定性编译工具)
date: 2023-07-03 00:00:00
description: Deterministic Compilation by eBPF (基于eBPF的确定性编译工具)
tags: eBPF deterministic-build
categories: projects
featured: true
---

## 1. 关于Inclavare Containers

本项目和Inclavare Container社区在阿里巴巴编程之夏项目中合作完成，[项目地址](https://github.com/inclavare-containers/deterministic-builds)。

Inclavare是`enclave`的拉丁语源，它是机密计算领域中一种为了保护用户数据，将用户的敏感工作负载与不受信任的、不可控的 基础设施隔离的技术。

Inclavare Container是一种从硬件辅助可信执行环境（TEE）中启动受保护容器的安全容器运行时。

## 2. 项目介绍

确定性编译（Deterministic Compilation），或者说可复现构建（Reproducible Builds），是指从相同的源代码中编译出相同的二进制文件的技术。它在软件供应链的安全以及管理中有重要作用。

免费与开源软件的源代码可以供所有人来检查，但是大多数软件都是预编译分发的，很难确认编译出的二进制文件是否与源代码对应，而没有引入只隐藏于二进制文件的后门或漏洞。确定性编译技术目标是搭建一个与编译时间、机器、文件名等无关的编译环境，让第三方可以方便地验证二进制文件是否由指定版本的源代码编译而成。

同时在二进制文件的管理中，如果需要对二进制文件进行存储管理，那么最好不要让二进制文件中存储会随着环境改变的时间戳、文件路径等信息，以形成完全相同的二进制文件。而确定性编译技术可以免除编译器引入这些不确定因素。

检查两个二进制文件是否相同的一个简单的方法是消息摘要（message digest），包括MD5，SHA系列算法等。

已有的确定性编译方案通过对编译器进行配置、ptrace拦截系统调用实现。而本项目希望实现基于eBPF的确定性编译系统。eBPF是Linux内核提供的在内核中运行沙箱程序的技术，允许开发者在不改动内核代码的同时对Linux内核的安全、网络、监控等功能进行扩展。Kprobe是Linux内核提供的对内核内任意函数（包括系统调用）进行跟踪探测的机制，可以在指定的函数调用时、任意指令处与函数返回时执行额外的程序。Linux同样提供了很多tracepoint的tracing机制。eBPF可以把代码挂载到Kprobe机制上，相比于直接开发Kprobe模块拥有eBPF提供的安全保障。

使用eBPF来监控编译器会引入不确定因素的系统调用进行监控，并修改其返回值，我们可以开发出一个对编译器完全透明的确定性编译工具，只需要在后台运行eBPF程序，不需要在使用编译器时添加额外的配置与指令，也不会影响除了编译器与其他编译所需工具之外的其他进程。

本项目我们的主要目标是让工具可以实现Linux内核的确定性编译。

## 3. 开发情况

### 编译中的不确定因素

首先我们需要定位编译器引入的不确定因素，以及它们所对应的系统调用。经过调研与实验，编译器在运行中主要会受到时间、随机数、文件系统等影响，在字符串、符号名、可执行文件时间戳等包含不确定因素。

#### 方案

我们没有去阅读编译器代码，而是采用以下方法进行调查：一方面我们收集一些常见的不确定性因素（如宏定义等），进行实验并跟踪系统调用；一方面我们通过编译Linux内核查看编译出的二进制文件中存在哪些不确定性。

Linux内核的编译中，我们使用相同版本的编译器、相同的配置文件、相同的Linux内核源码版本，分别在不同的时间、不同的Linux系统、不同的文件路径上进行编译，检查不同情况下编译出的二进制文件vmlinux的区别。

#### 工具

strace可以用来监控编译器的系统调用。由于gcc编译器的预处理进行是在子进程中的，所以strace使用时需要使用`--follow-forks`或`-f`选项。

二进制文件的相关工具，用于比较二进制文件的可以使用xxd与diff（或colordiff、cmp）等配合，查看二进制文件内容的有strings、nm、readelf、objdump等，二进制文件的GUI编辑器有010editor等。

#### 结果

我们发现以下几种编译不确定性的来源：

##### 1

gcc编译器提供了一些预先定义好的宏定义，在预处理阶段会进行宏定义展开与include展开等工作。其中`__DATE__`，`__TIME__`分别可以展开为编译时的日期与时间的字符串。`__TIMESTAMP__`与`__FILE__`可以展开为被编译的文件的最后一次修改时间戳和相对路径的字符串。

##### 2

此外，在gcc中开启LTO优化时，会在.o文件中生成随机的符号名称，使用了系统的random相关系统调用，这会影响到.o文件的确定性。

##### 3

在编译与比较Linux kernel中，发现vmlinux中有一个字符串是编译信息的记录，有编译此内核的Linux版本、用户名与机器名、gcc版本、时间戳、内核功能选项等。经调查，在内核安装后`/proc/version`会提供此版本信息；在编译时`scripts/mk_compile_h`会在`compile.h`中获取并在代码中生成各种信息，在编译机器上使用了`date`, `whoami`, `uname`等指令。

```
Linux version 5.18.11-051811-generic (kernel@gloin) (gcc (Ubuntu 11.3.0-4ubuntu1) 11.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38.50.20220629) #202207121541 SMP PREEMPT_DYNAMIC Tue Jul 12 15:47:28 UTC 2022
```

我们还发现二进制文件中有5处随机数的差异。在用LD_PRELOAD方法简单拦截libc中相关时间函数后，发现时间戳与随机数都变成相同的值，说明内核编译中产生随机数与时间相关。

### 确定性编译工具

我们使用libbpf，使用C语言进行eBPF程序的开发。eBPF的开发工具还有Bpftrace，bcc等，但是尝试在这些框架下实现同样的修改系统调用功能时并不会获得预期的行为，具体原因还需要学习与调查。

#### eBPF修改系统调用的返回值

与ptrace、seccomp等不同的是，eBPF并不允许直接修改函数的参数、返回值等。但是eBPF提供了一个`bpf_probe_write_user`函数，该函数可以传入一个用户空间地址（`void*`类型）dst和一个eBPF程序地址（`void*`类型）src以及一个u32类型的长度参数，它可以将src buffer的内容写入到用户空间，达到修改用户空间内存的目的。而一部分系统调用传入一个指向结构体的指针，调用者期待从结构体中获得函数返回的数据，这个地址也会被kprobe传入eBPF程序中，那这个结构体里的返回结果就可以被修改，如`gettimeofday`，以`struct timeval *tv, struct timezone *tz`两个参数返回具体的值，函数返回值只是0与-1的状态码。同样的，如果系统调用传入的参数是一个字符串，那么这个字符串也可以被修改成一个不比原来长的另一个字符串。很遗憾函数的返回值仍然不能被修改，但是能修改结构体中的结果对实现确定编译的目标已经足够了。而如果系统调用没有传入一个指针，那么eBPF程序对修改这个函数就无能为力了，比如`time`，直接以`time_t`类型作为返回值。

离可以修改系统调用只剩一步之遥了：eBPF的进入与退出hook点都只能获取到部分上下文信息，进入函数的hook点可以获得函数参数，包括结构体指针，但是不能在此时修改结构体，因为会被后面的函数逻辑再次修改；退出函数的hook点是修改结果的时机，但是只提供了函数返回值，没有提供结构体指针。这时就需要使用到eBPF的MAP（相当于eBPF程序各个地方都可以访问到的全局key value存储），实现进入函数向退出函数传递指针数据，我们使用进入与退出函数都可以获得的tid（对于每一次目标函数调用都是唯一的）作为key，传递的指针作为value，这样就使得退出函数也能获得结构体指针。

所以修改系统调用的流程是：

1. 在进入Hook函数中，通过进程名过滤掉不需要的进程，向map存储\<tid, arg0\>
2. 在退出Hook函数中，通过进程名过滤掉不需要的进程，然后通过tid查询map获得arg0
3. 构建假的返回值，并使用bpf_probe_write_user修改用户态内存

最终我们使用这种方法可以修改`openat`,`newfstatat`,`read`,`getrandom`,`gettimeofday`,`clock_gettime`，`uname`等系统调用，去除文件系统、时间、随机数、主机带来的不确定性。

#### vDSO的处理

在使用strace date等命令，对时间相关的系统调用进行分析时，会发现`time`，`gettimeofday`，`clock_gettime`并不会出现在strace的结果中，使用eBPF也没有办法被触发。这是因为Linux的`vDSO`机制，它把time相关的系统调用结果映射到内存中，在调用这些系统调用时直接从用户内存中读取结果，可以减少用户态与内核态的切换开销，做到快速系统调用。vDSO在Linux运行中是默认开启的，因此在使用工具前我们需要在启动选项中把它关闭。

#### 与LD_PRELOAD配合

由于eBPF程序无法修改`time`系统调用的返回值，我们需要使用另外的方法修改它。`time`系统调用被libc中的`time`函数使用，所以我们可以使用`LD_PRELOAD`的方法，在用户态把libc中的`time`函数替换掉。但是这样就需要在环境变量中指定LD_PRELOAD的so文件，或是在系统配置文件`/etc/ld.so.preload`中指定所有可执行文件的preload路径，无法做到自动拦截指定的编译相关进程的调用。而通过对`LD_PRELOAD`机制的系统调用进行分析，我们同样可以通过拦截系统调用的方式，让不同进程的读到的`/etc/ld.so.preload`不同，只对编译相关进程加载preload文件。

#### 结果

以date命令为例，使用modify_time可以使输出的时间为固定值（在编译前在一个头文件中配置）：

```bash
$ date # normal
Mon Oct 17 04:25:13 PM CST 2022
$ sudo ./modify_time &
$ date # modified
Mon Oct 17 04:26:25 PM CST 2022
```

以下为eBPF程序的log：

```bash
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
			<...>-816502  [006] d..31 5707624.022076: bpf_trace_printk: [sys_exit_clock_gettime] OVERWRITING struct timespec64 at     00000000c626f924 from (1665995173, 362091123) to (1659312000, 0)
           <...>-816502  [006] d..31 5707624.022077: bpf_trace_printk: [sys_exit_clock_gettime] RESULT 0
```

### 测试工具

为了在多个环境中测试编译确定性，我们基于shell脚本与docker实现了部分编译自动配置与编译的功能，如C语言宏定义的编译与kernel源码的配置与编译。


## 参考资料

eBPF and syscall:

- [Can eBPF modify the return value or parameters of a syscall?](https://stackoverflow.com/questions/43003805/can-ebpf-modify-the-return-value-or-parameters-of-a-syscall)
- [Offensive BPF: Understanding and using bpf_probe_write_user](https://embracethered.com/blog/posts/2021/offensive-bpf-libbpf-bpf_probe_write_user/)
- [Capture vDSO in strace](https://stackoverflow.com/questions/38103583/capture-vdso-in-strace)
- [[iovisor-dev] kprobe modify syscall](https://lists.linuxfoundation.org/pipermail/iovisor-dev/2017-May/000740.html)

deterministic build:

- [Deterministic builds with clang and lld](https://blog.llvm.org/2019/11/deterministic-builds-with-clang-and-lld.html)
- [An introduction to deterministic builds with C/C++](https://blog.conan.io/2019/09/02/Deterministic-builds-with-C-C++.html)

# BPF详解

[TOC]

#性能排查入门

## 总体原则

1. 目标明确，比如目标是降时延，还是提高性价比，还是提升吞吐率。 **别上来就揪着一些奇怪的指标看，也许并不是个有用的东西**。

2. 负载业务画像方式：5W2H方式。 what：负载是什么（syscall的read/write)，谁产生的（进程PID，UID，IP）？负载为什么产生（调用栈，代码路径）？负载是如何变化的？

   

## LINUX性能排查60s

总体： uptime，dmesg|tail， free -m。

CPU bottom： mpstat， pidstat，top

memory bottom： vmstat

IO bottom： iostat 1, sar -n DEV, sar -n TCP,ETCP

### 1. uptime

​	1min/5min/15min的负载情况，check现场是否仍然有问题发生。若稳定，则有问题，若偏差较大，可能已经错过了问题点。这里，负载的单位是：正在&将要在cpu上运行的任务数 + 阻塞在不可屏蔽中断下的IO任务数

``` c
[root@VM1 ~]# uptime
 00:50:12 up 2 days, 12:42,  2 users,  load average: 0.48, 0.10, 0.03
```

### 2. dmesg | tail

查看是否有内核报错信息。

### 3. free -m

查看available列，可用内存是否足够。

### 4. mpstat -P ALL 1

逐个CPU查看负载情况，关注点：

- 负载问题在单核or多核？
- 问题在sys or usr or io？（%sys, %usr, %iowait）

``` c
[root@openEuler-sp1-performanceTest UnixBench]# mpstat  -P ALL 1
Linux 4.19.90_NEW-HOST-NOFB+ (openEuler-sp1-performanceTest)    10/18/2021      _aarch64_       (64 CPU)

08:57:21 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
08:57:22 AM  all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
08:57:22 AM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
08:57:22 AM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
08:57:22 AM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
08:57:22 AM    3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

```

### 5. pidstat 1

按照进程显示CPU使用情况，可以滚动打印。包含任务运行在哪个cpu上。

``` c
[root@openEuler-sp1-performanceTest UnixBench]# pidstat 1
Linux 4.19.90_NEW-HOST-NOFB+ (openEuler-sp1-performanceTest)    10/18/2021      _aarch64_       (64 CPU)

09:00:43 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
09:00:44 AM     0    278988    0.98    1.96    0.00    0.00    2.94     5  pidstat

09:00:44 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
09:00:45 AM     0      2139    0.00    1.00    0.00    0.00    1.00    37  irqbalance
09:00:45 AM     0    278988    0.00    1.00    0.00    0.00    1.00     5  pidstat
```

### 6. top

综合上述，涵盖了mpstat，pidstat，iostat，uptime等所有信息。

### 7. vmstat 1

有用信息：

- r：cpu上正在&将要在cpu上执行的进程数量，不含任何IO进程；
- free：空闲内存，单位kB
- si/so: 换入/换出的页数，若不为0，说明内存可能不够用了。

``` c
[root@VM1 ~]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 7309028  62912 452732    0    0     0     0   11    9  0  0 100  0  0
 0  0      0 7309028  62912 452732    0    0     0     0   21   19  0  0 100  0  0
```

### 8. iostat -xz 1

可以check IO设备的负载情况。

- r/s, w/s, d/s, rkB/s, wkB/s，dkB/s ： 读写请求数，读写带宽，d表示discard的请求。
- await， r_await, w_await, d_await：单位ms，每个IO请求的平均响应时间，包含队列中的时间以及处理IO请求的时间。
- avgqu-sz(或aqu-sz): 平均队列长度，若超过1则需要check下。
- util：设备繁忙程度，不准确，即使达到100%也可能有优化空间，比如多磁盘情况下。若util>60%则需要check下磁盘是否为瓶颈了。

``` c
[root@openEuler-sp1-performanceTest UnixBench]# iostat -xz
Linux 4.19.90_NEW-HOST-NOFB+ (openEuler-sp1-performanceTest)    10/18/2021      _aarch64_       (64 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.13    0.00    0.01    0.02    0.00   99.84

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz  aqu-sz  %util
dm-0             0.06      6.25     0.00   0.00   16.67    99.14    0.86      4.56     0.00   0.00    1.32     5.28    0.00      0.00     0.00   0.00    0.00     0.00    0.00   0.11
dm-1             0.00      0.01     0.00   0.00    7.69    72.21    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00   0.00
dm-10            0.00      0.01     0.00   0.00   13.68    70.74    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00   0.00
dm-11            0.50     14.26     0.00   0.00   25.30    28.33    0.65      4.60     0.00   0.00    4.65     7.06    0.00      0.00     0.00   0.00    0.00     0.00    0.02   0.19
dm-12            0.00      0.01     0.00   0.00   15.26    70.74    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00   0.00
nvme0n1          0.00      0.10     0.00   0.00    0.16    77.43    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00   0.00
nvme1n1          0.00      0.07     0.00   0.00    0.15    73.31    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00   0.00
```

### 9. sar -n DEV 1

网卡负载情况。

``` c
[root@openEuler-sp1-performanceTest UnixBench]# sar -n DEV 1
Linux 4.19.90_NEW-HOST-NOFB+ (openEuler-sp1-performanceTest)    10/18/2021      _aarch64_       (64 CPU)

09:14:50 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
09:14:51 AM enp125s0f3      1.00      0.00      0.05      0.00      0.00      0.00      0.00      0.00
09:14:51 AM    virbr0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:14:51 AM enp189s0f1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

```

### 10. sar -n TCP,ETCP 1

tcp连接数，繁忙程度等。

active： 主动建立连接数；

passive：被动接收连接数；

retrans：重传数。

``` c
[root@openEuler-sp1-performanceTest UnixBench]# sar -n TCP,ETCP 1
Linux 4.19.90_NEW-HOST-NOFB+ (openEuler-sp1-performanceTest)    10/18/2021      _aarch64_       (64 CPU)

09:15:45 AM  active/s passive/s    iseg/s    oseg/s
09:15:46 AM      0.00      0.00      1.00      0.00

09:15:45 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
09:15:46 AM      0.00      0.00      0.00      0.00      0.00

```



## BCC性能排查top 11

总体： profile

CPU bottom： runqlat，execsnoop,

memory bottom： cachestat

IO bottom： tcpconnect, tcpaccept, tcpretrans, opensnoop, ext4slower, biolatency, biosnoop

仅挑出个别的来描述：

### 1. execsnoop

​	跟踪execve系统调用接口，去snoop所有创建出来的进程comm， pid等。

### 2. profile

打印所有进程的调用栈及调用次数，与perf record -g之后的perf script输出类似。

### 3. runqlat

跟踪re调度实体，打印出当前任务不在running状态的时间有多少，也就是没有cpu可用的状态。

### 4. cachestat

只是看文件系统层面上的cache是否命中，不是硬件cache，目前看用处不大。

### 5. ext4slower

这个有点意思，可以打印出整个IO路径中时延较长的ops的comm，pid，操作的文件大小、偏移（读写点相比文件头的偏移）、IO操作时延等。

### 6.  biolatency

所有bio的时延统计图，方便IO问题定位。



# BPF原理

## bpf设计理念

1. 低开销 - 对性能影响小
2. 更安全，不会轻易令内核crash

在linux内核中运行sandbox，无需改内核源码，无需加载内核模块。令内核可编程。

eBPF程序代码为只读，不允许修改

eBPF程序不能直接访问内核memory，必须通过helper calls

 ## bpf实现原理

high level看：

​	bpf程序 经过LLVM翻译成BPF字节码，然后通过`JIT`或者解释器来真正的执行。

![step.png](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5C4040745124-60f7d73775abe.png)



![bpftrace_mid-level_internals.png](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5C2611460215-60f66e35edf77.png)



![image-20210917084813861](D:\at_work\Documents\我的总结文档\images\image-20210917084813861.png)

- bpftrace语法 -> AST语法树  
  - // bpftrace的解析工具
  - 需要经过BPF verifier的验证，确保语法都OK才允许仅需走下去。

​		通过`bpftrace -e 'kprobe:do_nanosleep {printf("PID %d sleeping...\n",pid);}' -d`来查看其翻译解析情况。

``` c
bpftrace -e 'kprobe:do_nanosleep {printf("PID %d sleeping...\n",pid);}' -d
BTF: failed to find BTF data
Program
 kprobe:do_nanosleep
  call: printf
   string: PID %d sleeping...\n
   builtin: pid
......
```

​		其中有用到libbpf.so等动态链接库的内容。若涉及到内核，如pid需要向内核syscall获取，则通过helper去跟内核进行交互，内核中include/uapi/bpf.h文件中有保存了与bpf的接口函数。

- AST语法 -> LLVM IR

  - 经过Code Generation & IR Builder模块的处理，AST会被转化成LLVM IR。

  ​	`bpftrace -d`也可以看到LLVM IR的详细翻译情况。

- LLVM IR -> bytecode

  ​	通过LLVM编译器生成。以上都是在用户态完成的。

- bytecode ---> machine code

  - 从这一步开始进入内核，
  - 通过内核的jit编译，区分架构。这个动作是实时翻译的。

​	  可以通过`bpftool prog dump jited id 3`来查看汇编代码

- **JIT和解释器：** 

​	二者都是把字节码翻译成机器码（用到了虚拟指令，因此也成为虚拟机方式）；

​	都是经过编译 + 运行 两个阶段。

​	解释器编译后，将机器码存放在内存中，使用后就没有了。因此每次使用都要重新编译，而`JIT`第一次编译后会保存，后续使用的时候就可以直接运行了。

​	仅编译时用到了虚拟机，运行时就没有虚拟机什么事了，直接就是跑程序。



## bpf的组件

1. JIT   compilation

eBPF代码编译出来的是bytecode，所以可以跨平台。

1. 使用LLVM，将C代码编译成bytecode

2. 运行时使用JIT翻译成机器码。

3. map - 内核态与用户态共享的一块内存空间。

4. helper calls - 内核中已经写好的bpf函数接口

5. verifier - 权限验证&出口验证，保证eBPF的安全性

   

### 调用栈回溯

​		理论上可以通过栈帧指针（x29，FP reg）来进行追溯，x86也是这样做的。当然，需要编译选项显性指明`-fno-omit-frame-pointer`才可以。

``` c
00000000004012fc <cleanup>:
  4012fc:   b4000040    cbz x0, 401304 <cleanup+0x8>
  401300:   d65f03c0    ret
  401304:   a9be7bfd    stp x29, x30, [sp, #-32]!
  401308:   910003fd    mov x29, sp
  40130c:   f9000bf3    str x19, [sp, #16]
。。。。。。
  401338:   f9400bf3    ldr x19, [sp, #16]
  40133c:   a8c27bfd    ldp x29, x30, [sp], #32
  401340:   d65f03c0    ret
```

​		函数嵌套、中断、syscall等会打断前一个进程的情况下，会根据调用者的需要自行进行相关寄存器和状态的保存，也就是说占用的栈大小是不确定的。如call一个函数前，编译器会先store会修改的寄存器们，计算并保存x29，然后才b过去，此时硬件会把返回地址写到LR，在ret之前，软件同样的会自行load回原来的寄存器们，等运行到ret指令时，将LR写回PC调回来，就完成一次函数调用了。这个过程中，栈帧指针可以用来进行调用栈回溯。知道了栈帧，就知道了函数的返回地址，因为返回地址一定是放在栈顶的。

​		若栈帧被omit了，也可以通过其他软件方案获取调用栈，比如增加debug信息（dwarf格式的的调试信息，以及orc格式等，都是在elf中额外添加冗余段，进行相关信息的记录），但是，这些方式在bpf中不支持，在bcc/bpftrace中是支持的。



### kprobe

​	动态插桩，原理是：将kprobe要观测的地址内容复制到另一个地址中，源地址的**第一个指令**被修改为一个单步中断（x86上是INT3指令，arm上实测是brk #0x5指令，brk会产生异常进入异常处理函数处理），当程序运行至此，则触发中断或直接跳转，在中断处理函数中进行kprobe相应的统计，然后继续执行正常的内容。当不再观测时，修改回原来的内容即可。

​	详细来说：

1. 设置kprobe点（init_kprobes)：
   - 通过kallsyms查找函数入口的地址a，然后把这个地址的指令替换为INT3，原指令保存在一块kmalloc出来的内存中，内存地址保存在struct kprobe中.
   - 注册一个struct kprobe结构体，并以a作为hash值保存到kprobe_table的hash表中，相同hash值的struct以链表形式进行连接。
   - 注册DIE notifier函数，在获取int3/brk/单步等异常时可以直接通知到回调函数kprobe_exceptions_notify()，注意，notifier中会判断是int3还是单步触发的这个异常，然后调用不同的处理函数。若为前者则调用kprobe_handler，后者则调用post_kprobe_handler。
2. 指令执行INT3/BRK #5，触发**int3/brk异常**
   1. 可以进入kprobe_handler回调函数，在其中会调用pre_handler()函数；
   2. 同时，设置执行模式为单步调试模式
   3. 修改PC值，为原来的指令
3. 指令执行了一条后，触发**单步异常**
   1. 进入post_kprobe_handler回调函数，在其中会调用post_handler()函数，解除单步异常，并恢复原来的PC地址内容。

![img](D:\at_work\Documents\我的总结文档\images\2276022-20210110075907892-825572189.png)

​	kretprobe的原理是：在kprobe函数入口处插桩，当命中时，将返回地址修改为一个trampoline函数地址。返回时就会跳到该函数去处理，完成后再返回之前保存的地址即可。

​	出于安全的考虑，部分函数不允许kprobe，且修改地址时会stop_machine()禁止其他cpu核执行指令。

​	bpftrace监控kprobe方式：

``` c
bpftrace -e 'kprobe:vfs_* { @[probe] = count() }'
```

#### kprobe 源码

​	可以通过debugfs方式使用，也可以通过自行编写ko方式注册使用。

​	bpftrace是通过debugfs方式，向`"/sys/kernel/debug/tracing/kprobe_events"`写入用户给的kprobe函数名，然后进行跟踪。在内核中，这个子项注册了`struct file_operations kprobe_events_ops`（trace_kprobe.c），收到write时会调用`probes_write`函数，然后参数copy到内核态，再调用`__trace_kprobe_create`进行参数解析，之后调用`register_trace_kprobe`来真正注册probe事件。

​	在`register_trace_kprobe`函数中，会先判断该函数是否被标记为notrace的，若是则直接报`warning：Could not probe notrace function **`，若不是，则进入到register_kprobe或register_kretprobe来进行处理。

​	需要注意的是，对于`notrace`的函数，有两种方式可以追踪。

way 1： 通过自行编写ko的方式，因为直接调用`register_kprobe`函数进行注册，绕过了`within_notrace_func`的检查；

way 2： 内核选项开启`CONFIG_KPROBE_EVENTS_ON_NOTRACE`选项，这样`within_notrace_func`函数会直接返回`False`.



ps. kprobes过程中若有任何error，可以通过`cat tracing/error_log`来查看具体错误信息。

ps. kprobes过程中，可以tail /var/log/messages查看pr_warn的打印信息。

### uprobe

​	与kprobe类似，对用户态程序进行插桩。若对某一个文件进行插桩，则所有用到该文件的程序，无论是否正在运行，都会被插桩修改。

​	bpftrace监控uprobe方式：

``` c
bpftrace -e 'uprobe:/lib64/libc.so.6:p* {@[probe]=count();}'
```

### tracepoint

​	内核中预置的trace点。内核编译时，遇到trace_*函数时，会对函数开头和结尾进行修改。

​	在开头，x86上编译成5Byte的nop指令，arm上为4byte的inline nop指令，但是若开pac时则不会inline，而是编译成如下格式的函数。

``` c
ffff800010b0e9f0 <trace_xhci_dbg_cancel_urb>:
ffff800010b0e9f0:   d503233f    paciasp
ffff800010b0e9f4:   d50323bf    autiasp
ffff800010b0e9f8:   d65f03c0    ret
ffff800010b0e9fc:   d503201f    nop
```

​	在结尾，会蹦床跳转到一个遍历所有回调函数的地方去，这里会损耗一定的性能。

当开启tracepoint跟踪时，会将开头的nop修改为jmp或其他跳转指令，同时将回调函数注册过去，从而结尾蹦床时能找到该函数。当关闭tracepoint时，则将开头改回nop，同时注销回调函数。







## bpf 配置要求

内核要求：

- CONFIG_BPF
- CONFIG_BPF_SYSCALL
- CONFIG_BPF_EVENTS
- CONFIG_BPF_JIT
- CONFIG_HAVE_EBPF_JIT

可以通过直接 man+命令，查看REQUIREMENTS域段说明。

## bpf 性能开销

可以通过man+命令，从OVERHEAD域段说明中获取。

## bpf调试

### bcc调试

1. printk方式。
   1. 在bcc代码中，添加`bpf_trace_printk("a=%llx", req);`语句来打印，可以在`/sys/kernel/debug/tracing/trace_pipe`中查看打印结果。
   2. 在创建bpf对象是指定调试等级。`b=BPF(text=bpf_text, debug=0x2)`，不同等级可以在`src/cc/bpf_module.h`文件中查看，如debug=0x1f会打印全部信息，0x1打印LLVM IR， 0x8打印内嵌汇编，0x20打印BTF调试信息。
   3. bpftool： 显示当前正在运行的程序，列出BPF指令，和map进行交互等。
   4. dmesg。

### bpftrace调试







# bcc/bpftrace基础性能工具

获取帮助信息的途径：

1. man + 命令，可以获取dependency（包括内核配置、工具包等）、性能开销、函数打印域段说明等；
2. tools/doc/*_example.txt，可以获取很多example及对应的输出结果和结果解读。
3. 



## bpftrace 使用帮助

### 查看可以trace的点

``` bash
# 列出所有支持的参数，包括tracepoint/kprobe/kfunc/uprobe/usdt等
$ bpftrace -l
# 仅查看硬件探针
$ bpftrace  -lv 'h:*'
hardware:backend-stalls:
hardware:branch-instructions:
hardware:branch-misses:
hardware:bus-cycles:

# 有些函数看不到，需要添加 --unsafe才能显示。
bpftrace --unsafe -l 'uprobe:/usr/lib64/libc.so.6:*memcpy*'
```

### 查看函数参数

```shell
# 查看内核结构体的具体参数list
bpftrace -lv "struct file"
# 查看内核函数的参数
bpftrace -lv 'tracepoint:timer:hrtimer_start'
# 查看内核结构体的成员偏移(!!!，需要内核使能DEBUG_INFO_BTF)
bpftool btf dump file /sys/kernel/btf/vmlinux  | grep -i "STRUCT 'file'" -A20    
# 打印参数的成员变量
bpftrace -e 'kprobe:do_iter_readv_writev { printf("args: %lx : %s \n", ((struct file *)arg1)->f_flags, str(((struct file *)arg1)->f_path.dentry->d_iname[3])); }' --include linux/fs.h
```

```shell
# 统计函数入参出现次数
./argdist -C 'p::prep_new_page(struct page *page, unsigned int order, gfp_t gfp_flags,unsigned int alloc_flags):u64:order'
```





### 统计函数运行时长

``` shell
bpftrace -e 'kprobe:vfs_read /pid == 30153/ { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'
```

可参考bpftrace工具 `funclatency`的使用。

#### 其他debug方式

| 函数参数相关                                | 命令                                                         | 结果 |
| ------------------------------------------- | ------------------------------------------------------------ | ---- |
| 参数强转                                    | (uint32) a; (int64) b;                                       |      |
| 统计struct结构体大小                        | bpftrace -e 'hardware:bus-cycles:1 {printf("%u\n", (struct file *)0+1); }' |      |
| 统计struct某参数偏移                        | 直接编内核ko：printk("lru=%d\n", offsetof(struct page, lru)); |      |
|                                             |                                                              |      |
| **函数统计**                                | **命令**                                                     |      |
| 统计用户态的栈的前20行                      | bpftrace -e 'tracepoint:syscalls:sys_enter_recvfrom /pid==3406489/{ @[ustack(20)] = count(); }' |      |
|                                             |                                                              |      |
| **定时**                                    |                                                              |      |
| 统计30秒内cache-missed次数超过1000000的进程 | bpftrace -e 'hardware:cache-misses:1000000 { @[pid] = count(); } interval:s:30 { exit(); }' |      |
| 每秒打印一次结果                            | bpftrace -e 't:block:block_rq_i* {@[probe] = count();} interval:s:1 {print(@); clear(@)；}' |      |
|                                             |                                                              |      |
| **打印相关**                                |                                                              |      |
| 仅打印出现次数最多的5个函数                 | btftrace -e 'kprobe:vfs_* {@[probe]=count();} END{print(@，5)； clear(@);}'' |      |
| print时将ns转为s                            | print(@, 0, 1000000) // top=0表示全打印；1000000表示结果需要除以1000000. |      |
| 带条件过滤的，注意==前后必须要有空格        | bpftrace --unsafe -e 'uprobe:/usr/lib64/libc.so.6:__memcpy_falkor/comm == "dhry2reg"/ {printf("0x%x %p %u\n", arg0, arg1, arg2)}' |      |
|                                             |                                                              |      |
| **kprobes相关**                             |                                                              |      |
|                                             | kprobes过程中若有任何error，可以通过`cat tracing/error_log`来查看具体错误信息； 以及tail /var/log/messages查看pr_warn的打印信息。 |      |
|                                             |                                                              |      |
| **运行命令**                                |                                                              |      |
| 运行shell命令                               | bpftrace --unsafe -e 'BEGIN { printf("Hello Offensive BPF!\n");  system("whoami"); }' |      |
|                                             |                                                              |      |
|                                             |                                                              |      |








### 通过用户态函数跟踪

```bash

# objdump -tT test |grep main
0000000000000000       F *UND*	0000000000000000              __libc_start_main@@GLIBC_2.2.5
0000000000401126 g     F .text	0000000000000020              main
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 __libc_start_main
# 根据地址
$ bpftrace -e 'uprobe:/root/wrks/test:0x401126 { printf("test\n");}'
# 根据函数名
$ bpftrace -e 'uprobe:/root/wrks/test:main { printf("arg0: %d\n", arg0);}'

```



#### glibc函数调试

``` c
// 调用栈打印
bpftrace -e 'uprobe:/lib64/libc.so.6:__memmove_avx_unaligned_erms { printf("%s\n", ustack(bpftrace));}'
    
// 函数入参打印
bpftrace -e 'uprobe:/lib64/libc.so.6:__memmove_avx_unaligned_erms { printf("%d %d\n", arg1, arg2);}'    
    
// 有些函数看不到，需要添加 --unsafe才能显示。
bpftrace --unsafe -l 'uprobe:/usr/lib64/libc.so.6:*memcpy*'

```



## 调试/多用途



### trace

​	可以获取函数、函数**参数**、函数**返回值**，逐个打印。

``` c
./trace 'down_write "%llx" arg1'
```



### argdist

​	可以将函数、函数参数、函数返回值统计出来，并以出现频次或直方图的形式呈现。

​	-C： 统计频次。

​	-H： 统计直方图，2的幂次方。

​	注意下语法，函数名需要写完整，即包含函数参数类型及参数名等。

比如，跟踪tracepoint的read()函数参数buf的值，将char *转成u64输出：

​	./argdist -C 't:syscalls:sys_enter_read():u64:args->buf'

### funccount

​	统计函数调用次数。比如：

1. 各个系统调用的次数： /usr/share/bcc/tools/funccount 't:syscalls:sys_enter_*' -i 1

2. 所有vfs的操作次数：/usr/share/bcc/tools/funccount vfs_*

3. malloc的次数：/usr/share/bcc/tools/funccount 'c:malloc'

4. 用户态mutex调用次数：funccount c:pthread_mutex_lock

5. 频繁调用的字符串函数统计：funccount c:str* 

   可以追踪的函数包括：

   ​	内核态，tracepoint(t:开头)， lib库（lib库名开头，比如libc中的就是c:），用户态USDT(u:)，kprobe，自定义函数(path:，path为文件名路径)

### stackcount

​	统计函数栈调用次数，可以用来画火焰图，作用于perf probe类似。用法与funccount类似。比如：

1. 某个热点函数是被谁调用的： ./stackcount 'ktime_get'

### 	



## cpu

execsnoop, runqlat, runqlen, cpudist,profile, offcputime, syscount, softirq, hardirq

## IO

biolatency, biosnoop, biotop, bitesize

## memory

memleak

## 网络

tcpconnect, tcpaccept, tcplife, tcpretrans

### 文件系统

opensnoop, filelife, vfsstatt, fileslower, cachestat, writeback, dcstat, xfsslower, xfsdist, ext4dist

## 编程语言

javastat, javacalls,javathreads, javaflow, javagc

### 应用程序相关

mysqld_qslower, signals, killsnoop

### 内核

wakeuptime, offwaketime

### 安全

capable




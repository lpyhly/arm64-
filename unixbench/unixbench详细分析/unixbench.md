# unixbench

[TOC]

## 源码库

库的路径：https://github.com/kdlucas/byte-unixbench， 当前仍在继续维护。



# 竞分情况

详见onenote。

##1630V100样片

1630V100测试环境： 123#  openEuler 22.03， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

| // 单进程                             |                |                |          |
| ------------------------------------- | -------------- | -------------- | -------- |
| System Benchmarks Index Values        | BASELINE       | RESULT         | INDEX    |
| Dhrystone 2 using register variables  | 116700         | 72230453.1     | 6189.4   |
| Double-Precision Whetstone            | 55             | 8795.2         | 1599.1   |
| Execl Throughput                      | 43             | 6492.1         | 1509.8   |
| File Copy 1024 bufsize 2000 maxblocks | 3960           | 1107709        | 2797.2   |
| File Copy 256 bufsize 500 maxblocks   | 1655           | 323892         | 1957.1   |
| File Copy 4096 bufsize 8000 maxblocks | 5800           | 2718309        | 4686.7   |
| Pipe Throughput                       | 12440          | 1754576.2      | 1410.4   |
| Pipe-based Context Switching          | 4000           | 198375.4       | 495.9    |
| Process Creation                      | 126            | 11848.2        | 940.3    |
| Shell Scripts (1 concurrent)          | 42.4           | 10508.3        | 2478.4   |
| Shell Scripts (8 concurrent)          | 6              | 5013.5         | 8355.9   |
| System Call Overhead                  | 15000          | 1285256        | 856.8    |
|                                       |                |                | ======== |
| System Benchmarks Index Score         |                |                | 2014.8   |
|                                       |                |                |          |
| **// 160进程**                        | -------------- | -------------- | -----    |
| System Benchmarks Index Values        | BASELINE       | RESULT         | INDEX    |
| Dhrystone 2 using register variables  | 116700         | 11620071811    | 995721.7 |
| Double-Precision Whetstone            | 55             | 1407408.3      | 255892.4 |
| Execl Throughput                      | 43             | 28344          | 6591.6   |
| File Copy 1024 bufsize 2000 maxblocks | 3960           | 414924         | 1047.8   |
| File Copy 256 bufsize 500 maxblocks   | 1655           | 107748         | 651      |
| File Copy 4096 bufsize 8000 maxblocks | 5800           | 1480491        | 2552.6   |
| Pipe Throughput                       | 12440          | 279843929.6    | 224954.9 |
| Pipe-based Context Switching          | 4000           | 26669899.3     | 66674.7  |
| Process Creation                      | 126            | 82504.9        | 6548     |
| Shell Scripts (1 concurrent)          | 42.4           | 110575         | 26079    |
| Shell Scripts (8 concurrent)          | 6              | 13997.3        | 23328.8  |
| System Call Overhead                  | 15000          | 6419313.6      | 4279.5   |
|                                       |                |                | ======== |
| System Benchmarks Index Score         |                |                | 17357.3  |





## Intel 6248

`测试环境：118# Intel 6248，关SMT，cpu关turbo定频2.5GHz，ddr 2933MHz*12根*16GB， openEuler21.03，linux5.10.0_yhy`



## AMD 7742

`测试环境： 120# AMD 7742, 关SMT，关turbo，CPU频率2.23GHz定频，ddr 32根*2933MHz * 16GB  `



# 性能分析

## 影响因素剖析

### kpti

频繁的svc调用时，若开启kpti，则每次切换el等级时，需要切换内核页表与用户态页表，同时

chickenbit

### L3容量的影响

L3 裁way（28条，裁7条），测试1die 40c的性能。openEuler 22.03 sp1.

| System Benchmarks  Index Values        | 7M L3    | mfill.4 0xC605C304f0 0x7f     7M*3/4的L3 | 降低比例 |
| -------------------------------------- | -------- | ---------------------------------------- | -------- |
| Dhrystone 2 using register variables   | 257681.5 | 257592.6                                 | 100%     |
| Double-Precision Whetstone             | 66184.1  | 66183.6                                  | 100%     |
| Execl Throughput                       | 10978.8  | 11434.2                                  | 96%      |
| File Copy 1024 bufsize 2000  maxblocks | 3635.5   | 3816.9                                   | 95%      |
| File Copy 256 bufsize 500  maxblocks   | 2109.4   | 2339.2                                   | 90%      |
| File Copy 4096 bufsize 8000  maxblocks | 8079.7   | 9023.8                                   | 90%      |
| Pipe Throughput                        | 57768.1  | 57740.5                                  | 100%     |
| Pipe-based Context Switching           | 34515.6  | 34574.4                                  | 100%     |
| Process Creation                       | 10384.8  | 10337                                    | 100%     |
| Shell Scripts (1 concurrent)           | 37353.5  | 37344.1                                  | 100%     |
| Shell Scripts (8 concurrent)           | 37627.4  | 37599.8                                  | 100%     |
| System Call Overhead                   | 2205.8   | 2469.3                                   | 89%      |
|                                        | ======== | ========                                 | #VALUE!  |
| System Benchmarks Index Score          | 17351.4  | 17956.5                                  | 97%      |

### TLBI性能影响

手动在软件中添加tlbi时延，看看性能下降情况。

在内核中，`tlbi`指令最终调用接口仅在`tlbflush.h`中：

修改的patch如下：

```c

```

性能对比：

```c
Dhrystone 2 using register variables         116700.0   62848997.6   5385.5     
Double-Precision Whetstone                       55.0       7630.3   1387.3     
Execl Throughput                                 43.0       5443.4   1265.9     
File Copy 1024 bufsize 2000 maxblocks          3960.0    1197590.0   3024.2     
File Copy 256 bufsize 500 maxblocks            1655.0     350188.3   2115.9     
File Copy 4096 bufsize 8000 maxblocks          5800.0    2579607.7   4447.6     
Pipe Throughput                               12440.0    1611476.3   1295.4     
Pipe-based Context Switching                   4000.0     208375.5    520.9     
Process Creation                                126.0       9664.8    767.0     
Shell Scripts (1 concurrent)                     42.4       9617.2   2268.2     
Shell Scripts (8 concurrent)                      6.0       4075.9   6793.1     
System Call Overhead                          15000.0    1052480.6    701.7     
                                                                   ========     
System Benchmarks Index Score                                        1840.0     
```

###  进程绑核的影响

| v200,  2P开SMT，48C96T，所用核数：0-47,128-175 | 绑核收益 | 一一绑核 | 一一绑核 | 一一绑核 | 范围绑核 | 范围绑核 | 范围绑核 |
| ---------------------------------------------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- |
| unixbench-dhry2reg_cross_chip_near_96c_smt_hyp | 10%      | 2722.05  | 2729.07  | 2714.93  | 2459.02  | 2437.72  | 2514.42  |
| unixbench-syscall_cross_chip_near_96c_smt_hyp  | 11%      | 49.6788  | 49.6919  | 49.6783  | 44.9362  | 44.9682  | 44.7944  |
| unixbench-execl_cross_chip_near_96c_smt_hyp    | 5%       | 106.96   | 99.5597  | 98.6424  | 96.4059  | 96.3798  | 97.5629  |
| unixbench-file1024_cross_chip_near_96c_smt_hyp | 31%      | 22.639   | 16.1831  | 21.9436  | 15.3708  | 14.4647  | 16.7173  |
| unixbench-file256_cross_chip_near_96c_smt_hyp  | 68%      | 14.9226  | 16.154   | 15.6198  | 10.4907  | 8.58919  | 8.67573  |
| unixbench-file4096_cross_chip_near_96c_smt_hyp | 42%      | 56.2095  | 67.2265  | 60.4342  | 39.9153  | 46.6231  | 43.0648  |
| unixbench-pipe_cross_chip_near_96c_smt_hyp     | 6%       | 1101.08  | 1102.05  | 1100.56  | 1051.31  | 1027.01  | 1038.4   |
| unixbench-ctx_cross_chip_near_96c_smt_hyp      | 589%     | 767.171  | 765.602  | 766.436  | 124.513  | 102.827  | 106.274  |
| unixbench-spawn_cross_chip_near_96c_smt_hyp    | 29%      | 72.3795  | 81.4074  | 79.4081  | 60.0624  | 61.0861  | 59.0574  |
| unixbench-whets_cross_chip_near_96c_smt_hyp    | -3%      | 1101.96  | 1097.08  | 1096.95  | 1154.28  | 1127.02  | 1124.77  |
| unixbench-shell1_cross_chip_near_96c_smt_hyp   | -7%      | 166.274  | 173.511  | 169.752  | 182.704  | 182.365  | 182.454  |
| unixbench-shell8_cross_chip_near_96c_smt_hyp   | -35%     | 116.528  | 118.021  | 116.076  | 180.382  | 177.674  | 180.729  |





## 优化手段

### 内核安全特性配置

|                         |                                        |      |
| ----------------------- | -------------------------------------- | ---- |
| kpti=off                | 920B 上以后支持 E0PD，这个kpti自动不开 |      |
| mitigation=off          | 关闭所有安全漏洞相关的，确有效果的     |      |
| Randomize slab freelist | 默认开的安全选项，要重编译             |      |
| hardend usercopy        | 可以 CONFIG 关掉，需要重编译           |      |
| forced context tracking | 默认不开                               |      |
|                         |                                        |      |
|                         |                                        |      |
|                         |                                        |      |
|                         |                                        |      |

### others
userspace page fault，看这个影响 fork，当前 mysql、redis 等用例，没有频繁 fork
Missing CPU powersaving states，intel idle 节能相关，当前鲲鹏机器默认应该是 performance 模式

透明大页

### fault around

fault around 可以 通过 sysfs 修改，unixbench 的 execl 项时尝试改过，但是没有默认改，加载新程序pagefault 时有影响

![img](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5C55697381-885F-4240-BCEC-6594852D1144.png)



# 新版unixbench的那些变化

## 官方版本： 自5.1.3（2011年）后合入的有效修改如下

github： https://github.com/kdlucas/byte-unixbench

### 新增环境变量

1. UB_GCC_OPTIONS： 57847393139bd504d101c2652e1965b07b0d5561

   gcc的参数可以直接通过UB_GCC_OPTIONS传递，而不需要修改Makefile了。

   UB_TMPDIR

   UB_RESULTDIR

   UB_OUTPUT_FILE_NAME

   UB_OUTPUT_CSV: 可以以csv格式输出

### 编译选项

1. CFLAGS，LDFLAGS可以先自行指定部分，然后Makefile中再+=另一部分。 	aeed2ba662a9220089aee33be4123481dab0b524

2. 默认编译选项修改为O3，并添加mtune。 c22488d72c533a41cbc756806e1dad0c74e38047

3. 旧的默认选项：-O2       -fomit-frame-pointer -fforce-addr -ffast-math -Wall -pedantic
  
   新的默认选项： -O3       -ffast-math -pedantic. linux系统的话，再增加 -march=native -mtune=native
   
   删除了-ansi的编译选项，从而让gcc发挥更好的效果。

### Run文件

1. 1. 核数检测由/proc/cpuinfo修改为只检测active的核数。 c49ad96cae0e067034350309b611ed798834cdda

   2. 处理可能出现的log(0)问题。cada60619c

   3. 寻找Bin路径时使用FindBin函数库帮助，而不是直接指定。 e0b8c00209e

### ctx

1. 解决pipe管道通信中有一方意外退出，导致另外一方报Broken pipe错误的问题。 088ceb2

- 方式：对read/write的返回值的errno进行SIGPIPE的判断，若法穿一方意外退出，则另一方应该受到SIGPIPE信号。若收到，则忽略SIGPIPE信号，转而发送一个SIGALRM的计数器到时信号给父进程，令其按照正常结束来处理。

### whetstone

1. 修改计时方式：使用gettimeofday代替getrusage计时，从而统计的是wall time而不是进程的运行时间。更精准。这个只是对于进程数>核数的case有修复意义。618e0c074fd1

### arith
1. 修改iter变量为volatile，防止被优化。379a68e4b199b

 

## aliyun版本：v5.1.6 （2021年1月）

https://github.com/aliyun/byte-unixbench

### syscall

1. 使用syscall(SYS_getpid)替换以前的getpid()，从而越过glibc确保每次都能够直接调用svc，对于glibc2.3 - 2.24版本存在的cached PIDs情况进行了规避。glibc2.25以后就没有影响了。

### whets

对于whestone中的N8测试项，随着迭代次数的增加，运算难度也迅速增加。而迭代次数与cpu主频正相关。

``` c
x = 0.75;
timea = dtime();
 {
    for (ix=0; ix<xtra; ix++) // xtra是动态训练出来的，cpu主频越高xtra越高。
      {
+   x = 0.75;            // 这个patch将x固定，从而保证无论xtra是多少，都不会影响运算的难度。
    for(i=0; i<n8; i++)
      {
         x = sqrt(exp(log(x)/t1));
      }
      }
 }
timeb = dtime()-timea;
```

- 在一台高主频的vm，做单线程运算的耗时统计如下：

| 运算量 | 耗时  |
| ------ | ----- |
| 1000   | 0.820 |
| 2000   | 1.509 |
| 3000   | 2.216 |
| 4000   | 2.909 |
| 5000   | 6.527 |

也就是说，高主频的cpu做的次数更多，但耗时主要出现在后面一段，那么相比低主频的cpu来说，最终的得分就不会与cpu频率成线性关系了。

### context1

1. 将父子进程绑到不同核。

   - 所有的父进程绑在前一半核，所有子进程绑在后一半核。

   - 为了修复内核在使用wake_wide机制之后，与修复之前的性能差异。使用wake_wide之前fork后父子进程会在同一个核上，wake_wide机制则会令父子进程在不同核上。

     ```c
                 (no using wake_wide) kernel 3.10  score 977
                 (using wake_wide)    kernel 4.9   score 227
     ```

   - 合入这个patch后，ctx性能在低版本内核会下降。
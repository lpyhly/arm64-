[TOC]

# 测试代码行为分析

## 多核波动性非常大

涉及到进程调度，所以波动性非常大，在小用例/原始用例，4k/64k页表下都试过。

- 在鲲鹏波动异常大，但是在x86下波动就可控。
- 单核性能基本稳定，多进程就大幅波动。

|                                                    | 测试1   | 测试2   | 测试3   | 测试4   |
| -------------------------------------------------- | ------- | ------- | ------- | ------- |
| *64k页表下的EMU小用例， 1630v100样片**单进程**绑核 | 1059.15 | 1067.63 | 1060.65 | 1054.76 |
| 64k页表下的EMU小用例， 1630v100样片 160核关SMT     | 17324.8 | 23428.8 | 21505.6 | 74033.6 |
| 4k页表下的原始用例， 1630v100样片 单P80核关SMT     | 66510.1 | 39145.1 | 39510   | 40383.2 |
| Intel 6248原始用例，2P40核关SMT定频                | 13772   | 10876.1 | 12044.6 | 13576.7 |
| AMD 7742原始用例，2P 128核关SMT定频                | 48274.4 | 50309.5 | 51520.3 | 48512.0 |



# 4k页表性能分析

## 单核

## 多核

1620上测试4k页表的128c的ctx：

``` shell
带宽： 共44G，读33GB，写10GB（理论读带宽333GB，仅20%不到）
                    Start to monitor The System DDR Bandwidth
######################################        ######################################
|        CPU0 Bandwidth (MB/s)       |        |        CPU1 Bandwidth (MB/s)       |
######################################        ######################################
|         Die0          Die1         |        |         Die0          Die1         |
--------------------------------------        --------------------------------------
| RD:    8297.23       5509.65       |        | RD:   11747.09       8260.31       |
--------------------------------------        --------------------------------------
| WR:    2187.84       1174.85       |        | WR:    4114.37       2555.24       |
--------------------------------------        --------------------------------------
|   CPU0 Total Bandwidth: 17169.57   |        |   CPU1 Total Bandwidth: 26677.01   |
####################################################################################
|                   System Total Read Bandwidth:   33814.28                        |
------------------------------------------------------------------------------------
|                   System Total Write Bandwidth:   10032.30                       |
------------------------------------------------------------------------------------
|                     System Total Bandwidth:   43846.58                           |
####################################################################################
```





# 64k页表性能分析

## 单核

### 波形整体情况

1630V200的1c波形来看： 300000/128493=2.3348 us

![image-20221205171231424](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205171231424.png)

 波形上看，还算稳定。

一次loop包括： 父写子读+子写父读，因此有4次svc调用。时间分解(ns)：

**982**/275/608/216/

**968**/278/610/218/

**983**/274/615

![image-20221205171308928](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205171308928.png)

 

父进程： 0x400a**e8**(write) + 0x400b**0c**(read)

子进程：0x400a**4c**（read) + 0x400a**7c**(write)

但是，从commit的情况看，呈现规律性的两个write先commit，然后是两个read。

- 父write上半段->（进程切换） -> 子read下半段 -> （返回用户态） -> 子write完成 -> 子read上半段 ->（没数据，发生进程切换）

- -> 父write下半段->（返回用户态） -> 父read完成（已有数据，可以顺利完成） -> 父write上半段

-  因此，在el0呈现的就是pw->cw->cr->pr, p:parent, c:children





逻辑是：

| 行为主体     | 具体操作                                                     |
| ------------ | ------------------------------------------------------------ |
| cr           | fork后先进入子进程（这个不一定，原理上是一样的），read(p1)，因为没有writer所以必须死等，这里有一个调度点，会发生进程切换； |
| pw  pr       | 执行父进程的write(p1)，内核中会发现此时已经有reader在等待了，因此可以直接copy到reader那边，copy后会置signal表示需要唤醒子进程去接收数据了；然后父进程的write()返回用户态继续执行read(p2)，同理因为没有writer所以必须死等，会发生**进程切换**； |
| cr(7c  4c  ) | 子进程完成read(p1)，然后继续执行write(p2)，write在内核中没有看到有在等待的reader，所以直接运行完返回用户态，while循环继续下一轮，调用read(p1)。 |

#### 一次没有进程切换的write操作：

​		（返回用户态） -> 子write完成

![image-20221205172239678](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172239678.png)

#### 一次有reader的write操作：

​		父write上半段->（进程切换） -> 子read下半段 

1. 先到pipe_write，做copy_page（同上）

2. 发现reader，表示需要进程切换，因此可以做切换的准备工作了。这里，从__wake_up_common函数开始与上面不同。	__
3. __update各个进程时间片，重新排优先级等	
4. 算完后返回pipe_write函数，这里已经把need_resched置位了。	
5. 返回到了__wake_up_common函数，这里与上面是一致的，直到ret_to_user	
6. 在这里有个调度点，会发生真正的进程切换。	
7. 进程切换完毕，前一个进程此时在做pipe_read	
8. 此时已经有数据了，所以pipe_read可以继续完成了	
9. ret_to_user没有进程要切换，可以顺利返回用户态，结束read()的调用。

![image-20221205172125571](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172125571.png)



![image-20221205172153499](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172153499.png)

#### 一次有writer的read操作：

![image-20221205172319675](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172319675.png)

#### 一次没有writer的read操作： 

子read上半段 ->（没数据，发生进程切换）

![image-20221205172403126](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172403126.png)

![image-20221205172413922](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172413922.png)



### 函数级分析

#### 单次循环的函数占比统计

![image-20221205172522829](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172522829.png)

调度准备：

__wake_up_common（47109888825.8） -> __wake_up_common(47109889074.2)

  

调度过程：

return_to_user(9141.88)->schedule(9162.23) -> __switch_to(9343.48)->__switch_to(9412.59) ->schedule(9434.2) 

  

read的调度过程：

schedule(90160) ->__switch_to(90411)->__switch_to(90483.8)->schedule(90504)



#### 单板上测试（热点都在关中断中，因此看不到真正热点）

在44#上测试，单核单进程的热点图如下： `92% sys + 8% usr`

``` c
+   29.79%  context1  [kernel.vmlinux]  [k] finish_task_switch                  
+   21.51%  context1  [kernel.vmlinux]  [k] __wake_up_common_lock                
+    4.67%  context1  [kernel.vmlinux]  [k] el0_svc_handler                     
+    4.00%  context1  libc-2.29.so      [.] __GI___libc_write                   
+    2.83%  context1  libc-2.29.so      [.] read                                
+    2.65%  context1  [kernel.vmlinux]  [k] ktime_get_coarse_real_ts64          
```

在44#上测试，64c的热点图如下：

```c
+   43.41%  context1         [kernel.vmlinux]  [k] __wake_up_common_lock // 热点在开中断……
+   34.36%  context1         [kernel.vmlinux]  [k] finish_task_switch  // 热点也在开中断……
+    7.13%  swapper          [kernel.vmlinux]  [k] finish_task_switch
+    2.19%  swapper          [kernel.vmlinux]  [k] arch_cpu_idle           
```
#####  __wake_up_common_lock

调用关系：

占大头的：`write->el0_svc->vfs_write->pipe_write->__wake_up_sync_key->__wake_up_common_lock`

占小头的：`read ->el0_svc->vfs_read ->pipe_read->__wake_up_sync_key->__wake_up_common_lock `

##### finish_task_switch

调用关系： read ->el0_svc->vfs_read ->pipe_read->pipe_wait->schedule->__sched_text_start->finish_task_switch

### 指令级分析

top 时延(1630V200)

![image-20221205172705069](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172705069.png)

### topdown级分析

``` shell
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                    12568876586
        Total Execution Cycles:                          8682135681
        Instructions Per Cycle:                               1.448

        Front-End Bound:                                    13.796%
                Front-End Latency:                           8.555%
                        Idle_by_itlb_miss:                   0.349%
                        Idle_by_icache_miss:                39.227%
                        BP_Misp_Flush:                       0.767%
                        OoO_Flush:                           2.336%
                        SP_Flush:                            0.615%
                Front End Bound Bandwidth:                   5.241%

        Bad Speculation:                                     2.541%
                Branch Mispredicts:                          0.944%
                        Indirect Branch:                     5.335%
                        Push Branch:                         0.148%
                        Pop Branch:                          0.052%
                        Other Branch:                      100.000%
                Machine Clears:                              1.597%
                        Nuke Flush:                          0.000%
                        Other Flush:                       100.000%

        Retiring:                                           24.128%

        Back-End Bound:                                     59.536%
                Resource Bound:                              1.806%
                        Rob_stall:                           0.010%
                        Ptag_stall:                          0.000%
                        MapQ_stall:                         99.988%
                        PCBuf_stall:                         0.001%
                        Other_stall:                        -0.000%
                Core Bound:                                 78.673%
                        FDIV_Stall:                          0.000%
                        DIV_Stall:                           0.000%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                     78.673%
                                0_ports_serialize:          33.628%
                                0_ports_non_serialize:      12.811%
                                1_ports:                    16.421%
                                2_ports:                    11.521%
                                3_ports:                    10.212%
                                4_ports:                     7.381%
                                5_ports:                     4.638%
                                6_ports:                     3.442%
                Memory Bound:                               16.133%
                        L1 Bound:                           15.967%
                                DTLB:                        0.260%
                                Misalign:                    0.533%
                                Resource_Full:               0.000%
                                Instruction_Type:            0.609%
                                Forward_hazard:              8.704%
                                Structure_hazard:            1.019%
                                Pipeline:                    0.072%
                        L2 Bound:                            0.165%
                                buffer pending:              0.000%
                                snoop pending:               0.139%
                                Arb idle:                    0.157%
                                Pipeline:                   -0.131%
                        L3 Bound:                            0.001%
                        Mem Bound:                           0.000%
                                Local Dram:                  0.000%
                                Remote Dram:                 0.000%
                                Remote Cache:                0.000%
                        Store Bound:                         0.000%
                                SCA:                         6.584%
                                Head:                        1.439%
                                Order:                     106.076%
                                Other:                       0.002%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++                                
     2.001560470           1,000.50 msec cpu-clock                 #    1.001 CPUs utilized
     2.001560470            614,761      context-switches          #    0.614 M/sec
     2.001560470                  1      cpu-migrations            #    0.001 K/sec
     2.001560470                  0      page-faults               #    0.000 K/sec
     2.001560470      2,898,729,501      cycles                    #    2.897 GHz                      (69.17%)
     2.001560470      4,203,591,925      instructions              #    1.45  insn per cycle           (68.77%)
     2.001560470    <not supported>      branches
     2.001560470          4,514,703      branch-misses                                                 (68.37%)
     2.001560470      1,340,138,873      L1-dcache-loads           # 1339.465 M/sec                    (68.41%)
     2.001560470          1,234,679      L1-dcache-load-misses     #    0.09% of all L1-dcache accesses  (68.81%)
     2.001560470              8,022      LLC-loads                 #    0.008 M/sec                    (69.21%)
     2.001560470                953      LLC-load-misses           #   11.88% of all LL-cache accesses  (69.61%)
     2.001560470        729,026,185      L1-icache-loads           #  728.660 M/sec                    (69.61%)
     2.001560470         77,778,734      L1-icache-load-misses     #   10.67% of all L1-icache accesses  (69.62%)
     2.001560470      1,379,449,893      dTLB-loads                # 1378.756 M/sec                    (62.02%)
     2.001560470            317,281      dTLB-load-misses          #    0.02% of all dTLB cache accesses  (62.02%)
     2.001560470        729,172,577      iTLB-loads                #  728.806 M/sec                    (61.97%)
     2.001560470                 22      iTLB-load-misses          #    0.00% of all iTLB cache accesses  (61.57%)
     2.001560470    <not supported>      L1-dcache-prefetches
     2.001560470    <not supported>      L1-dcache-prefetch-misses

```



## 多核

1630V200的emu上测试结果：

![image-20221205172911527](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205172911527.png)



### topdown分析

``` shell
# 1630v100样片_emu_unixbench_32c
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                   410884008775
        Total Execution Cycles:                        276372950646
        Instructions Per Cycle:                               1.487

        Front-End Bound:                                    12.959%
                Front-End Latency:                           7.963%
                        Idle_by_itlb_miss:                   0.531%
                        Idle_by_icache_miss:                40.484%
                        BP_Misp_Flush:                       0.629%
                        OoO_Flush:                           2.040%
                        SP_Flush:                            0.520%
                Front End Bound Bandwidth:                   4.996%

        Bad Speculation:                                     2.240%
                Branch Mispredicts:                          0.800%
                        Indirect Branch:                     4.006%
                        Push Branch:                         0.108%
                        Pop Branch:                          0.040%
                        Other Branch:                      100.000%
                Machine Clears:                              1.441%
                        Nuke Flush:                          0.685%
                        Other Flush:                        99.315%

        Retiring:                                           24.778%

        Back-End Bound:                                     60.022%
                Resource Bound:                              5.914%
                        Rob_stall:                           0.046%
                        Ptag_stall:                          0.000%
                        MapQ_stall:                         99.774%
                        PCBuf_stall:                         0.180%
                        Other_stall:                         0.000%
                Core Bound:                                 71.544%
                        FDIV_Stall:                          0.000%
                        DIV_Stall:                           0.000%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                     71.544%
                                0_ports_serialize:          34.935%
                                0_ports_non_serialize:      11.627%
                                1_ports:                    15.662%
                                2_ports:                    11.453%
                                3_ports:                    10.371%
                                4_ports:                     7.521%
                                5_ports:                     4.968%
                                6_ports:                     3.518%
                Memory Bound:                               19.082%
                        L1 Bound:                           17.391%
                                DTLB:                        0.253%
                                Misalign:                    0.467%
                                Resource_Full:               0.000%
                                Instruction_Type:            0.530%
                                Forward_hazard:              7.649%
                                Structure_hazard:            1.610%
                                Pipeline:                    0.167%
                        L2 Bound:                            0.342%
                                buffer pending:              0.000%
                                snoop pending:               0.085%
                                Arb idle:                    0.285%
                                Pipeline:                   -0.028%
                        L3 Bound:                            1.349%
                        Mem Bound:                           0.000%
                                Local Dram:                  0.000%
                                Remote Dram:                 0.000%
                                Remote Cache:                0.000%
                        Store Bound:                         0.000%
                                SCA:                         6.210%
                                Head:                        1.229%
                                Order:                     114.955%
                                Other:                       0.002%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

```

``` shell
# 1630v100样片_emu_unixbench_40c
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                   481786183375
        Total Execution Cycles:                        345009667408
        Instructions Per Cycle:                               1.396

        Front-End Bound:                                    12.125%
                Front-End Latency:                           7.459%
                        Idle_by_itlb_miss:                   0.466%
                        Idle_by_icache_miss:                37.051%
                        BP_Misp_Flush:                       0.609%
                        OoO_Flush:                           1.911%
                        SP_Flush:                            0.513%
                Front End Bound Bandwidth:                   4.666%

        Bad Speculation:                                     2.164%
                Branch Mispredicts:                          0.789%
                        Indirect Branch:                     3.680%
                        Push Branch:                         0.214%
                        Pop Branch:                          0.037%
                        Other Branch:                      100.000%
                Machine Clears:                              1.375%
                        Nuke Flush:                          1.256%
                        Other Flush:                        98.744%

        Retiring:                                           23.274%

        Back-End Bound:                                     62.437%
                Resource Bound:                             10.817%
                        Rob_stall:                           0.050%
                        Ptag_stall:                          0.001%
                        MapQ_stall:                         99.649%
                        PCBuf_stall:                         0.301%
                        Other_stall:                        -0.000%
                Core Bound:                                 62.149%
                        FDIV_Stall:                          0.000%
                        DIV_Stall:                           0.000%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                     62.148%
                                0_ports_serialize:          38.721%
                                0_ports_non_serialize:      10.937%
                                1_ports:                    14.821%
                                2_ports:                    10.790%
                                3_ports:                     9.761%
                                4_ports:                     7.092%
                                5_ports:                     4.631%
                                6_ports:                     3.308%
                Memory Bound:                               23.785%
                        L1 Bound:                           20.493%
                                DTLB:                        0.230%
                                Misalign:                    0.434%
                                Resource_Full:               0.000%
                                Instruction_Type:            0.492%
                                Forward_hazard:              7.156%
                                Structure_hazard:            1.593%
                                Pipeline:                    0.197%
                        L2 Bound:                            0.373%
                                buffer pending:              0.000%
                                snoop pending:               0.081%
                                Arb idle:                    0.309%
                                Pipeline:                   -0.017%
                        L3 Bound:                            2.919%
                        Mem Bound:                           0.000%
                                Local Dram:                  0.000%
                                Remote Dram:                 0.000%
                                Remote Cache:                0.000%
                        Store Bound:                         0.000%
                                SCA:                         5.763%
                                Head:                        1.143%
                                Order:                     120.354%
                                Other:                       0.001%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
     2.001340620           1,000.29 msec cpu-clock                 #    1.000 CPUs utilized
     2.001340620            531,363      context-switches          #    0.531 M/sec
     2.001340620                  1      cpu-migrations            #    0.001 K/sec
     2.001340620                  0      page-faults               #    0.000 K/sec
     2.001340620      2,898,111,652      cycles                    #    2.897 GHz                      (69.18%)
     2.001340620      4,260,866,205      instructions              #    1.47  insn per cycle           (68.78%)
     2.001340620    <not supported>      branches
     2.001340620          3,625,580      branch-misses                                                 (68.38%)
     2.001340620      1,340,843,499      L1-dcache-loads           # 1340.454 M/sec                    (68.41%)
     2.001340620          8,964,595      L1-dcache-load-misses     #    0.67% of all L1-dcache accesses  (68.81%)
     2.001340620          3,570,354      LLC-loads                 #    3.569 M/sec                    (69.21%)
     2.001340620                665      LLC-load-misses           #    0.02% of all LL-cache accesses  (69.61%)
     2.001340620        714,592,023      L1-icache-loads           #  714.384 M/sec                    (69.61%)
     2.001340620         76,967,439      L1-icache-load-misses     #   10.77% of all L1-icache accesses  (69.61%)
     2.001340620      1,377,290,968      dTLB-loads                # 1376.891 M/sec                    (62.01%)
     2.001340620            274,065      dTLB-load-misses          #    0.02% of all dTLB cache accesses  (62.01%)
     2.001340620        714,556,751      iTLB-loads                #  714.349 M/sec                    (61.98%)
     2.001340620                 30      iTLB-load-misses          #    0.00% of all iTLB cache accesses  (61.58%)
     2.001340620    <not supported>      L1-dcache-prefetches
     2.001340620    <not supported>      L1-dcache-prefetch-misses

```



### 函数级分析

![image-20221205202954449](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205202954449.png)

​		





### 指令级分析

![image-20221226103616555](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221226103616555.png)

长时延有上千ns，主要分布为：

1. `__cpu_do_idle`函数中的`wfi`指令（`ret`的前一条）

   ``` c
   ffff800010c80ed0 <__cpu_do_idle>:
   ffff800010c80ed0:       d5033f9f        dsb     sy
   ffff800010c80ed4:       d503207f        wfi
   ffff800010c80ed8:       d65f03c0        ret
   ffff800010c80edc:       d503201f        nop
   ```

2. `queued_spin_lock_slowpath`中的`wfe`指令（`ldar`的前一条）

   ``` c
   ffff8000100d037c:       4a000021        eor     w1, w1, w0
   ffff8000100d0380:       35000041        cbnz    w1, ffff8000100d0388 <queued_spin_lock_slowpath+0x118>
   ffff8000100d0384:       d503205f        wfe
   ffff8000100d0388:       88dffc60        ldar    w0, [x3]
   ffff8000100d038c:       b90013e0        str     w0, [sp, #16]
   ffff8000100d0390:       72001c1f        tst     w0, #0xff
   ffff8000100d0394:       54fffec1        b.ne    ffff8000100d036c <queued_spin_lock_slowpath+0xfc>  // b.any
   ```

3. `update_cfs_group`中的`ldr`操作。单次跨P的rs读耗时1700ns，从数据流上看到有大量的同地址rs/atomic store操作在发送，没有true share/false share。

   ```c
   ffff8000100b6628:       d2800043        mov     x3, #0x2                        // #2
   ffff8000100b662c:       eb0300bf        cmp     x5, x3
   ffff8000100b6630:       9a8320a5        csel    x5, x5, x3, cs  // cs = hs, nlast
   ffff8000100b6634:       f9405083        ldr     x3, [x4, #160]
   ffff8000100b6638:       f9408000        ldr     x0, [x0, #256] // 热点1
   ffff8000100b663c:       eb05007f        cmp     x3, x5
   ffff8000100b6640:       f9408084        ldr     x4, [x4, #256]  // 热点2
   ffff8000100b6644:       9a852063        csel    x3, x3, x5, cs  // cs = hs, nlast
   ffff8000100b6648:       cb040000        sub     x0, x0, x4
   ffff8000100b664c:       ab000060        adds    x0, x3, x0
   ffff8000100b6650:       9b037c43        mul     x3, x2, x3
   ffff8000100b6654:       54000040        b.eq    ffff8000100b665c <update_cfs_group+0x5c>  // b.none
   ```

![image-20221205194636525](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221205194636525.png)

其中，x0是cfs_rq， x4是cfs_rq->tg(cfs_rq偏移128#取指）。



# 源码解读

## read

pipe_read -> pipe_wait

### pipe_wait

pipe_read -> pipe_wait

在4.19内核中，context-switch的性能很大程度上与pipe_wait函数相关。

- 但是在新内核中，该函数被删掉了，具体原因及影响待研究。

``` c
/* Drop the inode semaphore and wait for a pipe event, atomically */
void pipe_wait(struct pipe_inode_info *pipe)
{
    DEFINE_WAIT(wait);

    /*
     * Pipes are system-local resources, so sleeping on them
     * is considered a noninteractive wait:
     */
    	//
    prepare_to_wait(&pipe->wait, &wait, TASK_INTERRUPTIBLE);
    	// 会关中断操作
    pipe_unlock(pipe); // 释放锁
    schedule();
    	// 陷入调度，直到当前任务再次被唤醒。
    finish_wait(&pipe->wait, &wait);
    pipe_lock(pipe); // 会调用mutex抢锁，抢不到则一直继续。
}

```

- | daif状态 | 动作                  | 说明                   |
  | -------- | --------------------- | ---------------------- |
  | 0        |                       | 进入handler处理函数    |
  | 2        | prepare_to_wait       | pipe_wait()中调用的    |
  | 0        | prepare_to_wait       |                        |
  | 2        | __schedule            | 开始进行任务切换       |
  | ->0      | finish_context_switch | 切换完成 ->finish_wait |
  | ->2      | __wake_up_common_lock |                        |
  | ->0      | __wake_up_common_lock |                        |
  | ->f      | do_el0_svc            |                        |

其中的pipe_lock：

``` c
pipe_lock -> pipe_lock_nested -> mutex_lock_nested->__mutex_lock->__mutex_lock_common
```



# 影响因素
## 虚拟机：halt polling

- 内核参数调整：打开halt polling，可以让Pipe-based Context Switching性能提升一倍多，但是对应功耗也显著提升，默认云厂商好像都不打开该选项。
- halt polling: 针对虚拟机设计的，当vcpu发出wfi时，host会先polling一段时间，看看是否有IRQ来到当前vm，若一段时间内都没有，再跳出到host去执行schedule调度。

## audit

pipe管道上存在系统调用接口read()  write()，因此audit_syscall会影响性能。

从热点图中可以看到，`ktime_get_coarse_real_ts64`产生的途径有2个，一是在audit syscalll的时候需要记录时间，另外是在pipe管道读写的时候，需要更新file的atime和mtime.

``` c
- ktime_get_coarse_real_ts64                
   - __audit_syscall_entry           
        syscall_trace_enter                 
        el0_svc_handler                     
      - el0_svc                             
         +  __GI___libc_write          
         +  read                                        
   -  current_time                    
      -  file_update_time              
           pipe_write                                  
           vfs_write                        
           el0_svc                          
           __GI___libc_write                                         
      -  atime_needs_update            
           touch_atime                      
           pipe_read                        
           vfs_read                         
           el0_svc                          
           read                                                      
```

## chicken bit

关chicken bit提升会比较大，1620上实测20核从300+提升到600+分

## context tracking force

如果内核中开了context tracking force，则性能下降会非常明显。当然，如果只开了context_tracking，则没有明显影响。

## 线性度

x86 2p&1p线性度好；我们2p比1p还差。

1p分数比x86高；2p分数比x86低；

## 调度

每个进程会fork一个子进程，然后两个进程之间通过pipe管道互相读写。因此若启动20份则意味着共有40个进程。

### 实验1

在1620上实测，进程是有可能发生schedule的，但是多数情况下还是会被均匀分布在各个核上，每个核上会分配2个进程，并且这两个进程刚好是一对父子进程。

### 实验2

在1620上实测，使用64个核（c0-c63，不跨片）跑64组进程时，进程间可能存在分数不均衡。选择分数为236（COUNT 283463次）的进程组以及分数为104（COUNT 124641次）的进程组进行对比。

| 完成次数 | 父进程调度的cpu   | 子进程调度的cpu |      |      |      |
| -------- | ----------------- | --------------- | ---- | ---- | ---- |
| 283463   | 282000次调度在c50 | 281530调度在c50 |      |      |      |
| 124641   | 123650次调度在c59 | 123420调度在c59 |      |      |      |

由此可以看到，性能差异与核间调度无关，需要寻找其他可能性。

- 发现numa0的sys为98%， 但numa1的sys只有75%+；且随着时间变化，负责会有差别；一段时间后，numa0的sys为92%， 但numa1的sys为95%；一段时间后，numa0的sys为86%， 但numa1的sys为96%……
  - 怀疑存在共享变量并且有抢锁的行为。

### 实验3

在1620和6248上实测，1P多核不绑核跑的时候，调度次数基本一致，最大调度时延来看，1620很高，调度存在不均衡现象；（均为openEuler 22.03 4kpagesize版本）

|  Task（context1）       |   Runtime ms  | Switches | Avg delay ms    | Max delay ms    |
| -------- | ----------------- | --------------- | ---- | ---- |
|  AMD 2P128c（48291分）  | 108352.072 ms |     6992 | 0.555 ms | 280.612 ms |
| AMD 1P64c | 4937.344 ms | 138 | avg:   0.178 ms | max:  12.448 ms |
| 1620 2P128c（10146分） | 120590.556 ms | 728082 | avg:   0.041 ms | max: 183.722 ms |
| 1620 1P64c | 30441.727 ms | 2321 | avg:   0.062 ms | max:  62.043 ms |
| Intel 6248 2P40c | 107757.683 ms | 15940 | avg:   0.018 ms | max:  16.047 ms |

AMD、1620已检查过sched配置，是一致的。

	kernel.sched_latency_ns = 24000000
	kernel.sched_migration_cost_ns = 5000000
	kernel.sched_min_granularity_ns = 10000000
	kernel.sched_nr_migrate = 32
	kernel.sched_rr_timeslice_ms = 100
	kernel.sched_rt_period_us = 1000000
	kernel.sched_rt_runtime_us = 950000
	kernel.sched_schedstats = 0
	kernel.sched_tunable_scaling = 1
	kernel.sched_wakeup_granularity_ns = 15000000


###  不同内核下的调度策略



## 软件行为

两个管道，分别为父写子读，子写父读。

check_and_switch_context中存在cas锁操作。

``` c
__schedule
    -> context_switch
		->switch_mm 	// 用户进程地址空间的转换
            -> __switch_mm
                -> check_and_switch_context

void check_and_switch_context(struct mm_struct *mm, unsigned int cpu)
 {
     unsigned long flags;
     u64 asid, old_active_asid;

     if (system_supports_cnp())
         cpu_set_reserved_ttbr0();

     asid = atomic64_read(&mm->context.id);

     /*
      * The memory ordering here is subtle.
      * If our active_asids is non-zero and the ASID matches the current
      * generation, then we update the active_asids entry with a relaxed
      * cmpxchg. Racing with a concurrent rollover means that either:
      *
      * - We get a zero back from the cmpxchg and end up waiting on the
      *   lock. Taking the lock synchronises with the rollover and so
      *   we are forced to see the updated generation.
      *
      * - We get a valid ASID back from the cmpxchg, which means the
      *   relaxed xchg in flush_context will treat us as reserved
      *   because atomic RmWs are totally ordered for a given location.
      */
     old_active_asid = atomic64_read(&per_cpu(active_asids, cpu));
     if (old_active_asid &&
         !((asid ^ atomic64_read(&asid_generation)) >> asid_bits) &&
         atomic64_cmpxchg_relaxed(&per_cpu(active_asids, cpu),
                      old_active_asid, asid))
         goto switch_mm_fastpath;

     raw_spin_lock_irqsave(&cpu_asid_lock, flags);
     /* Check that our ASID belongs to the current generation. */
     asid = atomic64_read(&mm->context.id);
     if ((asid ^ atomic64_read(&asid_generation)) >> asid_bits) {
         asid = new_context(mm);
         atomic64_set(&mm->context.id, asid);
     }

     if (cpumask_test_and_clear_cpu(cpu, &tlb_flush_pending))
         local_flush_tlb_all();

     atomic64_set(&per_cpu(active_asids, cpu), asid);
     raw_spin_unlock_irqrestore(&cpu_asid_lock, flags);

 switch_mm_fastpath:

     arm64_apply_bp_hardening();

     /*
      * Defer TTBR0_EL1 setting for user threads to uaccess_enable() when
      * emulating PAN.
      */
     if (!system_uses_ttbr0_pan())
         cpu_switch_mm(mm->pgd, mm);
 }
```


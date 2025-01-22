# ub-shell

[TOC]



## 影响因素

1620 44#上check（linux 5.10.0-4.17.0.28.oe1.aarch64, openEuler21.03, 64k pagesize)， 单核绑核shell1测试3s，结果稳定。

|                        | 测试方式                                                     | 分数                   |
| ---------------------- | ------------------------------------------------------------ | ---------------------- |
| A0                     | 虚拟机内绑核，使用minifs的image &  fs                        | 2443                   |
| A1 内核影响            | 虚拟机内绑核，使用44#的vmlinux（5.10.0-4.17.0.28.oe1.aarch64 64k，21.03） + minifs的fs | 2240                   |
| A2 虚拟机无影响        | 44#host上chroot跑，使用LD_LIBRARY_PATH和PATH                 | 2216，性能基本同A1     |
|                        |                                                              |                        |
| B0                     | 44#host上直接跑unixbench（/bin/sh为静态编译版本）            | 1382                   |
| B1 LD_LIBRARY_PATH影响 | B0 + 使用minifs的tools下的工具（包含的是awk,grep,wc)  LD_LIBRARY_PATH  + PATH | 1349                   |
| B2 静态编译影响        | B1 + 将minifs的/usr/bin下的tee,rm,sh,od.sort（均为静态编译）全部copy到tools下 | 2000                   |
| B3  multibyte编码影响  | B2 +  LC_ALL=POSIX配置  LC_ALL=POSIX  LD_LIBRARY_PATH=/home/hly/cocobean/src/emu/initrd/tools/lib64   PATH=/home/hly/cocobean/src/emu/initrd/tools/bin:$PATH  bash run_1c.sh | 2198，基本可以比肩A2了 |

 综上，影响性能的因素有：

1. 动态链接库的库文件个数，包含两部分，一是/usr/lib64:/usr/lib等固有文件夹内的文件数量，二是LD_LIBRARH_PATH指定的路径下的文件数量。
2. 使用到的tee/rm/sh/od/sort/awk/grep/wc的编译版本，静态编译比动态编译好；
3. /bin/sh的版本，静态比动态好；
4. LC编码影响，POSIX即C语言，使用的是固定宽度编码，性能比UTF-8要高。UTF-8目前有些优化的patch，但无法合入主线。
5. 内核影响：部分系统调用、track、tracing、debug等会影响性能。



### 内核版本 - osq_lock

96核shell scripts性能差

内核版本有关系： shell1 96核：

linux 4.19 - 13000分

linux 5.3 - 22000分 （no audit, 64k pagesize）

5.3的热点在osq_lock - 13%，打了patch后提升到24000分。

在229#上的测试结果：shell1/shell8 = 24000/19000，已经与intel 6248持平。



### 绑核 

./Run shell1  --- 490

taskset -c 1 ./Run shell1 --- 1175

taskset -c 0-63 ./Run shell1 --- 486

taskset -c 0-31 ./Run shell1 --- 489

taskset -c 0-3 ./Run shell1  --- 772

taskset -c 0-1 ./Run shell1  --- 1367

 

### audit -- 无影响

默认开 -- 490

关     -- 492



### hugepage - 无影响



## 使用说明

shell1：

​		bin/looper 60 multi.sh 1

shell8：

​		bin/looper 60 multi.sh 8

 参数说明：

| **$1** | **持续时间 60s，通过SIGALARM来计时，到时间后停止任务。** |
| ------ | -------------------------------------------------------- |
| $2     | 通过fork子线程来执行multi.sh文件。                       |
| $3     | multi.sh文件：                                           |

``` shell
# multi.sh文件：
	同时执行tst.sh的个数，每个tst.sh都用&后台执行。
# tst.sh
sort >sort.$$ <sort.src
od sort.$$ | sort -n -k 1 > od.$$
grep the sort.$$ | tee grep.$$ | wc > wc.$$
rm sort.$$ grep.$$ od.$$ wc.$$
```

全部都是文件行为。

共享文件：sort.src，所有进程都读它。

后面的sort, od, grep, tee, wc, rm 操作的都是文件。所以文件锁是比较关键的部分。



## 4k页表

### 单核

### 多核

#### 1630v100

测试环境： 123#  openEuler 22.03， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

分数：2351.9分。

特征：

- 带宽不高，只有不到1GB
- BE占比91%，其中都bound在了memory处，有时候是L1 bound，有时是L3 bound。

```shell
 26.36%  [kernel]                  [k] ptep_clear_flush
 10.77%  [kernel]                  [k] filemap_map_pages
  8.33%  [kernel]                  [k] ptep_set_access_flags
  5.45%  [kernel]                  [k] page_remove_file_rmap
  4.40%  [kernel]                  [k] release_pages
  4.20%  [kernel]                  [k] zap_pte_range
  2.30%  [kernel]                  [k] change_protection_range
  1.95%  [kernel]                  [k] finish_task_switch
```

```shell
******************DDR*******************************
TIME    SCCL    DDR_READ        DDR_WRITE       MATA_TOTAL      CROSS_CHIP      CROSS_DIE       INNER_DIE
10      sccl11  0.056 GB/s      0.144 GB/s      0.162 GB/s      0.027 GB/s      0.031 GB/s      0.103 GB/s
10      sccl9   0.142 GB/s      0.289 GB/s      0.329 GB/s      0.093 GB/s      0.071 GB/s      0.165 GB/s
10      sccl1   0.301 GB/s      0.092 GB/s      0.603 GB/s      0.021 GB/s      0.102 GB/s      0.480 GB/s
10      sccl3   0.285 GB/s      0.093 GB/s      0.555 GB/s      0.046 GB/s      0.154 GB/s      0.355 GB/s
********************L3C*****************************
TIME    SCCL    CPIPE_READ      CPIPE_WRI       CPIPE_HIT_R     CPIPE_hit_W     SPIPE_READ      SPIPE_WRI       SPIPE_hit_R     SPIPE_hit_W     L3_HIT_ALL      RET_TO_CPU_DATA
10      sccl11  0.333 GB/s      0.000 GB/s      0.024 GB/s      0.000 GB/s      0.348 GB/s      0.000 GB/s      0.125 GB/s      0.000 GB/s      0.291 GB/s      0.340 GB/s
10      sccl9   0.381 GB/s      0.000 GB/s      0.028 GB/s      0.000 GB/s      0.364 GB/s      0.000 GB/s      0.139 GB/s      0.000 GB/s      0.337 GB/s      0.388 GB/s
10      sccl1   1.061 GB/s      0.000 GB/s      0.079 GB/s      0.000 GB/s      1.068 GB/s      0.000 GB/s      0.396 GB/s      0.000 GB/s      0.894 GB/s      1.029 GB/s
10      sccl3   0.762 GB/s      0.000 GB/s      0.053 GB/s      0.000 GB/s      0.766 GB/s      0.000 GB/s      0.284 GB/s      0.000 GB/s      0.623 GB/s      0.736 GB/s

```

```shell
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                      848015122
        Total Execution Cycles:                         12032698440
        Instructions Per Cycle:                               0.070

        Front-End Bound:                                     7.215%
                Front-End Latency:                           6.773%
                        Idle_by_itlb_miss:                   0.004%
                        Idle_by_icache_miss:                 1.045%
                        BP_Misp_Flush:                       0.064%
                        OoO_Flush:                           0.178%
                        SP_Flush:                            0.032%
                Front End Bound Bandwidth:                   0.442%

        Bad Speculation:                                     0.193%
                Branch Mispredicts:                          0.076%
                        Indirect Branch:                     2.620%
                        Push Branch:                         1.283%
                        Pop Branch:                          0.018%
                        Other Branch:                      100.000%
                Machine Clears:                              0.117%
                        Nuke Flush:                          1.245%
                        Other Flush:                        98.755%

        Retiring:                                            1.175%

        Back-End Bound:                                     91.417%
                Resource Bound:                              7.656%
                        Rob_stall:                           1.534%
                        Ptag_stall:                          0.000%
                        MapQ_stall:                         91.424%
                        PCBuf_stall:                         7.043%
                        Other_stall:                         0.000%
                Core Bound:                                  4.937%
                        FDIV_Stall:                          0.000%
                        DIV_Stall:                           0.004%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                      4.933%
                                0_ports_serialize:          81.985%
                                0_ports_non_serialize:       9.693%
                                1_ports:                     0.783%
                                2_ports:                     0.718%
                                3_ports:                     0.414%
                                4_ports:                     0.297%
                                5_ports:                     0.199%
                                6_ports:                     0.171%
                Memory Bound:                               82.193%
                        L1 Bound:                           28.829%
                                DTLB:                        0.005%
                                Misalign:                    0.003%
                                Resource_Full:               0.000%
                                Instruction_Type:            0.133%
                                Forward_hazard:              0.340%
                                Structure_hazard:            0.184%
                                Pipeline:                    0.033%
                        L2 Bound:                            0.311%
                                buffer pending:              0.000%
                                snoop pending:               0.052%
                                Arb idle:                    0.190%
                                Pipeline:                    0.068%
                        L3 Bound:                           53.053%
                        Mem Bound:                           0.000%
                                Local Dram:                  0.000%
                                Remote Dram:                 0.000%
                                Remote Cache:                0.000%
                        Store Bound:                         0.000%
                                SCA:                         0.179%
                                Head:                        0.030%
                                Order:                     130.073%
                                Other:                       0.000%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

```





## 64k页表

### 单核

1620上openEuler 21.03， 内核5.10 64k pagesize， 虚拟机内，sh使用静态链接，sort/od/sh/tee/rm均使用静态链接的busybox格式，awk/grep/wc使用tools下的动态链接，通过LD_LIBRARY_PATH和PATH来指定。

[1620_shell1_1c_2024score_minifs_toolsbin]: ../images/shell1_1c_2024score.data.svg
[6248_shell1_1c_1584score_oe2103]: ../images/shell1_1c_82_1584score.svg



1620 44#（2.6GHz，2666内存）上跑emu_unixbench的looper1小用例，性能1300分(tmpfs上为1363分），topdown如下：

```c
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                     8327995440
        Total Execution Cycles:                          7811369339
        Instructions Per Cycle:                               1.066

        Front-End Bound:                                   37.378%
                Front-End Latency:                         33.204%
                        iTLB Miss:                           1.248%
                                L1 iTLB Miss:                0.564%
                                L2 iTLB Miss:                0.684%
                        iCache Miss:                        58.587%
                                L1 iCache Miss:             11.122%
                                L2 iCache Miss:             47.465%
                        BP_Misp_Flush:                       3.596%
                        OoO Rob Flush:                       0.000%
                        Static Predictor Flush:              2.050%
                Front End Bound Bandwidth:                   4.174%

        Bad Speculation:                                     7.980%
                Branch Mispredicts:                          6.895%
                        Indirect Branch:                     0.411%
                        Push Branch:                         0.059%
                        Pop Branch:                          0.280%
                        Other Branch:                        6.484%
                Machine Clears:                              1.085%
                        Nuke Flush:                          0.302%
                        Other Flush:                         0.783%

        Retiring:                                           26.653%

        Back-End Bound:                                     27.988%
                Resource Bound:                              8.593%
                        Sync_stall:                          0.000%
                        Rob_stall:                           0.388%
                        Ptag_stall:                          0.570%
                        SaveOpQ_stall:                       0.000%
                        PC_buffer_stall:                     0.099%
                Core Bound:                                 54.971%
                        Divider:                             0.008%
                        FSU_stall:                           0.046%
                        Exe Ports Util:                     54.918%
                                ALU BRU IssueQ Full:         2.163%
                                LS IssueQ Full:             13.426%
                                FSU IssueQ Full:             0.211%
                Memory Bound:                               28.031%
                        L1 Bound:                           11.002%
                        L2 Bount:                            1.712%
                        Intra Cluster Remote L2 Bound:       0.029%
                        Local LLC Bound:                     0.531%
                        Inter Cluster Local LLC Bound:       5.330%
                        Intra Chip Remote LLC Bound:        -5.608%
                        Inter Chip Remote LLC Bound:        11.766%
                        Intra Chip Local DDR Bound:          0.609%
                        Intra Chip Remote DDR Bound:         0.347%
                        Inter Chip Remote DDR Bound:         0.170%
                        Store Bound:                         2.143%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

```










### 多核

fork/exec/exit以及进程切换、进程迁移、进程唤醒的操作，频率都比spawn多核要少90%，因此需要重点关注的不是进程创建删除的生命周期，而是elf文件加载、运行、管道等操作。

在1620（44#）上，4.19内核，1p64c上跑64进程。

#### 负载

核间比较均衡，user=6%，sys=19%，idle=75%。

#### 热点函数

perf top 结果：

``` c
  20.44%  [kernel]                   [k] osq_lock
   7.78%  [kernel]                   [k] copy_page
   5.08%  [kernel]                   [k] finish_task_switch
   4.17%  [kernel]                   [k] rwsem_spin_on_owner
   2.49%  [kernel]                   [k] __do_softirq
   1.66%  [kernel]                   [k] run_timer_softirq
   1.45%  [kernel]                   [k] clear_page
```

perf report --no-children 结果：

``` c
+    9.34%  sh               [kernel.vmlinux]    [k] osq_lock                  
+    8.47%  swapper          [kernel.vmlinux]    [k] arch_cpu_idle             
+    4.06%  tee              [kernel.vmlinux]    [k] osq_lock                  
+    3.19%  rm               [kernel.vmlinux]    [k] osq_lock                  
+    3.17%  sh               [kernel.vmlinux]    [k] copy_page                 
+    3.17%  swapper          [kernel.vmlinux]    [k] finish_task_switch        
+    1.74%  sh               [kernel.vmlinux]    [k] rwsem_spin_on_owner       
+    1.58%  swapper          [kernel.vmlinux]    [k] __do_softirq              
+    1.57%  rm               [kernel.vmlinux]    [k] rwsem_spin_on_owner       
+    1.38%  multi.sh         [kernel.vmlinux]    [k] copy_page                 
+    1.21%  od               libc-2.29.so        [.] __vfprintf_internal       
+    1.08%  swapper          [kernel.vmlinux]    [k] run_timer_softirq         
+    0.93%  sort             libc-2.29.so        [.] __strcoll_l               
+    0.84%  sort             [kernel.vmlinux]    [k] copy_page                 
+    0.72%  sh               [kernel.vmlinux]    [k] finish_task_switch        
+    0.64%  sh               [kernel.vmlinux]    [k] clear_page                
+    0.62%  tee              [kernel.vmlinux]    [k] rwsem_spin_on_owner       
+    0.60%  od               libc-2.29.so        [.] _IO_fwrite                
```

主要集中在osq_lock, idle, rwsem_spin_on_owner, copy_page/clear_page上面。

- 关于osq_lock: 

  ​	shell指令sh，tee，rm,od, sort，looper，multi.sh, expr,wc都会调用到。主要调用者为sh/tee/rm，三者的调用路径是：

  ``` c
  sh-> reader_loop-> execute_command  // bash的处理函数
      -> execute_command _internal -> do_redirections // bash的处理函数
      	-> __GI___libc_open -> open -> path_openat -> do_last -> down_write
  tee -> __fopen_internal -> _IO_file_open 
      	-> __GI___libc_open
  rm -> unlinkat
      	-> __arm64_sys_unlinkat -> do_unlinkat -> down_write
  ```

- 关于rwsem_spin_on_owner：

  ​	与osq_lock类似，都是down_write产生的。

- idle

  函数finish_task_switch， arch_cpu_idle, __do_softirq都与idle相关。

- copy_page/clear_page

  - 在fork过程中，新进程的前3级页表（pte除外）的创建(dup_mm)需要用到clear_page进行页表的清零初始化。

  - 在do_wp_copy分配新页表的时候，会调用copy_page进行页表的复制。

#### 调度

``` c
// 1s内： 64c跑64进程。
9.144198230                  0      sched:sched_kthread_stop
9.144198230                  0      sched:sched_kthread_stop_ret
9.144198230             29,524      sched:sched_waking
9.144198230             29,520      sched:sched_wakeup
9.144198230              6,559      sched:sched_wakeup_new
9.144198230             71,693      sched:sched_switch
9.144198230             13,536      sched:sched_migrate_task
9.144198230              6,525      sched:sched_process_free
9.144198230              6,560      sched:sched_process_exit
9.144198230                  0      sched:sched_wait_task
9.144198230             10,508      sched:sched_process_wait
9.144198230              6,558      sched:sched_process_fork
9.144198230              6,570      sched:sched_process_exec
9.144198230                  0      sched:sched_stat_wait
9.144198230                  0      sched:sched_stat_sleep
9.144198230                  0      sched:sched_stat_iowait
9.144198230                  0      sched:sched_stat_blocked
9.144198230     14,803,722,670      sched:sched_stat_runtime
9.144198230                  0      sched:sched_pi_setprio
9.144198230                  0      sched:sched_process_hang
9.144198230                  0      sched:sched_move_numa
9.144198230                  0      sched:sched_stick_numa
9.144198230                  0      sched:sched_swap_numa
9.144198230                  0      sched:sched_wake_idle_without_ipi
```

#### idle

#### osq_lock

调用关系1：open

`reader_loop ->  execute_command_internal -> do_redirections -> do_sys_open -> path_openat->down_write ->osq_lock`

调用关系2：unlink

`unlinkat(svc)->__arm64_sys_unlinkat->do_unlinkat->down_write`

从热点上看，sh/rm/tee/wc/sort/multi.sh/expr/looper/grep/od这些进程全都会调用到osq_lock，都是从svc的open或unlink操作调用到down_write使用到的，怀疑和管道的操作是有关系的。

#### copy_page

调用关系1： 写时复制

`do_mem_abort->do_page_fault->handle_mm_fault->wp_page_copy->copy_page`

调用关系2： 页表查找的fault

`do_mem_abort->do_translation_fault->do_page_fault->copy_page`

从热点上看，sh/rm/tee/wc/sort/multi.sh/expr/looper/grep/od这些进程全都会调用到copy_page，都是从do_mem_abort的缺页异常调用的，应该是运行新进程的时候会发生缺页。

####  clear_page

``` c
// perf report --no-children
// 1620, 32c shell1热点
 clear_page <-                                            
 get_page_from_freelist                                    
 __alloc_pages_nodemask                                    
  - 7.36% alloc_pages_current                              
     - 4.07% __pte_alloc                                   
        - 3.12% copy_pte_range                             
             copy_page_range                               
             dup_mmap                                      
             dup_mm                                        
             copy_process                                  
             _do_fork                                      
             __do_sys_clone                                
             __arm64_sys_clone                             
             do_el0_svc                                    
             el0_sync_handler                              
             el0_sync                                      
             __libc_fork                                   
             make_child                                    
        - 0.90% do_anonymous_page                          
           - handle_mm_fault                               
              - 0.81% __get_user_pages                     
                   __get_user_pages_remote                 
                   get_user_pages_remote                   
                   get_arg_page                            
                   copy_string_kernel                      
                   __do_execve_file                        
                   __arm64_sys_execve                      
                   do_el0_svc                              
                   el0_sync_handler                        
                 - el0_sync                                
                      0.81% __GI___execve                  
     - 2.43% __pmd_alloc                                   
        - 1.56% copy_page_range                            
             dup_mmap                                      
             dup_mm                                        
             copy_process                                  
             _do_fork                                      
             __do_sys_clone                                
             __arm64_sys_clone                             
             do_el0_svc                                    
             el0_sync_handler                              
             el0_sync                                      
             __libc_fork                                   
             make_child                                    
        - 0.87% handle_mm_fault                            
           + 0.76% __get_user_pages                        
     - 0.50% __do_fault                                    
          do_fault                                         
          handle_mm_fault                                  
          do_page_fault                                    
          do_translation_fault                             
          do_mem_abort                                     
  - 0.82% alloc_pages_vma                                  
       do_anonymous_page                                   
     - handle_mm_fault                                     
        + 0.69% __get_user_pages                                                                  
     
```

来源：

 1. fork子进程时，需要复制父进程的vma页表。

     `fork->copy_process->dup_mm->dup_mmap->copy_page_range->__pte_alloc->alloc_pages_current->__alloc_pages_nodemask->clear_page `

2. exec执行shell命令时，需要将elf文件头copy到内核去处理，引发了缺页异常。

   `exec->__arm64_sys_execve->copy_string_kernel->get_arg_page->get_user_pages_remote->__get_user_pages->handle_mm_fault->alloc_pages_vma->__alloc_pages_nodemask->clear_page`
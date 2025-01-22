[TOC]

# spawn测试项介绍

该项测试进程创建的性能，主要流程：父进程通过fork()接口创建子进程并wait()阻塞等待，子进程只执行exit(0)退出指令。父进程继续创建新的子进程，重复该操作。

## 4k页表
### 单核
### 多核

#### 1620
##### Topdown

``` shell
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                      435038521
        Total Execution Cycles:                          3936337222
        Instructions Per Cycle:                               0.111

        Front-End Bound:                                    8.115%
                Front-End Latency:                          7.813%
                        iTLB Miss:                           1.134%
                                L1 iTLB Miss:                0.109%
                                L2 iTLB Miss:                1.025%
                        iCache Miss:                        10.944%
                                L1 iCache Miss:              1.517%
                                L2 iCache Miss:              9.427%
                        BP_Misp_Flush:                       0.459%
                        OoO Rob Flush:                       0.000%
                        Static Predictor Flush:              0.405%
                Front End Bound Bandwidth:                   0.302%

        Bad Speculation:                                     2.227%
                Branch Mispredicts:                          1.489%
                        Indirect Branch:                     0.081%
                        Push Branch:                         0.010%
                        Pop Branch:                          0.071%
                        Other Branch:                        1.407%
                Machine Clears:                              0.739%
                        Nuke Flush:                          0.445%
                        Other Flush:                         0.294%

        Retiring:                                            2.763%

        Back-End Bound:                                     86.895%
                Resource Bound:                             29.255%
                        Sync_stall:                          0.000%
                        Rob_stall:                           3.842%
                        Ptag_stall:                          1.177%
                        SaveOpQ_stall:                       0.000%
                        PC_buffer_stall:                     0.510%
                Core Bound:                                 31.827%
                        Divider:                             0.002%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                     31.826%
                                ALU BRU IssueQ Full:         7.819%
                                LS IssueQ Full:             48.686%
                                FSU IssueQ Full:             0.000%
                Memory Bound:                               37.978%
                        L1 Bound:                           17.840%
                        L2 Bount:                            0.337%
                        Intra Cluster Remote L2 Bound:       0.361%
                        Local LLC Bound:                     0.558%
                        Inter Cluster Local LLC Bound:       7.653%
                        Intra Chip Remote LLC Bound:        -5.781%
                        Inter Chip Remote LLC Bound:        12.241%
                        Intra Chip Local DDR Bound:          1.977%
                        Intra Chip Remote DDR Bound:         2.251%
                        Inter Chip Remote DDR Bound:         0.219%
                        Store Bound:                         0.322%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```



```shell
                    Start to monitor The System DDR Bandwidth
######################################        ######################################
|        CPU0 Bandwidth (MB/s)       |        |        CPU1 Bandwidth (MB/s)       |
######################################        ######################################
|         Die0          Die1         |        |         Die0          Die1         |
--------------------------------------        --------------------------------------
| RD:    7625.85       9773.43       |        | RD:    5793.33      10169.74       |
--------------------------------------        --------------------------------------
| WR:    1631.24       3020.96       |        | WR:    1847.31       3496.63       |
--------------------------------------        --------------------------------------
|   CPU0 Total Bandwidth: 22051.48   |        |   CPU1 Total Bandwidth: 21307.00   |
####################################################################################
|                   System Total Read Bandwidth:   33362.35                        |
------------------------------------------------------------------------------------
|                   System Total Write Bandwidth:    9996.13                       |
------------------------------------------------------------------------------------
|                     System Total Bandwidth:   43358.48                           |
####################################################################################

```

#### 1630V100

测试环境： 123#  openEuler 22.03硬盘文件系统， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

测试分数： 6348分

```shell
45.08%  [kernel]       [k] ptep_clear_flush
 7.96%  [kernel]       [k] osq_lock
 7.25%  [kernel]       [k] ptep_set_access_flags
 4.35%  [kernel]       [k] dup_mmap
 4.16%  [kernel]       [k] tlb_flush_mmu
 3.05%  [kernel]       [k] filemap_map_pages
 2.45%  [kernel]       [k] release_pages
 1.55%  [kernel]       [k] page_remove_file_rmap
 1.51%  [kernel]       [k] rwsem_wake.constprop.0.isra.0
 1.27%  [kernel]       [k] native_queued_spin_lock_slowpath
 0.91%  [kernel]       [k] release_task
 0.90%  [kernel]       [k] exit_notify
 0.82%  [kernel]       [k] copy_process
```

其中，`ptep_clear_flush`的热点99%在于：

```c
  tlbi    vale1is, x1   
  dsb     ish                               
```

```shell
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                    87321063682
        Total Execution Cycles:                        894618668728
        Instructions Per Cycle:                               0.098

        Front-End Bound:                                    13.167%
                Front-End Latency:                          12.702%
                        Idle_by_itlb_miss:                   1.042%
                        Idle_by_icache_miss:                 4.311%
                        BP_Misp_Flush:                       0.171%
                        OoO_Flush:                           0.080%
                        SP_Flush:                            0.220%
                Front End Bound Bandwidth:                   0.465%

        Bad Speculation:                                     0.543%
                Branch Mispredicts:                          0.430%
                        Indirect Branch:                     1.284%
                        Push Branch:                         0.634%
                        Pop Branch:                          0.009%
                        Other Branch:                      100.000%
                Machine Clears:                              0.112%
                        Nuke Flush:                         11.011%
                        Other Flush:                        88.989%

        Retiring:                                            1.627%

        Back-End Bound:                                     84.664%
                Resource Bound:                             57.378%
                        Rob_stall:                           1.098%
                        Ptag_stall:                          0.010%
                        MapQ_stall:                         98.537%
                        PCBuf_stall:                         0.355%
                        Other_stall:                         0.000%
                Core Bound:                                -36.712%
                        FDIV_Stall:                          0.000%
                        DIV_Stall:                           0.001%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                    -36.714%
                                0_ports_serialize:          21.713%
                                0_ports_non_serialize:      76.099%
                                1_ports:                     1.123%
                                2_ports:                     1.144%
                                3_ports:                     0.911%
                                4_ports:                     0.690%
                                5_ports:                     0.469%
                                6_ports:                     0.384%
                Memory Bound:                               83.535%
                        L1 Bound:                           73.250%
                                DTLB:                        0.150%
                                Misalign:                    0.011%
                                Resource_Full:               0.000%
                                Instruction_Type:            0.139%
                                Forward_hazard:              1.341%
                                Structure_hazard:            0.457%
                                Pipeline:                    0.066%
                        L2 Bound:                            0.587%
                                buffer pending:              0.000%
                                snoop pending:               0.042%
                                Arb idle:                    0.516%
                                Pipeline:                    0.029%
                        L3 Bound:                            9.689%
                        Mem Bound:                           0.000%
                                Local Dram:                  0.000%
                                Remote Dram:                 0.000%
                                Remote Cache:                0.000%
                        Store Bound:                         0.009%
                                SCA:                         0.352%
                                Head:                      104.199%
                                Order:                      31.816%
                                Other:                       0.052%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

```python
******************DDR*******************************
TIME    SCCL    DDR_READ        DDR_WRITE       MATA_TOTAL      CROSS_CHIP      CROSS_DIE       INNER_DIE
3       sccl9   3.073 GB/s      4.415 GB/s      6.022 GB/s      1.657 GB/s      1.829 GB/s      2.536 GB/s
3       sccl3   1.552 GB/s      3.408 GB/s      4.306 GB/s      0.187 GB/s      1.274 GB/s      2.845 GB/s
3       sccl11  0.756 GB/s      2.144 GB/s      2.287 GB/s      0.279 GB/s      0.603 GB/s      1.406 GB/s
3       sccl1   1.898 GB/s      3.693 GB/s      4.956 GB/s      0.339 GB/s      1.513 GB/s      3.104 GB/s
********************L3C*****************************
TIME    SCCL    CPIPE_READ      CPIPE_WRI       CPIPE_HIT_R     CPIPE_hit_W     SPIPE_READ      SPIPE_WRI       SPIPE_hit_R     SPIPE_hit_W     L3_HIT_ALL      RET_TO_CPU_DATA
3       sccl9   4.178 GB/s      0.000 GB/s      0.376 GB/s      0.000 GB/s      4.151 GB/s      0.000 GB/s      1.003 GB/s      0.000 GB/s      3.210 GB/s      4.075 GB/s
3       sccl3   7.914 GB/s      0.000 GB/s      0.612 GB/s      0.000 GB/s      7.828 GB/s      0.000 GB/s      1.817 GB/s      0.000 GB/s      6.424 GB/s      7.706 GB/s
3       sccl11  4.248 GB/s      0.000 GB/s      0.390 GB/s      0.000 GB/s      4.227 GB/s      0.000 GB/s      1.029 GB/s      0.000 GB/s      3.279 GB/s      4.124 GB/s
3       sccl1   7.949 GB/s      0.000 GB/s      0.620 GB/s      0.000 GB/s      7.871 GB/s      0.000 GB/s      1.817 GB/s      0.000 GB/s      6.532 GB/s      7.795 GB/s

```

若是单die多核(40核，0-39)，9695.4分，且热点会变化：

```shell
   9.68%  [kernel]               [k] filemap_map_pages
   8.89%  [kernel]               [k] release_pages
   7.27%  [kernel]               [k] ptep_clear_flush
   5.09%  [kernel]               [k] native_queued_spin_lock_slowpath
   3.87%  [kernel]               [k] exit_notify
   3.58%  [kernel]               [k] zap_pte_range
   3.47%  [kernel]               [k] release_task
   3.40%  [kernel]               [k] page_remove_file_rmap
   3.18%  [kernel]               [k] wake_up_new_task
   3.13%  [kernel]               [k] copy_process
   1.93%  [kernel]               [k] __pagevec_lru_add
```

单P 2die 80核（0-79,80进程），9513.7分

```shell
  14.42%  [kernel]               [k] native_queued_spin_lock_slowpath
   8.52%  [kernel]               [k] exit_notify
   7.79%  [kernel]               [k] release_task
   7.37%  [kernel]               [k] filemap_map_pages
   7.20%  [kernel]               [k] copy_process
   6.80%  [kernel]               [k] release_pages
   6.77%  [kernel]               [k] ptep_clear_flush
   3.08%  [kernel]               [k] osq_lock
   2.38%  [kernel]               [k] wake_up_new_task

```

单P 跨die 40核（0-19,40-59,40进程）， 6523.2分

```shell
  22.50%  [kernel]                 [k] ptep_clear_flush
  10.95%  [kernel]                 [k] filemap_map_pages
   8.00%  [kernel]                 [k] release_pages
   5.29%  [kernel]                 [k] page_remove_file_rmap
   4.37%  [kernel]                 [k] ptep_set_access_flags
   2.66%  [kernel]                 [k] dup_mmap
   2.61%  [kernel]                 [k] tlb_flush_mmu
   2.36%  [kernel]                 [k] zap_pte_range
   1.89%  [kernel]                 [k] wake_up_new_task
   1.29%  [kernel]                 [k] down_write
```

跨P 40核（0-19,80-99,40进程）, 6751.8分

```shell
 23.44%  [kernel]                    [k] ptep_clear_flush
 11.05%  [kernel]                    [k] filemap_map_pages
  7.52%  [kernel]                    [k] release_pages
  5.03%  [kernel]                    [k] page_remove_file_rmap
  4.62%  [kernel]                    [k] ptep_set_access_flags
  2.80%  [kernel]                    [k] tlb_flush_mmu
  2.73%  [kernel]                    [k] dup_mmap
```



#### Intel 6248

`测试环境：118# Intel 6248，关SMT，cpu关turbo定频2.5GHz，ddr 2933MHz*12根*16GB， openEuler21.03，linux5.10.0_yhy`

单P 20c（0-19），分数8877.3分

```shell
   9.13%  [kernel]             [k] filemap_map_pages
   4.05%  [kernel]             [k] native_queued_spin_lock_slowpath.part.0
   3.82%  [kernel]             [k] release_pages
   3.04%  [kernel]             [k] zap_pte_range.isra.0
   2.60%  [kernel]             [k] _raw_spin_lock
   2.32%  [kernel]             [k] native_irq_return_iret
   2.22%  [kernel]             [k] page_remove_file_rmap
   2.21%  [kernel]             [k] mark_page_accessed
   2.11%  [kernel]             [k] clear_page_erms
   1.79%  [kernel]             [k] __slab_free
```

跨P 20c（0-9,20-29），分数6334.4分

```shell
  11.08%  [kernel]                 [k] filemap_map_pages
   4.04%  [kernel]                 [k] zap_pte_range.isra.0
   3.69%  [kernel]                 [k] release_pages
   3.20%  [kernel]                 [k] native_queued_spin_lock_slowpath.part.0
   2.57%  [kernel]                 [k] osq_lock
   2.24%  [kernel]                 [k] page_remove_file_rmap
   2.16%  [kernel]                 [k] _raw_spin_lock
   1.90%  [kernel]                 [k] native_irq_return_iret
   1.71%  [kernel]                 [k] up_write
```

其中，

```shell
filemap_map_pages:
28.41 │       mov    0x8(%r15),%rdx
  0.18 │       lea    -0x1(%rdx),%rax
  0.08 │       and    $0x1,%edx
  0.17 │       cmove  %r15,%rax
  5.40 │       mov    (%rax),%rax
  0.19 │       test   $0x1,%al
  0.01 │     ↓ jne    1b7
  0.09 │       mov    0x34(%r15),%eax
  0.04 │ e0:   test   %eax,%eax
       │     ↓ je     1b7
  0.07 │       lea    0x1(%rax),%edx
 30.19 │       lock   cmpxchg %edx,0x34(%r15)
  0.02 │     ↑ jne    e0

```

整芯片40c，分数8080分；

#### AMD 7742

`测试环境： 120# AMD 7742, 关SMT，关turbo，CPU频率2.23GHz定频，ddr 32根*2933MHz * 16GB  `

单片40核，分数：6935.2分（taskset -c 0-39 ./Run -i 1  -c 40  spawn）

```shell
  15.36%  [kernel]                             [k] osq_lock
  14.78%  [kernel]                             [k] filemap_map_pages
   6.24%  [kernel]                             [k] zap_pte_range
   4.50%  [kernel]                             [k] native_queued_spin_lock_slowpath
   2.98%  [kernel]                             [k] find_idlest_group
   2.06%  [kernel]                             [k] page_remove_file_rmap
   1.98%  [kernel]                             [k] down_write
   1.96%  [kernel]                             [k] release_pages

```

跨P40核，分数： 7626.8分(taskset -c 0-19,64-83 ./Run -i 1  -c 40  spawn)

```shell
  13.52%  [kernel]                             [k] filemap_map_pages
   8.12%  [kernel]                             [k] osq_lock
   7.30%  [kernel]                             [k] native_queued_spin_lock_slowpath
   6.22%  [kernel]                             [k] zap_pte_range
   4.84%  [kernel]                             [k] find_idlest_group
   2.26%  [kernel]                             [k] page_remove_file_rmap
   2.24%  [kernel]                             [k] release_pages
   2.10%  [kernel]                             [k] copy_page
   1.82%  [kernel]                             [k] down_write
   1.49%  [kernel]                             [k] __handle_mm_fault

filemap_map_pages:
  0.00 │       mov    0x20(%rsp),%r14
  0.00 │       mov    (%rax),%rax
 38.30 │       mov    0x50(%rax),%rax
  0.44 │       add    $0xfff,%rax
  0.02 │       shr    $0xc,%rax
       │       cmp    %rax,%r14
  0.11 │     ↑ jae    2db
```

单片20核，分数：6004分 (taskset -c 0-19 ./Run -i 1  -c 20  spawn)

跨片20核，分数：4376分(taskset -c 0-9,64-73 ./Run -i 1  -c 20  spawn)

单片64核，分数：6545分

跨片64核，分数：7459分

整芯片128核，分数：7205分




## 64k页表

### 单核

#### 热点

![image-20201020204859348](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20201020204859348.png)

#### 函数级分析

波形上分析的函数占比：

​	![image-20221130145720894](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221130145720894.png)

从emu波形上抓trace分析，

![image-20221130144835345](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221130144835345.png)

![image-20221130144917430](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221130144917430.png)





### 多核

#### 1620

1620服务器（42#）， 单片64c，openEuler21.03版本，

![image-20221109110841446](D:\at_work\Documents\我的总结文档\images\image-20221109110841446.png)

其中，`ptep_clear_flush`热点主要在`tlbi+dsb`，调用栈是从`el0_da`以及`el1_abort`的`do_mem_abort`来的。

`filemap_map_pages`热点主要在`cas`锁上，主要来源是`el0_ia`和`el0_da`；

`release_pages`热点主要在`ldaddal`上，来源是`do_exit->exit_mkmap->tlb_finish_mmu->tlb_flush_mmu->free_pages_and_swap_cache`

`page_remove_file_rmap`在`ldaddal`上；`down_write`在`cas`上；`dup_mmap`在`tlbi+dsb`





| `ptep_clear_flush`热点主要在`tlbi+dsb`                       | `filemap_map_pages`热点主要在`cas`锁上。                     |      |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| <img src="D:\at_work\Documents\我的总结文档\images\image-20221109111249102.png" alt="image-20221109111249102" style="zoom:50%;" /> | <img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221109111752234.png" alt="image-20221109111752234" style="zoom:50%;" /> |      |
| <img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221109152357561.png" alt="image-20221109152357561" style="zoom:50%;" /> | <img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221109152640216.png" alt="image-20221109152640216" style="zoom:50%;" /> |      |
|                                                              |                                                              |      |

| `release_pages`                                              | `dup_fd` |      |
| ------------------------------------------------------------ | -------- | ---- |
| <img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221109151717395.png" alt="image-20221109151717395" style="zoom:50%;" /> |          |      |
| <img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221109153528727.png" alt="image-20221109153528727" style="zoom:50%;" /> |          |      |
|                                                              |          |      |



##### TopDown

![image-20201020204922560](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20201020204922560.png)

#### 1630V100

测试环境： 123#  openEuler 22.03硬盘文件系统， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

##### TopDown

1630V100样片上测试，关SMT 160C，热点主要集中在qspinlock & ptep_clear_flush，其中第二个主要在


```shell
11.74%  [kernel]       [k] ptep_clear_flush   # dvm sync                              
11.55%  [kernel]       [k] queued_spin_lock_slowpath                        
 7.26%  [kernel]       [k] arch_local_irq_enable                            
 6.60%  [kernel]       [k] arch_local_irq_enable                            
 6.55%  [kernel]       [k] release_task                                     
 3.79%  [kernel]       [k] copy_page                                        
 3.60%  [kernel]       [k] ptep_set_access_flags                            
 3.28%  [kernel]       [k] arch_cpu_idle                                    
 3.22%  [kernel]       [k] dup_mm                                           
 2.94%  [kernel]       [k] dup_fd                                           
 2.74%  [kernel]       [k] flush_tlb_mm                                     
 2.67%  [kernel]       [k] filp_close                                       
 2.46%  [kernel]       [k] osq_lock                                         
 2.34%  [kernel]       [k] clear_page                                       
```


	+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
		Total Execution Instruction:                    66157854732
		Total Execution Cycles:                        531544157564
		Instructions Per Cycle:                               0.124
		
	Front-End Bound:                                    35.729%
		Front-End Latency:                          35.136%
			Idle_by_itlb_miss:                   1.324%
			Idle_by_icache_miss:                 8.471%
			BP_Misp_Flush:                       0.234%
			OoO_Flush:                           0.121%
			SP_Flush:                            0.290%
		Front End Bound Bandwidth:                   0.593%
	
	Bad Speculation:                                     0.761%
		Branch Mispredicts:                          0.591%
			Indirect Branch:                     1.677%
			Push Branch:                         0.837%
			Pop Branch:                          0.014%
			Other Branch:                      100.000%
		Machine Clears:                              0.171%
			Nuke Flush:                          4.510%
			Other Flush:                        95.490%
	
	Retiring:                                            2.074%
	
	Back-End Bound:                                     61.435%
		Resource Bound:                             27.007%
			Rob_stall:                           7.074%
			Ptag_stall:                          0.319%
			MapQ_stall:                         91.422%
			PCBuf_stall:                         1.185%
			Other_stall:                         0.000%
		Core Bound:                                 13.502%
			FDIV_Stall:                          0.000%
			DIV_Stall:                           0.005%
			FSU_stall:                           0.000%
			Exe Ports Util:                     13.498%
				0_ports_serialize:          18.638%
				0_ports_non_serialize:      75.340%
				1_ports:                     2.003%
				2_ports:                     1.947%
				3_ports:                     1.093%
				4_ports:                     0.862%
				5_ports:                     0.449%
				6_ports:                     0.347%
		Memory Bound:                               59.816%
			L1 Bound:                           39.769%
				DTLB:                        0.330%
				Misalign:                    0.007%
				Resource_Full:               0.002%
				Instruction_Type:            0.202%
				Forward_hazard:              1.920%
				Structure_hazard:            0.994%
				Pipeline:                    0.098%
			L2 Bound:                            0.946%
				buffer pending:              0.000%
				snoop pending:               0.079%
				Arb idle:                    0.768%
				Pipeline:                    0.100%
			L3 Bound:                           17.862%
			Mem Bound:                           0.000%
				Local Dram:                  0.000%
				Remote Dram:                 0.000%
				Remote Cache:                0.000%
			Store Bound:                         1.239%
				SCA:                         2.930%
				Head:                       35.935%
				Order:                      68.539%
				Other:                       0.113%





## 软件流程

``` c
static __latent_entropy struct task_struct *copy_process(
          unsigned long clone_flags,
          unsigned long stack_start,
          unsigned long stack_size,
          int __user *child_tidptr,
          struct pid *pid,
          int trace,
          unsigned long tls,
          int node)
{
    int retval;
    struct task_struct *p;
......
    //分配新的task_struct和内核栈，然后直接copy父进程的task_struct过来。
    p = dup_task_struct(current, node);  
......
    //权限处理
    retval = copy_creds(p, clone_flags);
......
    //设置调度相关变量
    retval = sched_fork(clone_flags, p);    
......
    //初始化文件相关变量, copy父进程打开的文件
   	// -> dup_fd fd代表打开的文件，统统复制到子进程去
    retval = copy_files(clone_flags, p);
    // copy 父进程fs_struct结构
    // 一个进程有自己的根目录和根文件系统 root，也有当前目录 pwd 和当前目录的文件系统
    retval = copy_fs(clone_flags, p);  
......
    //初始化信号相关变量
    init_sigpending(&p->pending);
    retval = copy_sighand(clone_flags, p);
    retval = copy_signal(clone_flags, p);  
......
    //拷贝进程内存空间
    // 父进程的内存空间
    // -> dup_mm
    //	-> dup_mmap 复制父进程的vma对应的pte页表项到子进程的页表项，而不是真正复制内容。
    retval = copy_mm(clone_flags, p);
...... 
    // copy_thread 若新进程不是内核线程，则将父进程的寄存器值复制到子进程中。   	
......    
    //初始化亲缘关系变量
    INIT_LIST_HEAD(&p->children);
    INIT_LIST_HEAD(&p->sibling);
......
    //建立亲缘关系
    //源码放在后面说明  
};

```



# 测试数据分析

|                 | 1620 | 6248关turbo | 688（3.0G） |
| --------------- | ---- | ----------- | ----------- |
| 单核默认预取    | 1072 | 1148        | 1640        |
| 单核关预取      | 1046 | -           | 1620        |
| 单die 20c开预取 | 305  | 434         | 341         |
| 单die 20c关预取 | 298  | -           | 337         |
|                 |      |             |             |
|                 |      |             |             |





# spawn的软件调用流程

## ub中的测试代码

主体思想是：

``` c
 while (1) {
         if ((slave = fork()) == 0) {
                 /* slave .. boring */
                 /* kill it right away */
                 exit(0);
         } else
                 /* master */
                 wait(&status);
         iter++;
         }
```

总共有3个系统调用函数，其中

父进程调用： fork + wait

子进程调用：exit

同一时间只有1个父进程和1个子进程。

## x86

entry_SYSCALL_64 -> do_syscall_64 (57%)-> 

有3大类： exit(30%)，fork(22%)，wait(4%)。重点在前两者。

![image-20201117105923494](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20201117105923494.png)

## exit

- 执行exit()退出后，会有大量的tlb flush操作出来。

  原因：

  因为是销毁进程，mm的引用会被降到0，那么就需要调用mmput->exit_mmap，当进程中的最后一个线程退出或进程成功调用`execve()`时会发生这种情况。

	1. 释放进程映射的内存 remove_vm_struct
	2. 释放vma所用的内存 unmap_vmas
	3. 释放进程页表所用内存clear_page_range（pud，pmd，pte） -> tlb_flush_mmu

``` c
do_group_exit -> do_exit 
-> mmput
	-> exit_mmap 
-> put_files_struct
-> do_notify_parent

void exit_mmap(struct mm_struct *mm)
{
    vma = mm->mmap; // 获取vma
    flush_cache_mm(mm); // 空函数
	tlb_gather_mmu(&tlb, mm, 0, -1); // 初始化mmu_gather结构体，为后续销毁页表做准备。
    unmap_vmas(&tlb, vma, 0, -1); // unmap所有vma所用的page。
    	// 调用unmap_page_range，通过va找到pte页表，并解开和pa的映射。
    	// 对于共享的pa则不用释放，只unmap即可。
    	
	free_pgtables(&tlb, vma, FIRST_USER_ADDRESS, USER_PGTABLES_CEILING);
    	// 真正释放页表，通过unmap后若页表没有人再使用了（pte没有映射），则需要释放掉。
    	// -> 遍历vma链表，调用free_pgd_range释放所有vma的pages。
    	// 释放之前，先unlink反向页表映射，包含匿名页和file-based页，
    	//    通过unlink_anon_vmas和unlink_file_vma函数。
	tlb_finish_mmu(&tlb, 0, -1); 
    	// 真正刷新TLB,根据mmu_gather结构体来释放页表框。
}

```

``` c
tlb_finish_mmu
    -> arch_tlb_finish_mmu
    	-> tlb_flush_mmu
    		-> tlb_flush_mmu_tlbonly(tlb);
			-> tlb_flush_mmu_free(tlb); 
				for(batch_nr次) 
                -> free_pages_and_swap_cache
                  	-> release_pages
    	-> free_pages
            ->
```



## fork

``` c
_do_fork -> copy_process ->
-> copy_page_range->__pte_alloc + __pmd_alloc + __pud_alloc
    	-> __alloc_pages_nodemask 
    		-> get_page_from_freelist -> clear_page_erms(rep stos)
    		-> memcg_kmem_charge -> page_counter_try_charge
```









# spawn中的pagefault

主要是2类fault，一是读的时候发现没有页表，进入read fault流程。二是写的时候发生cow fault，进入cow流程。

二者都是由硬件触发pagefault后进入pagefault的处理流程，

``` c
page_fault
->do_pagefault
-> handle_mm_fault
-> __handle_mm_fault
-> handle_pte_fault
```

然后，会进行判断，是进入到哪一种fault。

## read fault

handle_pte_fault ->

do_fault->

do_read_fault

do_fault_around 

vm_ops -> map_pages

在filemap.c:2628中: vma->vm_ops = &generic_file_vm_ops;

然后由此进入filemap_map_pages函数。



## cow fault

## 关于copy_page

``` c
// 关于copy_page的追踪。
el0_da -> do_mem_abort -> do_page_fault -> 
    handle_mm_fault -> __handle_mm_fault
		handle_pte_fault
			do_wp_page
				wp_page_copy
					alloc_page_vma(alloc_pages_vma) // 分配新页
						__alloc_pages_nodemask  （伙伴分配机制的核心）
							get_page_from_freelist
    				cow_user_page // 页表copy
    					copy_user_highpage
    						copy_user_page -> __cpu_copy_user_page
    							copy_page(kto, kfrom) -> memcpy(to, from, PAGE_SIZE)
    						flush_dcache_page // 只是标脏，后续才刷。
					ptep_clear_flush_notify // 刷tlb
```

``` c
// intel 上。
    do_wp_page
    	wp_page_copy
          	alloc_page_vma
        		__alloc_pages_nodemask  （伙伴分配机制的核心）
					get_page_from_freelist
	        ptep_clear_flush_notify(vma, vmf->address, vmf->pte);
				ptep_clear_flush    	
					flush_tlb_page                
						flush_tlb_mm_range
```

其中，copy_page的实现是依赖架构的，Intel是用rep movsq实现的，arm用ldp/stnp实现。

### 什么时候调用copy_page？

在64k页表下check，copy_page均在pagefault中被调用，主要有4个地方。

1. el1的ret_from_fork的schedule中发生el1的缺页。
2. el0的__run_exit_handlers中（lib库函数）
3. el0的__run_fork_handlers中（lib库函数）
4. el0的_dl_fixup中(lib库函数)

在4k页表下，前3项也都是存在的。



### cow为什么要flush tlb？

​	因为cow需要更新pte entry的属性，因此需要先flush掉之前的tlb entry。这里使用的是flush_tlb_page，按照page来flush。va是从vmf中获取，



### 64k与4k的区别？

4k粒度下：

​	每次cow的pagefault需要映射一个4k页。

64k粒度下：

​	每次cow的pagefault需要映射一个64k页，把原进程的该PAGE完全copy过来，带宽就是4k的16倍了。



## 关于clear_page

``` c
// 关于clear_page的追踪，42#上测试64k pagesize下共8.6%的热点。
copy_process->
    dup_mm->dup_mmap->copy_page_range->
		copy_pte_range
    		// 只有当pgd == none时，才会进入到pte_alloc函数中，真正分配一个内存页过来。
			__pte_alloc-> pte_alloc_one->alloc_pages-> // pte对应的地址空间一定是一个page了。
				alloc_pages_current
					__alloc_pages_nodemask
						get_page_from_freelist // 详细解析。
							prep_new_page
								kernel_init_free_pages
									clear_highpage
										clear_page(zva) (4.5%)
		__pmd_alloc
    		// 只有当pgd->pgd == none时，才会进入到pte_alloc函数中，真正分配一个内存页过来。
			alloc_pages_current->clear_page (3%)
    dup_task_struct-> //（待分析）
    	__vmalloc_node_range->
    		alloc_pages_current->clear_page(1.1%)
```

### __GFP_ZERO标记位何时置位的？

  ​	1. arm64结构中：`pte_alloc_one`函数中，调用`alloc_pages(PGALLOC_GFP, 0);`  而定义为：


``` c
#define PGALLOC_GFP (GFP_KERNEL | __GFP_ZERO)
```

​		  2. x86中，`pte_alloc_one`函数中，调用`alloc_pages(__userpte_alloc_gfp, 0)`; 定义中：

``` c
#define PGALLOC_GFP (GFP_KERNEL_ACCOUNT | __GFP_ZERO)
gfp_t __userpte_alloc_gfp = PGALLOC_GFP | PGALLOC_USER_GFP; 
```



###  为什么要让页表清零？

  如果不清零的话，页表中包含了以前的内容，可能会有问题。

  

### 为什么会出现`pgd->pgd== none`的情况？

新的task中，页表可以先不copy，但是需要先预留出新mm的各级页表的地址空间，并且这里直接对页表进行了清零操作，也就有大量的`clear_page`操作了。

``` c
copy_process-> copy_mm(CLONE_FLAGS, p) 
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
    dup_mm(tsk, current->mm)
}
static struct mm_struct *dup_mm(struct task_struct *tsk, struct mm_struct *oldmm)
{
    struct mm_struct *mm;
	mm = allocate_mm();
    memcpy(mm, oldmm, sizeof(*mm)); // 首先复制过去。
    if (!mm_init(mm, tsk, mm->user_ns)) // 然后mm_init改变其中的很多值，包含pgd成员。
    	goto fail_nomem;
    dup_mmap(mm, oldmm);
    // 实际trace发现，mm->pgd->pgd全部为0（但是mm->pgd不为NULL)， oldmm->pgd->pgd全部有值。
}
// mm_init -> mm_alloc_pgd
static inline int mm_alloc_pgd(struct mm_struct *mm)
{
    mm->pgd = pgd_alloc(mm);
    if (unlikely(!mm->pgd))
        return -ENOMEM;
    return 0;
}

#define ARM64_HW_PGTABLE_LEVEL_SHIFT(n) ((PAGE_SHIFT - 3) * (4 - (n)) + 3) 
	// 64K: n = 4 - PGTABLE_LEVELS = 1, ((16-3) * (4-1) + 3) = 42
	// 4K:  n = 4 - PGTABLE_LEVELS = 0, ((12-3) * (4-0) + 3) = 39
#define PGDIR_SHIFT     ARM64_HW_PGTABLE_LEVEL_SHIFT(4 - CONFIG_PGTABLE_LEVELS)
#define PTRS_PER_PGD        (1 << (VA_BITS - PGDIR_SHIFT)) // (1 << (48-42)) = 64
#define PGD_SIZE    (PTRS_PER_PGD * sizeof(pgd_t)) // level 0页表地址空间大小。
	// 64K:  64  * 8 Byte = 512  Byte， level 0只有64个entry
	// 4K:   512 * 8 Byte = 4096 Byte， level 0有512个entry
#define GFP_PGTABLE_KERNEL  (GFP_KERNEL | __GFP_ZERO)
#define GFP_PGTABLE_USER    (GFP_PGTABLE_KERNEL | __GFP_ACCOUNT)
pgd_t *pgd_alloc(struct mm_struct *mm)
{
    gfp_t gfp = GFP_PGTABLE_USER;
    if (PGD_SIZE == PAGE_SIZE) // 4k pagesize走这里
        return (pgd_t *)__get_free_page(gfp);
    else // 64k pagesize走这里
        return kmem_cache_alloc(pgd_cache, gfp);
}

void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
    void *ret = slab_alloc(cachep, flags, _RET_IP_);
    trace_kmem_cache_alloc(_RET_IP_, ret,
                   cachep->object_size, cachep->size, flags);
	// 通过trace可以check，都分配成功了(参数一一对应）。
    // 0.01%  call_site=ffff0000080a1464 ptr=0xffff802db3850a00 bytes_req=512 bytes_alloc=512 gfp_flags=GFP_KERNEL|__GFP_ZERO
    return ret;
}
```

通过上述代码追踪，可以看到fork时，对于新进程的vma->pgd域段，会先通过slab机制获取一段**预先清零**的地址空间（用来存放level 0页表），那么这时候pgd->pgd自然是NONE的，因为还没有真正写入页表。



### 页表创立的完整流程

1. 在`dup_mm`中的`mm_init`中，对于新的mm，会首先将它的level 0（pgd）页表地址按实际大小重新分配并清零。
   - 这里level 0的页表所占空间大小在4k时为4kB，刚好一个PAGE；在64kpagesize下，level 0只有64个entry，每个entry固定为64bit，因此实际大小只有512kB。

2. 在`dup_mm-> dup_mmap`中，调用`copy_page_range`函数。首先遍历level 0的每个entry，然后调用`copy_p4d_range`arm64由于没有p4d的概念(p4d == pgd)，因此可以忽略p4d这一层。
3. 继续下到pud这一层（`copy_pud_range`函数），对于64kB的pagesize，由于只有3层页表翻译，因此pud也被忽略（`pud == p4d`）； 对于4k页表则可以正常工作了。首先读取pgd的entry中的值发现是NULL，因此需要调用`pud_alloc`，先`kmem_cache_alloc`一段level 1页表，这里可以直接分配一个PAGE，因为页表大小刚好占用一个PAGE。然后把level 1页表的基地址填写到相应pgd的entry中，从而真正形成链表。
4. 然后到pmd这一层（`copy_pmd_range`函数）无论4k还是64k页表粒度，都需要建立level 2页表了。页表建立后，再遍历每个pmd entry，并逐个调用`copy_pte_range`了。
5. 到pte这一层，需要建立level 3页表了（`pte_alloc_map_lock`函数），过程同上。这样就可以建立起完整的4级/3级页表了。
6. 当前的pte entry依然是NULL的，因此`copy_pte_range`中对每个pte entry调用`copy_one_pte`，根据父进程的pte entry来构造子进程的pte entry并写入，同时对于cow的pte，将父子进程的pte entry都置为只读。

``` c
static inline unsigned long
copy_one_pte(struct mm_struct *dst_mm, struct mm_struct *src_mm,
        pte_t *dst_pte, pte_t *src_pte, struct vm_area_struct *vma,
        unsigned long addr, int *rss)
{
    pte_t pte = *src_pte;
    ......
    // 首先会处理一些页表项不为空但物理页不在内存中的情况（!pte_present(pte)分支）如被swap到交换分区中的页：
    if (unlikely(!pte_present(pte))) {
        ......
    }
    // 接下来处理物理页在内存中的情况
    /*
     * If it's a COW mapping, write protect it both
     * in the parent and the child
     */
    if (is_cow_mapping(vm_flags) && pte_write(pte)) { // 当前页所在的vma是否是私有可写的属性而且父进程页表项是可写：
        ptep_set_wrprotect(src_mm, addr, src_pte); // 设置父进程页表项为只读，使用cas写入。
        pte = pte_wrprotect(pte); // 为子进程设置只读的页表项值，这里还没有真正写到子进程页表项中。
    }
    ...... // 继续拼接子进程页表项的其他域段。
out_set_pte:
    set_pte_at(dst_mm, addr, dst_pte, pte); // 真正写子进程页表项的地方。
    return 0;
}
```

### 64k与4k的区别？

4k粒度下：

​	4级页表，level 3的页表大小为：512（level 0 entrys）* 512（level 1 entrys) * 512 (level 2 entrys) * 512 (level 3 entrys) * 8 Byte(每个entry的大小)  = 2^39 kB

64k粒度下：

​	3级页表，level 3的页表大小为：64（level 1 entrys) * 8k(level 2 entrys) * 8k(level 3 entrys) * 8 Byte(每个entry的大小) = 2^35 kB

### 有没有缓解方式？

1. 从源码中看，clear_page主要是页表的创立，如果页表项少一些会不会缓解？

   ​		vma为用户态的，因为内核态公用一份页表，所以不需要在vma中体现。那么，用户态程序可以使用大页？（貌似使用大页是通过lib库，但是可以通过标记位传递给内核，从而让内核感知。）

   ​		通过改成大页，去跑绑核4进程spawn，发现512M大页使用了9个，性能从1500下降到1200。同时，写带宽相比性能来说并没有降低，而且相对读带宽还提升了20%

2. 


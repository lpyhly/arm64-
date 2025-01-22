---
typora-copy-images-to: ..\images
---

待办事项：

	1. 单核file在不同fs下性能差异的原因分析（x86没有差异）
 	2. osq_lock产生的流程。
 	3. 相关的内核选项梳理。
   	4. struct      address_space 的影响梳理。  

[TOC]

## 影响因素

1620 96核file copy性能差

1. 问题1：osq_lock：跨片时延

   合入新的osq_lock patch（合入主线5.6，wfe睡眠不会一直读）后，1core/1die/1p 都不比intel差；

   2p还差一些，已验证是跨片时延的问题。（同样的核数，跨片比不跨片性能差一倍，与跨片时延数据吻合）。

2. 问题2：find_get_entry：L3 128Byte问题。

3. 1. find_get_entry，看是否能修改_refcount结构体为指针类型，防止修改时会打掉结构体的其他变量。
  
   2. page结构体只能是64B，所以两个page会互相影响。page->_refcount结构体改成指针方案比较困难。
   
   3. 修改struct      address_space的大小，原结构体大小160B， 改成256 Byte对齐，96核提升15%，与intel持平。
   



# 调优手段

​	http://3ms.huawei.com/hi/group/2351/thread_8789804.html?mapId=10616636&for_statistic_from=all_group_forum

### 内核版本

不同内核的数据：

- 4.19内核与4.14内核的差异点。（1620, 1c）

	| 测试项（单核）                        | 4.14内核 | 4.19内核 |
| ------------------------------------- | -------- | -------- |
| File copy 1024 bufsize 2000 maxblocks | 372.8    | 771      |
| File copy 256 bufsize 500 maxblocks   | 237.8    | 527      |
| File copy 4096 bufsize 8000 maxblocks | 1208.5   | 2161     |

- 5.8内核与4.19内核的差异点。（688, 20c）

  | 测试项（20c）                         | 4.19内核 | 5.8内核 |
  | ------------------------------------- | -------- | ------- |
  | File copy 1024 bufsize 2000 maxblocks | 114      | 167     |
  | File copy 256 bufsize 500 maxblocks   |          |         |
  | File copy 4096 bufsize 8000 maxblocks |          |         |
  
  #### 提升4.14内核性能的方式
  
  若单核热点中，发现`file_update_time`热点过高(可以高到30%，肉眼可见)。则可以check一下fs/inode.c文件中，`file_update_time`函数的实现方式，按照如下方式修改。
  
  ``` c
  -	if (IS_I_VERSION(inode))
  +	if (IS_I_VERSION(inode) && inode_iversion_need_inc(inode))
  ```
  
  完整patch：
  
  ``` c
  From 2b551b5737d7c7e90195d0dacbde3b932469b45a Mon Sep 17 00:00:00 2001
  From: root <root@localhost.localdomain>
  Date: Fri, 24 Jul 2020 10:50:59 -0400
  Subject: [PATCH] kernel414_inode_version_optimize_for_unixbench_filecopy
  
  Signed-off-by: root <root@localhost.localdomain>
  ---
   fs/inode.c         |  2 +-
   include/linux/fs.h | 18 ++++++++++++++++++
   2 files changed, 19 insertions(+), 1 deletion(-)
  
  diff --git a/fs/inode.c b/fs/inode.c
  index 61f9134..8419c4d 100644
  --- a/fs/inode.c
  +++ b/fs/inode.c
  @@ -1863,7 +1863,7 @@ int file_update_time(struct file *file)
   	if (!timespec_equal(&inode->i_ctime, &now))
   		sync_it |= S_CTIME;
   
  -	if (IS_I_VERSION(inode))
  +	if (IS_I_VERSION(inode) && inode_iversion_need_inc(inode))
   		sync_it |= S_VERSION;
   
   	if (!sync_it)
  diff --git a/include/linux/fs.h b/include/linux/fs.h
  index 7ba0531..0a8613a 100644
  --- a/include/linux/fs.h
  +++ b/include/linux/fs.h
  @@ -2048,6 +2048,24 @@ static inline void inode_inc_iversion(struct inode *inode)
          spin_unlock(&inode->i_lock);
   }
   
  +#define I_VERSION_QUERIED_SHIFT        (1)                                 
  +#define I_VERSION_QUERIED      (1ULL << (I_VERSION_QUERIED_SHIFT - 1))     
  +#define I_VERSION_INCREMENT    (1ULL << I_VERSION_QUERIED_SHIFT)           
  +/**                                                                           
  + * inode_iversion_need_inc - is the i_version in need of being incremented?   
  + * @inode: inode to check                                                     
  + *                                                                            
  + * Returns whether the inode->i_version counter needs incrementing on the next
  + * change. Just fetch the value and check the QUERIED flag.                   
  + */                                                                           
  +static inline bool                                                            
  +inode_iversion_need_inc(struct inode *inode)                                  
  +{                                                                             
  +//       return (inode->iversion) & I_VERSION_QUERIED;             
  +	return false;
  +}                                                                             
  +
  +
   enum file_time_flags {
   	S_ATIME = 1,
   	S_MTIME = 2,
  -- 
  1.8.3.1
  ```
  
  

### PAC

内核关掉ptrauth功能，否则对于read/write会有返回值校验。

### AUDIT

OS中使用`systemctl disable auditd`关闭audit功能，然后重启。 仅仅是auditctl -e0实测依然有audit函数被调用。

| 文件系统类型 | file1得分 | 备注 |
| ------------ | --------- | ---- |
| 开启auditd   | 2340      |      |
| 关闭auditd   | 2493      |      |

``` bash
# audit 调用栈展示
-    5.04%     1.97%  fstime  [kernel.vmlinux]  [k] __audit_syscall_exit  
   - 3.07% __audit_syscall_exit                                           
        1.41% kfree                                                       
        0.86% unroll_tree_refs                                            
        0.80% path_put                                                    
   + 1.97% 0x4f4e4d4c4b4a4948                                             
-    1.06%     1.06%  fstime  [kernel.vmlinux]  [k] __audit_syscall_entry 
     0x4f4e4d4c4b4a4948                                                   
   - c_test                                                               
      - 0.58% read                                                        
           el0_sync                                                       
           el0_sync_handler                                               
           el0_svc                                                        
           do_el0_svc                                                     
           el0_svc_common.constprop.0                                     
     0.00%     0.00%  perf    [kernel.vmlinux]  [k] __audit_syscall_entry 
```



### 文件系统类型

在openEuler 21.03-1620环境上实测单核绑核fstime：

| 文件系统类型 | file1得分 | 挂载点                                                       |
| ------------ | --------- | ------------------------------------------------------------ |
| tmpfs        | 2493      | mount -t tmpfs tmpfs /tmpfsmnt<br />tmpfs on /tmpfsmnt type tmpfs (rw,relatime) |
| ramfs        | 2489      | mount -t ramfs none /ramfsmnt<br /> // none on /ramfsmnt type ramfs (rw,relatime) |
| ext4         | 1469      | /dev/mapper/openeuler-root on / type ext4 (rw,relatime)      |
|              |           |                                                              |

详细实现细节见《文件系统类型梳理.md》

### spill/WU bypass

​		单核场景下，4096子项的总数据量为read（8MB） + write（8MB），涉及到的地址空间为8M * 3（f -> buffer -> g，内核态->用户态 -> 内核态），因此会发生L2 cache替换，下去的数据到其他L3或者到DDR的选择，需要看后续spill其他L3的时延更长，还是从DDR load数据的时延更长。

​		同时，单次操作4096B可以触发stream write了，也就涉及到WU bypass策略，是直接到ddr还是share到其他L3。

​		从关预取上看，只有在write的时候触发了streamwrite，也仅仅是写到了L3。read的时候没有触发ss。

![image-20221029154148237](D:\at_work\Documents\我的总结文档\images\image-20221029154148237.png)

 # 测试项说明

## 流程源码解析

``` c
 switch (test) {
 case 'c':
     w_test(2);
     r_test(2); // 精简版删掉了读。
     status = c_test(seconds);
     break;
```

以file1为例进行解析：

1. 准备环节（w_test）： 不断write()来写f文件（/tmp/dummy0），每次写一个bufsize大小，使其size到2000KB，写入的内容是“0,1,2，……”；
2. 循环copy环节：
3. 1. 先read(file1, buf, bufsize)从`dummy0`读1k（bufsize）大小内容放到buf中；
   2. 再write(file2,buf,bufsize)从buf中往`dummy1`写1k（bufsize）大小内容。
   3. 到达2000KB（文件大小）后，lseek()返回到dummy0、dummy1的文件起始位置。
4. 测试用例中是按照时间计算，到时间了就发送一个sigalarm退出循环。

##　三个子项的测试方式

```
Usage: fstime [-c|-r|-w] [-b <bufsize>] [-m <max_blocks>] [-t <seconds>] [-d dir]
```

其中，

​	-r为只读，-w为只写，-c为对文件进行先读后写（每次操作bufsize大小）。

​	-d 指定目录，若指定在代码会在一开始就chdir过去，ub中为/tmp。因此会创建/tmp/dummy0和/tmp/dummy1两个文件。

​	-m，-b含义如下。

### **File Copy 1024 bufsize 2000 maxblocks** 

​	bufsize = 1024 

​	文件大小 = max_buffs * bufsize = max_blocks * 1024 = 2000 * 1024 = 2000 kB

​	count_per_buf = bufsize/COUNTSIZE(256) = 4

​	max_buffs = max_blocks * 1024/bufsize

###　**File Copy 256 bufsize 500 maxblocks**

​	bufsize = 256

​	文件大小 = 500 kB

### **File Copy 4096 bufsize 8000 maxblocks**

​	bufsize = 4096

​	文件大小 = 8000 kB



## 热点分析

从波形上看，

​	单核很正常，单次循环在***ns；

​	多核热点主要在qspinlock抢锁过程中。其中`read`很快，大部分时间在`write`上。

​	以1630的10c filecopy 256 Byte为例，`read`用时为316ns，而`write`用时为9200ns。

###　ext4上的热点分析

#### 单核

![image-20200711114301374](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20200711114301374.png)

 	![image-20210209104516133](D:\at_work\Documents\我的总结文档\images\image-20210209104516133.png)

#### 多核

![image-20200711114316035](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20200711114316035.png)



#### vs Intel热点

![image-20200711114401891](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20200711114401891.png)

6148热点： 1716.6分  ./Run -i 1 -c 80 fstime

![image-20200711114425642](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20200711114425642.png)

![image-20200711114452285](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20200711114452285.png)



### ramfs/tmpfs上的热点分析

#### 64k页表 - 单核绑核

1620 openEuler 21.03上（linux-5.10.0-4.17.0.28.oe1.aarch64，libc2.31），内核关闭ptrauth，挂载tmpfs。


##### 函数级分析

``` c
52.66% write                                                                          
 - 46.48% el0_sync                                                                    
    - el0_sync_handler                                                                
       - el0_svc                                                                      
          - do_el0_svc                                                                
             - 41.99% el0_svc_common.constprop.0                                      
                - 36.36% invoke_syscall                                               
                   - 35.98% __arm64_sys_write                                         
                      - ksys_write                                                    
                         - 34.87% vfs_write                                           
                            - 34.20% vfs_write.part.0                                 
                               - 30.70% new_sync_write                                
                                  - 29.32% generic_file_write_iter                    
                                     - 27.24% __generic_file_write_iter               
                                        - 23.21% generic_perform_write                
                                           - 9.58% iov_iter_copy_from_user_atomic     
                                                9.15% __arch_copy_from_user           
                                           - 6.85% shmem_write_begin                  
                                              - shmem_getpage_gfp                     
                                                 - 4.21% find_lock_entry              
                                                    - 1.67% find_get_entry            
                                                         0.73% xas_start              
                                                      1.18% page_cache_get_speculative
                                                   1.08% trylock_page                 
                                                   0.63% mark_page_accessed           
                                             2.62% shmem_write_end                    
                                             1.23% put_page                           
                                             0.93% unlock_page                        
                                             0.63% fault_in_pages_readable            
                                        - 2.04% file_update_time                      
                                             1.64% ktime_get_coarse_real_ts64         
                                       1.08% down_write                               
                                    1.05% up_write                                    
                                 0.55% init_sync_kiocb                                
                                 0.55% file_end_write                                 
                           0.51% __fdget_pos                                          
                - 2.93% syscall_trace_exit                                            
                   - __audit_syscall_exit                                             
                        0.75% kfree                                                   
                - 2.17% syscall_trace_enter                                           
                     1.28% ktime_get_coarse_real_ts64                                 
               3.68% local_daif_restore.constprop.0                                   
45.32% read                                                                           
 - 39.75% el0_sync                                                                    
    - el0_sync_handler                                                                
    - el0_svc                                                                         
       - do_el0_svc                                                                   
          - 34.87% el0_svc_common.constprop.0                                         
             - 28.25% invoke_syscall                                                  
                - 28.08% __arm64_sys_read                                             
                   - 27.98% ksys_read                                                 
                      - 27.20% vfs_read                                               
                         - 23.67% new_sync_read                                       
                            - 21.29% shmem_file_read_iter                             
                               - 12.66% copy_page_to_iter                             
                                  - 12.63% copy_page_to_iter_iovec                    
                                       12.10% __arch_copy_to_user                     
                               - 6.14% shmem_getpage_gfp                              
                                  - 4.21% find_lock_entry                             
                                       1.58% find_get_entry                           
                                       1.23% page_cache_get_speculative               
                                    0.86% trylock_page                                
                               - 1.39% touch_atime                                    
                                  - 1.24% atime_needs_update                          
                                       0.83% ktime_get_coarse_real_ts64               
                              1.15% put_page                                          
                              0.75% unlock_page                                       
                         - 1.23% rw_verify_area                                       
                            - 0.88% security_file_permission                          
                                 fsnotify_perm.part.0                                 
                           0.58% init_sync_kiocb                                      
             - 3.07% syscall_trace_exit                                               
                - __audit_syscall_exit                                                
                     0.70% kfree                                                      
                     0.50% path_put                                                   
             - 2.76% syscall_trace_enter                                              
                  1.78% ktime_get_coarse_real_ts64                                    
            4.05% local_daif_restore.constprop.0                                      
```

1. 函数调用比例

   ​	53%在write()函数中，45%在read()函数中。这俩是直接从lib库调用到内核系统调用中。

   ` 	write -> __arm64_sys_write  -> vfs_write -> generic_file_write_iter `

   ​	write/read函数具体过程详见《file结构体及各种文件操作.md》

   - read操作主要有两个动作： copy_to_user执行内核态到用户态的拷贝。

2. 函数调用频率

   ​	1620上统计，调用频率为70w/s，也就是1微秒会调用0.7次read + 0.7次write。 每次调用路径都不会落盘，read是从内存中读取，且会在cache命中（下面可以看到，L3 miss率仅4%，且只有2M size）；write是写到内存中，也会落到cache中。单次操作为

   ``` shell
   $ ./syscount  -p 9490 -d 1                                   
   Tracing syscalls, printing top 10... Ctrl+C to quit.         
   [09:59:57]                                                   
   SYSCALL                   COUNT                              
   read                     700802                              
   write                    700408                              
   lseek                       700                              
   ```

   

3. 函数调用时延



##### PMU级分析

``` c
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                    35713466425
        Total Execution Cycles:                         23507307516
        Instructions Per Cycle:                               1.519

        Front-End Bound:                                   32.586%
                Front-End Latency:                         25.614%
                        iTLB Miss:                           1.080%
                                L1 iTLB Miss:                1.078%
                                L2 iTLB Miss:                0.002%
                        iCache Miss:                         7.924%
                                L1 iCache Miss:              6.741%
                                L2 iCache Miss:              1.183%
                        BP_Misp_Flush:                       0.652%
                        OoO Rob Flush:                       0.000%
                        Static Predictor Flush:              0.113%
                Front End Bound Bandwidth:                   6.972%

        Bad Speculation:                                     9.103%
                Branch Mispredicts:                          1.882%
                        Indirect Branch:                     1.818%
                        Push Branch:                         0.792%
                        Pop Branch:                          1.030%
                        Other Branch:                        0.064%
                Machine Clears:                              7.221%
                        Nuke Flush:                          1.222%
                        Other Flush:                         5.999%

        Retiring:                                           37.981%

        Back-End Bound:                                     20.329%
                Resource Bound:                             10.081%
                        Sync_stall:                          0.000%
                        Rob_stall:                           0.008%
                        Ptag_stall:                          0.032%
                        SaveOpQ_stall:                       0.000%
                        PC_buffer_stall:                     0.000%
                Core Bound:                                 57.675%
                        Divider:                             0.000%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                     57.675%
                                ALU BRU IssueQ Full:         0.061%
                                LS IssueQ Full:              5.465%
                                FSU IssueQ Full:             0.000%
                Memory Bound:                               21.657%
                        L1 Bound:                           19.227%
                        L2 Bount:                            0.771%
                        Intra Cluster Remote L2 Bound:       0.002%
                        Local LLC Bound:                     0.017%
                        Inter Cluster Local LLC Bound:       0.392%
                        Intra Chip Remote LLC Bound:        -0.242%
                        Inter Chip Remote LLC Bound:         0.494%
                        Intra Chip Local DDR Bound:          0.007%
                        Intra Chip Remote DDR Bound:         0.008%
                        Inter Chip Remote DDR Bound:         0.001%
                        Store Bound:                         0.980%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
L1 Data Miss Ratio     L1 Instructions Miss Ratio    L2 Data Miss Ratio    
    0.964%                      1.091%                   21.448%                              
L2 Instructions Miss Ratio     L2 Total Miss Ratio    LLC Miss Ratio
    1.741%                      20.015%               4.512%                            
```

ipc=1.52比较低，

topdown上看，FE 32%， BE 20%， Badspec 9%， Retire 38%， FE有些高。

BE方面，带宽很低，基本都是L1 bound， L1的 DC miss率1%，还可以；

FE方面，主要bound在FE Latency上， 统计上看L1 IC miss率1%，有些高了。

##### 指令级分析

​	从1630V200-B601-0928的emu波形中分析，最热点的两个函数是：

- `__arch_copy_from_user`
- ``__arch_copy_to_user`` 

这两个函数的指令级实现详见`arch/arm64/lib/copy_to_user.S`和`arch/arm64/lib/copy_from_user.S`，本质上是一致的，调用了

从emu波形中看到的实际trace流及指令commit时延如下（指令后面的就是时延），可以看到：

![image-20221029150906114](D:\at_work\Documents\我的总结文档\images\image-20221029150906114.png)

![image-20221029150715719](D:\at_work\Documents\我的总结文档\images\image-20221029150715719.png)

- copy_from_user的时延基本上是0.3->0.5 ns，基本上都是L1 cache命中的。因为单次操作4k地址，而这4k已经在copy_to_user的函数使用时被load到L1 cache了，4kB*2（内核态4k+用户态4k）是可以被L1 cache 全部hold住的。
- copy_to_user的时延占大头。这里要区分开预取和关预取。开预取场景下，因为地址连续，因此是可以提前预取到L1/L2 cache中的。而关预取场景，总的数据量是32MB，一定是local L2/L3 miss的，那就看从ddr还是其他L3拿数据了。从V200的实际波形上统计，大致有20%从ddr拿数据，80%从其他L3拿的。

![image-20221029154559553](D:\at_work\Documents\我的总结文档\images\image-20221029154559553.png)

##### 代码更新

在5.10版本中，还是会区分ARM64_UAO特性，若实现了该特性，则使用`ldtr*2/sttr*2`指令，若没有实现则使用`ldp/stp`指令。

``` c
// "arch/arm64/include/asm/alternative.h"
		.macro uao_ldp l, reg1, reg2, addr, post_inc
                alternative_if_not ARM64_HAS_UAO
8888:                   ldp     \reg1, \reg2, [\addr], \post_inc;
8889:                   nop;
                        nop;
                alternative_else
                        ldtr    \reg1, [\addr];
                        ldtr    \reg2, [\addr, #8];
                        add     \addr, \addr, \post_inc;
                alternative_endif

                _asm_extable    8888b,\l;
                _asm_extable    8889b,\l;
        .endm
```

但是后面开始无差别使用



#### 64k页表 - 多核绑核

​		从read/write的详细流程上可以看出(《file结构体及各种文件操作.md》).

- read主要流程： 不操作inode锁，仅对struct page进行计数（filemap_get_pages），内容拷贝调用的是copy_to_user，read结束后修改回page->count（folio_put）。

- write主要流程：先获取inode锁，再执行copy_from_user进行内容拷贝，最后释放inode锁。
  - copy_from_user

    以fs1024为例对该函数进行trace，发现每次的size为1k， 内核态的地址一直是不变的，也就是用户态总是copy到了内核的同一个地址域段。

  

而多核热点来看，多核同时read的问题不大，冲突最严重的就是write（95%）中对于inode中读写信号量i_rwsem的osq_lock操作。

``` c
down_write(&inode->i_rwsem) -> __down_write
    
static inline void __down_write(struct rw_semaphore *sem)
{
    long tmp = RWSEM_UNLOCKED_VALUE;

    if (unlikely(!atomic_long_try_cmpxchg_acquire(&sem->count, &tmp,
                              RWSEM_WRITER_LOCKED))) // count变量的cas操作
        rwsem_down_write_slowpath(sem, TASK_UNINTERRUPTIBLE); //真正用到osq_lock的地方
    else
        rwsem_set_owner(sem);//  owner变量的STSET
        				//	atomic_long_set(&sem->owner, (long)current);
}

struct rw_semaphore {
    atomic_long_t count; // 8B
    struct list_head wait_list; // 16B
    raw_spinlock_t wait_lock; // 4B
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* spinner MCS lock */ // 4B
    /*
     * Write owner. Used as a speculative check to see
     * if the owner is running on the cpu.
     */
    struct task_struct *owner; //8B
#endif
};
```

查看*sem的地址，看到了3种偏移量： 0x10, 0x50, 0x90.

### 4k页表 - 单核绑核

在1650的EMU上看：openEuler 22.03 sp2，小文件系统，tmpfs， 2.5GHz+7200 DDR； 

- file4096为6874分，主要topdown为：

  ![image-20240330162039161](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20240330162039161.png)
  - 在backend bound中，L1 bound很高，但是再细分没有看到瓶颈这里，需要去check core bound的分解。
  - 其中，core bound几乎全部在ptag stall，原因可能有2个：本身就是数量不足，另一个释放的慢（后端原因，做的慢了）。1650上ptag为256个（和1630v200一样），其中开smt的话会预留32个给T1。	![image-20240330162122984](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20240330162122984.png)

  - non-serialize这个是非串行（没依赖关系）指令执行时没有exec port，这个一般无法确定是哪的问题。

  



## 内核源码解读

### struct   address_space 

这是pagecache的核心数据结构，每个file都有自己的address_space，每个address_space对应不同的inode，也就是不同的owner（O_DIRECT读，文件读同一部分内容时是不同的inode）

在`generic_file_write_iter -> down_write-> ... -> osq_lock`中，**函数`down_write(&inode->i_rwsem)  `中的参数就是`struct rw_semaphore`。** 

这里，`inode`是`file.f_mapping.host`成员，其中`file.f_mapping`为`struct addreess_space`类型。

``` c
struct address_space {
    struct inode        *host;  /* owner: inode, block_device */     // 0
    struct xarray       i_pages;                                     // 8
    gfp_t           gfp_mask;                                        // 
    atomic_t        i_mmap_writable;
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
    /* number of thp, only for non-shmem files */
    atomic_t        nr_thps;
#endif
    struct rb_root_cached   i_mmap;
    struct rw_semaphore i_mmap_rwsem;
    unsigned long       nrpages;
    unsigned long       nrexceptional;
    pgoff_t         writeback_index;
    const struct address_space_operations *a_ops;
    unsigned long       flags;
    errseq_t        wb_err;
    spinlock_t      private_lock;
    struct list_head    private_list;
    void            *private_data;
} __attribute__((aligned(sizeof(long)))) __randomize_layout;
```



### struct rw_semaphore

其中含有count，wait_lock，owner变量，是与osq_lock有关的同128Byte不同64Byte的变量。

``` c
struct rw_semaphore {
    atomic_long_t count; // 8B
    struct list_head wait_list; // 16B
    raw_spinlock_t wait_lock; // 4B
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* spinner MCS lock */ // 4B
    /*
     * Write owner. Used as a speculative check to see
     * if the owner is running on the cpu.
     */
    struct task_struct *owner; //8B
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

count与owner相隔32B，实测当count的pa为0xa0时，因为count和owner不在同一个cacheline。





### osq_lock

​	调用栈：  ` down_write -> rwsem_down_write_failed -> rwsem_optimistic_spin -> osq_lock`

​	



### 其中的atomic操作

#### 单核

流程是：

1. pagecache_get_page中，先find_get_entry获取page，然后对page进行加锁操作。前者是page->_refcount的atomic_add_unless(cas操作)，后者是page->flags的atomic_long_fetch_or_acquire操作。

2. simple_write_end中，需要将写完的page释放掉，做法是：先unlock_page，再put_page。前者为page->flags的atomic_fetch_andnot操作，后者是page->_refcount的atomic_sub操作。

 

主要有4种操作：

​	增加page计数，减少page计数，获取page，释放page。以后两者为例分析，前两者只是简化版了。

**获取page**

1. 从radix tree中寻找page，这里整个pagecache就是一个radix tree。
2. 若找到，则获取page，当refcount不为0时令page->_refcount++（cas）
3. 检查步骤1,2的page是否相同，若不同则put_page后重新获取.

**释放page**（这里用不到，只会减少page计数，那里没有cas）

1. 先读page->_refcount，然后cas检查并置0（若中间被别人修改了refcount则报错）
2. 移除出pagecache。
3. 释放page。



L3看到的同一条cacheline上原子操作的pattern为：

​	rs->cas -> rs ->  -> rs -> 

关于软件上连续4个同地址atomic操作的问题，对应的软件行为是：

-  4个rs的行为一致，都是用来判断 `page->mapping != mapping`是否成立，mapping指向inode，因为是rcu锁，所以struct page可能会在获取到之后被释放。



page结构体的_refcount和flags变量是在同一个cacheline中。在一次写操作中会进行：

1. cas:  find_get_entry

   ``` c
   find_get_entry
       -> page_cache_get_speculative
       	-> get_page_unless_zero
       		-> page_ref_add_unless(page, 1, 0)
       			-> atomic_fetch_add_unless(atomic_cas操作)
   ```

   作用是获取page（_refcount的atomic_add_unless(包含rs+cas操作))。 

   ``` c
   // include/linux/atomic.h
   // find_get_entry
   //	-> pagecache_get_page
   //       -> atomic_fetch_add_unless
   static inline int atomic_fetch_add_unless(atomic_t *v, int a, int u)
   {
       int c = atomic_read(v); // rs读
       do {
           if (unlikely(c == u))
               break;
       } while (!atomic_try_cmpxchg(v, &c, c + a)); // cas锁操作
       return c;
   }
   
   #define __atomic_try_cmpxchg(type, _p, _po, _n)             \
   ({                                  \
       typeof(_po) __po = (_po);                   \
       typeof(*(_po)) __r, __o = *__po;                \
       __r = atomic_cmpxchg##type((_p), __o, (_n));            \
       if (unlikely(__r != __o))                   \
           *__po = __r;                        \
       likely(__r == __o);                     \
   })
   #define atomic_try_cmpxchg(_p, _po, _n)     __atomic_try_cmpxchg(, _p, _po, _n)
   ```

2. lock_page

 **注意： 一般来说，只有写才会调用这个接口，以保证此时没有其他人在修改。若读的时候发现page不是最新的，需要update时，也会先lock_page的**

**注意2： lock_page这个接口虽然是加锁，但是实际上和锁没有关系，只是调用了ldset的原子操作而已。**

​      在不同的fs中，调用的接口也不一样,lock_page/lock_page_killable等，反正是在find_get_entry后面使用。 无论哪一种形式，都会调用trylock_page(page)

   ``` c
lock_page -> trylock_page
   {
       test_and_set_bit_lock(PG_locked, &page->flags);
   }
   ```

   关于atomic_long_fetch_or_acquire，实际调用 `ldset a x0, x0, v->counter`。具体代码解析如下：

``` c
// atomic_long_fetch_or_acquire
// 第一步：：：asm-generic/atomic-long.h
#define ATOMIC_LONG_PFX(x)  atomic64 ## x
#define ATOMIC_LONG_FETCH_OP(op, mo)                    \
static inline long                          \
atomic_long_fetch_##op##mo(long i, atomic_long_t *l)            \
{                                   \
    ATOMIC_LONG_PFX(_t) *v = (ATOMIC_LONG_PFX(_t) *)l;      \
                                    \
    return (long)ATOMIC_LONG_PFX(_fetch_##op##mo)(i, v);        \
}
ATOMIC_LONG_FETCH_OP(or, _acquire) 
// atomic_long_fetch_or_acquire == atomic64_fetch_or_acquire
// 第二步：：：arch/arm64/include/asm/atomic_lse.h
#define ATOMIC64_FETCH_OP(name, mb, op, asm_op, cl...)          \
static inline long atomic64_fetch_##op##name(long i, atomic64_t *v) \
{                                   \
    register long x0 asm ("x0") = i;                \
    register atomic64_t *x1 asm ("x1") = v;             \
                                    \
    asm volatile(ARM64_LSE_ATOMIC_INSN(             \
    /* LL/SC */                         \
    __LL_SC_ATOMIC64(fetch_##op##name),             \
    /* LSE atomics */                       \
"   " #asm_op #mb " %[i], %[i], %[v]")              \
    : [i] "+r" (x0), [v] "+Q" (v->counter)              \
    : "r" (x1)                          \
    : __LL_SC_CLOBBERS, ##cl);                  \
                                    \
    return x0;                          \
}

#define ATOMIC64_FETCH_OPS(op, asm_op)                  \
    ATOMIC64_FETCH_OP(_relaxed,   , op, asm_op)         \
    ATOMIC64_FETCH_OP(_acquire,  a, op, asm_op, "memory")       \
    ATOMIC64_FETCH_OP(_release,  l, op, asm_op, "memory")       \
    ATOMIC64_FETCH_OP(        , al, op, asm_op, "memory")
ATOMIC64_FETCH_OPS(or, ldset)       
// atomic64_fetch_or_acquire == ldset a x0, x0, v->counter
```

3. copy操作

4. 解锁；

   以simple_write_end  ->  unlock_page为例：

   ``` c
   unlock_page(page) 
       -> clear_bit_unlock_is_negative_byte 
       	-> clear_bit_unlock // 清除PG_locked标志位
       		-> atomic_long_fetch_andnot_release
   ```

   ``` c
   atomic64_fetch_andnot -> atomic_fetch_and ->
       mvn + ldclr(atomic bit clear操作)
   ```

   ​	

5. 释放page

   在generic_file_buffered_read函数中的put_page->put_page_testzero->page_ref_dec_and_test(page)，使用了atomic_dec来判断page-1后是否为0，若为0则需要释放这个page。

因此，1个loop中会出现同cacheline的1个cas + 3个atomic load操作。



####　多核

多核瓶颈主要在osq_lock上，
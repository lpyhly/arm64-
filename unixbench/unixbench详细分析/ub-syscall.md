typora-copy-images-to: ..\images

# syscall

[TOC]

## 测试场景详解

主要测试5个系统调用。

``` c
while (1) {
 close(dup(0));
 getpid();
 getuid();
 umask(022);
 iter++;
}
```

​	dup是复制一个现存的文件描述符。从shell中运行一个进程，默认会有3个文件描述符存在(0、１、2)，0与进程的标准输入相关联，１与进程的标准输出相关联，2与进程的标准错误输出相关联，一个进程当前有哪些打开的文件描述符可以通过/proc/进程ID/fd目录查看。　

## 单进程

### 64K页表

### 4K页表

#### 1630V100样片

测试环境： 123#  openEuler 22.03硬盘文件系统， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

单进程，分数840分，热点在关中断中采不到。

``` shell
 33.39%  [kernel]                    [k] el0_svc_common.constprop.0
  7.08%  [kernel]                    [k] ktime_get_coarse_real_ts64
  4.15%  libc.so.6                   [.] __close
  3.64%  libc.so.6                   [.] __GI___dup
  3.57%  [kernel]                    [k] syscall_trace_enter
```





## 多进程

### 64K页表

#### 1616

在关lse时，1616有snoop delay的形式，会在抢锁的时候降低失败率；而1620没有这个机制，因此抢锁严重的情况下失败率就比较高了。

perf stat -e r6c -e r6d -e r6e -e r6f ./Run -c 96 syscall

其中r6e/r6f = 抢锁失败率。

####1620问题分析

主要问题： 

1. 多核cas性能差
2. 跨P存在目录模糊

64核（1P）时，关lse 1620比1616差340%, 开lse 1620比1616差90%。

8核（核0-7时）： 1620为2104分， 1616为2500分。

另外，实际场景下，统计1620在64核的exclusive失败率，为70%。

在2p时，1620比intel差，因为有4个numa节点，抢锁时会模糊目录，同时跨片时延也长。

#### 688问题分析

主要问题： 

1. 128B falseshare性能差

![image-20220724165147997](D:\at_work\Documents\我的总结文档\images\image-20220724165147997.png)

#### 1630V200问题分析

5.10内核上，统计20c syscall流程中，对于某一条128Byte的cacheline，L3可以看到的操作们：

![image-20220802104901800](D:\at_work\Documents\我的总结文档\images\image-20220802104901800.png)

20c 开预取syscall性能：

| 配置                | 20c syscall得分 |
| ------------------- | --------------- |
| 0715 default        | 104 分          |
| 0715 仅修改unlock   | 108 分          |
| 0715 仅修改snpdelay | 104 分          |
| 0715 两个都修改     | 102 分          |
| 0615 版本           | 102 分          |
| **0515** **版本**   | **116** **分**  |

 从数据上看，主要影响 还是 拍数增加（0515->0615版本差异）。原因分析：

- 0715版本中，会有连续cas失败的情况，应该rs和cas中间，被其他核插入做了修改。 而0615版本，没有far的cas，说明cas成功率很高，也就是前面rs读到的值和后面是匹配的。

- 因此可以推测，由于加拍，导致rs获取的E态还没等到cas就被其他核抢走了。
- 在此场景下，128Byte的falseshare会带来明显影响，因为有前后两个cacheline的RS操作，后面再跟一个CAS，而核的实现是CAS操作为nonflush的，需要等待前面两个RS都完成后才可以做。

### 4k页表



#### 1630V100样片

测试环境： 123#  openEuler 22.03硬盘文件系统， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx8 RDIMM）

160进程，分数4259分

```shell
25.36%  [kernel]                 [k] page_counter_try_charge
22.93%  [kernel]                 [k] __fget_files
14.26%  [kernel]                 [k] page_counter_cancel
11.56%  [kernel]                 [k] filp_close
10.94%  [kernel]                 [k] fput_many
10.63%  [kernel]                 [k] propagate_protected_usage
 1.17%  [kernel]                 [k] el0_svc_common.constprop.0
 0.27%  [kernel]                 [k] ktime_get_coarse_real_ts64
 0.22%  [kernel]                 [k] ptep_clear_flush
```

其中，热点指令为：

```shell
page_counter_try_charge:
       │       mov     x20, x21
  0.01 │       ldaddal x20, x0, [x19]
       │       add     x20, x20, x0
 52.88 │       ldr     x3, [x19, #32]
       │       mov     x0, x19
       │       mov     x1, x20
       │       cmp     x3, x20
 47.06 │     ↓ b.cc    b0
  0.01 │ 60: → bl      propagate_protected_usage
__fget_files:
0.00 │       ldr     w1, [x5, #68]
      │       tst     w22, w1
      │     ↓ b.ne    f4
51.39 │       mov     x4, x5
      │       ldr     x1, [x4, #56]!
 0.00 │ 84: ↑ cbz     x1, 3c
17.90 │       add     x2, x25, x1
      │       nop
      │       nop
      │       mov     x0, x4
      │       mov     x3, x1
      │       casal   x3, x2, [x4]
      │       mov     x0, x3
      │       mov     x3, x0
      │     ↓ b       b0
      │     → b       f_dupfd
30.68 │ b0:   cmp     x1, x3
      │     ↓ b.ne    10c

filp_close:
  0.00 │      ldr     x2, [x0, #56]
       │    ↓ cbz     x2, 84
 71.07 │      stp     x19, x20, [sp, #16]
       │      mov     w21, #0x0                       // #0
       │      mov     x19, x0
       │      ldr     x2, [x0, #40]
       │      mov     x20, x1
  0.72 │      ldr     x2, [x2, #120]
       │    ↓ cbz     x2, 44
       │    → blr     x2
       │      mov     w21, w0
  0.01 │44:   ldr     w0, [x19, #68]
       │    ↓ tbnz    w0, #14, 64
 28.20 │      mov     x1, x20
       │      mov     x0, x19
       │    → bl      dnotify_flush
       │      mov     x1, x20
```

40进程，绑核0-39，分数4802分

````shell
29.53%  [kernel]                    [k] page_counter_try_charge
15.58%  [kernel]                    [k] page_counter_cancel
15.24%  [kernel]                    [k] __fget_files
11.72%  [kernel]                    [k] propagate_protected_usage
 9.79%  [kernel]                    [k] fput_many
 5.20%  [kernel]                    [k] el0_svc_common.constprop.0
 4.53%  [kernel]                    [k] filp_close
 1.18%  [kernel]                    [k] ktime_get_coarse_real_ts64
 0.65%  libc.so.6                   [.] __close
 0.60%  libc.so.6                   [.] __GI___dup
 0.58%  [kernel]                    [k] syscall_trace_enter
 0.55%  libc.so.6                   [.] __GI___umask
 0.54%  libc.so.6                   [.] __GI___getuid
 0.52%  libc.so.6                   [.] __getpid
 0.41%  [kernel]                    [k] __audit_syscall_exit
````

跨P 40进程，绑核0-19,80-99，分数3993分

```shell
 30.62%  [kernel]                                 [k] page_counter_try_charge
 16.42%  [kernel]                                 [k] page_counter_cancel
 14.15%  [kernel]                                 [k] propagate_protected_usage
 13.78%  [kernel]                                 [k] __fget_files
  8.93%  [kernel]                                 [k] fput_many
  4.57%  [kernel]                                 [k] filp_close
  4.38%  [kernel]                                 [k] el0_svc_common.constprop.0
  0.97%  [kernel]                                 [k] ktime_get_coarse_real_ts64
  0.53%  libc.so.6                                [.] __close
  0.48%  libc.so.6                                [.] __GI___dup
```

#### Intel 6248

`测试环境：118# Intel 6248，关SMT，cpu关turbo定频2.5GHz，ddr 2933MHz*12根*16GB， openEuler21.03，linux5.10.0_yhy`



#### AMD 7742

`测试环境： 120# AMD 7742, 关SMT，关turbo，CPU频率2.23GHz定频，ddr 32根*2933MHz * 16GB  `



# 内核函数流程梳理

## 函数级分析（linux 6.6)

1. 整体流程：
   - 区分正常跳转or蹦床跳转，区别处理入口（kernel_ventry鲲鹏硬件上保证了不受影响，所以无需蹦床)；
   - 若非蹦床，则**调整栈指针**sp为sp-PT_REGS_SIZE，然后在通用框架`el\el\ht\()_\regsize\()_\label` 中，先调用`kernel_entry`**构建pt_regs，并使能部分特性；**
   - 然后，根据跳转过来的el & t/h & 异常类型，分别进入不同的handler入口，svc则进入`el0t_64_sync_handler`部分。
     - 在el0t_64_sync_handler中，根据esr的EC类型，分别跳转到不同入口，svc进入el0_svc
     - el0_svc中，将开中断之前的事情处理完，包括trace、debug、fpsimd/sve/sme相关事项，然后**开中断**。
     - do_el0_svc->el0_svc_common，然后判断是走快速路径还是慢速路径。
       - 修改pt_regs中的syscallno和orig_x0；
       - 读取thread flags：
         - 若慢速路径，则先调用syscall_trace_enter；
         - invoke_syscall中，先调用syscall table对应的入口，等具体syscall处理完毕后，将返回值保存回pt_regs中；
       - 再次读取thread flags，
         - 若两次读取都可以走快速路径，则直接返回
         - 任意一次需要走慢速路径，则需要调用syscall_trace_exit处理了；
     - 返回用户态前的操作：
       - **关中断**
       - 检查调度&信号；若需要调度或处理信号，则进行处理；
       - trace/lockdep/MTE等特性的处理。若没有开相关特性则不做操作。
   - 跳转到ret_to_user/ret_to_kernel处理，这里只分析ret_to_user
     - 处理单步，然后进入kernel_exit
     - **从pt_regs中恢复出x0-x30，以及elr/spsr/sp系统寄存器；**
     - 处理pseudo_nmi/PAN/MTE/SSBS/PTRAUTH等特性；
     - eret返回。

2. 重点函数分析：

- **kernel_entry 0 ：** 
  - **在栈空间中构建一个pt_regs用来存放前一个任务的现场**，包括x0-x30，pc/pstate/sp/syscallno/stackframe等。调用完毕后，前一个任务的SP，PC，PSTATE分别存放在x21,x22,x23中。
    - 所有x0-x29通用寄存器入栈；
    - 安全考虑： 清空x0-x29；
  - **使能部分特性**，比如PAC，MTE，PSEUDO NMI等。
    - 安全考虑：sp_el0指针保护，不让内核态轻易访问到这个寄存器。
    - 安全考虑：ssbd特性（修改pstate.ssbs域段），pac特性的初始化使能（修改key register）。
    - elr_el0, spsr入栈，为了防止异常嵌套？	

- **el0_sync_handler**  ： 读取`esr_el1`中的EC值，判断进入到哪个处理分支去。调用关联的异常处理函数；
  - 若ec=0x15， 则进入el0_svc。
  - 
  - do_el0_svc
    - 判断sve状态，需要disable掉el0的sve能力，即控制el0的sve操作需要trap到el2.
    - el0_svc_common，**开中断**，判断svc num是否正确，调用syscall_table[scno]钩子函数来处理。
    - 如果遇到有trace，audit，debug，step等，需要调用trace方式。
- **kernel_exit 0:**
  - 若返回内核态，则需要先关闭daif（返回用户态的话，之前已经关过中断了）
  - pseudo_nmi的处理：恢复异常前的中断优先级；
  - 从pt_regs中率先读取出elr,spsr,sp_el0暂存到x21-x23;
  - 处理PAN/pid_in_contextidr/PTRAUTH/SSBS/MTE等特性；
  - 从pt_regs中恢复x0->x30，以及系统寄存器elr,spsr，再根据是否需要蹦床来恢复sp_el0/lr为不同的值。
  - eret返回。



## 代码级分析

结合linux 6.6 内核来分析。

### 背景知识

#### pt_regs

陷入异常时，内核中需要保存的信息都在`pt_regs`中写着了。

```c
/*                                                                            
 * This struct defines the way the registers are stored on the stack during an
 * exception. Note that sizeof(struct pt_regs) has to be a multiple of 16 (for
 * stack alignment). struct user_pt_regs must form a prefix of struct pt_regs.
 */                                                                           
struct pt_regs {                                                              
        union {                                                               
                struct user_pt_regs user_regs;                                
                struct {                                                      
                        u64 regs[31];                                         
                        u64 sp;    
                        u64 pc;    
                        u64 pstate;                                           
                };                                                            
        };                                                                    
        u64 orig_x0; 
#ifdef __AARCH64EB__                                                          
        u32 unused2;                                                          
        s32 syscallno;                                                        
#else                                                                         
        s32 syscallno;                                                        
        u32 unused2;                                                          
#endif                                                                        
        u64 sdei_ttbr1;                                                       
        /* Only valid when ARM64_HAS_GIC_PRIO_MASKING is enabled. */          
        u64 pmr_save;                                                         
        u64 stackframe[2];                                                    
                                                                              
        /* Only valid for some EL1 exceptions. */                             
        u64 lockdep_hardirqs;                                                 
        u64 exit_rcu;                                                         
};                    
struct user_pt_regs {                                        
        __u64           regs[31];       // x0-x30
        __u64           sp;                   
        __u64           pc;       // 也可能是elr or lr
        __u64           pstate;   // 也可能是spsr
};    
```



### 开头通用部分

#### kernel_ventry

"arch/arm64/kernel/entry.S"

进入内核后，每一个entry都会通过`kernel_ventry`的宏来引导进入到对应入口:

```asm
SYM_CODE_START(vectors)                                                
    kernel_ventry   1, t, 64, sync      // Synchronous EL1t            
    kernel_ventry   1, t, 64, irq       // IRQ EL1t                    
    kernel_ventry   1, t, 64, fiq       // FIQ EL1t                    
    kernel_ventry   1, t, 64, error     // Error EL1t                  
                                                                       
    kernel_ventry   1, h, 64, sync      // Synchronous EL1h            
    kernel_ventry   1, h, 64, irq       // IRQ EL1h                    
    kernel_ventry   1, h, 64, fiq       // FIQ EL1h                    
    kernel_ventry   1, h, 64, error     // Error EL1h                  
                                                                       
    kernel_ventry   0, t, 64, sync      // Synchronous 64-bit EL0      
    kernel_ventry   0, t, 64, irq       // IRQ 64-bit EL0              
    kernel_ventry   0, t, 64, fiq       // FIQ 64-bit EL0              
    kernel_ventry   0, t, 64, error     // Error 64-bit EL0            
                                                                       
    kernel_ventry   0, t, 32, sync      // Synchronous 32-bit EL0      
    kernel_ventry   0, t, 32, irq       // IRQ 32-bit EL0              
    kernel_ventry   0, t, 32, fiq       // FIQ 32-bit EL0              
    kernel_ventry   0, t, 32, error     // Error 32-bit EL0            
SYM_CODE_END(vectors)                                                  
```

`kernel_ventry`这个宏，对el0跳转过来的，第一条就会编译出b指令，原因是要区分kernel_ventry以及tramp_ventry两种场景。若需要tramp，则会走tramp_ventry，然后直接跳到kernel_ventry的第二条指令去执行（修改x30，然后ret）。这么操作是考虑到大小核场景，即tramp和不需要tramp的核共存的时候，怎么去保证切换的顺畅性。若不是el0跳转过来的或者是tramp跳转过来的，则从`tpidrro_el0`中恢复x30寄存器（在tramp_ventry中临时把x30保存到这个系统寄存器中的），就完成其使命了。

```asm
    .macro kernel_ventry, el:req, ht:req, regsize:req, label:req                
    .align 7                                                                    
.Lventry_start\@:                                                               
    .if \el == 0                                                                
    /*                                                                          
     * This must be the first instruction of the EL0 vector entries. It is      
     * skipped by the trampoline vectors, to trigger the cleanup.               
     */                                                                         
    b   .Lskip_tramp_vectors_cleanup\@                                          
    .if \regsize == 64                                                          
    mrs x30, tpidrro_el0     # read-only software thread ID reg
    msr tpidrro_el0, xzr                                                        
    .else                                                                       
    mov x30, xzr                                                                
    .endif                                                                      
.Lskip_tramp_vectors_cleanup\@:                                                 
    .endif
    
   sub sp, sp, #PT_REGS_SIZE    #修改栈指针，方便后面计算。
 #ifdef CONFIG_VMAP_STACK                             
  .....
 #endif
  b   el\el\ht\()_\regsize\()_\label   # 跳转到对应函数入口
```

#### el\el\ht\()_\regsize\()_\label 

然后，这个函数入口，又做了一层封装。其中包括3步：

1. kernel_entry保存pt_regs
2. 将栈指针mov到x0；（？？？为什么？？？）
3. 跳转到XX_handler这个c语言函数入口去处理；
4. ret_to_user or ret_kernel

```asm
SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)                             
    kernel_entry \el, \regsize                                                   
    mov x0, sp                                                                   
    bl  el\el\ht\()_\regsize\()_\label\()_handler                                
    .if \el == 0                                                                 
    b   ret_to_user                                                              
    .else                                                                        
    b   ret_to_kernel                                                            
    .endif                                                                       
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)                                     
    .endm                                                                        
```

##### kernel_entry

其中，`kernel_entry`部分：

**作用是：**

1. **在栈空间中构建一个pt_regs用来存放前一个任务的现场**，包括x0-x30，pc/pstate/sp/syscallno/stackframe等。调用完毕后，前一个任务的SP，PC，PSTATE分别存放在x21,x22,x23中。
2. **使能部分特性**，比如PAC，MTE，PSEUDO NMI等。



详细动作是：

1. DIT特性判断

```asm
.macro  kernel_entry, el, regsize = 64                                          
     .if \el == 0                                                                    
     alternative_insn nop, SET_PSTATE_DIT(1), ARM64_HAS_DIT   #数据独立计时
     .endif                                                                          
     .if \regsize == 32                                                              
     mov w0, w0              // zero upper 32 bits of x0                             
     .endif                                                                        
```

2. **保存x0-x29通用寄存器**，入栈。(注意，x29为FP/栈底指针，x30为LR，x31为xzr不能修改。sp不是通用寄存器)
   - 为什么不保存浮点寄存器？ -- 不建议内核使用浮点寄存器，若非要使用，则kernel_neon_begin()/kernel_neon_end()来先保存再使用。当然，在进程切换的时候，是需要进行浮点寄存器的保存恢复的。

3. 若从用户空间进入，

   - **将x0-x29全部清零（clear_gp_regs）**

   - 将用户栈指针SP_EL0放到x21；

   - **将当前进程的entry_task结构体指针保存到x28寄存器（**这里的tsk就是x28），x20只是作为tmp使用，entry_task是struct task_struct结构体指针，每个进程一个，进程号由TPIDR_EL0寄存器获取。

   - 用sp_el0来存放tsk地址（旧内核也有做，详见步骤10）；

   -  disable单步调试（x19，x20均为tmp）

   - ptrauth特性使能时+，若当前sctlr的enia是disabled，则无需更换key，只需要msr使能sctlr就可以（说明用户态中没有换过ia key）；若是enabled，则需要更换key（仅换ia key就可以），但sctlr可以不用再修改了。（因为存在EL切换，允许不同EL状态下使用不同的策略及key。）

   - apply_ssbd：ssbs特性使能时，若非硬件防护，而是firmware防护，则需要挂一下对应的callback函数。

   - MTE特性（mte_set_kernel_gcr）、SHADOW_CALL_STACK特性（scs_load）先不care。

```asm
  
     stp x0, x1, [sp, #16 * 0]                                                       
     stp x2, x3, [sp, #16 * 1]                                                       
     stp x4, x5, [sp, #16 * 2]                                                       
     stp x6, x7, [sp, #16 * 3]                                                       
     stp x8, x9, [sp, #16 * 4]                                                       
     stp x10, x11, [sp, #16 * 5]                                                     
     stp x12, x13, [sp, #16 * 6]                                                     
     stp x14, x15, [sp, #16 * 7]                                                     
     stp x16, x17, [sp, #16 * 8]                                                     
     stp x18, x19, [sp, #16 * 9]                                                     
     stp x20, x21, [sp, #16 * 10]                                                    
     stp x22, x23, [sp, #16 * 11]                                                    
     stp x24, x25, [sp, #16 * 12]                                                    
     stp x26, x27, [sp, #16 * 13]                                                    
     stp x28, x29, [sp, #16 * 14]                                                    
                                                                                     
     .if \el == 0                                                                    
     clear_gp_regs           // 将x0-x29全部清零
     mrs x21, sp_el0                                                                 
     ldr_this_cpu    tsk, __entry_task, x20                                          
     msr sp_el0, tsk                                                                 
                                                                                     
     /*                                                                              
      * Ensure MDSCR_EL1.SS is clear, since we can unmask debug exceptions           
      * when scheduling.                                                             
      */                                                                             
     ldr x19, [tsk, #TSK_TI_FLAGS]                                                   
     disable_step_tsk x19, x20                                                       
                                                                                     
     /* Check for asynchronous tag check faults in user space */                     
     ldr x0, [tsk, THREAD_SCTLR_USER]                                                
     check_mte_async_tcf x22, x23, x0          // 没有使能MTE时，为nop         
                                                                                     
 #ifdef CONFIG_ARM64_PTR_AUTH                                                        
 alternative_if ARM64_HAS_ADDRESS_AUTH                                               
     /*                                                                              
      * Enable IA for in-kernel PAC if the task had it disabled. Although            
      * this could be implemented with an unconditional MRS which would avoid        
      * a load, this was measured to be slower on Cortex-A75 and Cortex-A76.         
      *                                                                              
      * Install the kernel IA key only if IA was enabled in the task. If IA          
      * was disabled on kernel exit then we would have left the kernel IA            
      * installed so there is no need to install it again.                           
      */                                                                             
     tbz x0, SCTLR_ELx_ENIA_SHIFT, 1f                                                
     __ptrauth_keys_install_kernel_nosync tsk, x20, x22, x23                         
     b   2f                                                                          
 1:                                                                                  
     mrs x0, sctlr_el1                                                               
     orr x0, x0, SCTLR_ELx_ENIA                                                      
     msr sctlr_el1, x0                                                               
 2:                                                                                  
 alternative_else_nop_endif                                                          
 #endif      
      apply_ssbd 1, x22, x23                                                   
                                                                              
     mte_set_kernel_gcr x22, x23                                              
                                                                              
     /*                                                                       
      * Any non-self-synchronizing system register updates required for       
      * kernel entry should be placed before this point.                      
      */                                                                      
 alternative_if ARM64_MTE                                                     
     isb                                                                      
     b   1f                                                                   
 alternative_else_nop_endif                                                   
 alternative_if ARM64_HAS_ADDRESS_AUTH                                        
     isb                                                                      
 alternative_else_nop_endif                                                   
 1:                                                                           
                                                                              
     scs_load_current     // 若没使能该特性，则编译为空。                                                    
```

4. 若从内核态进入，则：

- 用x21存放用户栈指针，因为异常之前已经在el1了，有一份自己的pt_regs，因此用户栈指针是在pt_regs之前的另一个pt_regs。栈中存放的全部内容就是由struct pt_regs来标识，因此跳过一个pt_regs的size就足够了。

- （get_current_task）从sp_el0中获取当前的任务结构体放到tsk(x28)中。因为在第一次进入内核空间时已经通过SP_EL0存放tsk了。

```asm
.else                                 
add x21, sp, #PT_REGS_SIZE            
get_current_task tsk                  
.endif /* \el == 0 */                 
```

5. 剩余pt_regs寄存器的处理及保存（pc,pstate, stackframe, syscallno)

   - **保存lr/sp:**  首先读取elr_el1/spsr_el1到x22，x23中，方便后面使用，然后保存lr/sp_el0(目前存放在x21中)到pt_regs[LR, SP]中,这里是EL1的LR，SP寄存器，其中LR就是x30寄存器，之前没有保存过。 这里没及时将elr和spsr入栈的原因是，后面一些安全特性会对其进行处理，因此在处理完后才会入栈。这里的S_LR域段存储的是sp_el0，而不是当前的sp寄存器的值，二者之间相差PT_REGS_SIZE.

     （这里硬件已经清空PSTATE，UAO(user access only， 与PAN配合使用)了，包含PSTATE中的daif域段，因此不需要再调用local_daif_mask来关闭daif了）

   - **保存stackframe**：若用户态则不care，若内核态则保存FP和elr_el1（目前存放在x22）用来backtrace使用。

   - 更新栈底指针FP/x29。此时栈中已经存放了一份完整的pt_regs，栈是向低地址生长的，因此栈底指针为sp+sizeof(pt_regs)。这里仅用来dump pt_regs时使用。

   - 若支持SW TTBR0 PAN特性（privilleged access never, 软件的特性，与arm v8.1的PAN特性不同，后者是任何时候都禁止TTBR0_EL1的访问）
   
     - 若支持arm v8.1的硬件PAN特性，则不需要处理了。否则：
   
       - 内核态过来的？ 则关闭SPSR中的TTBR0 PAN的使能开关，后面的内核态就不能访问TTBR0_EL1寄存器了；同时_uaccess_ttbr0_disable中修改了ttbr0_el1的值。
   
       - 用户态过来的？不用关闭，因为本来就有TTBR0_EL1的访问权限。
   
   - **保存pc/pstate**: 安全特性处理完了，现在可以**保存返回后的PC和PSTATE的值**（其实是elr_el1, spsr_el1）入栈了。
   
   - **保存syscallno**：若从用户态过来，先给个初始syscall num表示默认情况下进入异常的原因不是svc调用，后面若发现确实是调用svc，则这个num会被覆盖。
   
   - 保存pmr: 若使能了pseudo_nmi特性，且gic支持prio_masking特性，则保存`SYS_ICC_PMR_EL1`的值到pt_regs中，同时将SYS_ICC_PMR_EL1赋予新值。
   
   ```asm
   mrs x22, elr_el1                                                      
   mrs x23, spsr_el1                                                     
stp lr, x21, [sp, #S_LR]                                              
                                                                         
   /*                                                                    
    * For exceptions from EL0, create a final frame record.              
    * For exceptions from EL1, create a synthetic frame record so the    
    * interrupted code shows up in the backtrace.                        
    */                                                                   
   .if \el == 0                                                          
   stp xzr, xzr, [sp, #S_STACKFRAME]     // 应该存放栈尾指针stackframe的位置
   .else                                                                 
   stp x29, x22, [sp, #S_STACKFRAME]                                     
   .endif                                                                
   add x29, sp, #S_STACKFRAME     //更新FP指针
   #ifdef CONFIG_ARM64_SW_TTBR0_PAN      // SW_PAN特性处理
   alternative_if_not ARM64_HAS_PAN  
       bl  __swpan_entry_el\el      
   alternative_else_nop_endif                                                 
   #endif
   
       stp x22, x23, [sp, #S_PC]                                              
                                                                 
       /* Not in a syscall by default (el0_svc overwrites for real syscall) */
       .if \el == 0                                                           
       mov w21, #NO_SYSCALL                                                   
       str w21, [sp, #S_SYSCALLNO]                                            
       .endif          
                                                                                  
   #ifdef CONFIG_ARM64_PSEUDO_NMI                                             
   alternative_if_not ARM64_HAS_GIC_PRIO_MASKING                              
       b   .Lskip_pmr_save\@                                                  
   alternative_else_nop_endif                                                 
                                                                              
       mrs_s   x20, SYS_ICC_PMR_EL1                                           
       str x20, [sp, #S_PMR_SAVE]                                             
       mov x20, #GIC_PRIO_IRQON | GIC_PRIO_PSR_I_SET                          
       msr_s   SYS_ICC_PMR_EL1, x20   //重新设置中断优先级。
                                                                              
   .Lskip_pmr_save\@:                                                         
   #endif     //pseudo_nmi
       /*                                                                     
        * Registers that may be useful after this macro is invoked:           
        *                                                                     
        * x20 - ICC_PMR_EL1                                                   
        * x21 - aborted SP                                                    
        * x22 - aborted PC                                                    
        * x23 - aborted PSTATE                                                
       */                                                                     
       .endm        
   ```
   
   

### el0t_64_sync_handler

"arch/arm64/kernel/entry-common.c"

 读取`esr_el1`中的EC值，判断进入到哪个处理分支去。调用关联的异常处理函数；

- 若ec=0x15， 则进入el0_svc。

#### el0_svc

作用：开中断之前必须做的事情处理完，如FP/SVE/SME寄存器状态，为do_el0_svc做准备。

详细动作包括：

1. user_mode过来时的一些debug/trace的记录，
   - debug_lock若开启：记录什么锁动作。
   - track_context若开启：先获取当前进程track的状态，然后记录一个用户态退出；
   - trace_irqflags若开启：记录一次关中断的动作。
   - mte若开启：没细看。

2. sme/sve状态的一些处理
   - 若使能了SME，则退出stream mode；
   - 若使能了SVE，且task使用了sve寄存器，则清空不与fpsimd共用的部分；原因是，在不支持sve的场景中是不需要保留的？？（感觉强词夺理了）
   - fpsimd寄存器是否使用，会在ctx switch的时候进行判断。内核中会默认保留fpsimd寄存器。在使用的时候，先保存再使用。

```c
static void noinstr el0_svc(struct pt_regs *regs) 
{                                                 
        enter_from_user_mode(regs);  // 若debug锁/trace进程/trace irq没开，则没有什么操作。
        cortex_a76_erratum_1463225_svc_handler();  //A76的bug
        fp_user_discard(); //
        local_daif_restore(DAIF_PROCCTX);  // 尽早开中断，把可以开中断处理的都放在do_el0_svc中。
        do_el0_svc(regs);  // 直接调用el0_svc_common函数
        exit_to_user_mode(regs);
}                                                 
```

```c
static __always_inline void __enter_from_user_mode(void)
{                                                       
        lockdep_hardirqs_off(CALLER_ADDR0);  // 若debug_locks=0则直接返回           
        CT_WARN_ON(ct_state() != CONTEXT_USER); // WARN_ON. 获取当前进程track的状态，通过percpu变量 context_tracking.state来读取。若未使能context_tracking，则状态为CONTEXT_DISABLED
        user_exit_irqoff(); // 若context_tracking使能则记录一个用户态退出。
        trace_hardirqs_off_finish(); // 若开了TRACE_IRQFLAGS,则trace中记录一次关中断的动作。
        mte_disable_tco_entry(current);  //mte相关操作
}                                                       
```

```c
static inline void fp_user_discard(void)                                        
{                                                                               
        /*                                                                      
         * If SME is active then exit streaming mode.  If ZA is active          
         * then flush the SVE registers but leave userspace access to           
         * both SVE and SME enabled, otherwise disable SME for the              
         * task and fall through to disabling SVE too.  This means              
         * that after a syscall we never have any streaming mode                
         * register state to track, if this changes the KVM code will           
         * need updating.                                                       
         */                                                                     
        if (system_supports_sme())  // 若CONFIG_ARM64_SME使能并且硬件支持SME
                sme_smstop_sm();   // SYS_SVCR_SMSTOP_SM_EL0寄存器写0      
                                                                                
        if (!system_supports_sve()) // 若CONFIG_ARM64_SVE使能并且硬件支持SVE，则要判断SVE状态
                return;  
                                                       
        if (test_thread_flag(TIF_SVE)) { //若SVE寄存器已经被使用，则需要清空
                unsigned int sve_vq_minus_one;                                  

                sve_vq_minus_one = sve_vq_from_vl(task_get_sve_vl(current)) - 1;
                sve_flush_live(true, sve_vq_minus_one);  //将sve寄存器中不与fpsimd共用的部分全部清零。（所以在进入内核态前，sve寄存器的状态一定要用户态自己保存）
        }                                                                       
}   
```

exit_to_user_mode:

1.  关中断；
2. 检查调度&信号，是内核返回用户态前夕的抢占点；若需要调度或处理信号，则进行处理
3. trace/lockdep/MTE等特性的处理。

```c
static __always_inline void exit_to_user_mode(struct pt_regs *regs)  
{                                                                    
        exit_to_user_mode_prepare(regs); // 判断是否有需要处理的其他work，比如pending的信号、调度（内核态返回用户态前夕会检查一次调度，无论是否开抢占，就是在这里）、浮点寄存器的恢复、uprobe、mte等等。如果有，就跳到do_notify_resume处理。
        mte_check_tfsr_exit();  // MTE相关     
        __exit_to_user_mode();                                       
} 

#define _TIF_WORK_MASK          (_TIF_NEED_RESCHED | _TIF_SIGPENDING | \         
                                 _TIF_NOTIFY_RESUME | _TIF_FOREIGN_FPSTATE | \   
                                 _TIF_UPROBE | _TIF_MTE_ASYNC_FAULT | \          
                                 _TIF_NOTIFY_SIGNAL)                             
static __always_inline void exit_to_user_mode_prepare(struct pt_regs *regs) 
{                                        
        unsigned long flags;                                                
                                                                            
        local_daif_mask();  // 关中断
                                                
        flags = read_thread_flags(); 
        if (unlikely(flags & _TIF_WORK_MASK))                               
                do_notify_resume(regs, flags); //判断thread_info中是否有相关置位，处理完全部置位后才返回      
        lockdep_sys_exit();   //若CONFIG_LOCKDEP未置位，则为空函数。
} 

static __always_inline void __exit_to_user_mode(void)   
{                                                       
        trace_hardirqs_on_prepare();  // 若开了TRACE_IRQFLAGS, trace中记录一次关中断的动作。 
        lockdep_hardirqs_on_prepare(); // 若开了PROVE_LOCKING，才进行处理
        user_enter_irqoff();  // 若开了context_tracking_enabled （CONFIG_CONTEXT_TRACKING_USER不使能则不会开），则在context tracking中记录当前cpu正在进入user态。具体来说，是通过__ct_user_enter调用context_tracking_user_enter->user_enter()来track进程切换，具体来说，是调用context_tracking->__context_tracking_enter来记录进程： 1） 此处切换到了user态，后面的运行为user态；2） 统计此前在内核态的总的运行时间，记录在struct task的vtime结构体中，单位是ns，但实际上用的linux的HZ timer来进行更新的。同时，令用户态的起始时间stime=0，以便退出用户态时统计用户态时间。中间包含了PMR，PAN, PAC, MTE，SSBS特性的处理，以及Cortex系列的ERRATUM，均按下不表。
        lockdep_hardirqs_on(CALLER_ADDR0);   // 若开了PROVE_LOCKING，才进行处理            
}                                                       
```

#####do_svc_common

作用：去syscall table中寻找对应系统调用号的处理函数

具体来说：

1. 修改栈上pt_regs中的syscallno以及orig_x0域段为真实的系统调用号；
2. 查看flag，判断syscall走快速模式还是慢速模式。
3. 查表并跳转到真正的系统函数。
   - svc号可以在include/uapi/asm-generic/unistd.h查询。
4. 

```c
void do_el0_svc(struct pt_regs *regs)                                         
{                                                                             
        el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);   
}                                                                             
static void el0_svc_common(struct pt_regs *regs, int scno, int sc_nr,   
                           const syscall_fn_t syscall_table[])          
{                                                                       
        unsigned long flags = read_thread_flags();                      
                                                                        
        regs->orig_x0 = regs->regs[0];                                  
        regs->syscallno = scno;
	    if (flags & _TIF_MTE_ASYNC_FAULT) {   // MTE相关，先pass  
               syscall_set_return_value(current, regs, -ERESTARTNOINTR, 0); 
               return;                                                      
        }                                                                    
                                                                            
        if (has_syscall_work(flags)) {     // 查看flag中是否由_TIF_SYSCALL_WORK标志位，若这个标志位置位，则走慢速模式。这个标志位被置位的情况有： 开syscall_trace， 开audit，ftrace中开了syscall tracepoint， 开seccomp，no hz模式， 开syscall模拟。
  /*          	Kernel何时会进入nohz模式？
	1）  Kernel 处于空闲时，也就是cpu_idle()中；
	2）  中断处理函数返回时，也就是irq_exit()；
	syscall_trace_enter函数会去处理syscall trace和audit的情况。开AUDIT的时候会调用audit_syscall_entry进行审核。
*/
                if (scno == NO_SYSCALL)                                          
                        syscall_set_return_value(current, regs, -ENOSYS, 0);     
                scno = syscall_trace_enter(regs);                                
                if (scno == NO_SYSCALL)                                          
                        goto trace_exit;                                         
        }                                                                        
                                                                                 
        invoke_syscall(regs, scno, sc_nr, syscall_table);  // invoke_syscall->__invoke_syscall调用syscall_table数组的第scno个指针，指向对应的系统调用函数。对应函数可以在include/uapi/asm-generic/unistd.h中查询。                      
                                                                                 
        /*                                                                       
         * The tracing status may have changed under our feet, so we have to     
         * check again. However, if we were tracing entry, then we always trace  
         * exit regardless, as the old entry assembly did.                       
         */                                                                      
        if (!has_syscall_work(flags) && !IS_ENABLED(CONFIG_DEBUG_RSEQ)) {     
            //若flag中TIF_SYSCALL_WORK不置位，则走快速模式
                flags = read_thread_flags();                                     
                if (!has_syscall_work(flags) && !(flags & _TIF_SINGLESTEP))  
                    //if分支是为了防止过程中trace status会被改变，因此要再测试一遍。正常来说，都会直接在if中return，若trace status被改变，则继续到下一步，syscall_trace_exit。
                        return;                                                  
        } 
trace_exit:                             
        syscall_trace_exit(regs);       
}  
```

##### invoke_syscall

作用：

1. __invoke_syscall调用syscall_table数组的第scno个指针，指向对应的系统调用函数。对应函数可以在include/uapi/asm-generic/unistd.h中查询。
2. 将返回值保存到`pt_regs`栈中。

```c
static void invoke_syscall(struct pt_regs *regs, unsigned int scno,                   
                           unsigned int sc_nr,                                        
                           const syscall_fn_t syscall_table[])                        
{                                                                                     
        long ret;                                                                     
                                                                                      
        add_random_kstack_offset();                                                   
                                                                                      
        if (scno < sc_nr) {                                                           
                syscall_fn_t syscall_fn;                                              
                syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];          
                ret = __invoke_syscall(regs, syscall_fn);                             
        } else {                                                                      
                ret = do_ni_syscall(regs, scno);                                      
        }                                                                             
                                                                                      
        syscall_set_return_value(current, regs, 0, ret);    
                                                                                      
        /*                                                                            
         * Ultimately, this value will get limited by KSTACK_OFFSET_MAX(),            
         * but not enough for arm64 stack utilization comfort. To keep                
         * reasonable stack head room, reduce the maximum offset to 9 bits.           
         *                                                                            
         * The actual entropy will be further reduced by the compiler when            
         * applying stack alignment constraints: the AAPCS mandates a                 
         * 16-byte (i.e. 4-bit) aligned SP at function boundaries.                    
         *                                                                            
         * The resulting 5 bits of entropy is seen in SP[8:4].                        
         */                                                                           
        choose_random_kstack_offset(get_random_u16() & 0x1FF);                        
} 

static long __invoke_syscall(struct pt_regs *regs, syscall_fn_t syscall_fn)       
{                                                                                 
        return syscall_fn(regs);                                                  
}                                                                                 
```



### 结尾框架部分

#### ret_to_user

就做了2件事：

1. 对单步调试场景进行判断及处理；

 	2. kernel_exit

```asm
SYM_CODE_START_LOCAL(ret_to_user)                                                
    ldr x19, [tsk, #TSK_TI_FLAGS]   // re-check for single-step                  
    enable_step_tsk x19, x2                                                      
#ifdef CONFIG_GCC_PLUGIN_STACKLEAK                                               
    bl  stackleak_erase_on_task_stack                                            
#endif                                                                           
    kernel_exit 0                                                                
SYM_CODE_END(ret_to_user)                                                        
```

`kernel_exit`是大头，里面包含了很多事情。

1. 若返回内核态，则需要先关闭daif（返回用户态的话，之前已经关过中断了）
2. pseudo_nmi的处理：恢复异常前的中断优先级；
3. 从pt_regs中率先读取出elr,spsr,sp_el0暂存到x21-x23;
4. 处理PAN/pid_in_contextidr/PTRAUTH/SSBS/MTE等特性；
5. 从pt_regs中恢复x0->x30，以及系统寄存器elr,spsr，再根据是否需要蹦床来恢复sp_el0/lr为不同的值。
6. eret返回。

```asm
    .macro  kernel_exit, el                                             
    .if \el != 0                                                        
    disable_daif                                                        
    .endif                                                              
                                                                        
#ifdef CONFIG_ARM64_PSEUDO_NMI                                          
alternative_if_not ARM64_HAS_GIC_PRIO_MASKING                           
    b   .Lskip_pmr_restore\@                                            
alternative_else_nop_endif                                              
                                                                        
    ldr x20, [sp, #S_PMR_SAVE]                                          
    msr_s   SYS_ICC_PMR_EL1, x20  // 恢复之前的中断优先级设置
                                                                        
    /* Ensure priority change is seen by redistributor */               
alternative_if_not ARM64_HAS_GIC_PRIO_RELAXED_SYNC                      
    dsb sy                       
alternative_else_nop_endif                                              
                                                                        
.Lskip_pmr_restore\@:                                                   
#endif                                                                  
                                                                        
    ldp x21, x22, [sp, #S_PC]       // load ELR, SPSR                   
                                                                        
#ifdef CONFIG_ARM64_SW_TTBR0_PAN                                        
alternative_if_not ARM64_HAS_PAN                                        
    bl  __swpan_exit_el\el                                              
alternative_else_nop_endif                                              
#endif                                                                  
                                                                        
    .if \el == 0                                                        
    ldr x23, [sp, #S_SP]        // load return stack pointer            
    msr sp_el0, x23                                                     
    tst x22, #PSR_MODE32_BIT        // native task?                     
    b.eq    3f                                                          
                                                                        
#ifdef CONFIG_ARM64_ERRATUM_845719                                      
alternative_if ARM64_WORKAROUND_845719                                  
#ifdef CONFIG_PID_IN_CONTEXTIDR                                         
    mrs x29, contextidr_el1                                             
    msr contextidr_el1, x29                                             
#else                                                                   
    msr contextidr_el1, xzr                                             
#endif                                                                  
alternative_else_nop_endif                                              
#endif                                                                  
3:                                                                      
    scs_save tsk   // 与SHADOW页表相关。
    /* Ignore asynchronous tag check faults in the uaccess routines */
    ldr x0, [tsk, THREAD_SCTLR_USER]                                  
    clear_mte_async_tcf x0  // mte相关
                                                                      
#ifdef CONFIG_ARM64_PTR_AUTH      
alternative_if ARM64_HAS_ADDRESS_AUTH                                 
    /*                                                                
     * IA was enabled for in-kernel PAC. Disable it now if needed, or 
     * alternatively install the user's IA. All other per-task keys an
     * SCTLR bits were updated on task switch.                        
     *                                                                
     * No kernel C function calls after this.                         
     */                                                               
    tbz x0, SCTLR_ELx_ENIA_SHIFT, 1f                                  
    __ptrauth_keys_install_user tsk, x0, x1, x2                       
    b   2f                                                            
1:                                                                    
    mrs x0, sctlr_el1                                                 
    bic x0, x0, SCTLR_ELx_ENIA                                        
    msr sctlr_el1, x0                                                 
2:                                                                    
alternative_else_nop_endif                                            
#endif                                                                
                                                                      
    mte_set_user_gcr tsk, x0, x1                                      
                                                                      
    apply_ssbd 0, x0, x1                                              
    .endif                                                            
                                                                      
    msr elr_el1, x21            // set up the return data             
    msr spsr_el1, x22                                                 
    ldp x0, x1, [sp, #16 * 0]                                         
    ldp x2, x3, [sp, #16 * 1]                                         
    ldp x4, x5, [sp, #16 * 2]                                         
    ldp x6, x7, [sp, #16 * 3]                                         
    ldp x8, x9, [sp, #16 * 4]                                         
    ldp x10, x11, [sp, #16 * 5]                                       
    ldp x12, x13, [sp, #16 * 6]                                       
    ldp x14, x15, [sp, #16 * 7]                                       
    ldp x16, x17, [sp, #16 * 8]                                       
    ldp x18, x19, [sp, #16 * 9]                                       
    ldp x20, x21, [sp, #16 * 10]                                      
    ldp x22, x23, [sp, #16 * 11]                                      
    ldp x24, x25, [sp, #16 * 12]                                      
    ldp x26, x27, [sp, #16 * 13]                                      
    ldp x28, x29, [sp, #16 * 14]                                      
                                                                      
    .if \el == 0                                                      
alternative_if ARM64_WORKAROUND_2966298                               
    tlbi    vale1, xzr                                                
    dsb nsh                                                           
alternative_else_nop_endif           
alternative_if_not ARM64_UNMAP_KERNEL_AT_EL0                 
    ldr lr, [sp, #S_LR]                                      
    add sp, sp, #PT_REGS_SIZE       // restore sp            
    eret                                                     
alternative_else_nop_endif                                   
#ifdef CONFIG_UNMAP_KERNEL_AT_EL0                            
    msr far_el1, x29                                         
                                                             
    ldr_this_cpu    x30, this_cpu_vector, x29                
    tramp_alias x29, tramp_exit                              
    msr     vbar_el1, x30       // install vector table      
    ldr     lr, [sp, #S_LR]     // restore x30               
    add     sp, sp, #PT_REGS_SIZE   // restore sp            
    br      x29                                              
#endif                                                       
    .else                                                    
    ldr lr, [sp, #S_LR]                                      
    add sp, sp, #PT_REGS_SIZE       // restore sp            

    /* Ensure any device/NC reads complete */                
    alternative_insn nop, "dmb sy", ARM64_WORKAROUND_1508412 
                                                             
    eret                                                     
    .endif                                                   
    sb                                                       
    .endm                                                    
```



#### svc相关的CONFIG

```c
// 功能特性    
CONFIG_ARM64_SVE
CONFIG_ARM64_SME
CONFIG_ARM64_SW_TTBR0_PAN
CONFIG_ARM64_PSEUDO_NMI
CONFIG_VMAP_STACK
// 安全特性
CONFIG_ARM64_PTR_AUTH   // pac
// DEBUG/TRACE特性
CONFIG_TRACE_IRQFLAGS    
CONFIG_DEBUG_RSEQ
CONFIG_PID_IN_CONTEXTIDR  
CONFIG_CONTEXT_TRACK_USER
   // lockdep特性
CONFIG_PROVE_LOCKING
CONFIG_LOCKDEP    
// ERRATUM
CONFIG_ARM64_ERRATUM_1463225 // for A76
CONFIG_ARM64_ERRATUM_845719
```









## 指令级分析

抓取1630v200的EMU波形来进行分析(内核5.10，但和1650的代码有一定区别)

路径：![image-20231016153415398](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20231016153415398.png)

#### kernel_entry

![image-20230925165756488](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230925165756488.png)

1. **svc指令本身耗时 30cycle才完成**（这里的代码中，因为没有编译CONFIG_UNMAP_KERNEL_AT_EL0，因此svc后面没有b指令）

   ![image-20230925165855335](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230925165855335.png)

2. sp指针修改 & 通用寄存器保存和清零（共计8.7*2.5=22拍，其中前半段耗时8拍，stp+清零耗时14拍，**第一个stp耗时最长，需要5拍**）

   ![image-20231019151613904](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20231019151613904.png)

3. 将当前tsk指针保存到sp_el0中

   ​		两个mrs都是紧接着1拍内同时commit的；

   ​		后面的ldr x28因为有依赖，所以要3拍才完成；

   ​		**然后msr sp_el0, 28耗时9拍**

   ![image-20230925173447956](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230925173447956.png)

4. 保存elr/spsr及剩余pt_regs成员

   ​	elr的保存紧接着直接commit，而spsr耗时2拍后完成；

   ​	stp 保存x30/x21/x22/x23： 第一、二条指令和上面一条mrs指令同时commit；而x22/x23的保存又**耗时5拍**，原因应该是还在**等前面的mrs指令**真正把值取回来。之前mrs的commit可能并没有实际完成取值操作。

   ​	然后 ，就准备跳转到el0_sync_handler了。

#### el0_sync_handler

1. 跳转过来后，第一条ldrb指令就耗时4拍才取回来，应该是cache miss导致的。
2. 然后判断到是svc，又跳转到el0_svc函数，这里又等了4拍。

所以，这个函数只是从sync到svc，就耗时8拍了。

#### el0_svc

这里前面的指令运行的都很顺利，8拍满发射运行。

《end here，to be continued》



### 5.10代码分析

``` c
SYM_CODE_START_LOCAL_NOALIGN(el0_sync)
    kernel_entry 0
    mov x0, sp
    bl  el0_sync_handler
    b   ret_to_user
SYM_CODE_END(el0_sync)
//  kernel_entry的具体实现：
.macro  kernel_entry, el, regsize = 64
.if \regsize == 32
mov w0, w0              // zero upper 32 bits of x0
.endif
stp x0, x1, [sp, #16 * 0]
stp x2, x3, [sp, #16 * 1]
stp x4, x5, [sp, #16 * 2]
stp x6, x7, [sp, #16 * 3]
stp x8, x9, [sp, #16 * 4]
stp x10, x11, [sp, #16 * 5]
stp x12, x13, [sp, #16 * 6]
stp x14, x15, [sp, #16 * 7]
stp x16, x17, [sp, #16 * 8]
stp x18, x19, [sp, #16 * 9]
stp x20, x21, [sp, #16 * 10]
stp x22, x23, [sp, #16 * 11]
stp x24, x25, [sp, #16 * 12]
stp x26, x27, [sp, #16 * 13]
stp x28, x29, [sp, #16 * 14]

.if \el == 0
clear_gp_regs     // 对应S2
mrs x21, sp_el0     
ldr_this_cpu    tsk, __entry_task, x20
msr sp_el0, tsk

/*
 * Ensure MDSCR_EL1.SS is clear, since we can unmask debug exceptions
 * when scheduling.
 */
ldr x19, [tsk, #TSK_TI_FLAGS]
disable_step_tsk x19, x20 

/* Check for asynchronous tag check faults in user space */
check_mte_async_tcf x19, x22 // 没有使能MTE时，为nop
apply_ssbd 1, x22, x23       // 没有使能ssbd时，为nop

ptrauth_keys_install_kernel tsk, x20, x22, x23
    // 没有使能pac时为nop，但这里使能了呀，为什么没找到？？？

scs_load tsk, x20
.else
add x21, sp, #S_FRAME_SIZE
get_current_task tsk
/* Save the task's original addr_limit and set USER_DS */
ldr x20, [tsk, #TSK_TI_ADDR_LIMIT]
str x20, [sp, #S_ORIG_ADDR_LIMIT]
mov x20, #USER_DS
str x20, [tsk, #TSK_TI_ADDR_LIMIT]
/* No need to reset PSTATE.UAO, hardware's already set it to 0 for us */
.endif /* \el == 0 */
mrs x22, elr_el1
mrs x23, spsr_el1
stp lr, x21, [sp, #S_LR]

/*
 * In order to be able to dump the contents of struct pt_regs at the
 * time the exception was taken (in case we attempt to walk the call
 * stack later), chain it together with the stack frames.
 */
.if \el == 0
stp xzr, xzr, [sp, #S_STACKFRAME]
.else
stp x29, x22, [sp, #S_STACKFRAME]
.endif
add x29, sp, #S_STACKFRAME
stp x22, x23, [sp, #S_PC]

/* Not in a syscall by default (el0_svc overwrites for real syscall) */
.if \el == 0
mov w21, #NO_SYSCALL
str w21, [sp, #S_SYSCALLNO]
.endif
.endm
```



`el0_sync`函数对应的反汇编：

```c
ffff800010011ac0 <el0_sync>:
	========= S1: 进入kernel_entry，先保存所有通用寄存器到内核栈。
ffff800010011ac0:   a90007e0    stp x0, x1, [sp]
ffff800010011ac4:   a9010fe2    stp x2, x3, [sp, #16]
ffff800010011ac8:   a90217e4    stp x4, x5, [sp, #32]
ffff800010011acc:   a9031fe6    stp x6, x7, [sp, #48]
ffff800010011ad0:   a90427e8    stp x8, x9, [sp, #64]
ffff800010011ad4:   a9052fea    stp x10, x11, [sp, #80]
ffff800010011ad8:   a90637ec    stp x12, x13, [sp, #96]
ffff800010011adc:   a9073fee    stp x14, x15, [sp, #112]
ffff800010011ae0:   a90847f0    stp x16, x17, [sp, #128]
ffff800010011ae4:   a9094ff2    stp x18, x19, [sp, #144]
ffff800010011ae8:   a90a57f4    stp x20, x21, [sp, #160]
ffff800010011aec:   a90b5ff6    stp x22, x23, [sp, #176]
ffff800010011af0:   a90c67f8    stp x24, x25, [sp, #192]
ffff800010011af4:   a90d6ffa    stp x26, x27, [sp, #208]
ffff800010011af8:   a90e77fc    stp x28, x29, [sp, #224]
    ========= S2: 若是从el0过来的，则对general register清零。
ffff800010011afc:   aa1f03e0    mov x0, xzr
ffff800010011b00:   aa1f03e1    mov x1, xzr
ffff800010011b04:   aa1f03e2    mov x2, xzr
ffff800010011b08:   aa1f03e3    mov x3, xzr
ffff800010011b0c:   aa1f03e4    mov x4, xzr
ffff800010011b10:   aa1f03e5    mov x5, xzr
ffff800010011b14:   aa1f03e6    mov x6, xzr
ffff800010011b18:   aa1f03e7    mov x7, xzr
ffff800010011b1c:   aa1f03e8    mov x8, xzr
ffff800010011b20:   aa1f03e9    mov x9, xzr
ffff800010011b24:   aa1f03ea    mov x10, xzr
ffff800010011b28:   aa1f03eb    mov x11, xzr
ffff800010011b2c:   aa1f03ec    mov x12, xzr
ffff800010011b30:   aa1f03ed    mov x13, xzr
ffff800010011b34:   aa1f03ee    mov x14, xzr
ffff800010011b38:   aa1f03ef    mov x15, xzr
ffff800010011b3c:   aa1f03f0    mov x16, xzr
ffff800010011b40:   aa1f03f1    mov x17, xzr
ffff800010011b44:   aa1f03f2    mov x18, xzr
ffff800010011b48:   aa1f03f3    mov x19, xzr
ffff800010011b4c:   aa1f03f4    mov x20, xzr
ffff800010011b50:   aa1f03f5    mov x21, xzr
ffff800010011b54:   aa1f03f6    mov x22, xzr
ffff800010011b58:   aa1f03f7    mov x23, xzr
ffff800010011b5c:   aa1f03f8    mov x24, xzr
ffff800010011b60:   aa1f03f9    mov x25, xzr
ffff800010011b64:   aa1f03fa    mov x26, xzr
ffff800010011b68:   aa1f03fb    mov x27, xzr
ffff800010011b6c:   aa1f03fc    mov x28, xzr
ffff800010011b70:   aa1f03fd    mov x29, xzr
    // mrs x21, sp_el0 指令
ffff800010011b74:   d5384115    mrs x21, sp_el0
    // ldr_this_cpu    tsk, __entry_task, x20，
    // 这里__entry_task表示当前进程的task指针，将其通过x28最终放到sp_el0中。
    // 这里是做了一个sp_el0的shadow copy，本身的值没有意义，目的是在内核态时不让
    // 其他人轻易访问及修改sp_el0的值。
    
    // __entry_task是per_cpu变量
    // 代码中定义：tsk .req    x28     // current thread_info
    // 代码中定义： DEFINE_PER_CPU(struct task_struct *, __entry_task);
ffff800010011b78:   f0008efc    adrp    x28, ffff8000111f0000 <cpu_number>
ffff800010011b7c:   910aa39c    add x28, x28, #0x2a8
    // tpidr_el1 每个核不一样，用来区分当前的核，smp_processor_id()也会用到，这里先暂存到x20，省的后来又得重新取。
ffff800010011b80:   d538d094    mrs x20, tpidr_el1 
ffff800010011b84:   f8746b9c    ldr x28, [x28, x20]
    // msr sp_el0, tsk
ffff800010011b88:   d518411c    msr sp_el0, x28
    
    // ldr x19, [tsk, #TSK_TI_FLAGS]
ffff800010011b8c:   f9400393    ldr x19, [x28]
    // disable_step_tsk x19, x20， 通过写mdscr_el1寄存器来禁止软件单步调试。
ffff800010011b90:   36a800b3    tbz w19, #21, ffff800010011ba4 <el0_sync+0xe4>
ffff800010011b94:   d5300254    mrs x20, mdscr_el1
ffff800010011b98:   927ffa94    and x20, x20, #0xfffffffffffffffe
ffff800010011b9c:   d5100254    msr mdscr_el1, x20
ffff800010011ba0:   d5033fdf    isb
    // disable_step_tsk函数结束。
ffff800010011ba4:   1400000b    b   ffff800010011bd0 <el0_sync+0x110>
ffff800010011ba8:   f0008ef7    adrp    x23, ffff8000111f0000 <cpu_number>
ffff800010011bac:   910082f7    add x23, x23, #0x20
ffff800010011bb0:   d538d096    mrs x22, tpidr_el1
ffff800010011bb4:   f8766af7    ldr x23, [x23, x22]
ffff800010011bb8:   b40000d7    cbz x23, ffff800010011bd0 <el0_sync+0x110>
ffff800010011bbc:   f9400397    ldr x23, [x28]
ffff800010011bc0:   37c80097    tbnz    w23, #25, ffff800010011bd0 <el0_sync+0x110>
ffff800010011bc4:   32013fe0    mov w0, #0x80007fff             // #-2147450881
ffff800010011bc8:   52800021    mov w1, #0x1                    // #1
ffff800010011bcc:   d503201f    nop
    // 跳转到这里。
ffff800010011bd0:   d503201f    nop
ffff800010011bd4:   d503201f    nop
ffff800010011bd8:   d503201f    nop
ffff800010011bdc:   d503201f    nop
ffff800010011be0:   d503201f    nop
ffff800010011be4:   d503201f    nop
    // mrs x22, elr_el1
ffff800010011be8:   d5384036    mrs x22, elr_el1
    // mrs x23, spsr_el1
ffff800010011bec:   d5384017    mrs x23, spsr_el1
    // stp lr, x21, [sp, #S_LR]
ffff800010011bf0:   a90f57fe    stp x30, x21, [sp, #240]
    // stp xzr, xzr, [sp, #S_STACKFRAME]
ffff800010011bf4:   a9137fff    stp xzr, xzr, [sp, #304]
    // stp x29, x22, [sp, #S_STACKFRAME]
	// add x29, sp, #S_STACKFRAME
ffff800010011bf8:   9104c3fd    add x29, sp, #0x130
    // stp x22, x23, [sp, #S_PC]
ffff800010011bfc:   a9105ff6    stp x22, x23, [sp, #256]
	// mov w21, #NO_SYSCALL
ffff800010011c00:   12800015    mov w21, #0xffffffff                // #-1
    // str w21, [sp, #S_SYSCALLNO]
ffff800010011c04:   b9011bf5    str w21, [sp, #280]
ffff800010011c08:   d503201f    nop
ffff800010011c0c:   d503201f    nop
    // 回到el0_sync函数。
    // mov x0, sp
ffff800010011c10:   910003e0    mov x0, sp
    // bl  el0_sync_handler
ffff800010011c14:   94293da9    bl  ffff800010a612b8 <el0_sync_handler>
    // b   ret_to_user
ffff800010011c18:   1400010f    b   ffff800010012054 <ret_to_user>
    // el0_sync函数结束
```

``` c
ffff800010022a68 <do_el0_svc>:
ffff800010022a68:   d503233f    paciasp
ffff800010022a6c:   a9bd7bfd    stp x29, x30, [sp, #-48]!
ffff800010022a70:   910003fd    mov x29, sp
ffff800010022a74:   a90153f3    stp x19, x20, [sp, #16]
ffff800010022a78:   aa0003f3    mov x19, x0
ffff800010022a7c:   a9025bf5    stp x21, x22, [sp, #32]
    // 先跳过去执行sve_user_discard函数
ffff800010022a80:   1400002a    b   ffff800010022b28 <do_el0_svc+0xc0>
ffff800010022a84:   d503201f    nop
==========================  el0_svc_common =====================
    // el0_svc_common函数起始点。
ffff800010022a88:   f9400261    ldr x1, [x19]
ffff800010022a8c:   d5384100    mrs x0, sp_el0
ffff800010022a90:   f9402275    ldr x21, [x19, #64]
ffff800010022a94:   f9400016    ldr x22, [x0]
ffff800010022a98:   f9008a61    str x1, [x19, #272]
ffff800010022a9c:   b9011a75    str w21, [x19, #280]
ffff800010022aa0:   2a1503f4    mov w20, w21
ffff800010022aa4:   f9400000    ldr x0, [x0]
ffff800010022aa8:   37a807a0    tbnz    w0, #21, ffff800010022b9c <do_el0_svc+0x134>
    // 这里开始开中断了。local_daif_restore(DAIF_PROCCTX)
ffff800010022aac:   d51b423f    msr daif, xzr
    //  if (has_syscall_work(flags))，这个函数就是判断flags的bit 8-12是否为1，
    //  若这个标志位不为0，则不进入函数，走快速模式。对应的汇编就是b.ne不跳转。
ffff800010022ab0:   f27812d6    ands    x22, x22, #0x1f00
    //  否则，需要走慢速模式。这里汇编是反着的。若进入这个循环则要跳转出去执行。
ffff800010022ab4:   54000601    b.ne    ffff800010022b74 <do_el0_svc+0x10c>  // b.any

==========================  invoke_syscall   =====================
    // invoke_syscall函数
    // 若为if (scno < sc_nr) 的else分支
    // sc_nr为syscall的总数量，这个版本为441个，即0x1b9个。
ffff800010022ab8:   7106e29f    cmp w20, #0x1b8
	// 若大于0x1b8，即大于等于0x1b9，也就是满足 scno >= sc_nr，即else分支。
    // 则跳转到sys_ni_syscall处理错误。
ffff800010022abc:   54000528    b.hi    ffff800010022b60 <do_el0_svc+0xf8>  // b.pmore
    // 然后是if分支
ffff800010022ac0:   2a1403e0    mov w0, w20
    // syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
    // 首先是array_index_nospec函数 -> array_index_mask_nospec
ffff800010022ac4:   f106e41f    cmp x0, #0x1b9
    // ngc等价于sbc指令。
ffff800010022ac8:   da1f03e0    ngc x0, xzr 
    // barrier指令
ffff800010022acc:   d503229f    csdb
ffff800010022ad0:   d00053e1    adrp    x1, ffff800010aa0000 <kimage_vaddr>
ffff800010022ad4:   0a000294    and w20, w20, w0
    // 然后是syscall_table[]
ffff800010022ad8:   91192021    add x1, x1, #0x648
ffff800010022adc:   aa1303e0    mov x0, x19
ffff800010022ae0:   f8747821    ldr x1, [x1, x20, lsl #3]
    // 跳去执行ret = __invoke_syscall(regs, syscall_fn); 
ffff800010022ae4:   d63f0020    blr x1
    // 返回后继续执行invoke_syscall函数的regs->regs[0] = ret;
ffff800010022ae8:   f9000260    str x0, [x19]
==========================  invoke_syscall end  =====================
    // el0_svc_common中，invoke_syscall后面判断has_syscall_work的语句。
    //  跳转表示要去走慢速模式。
ffff800010022aec:   b5000116    cbnz    x22, ffff800010022b0c <do_el0_svc+0xa4>
    // 快速模式：
    // 关中断
ffff800010022af0:   d5034fdf    msr daifset, #0xf
ffff800010022af4:   d5384100    mrs x0, sp_el0
    // flags = current_thread_info()->flags;
ffff800010022af8:   f9400000    ldr x0, [x0]
ffff800010022afc:   92783400    and x0, x0, #0x3fff00
ffff800010022b00:   926bdc00    and x0, x0, #0xffffffffffe01fff
    // if (!has_syscall_work(flags) && !(flags & _TIF_SINGLESTEP))的else分支，
    // 可以直接return
ffff800010022b04:   b4000080    cbz x0, ffff800010022b14 <do_el0_svc+0xac>
	// 否则，重新开中断继续到trace_exit处理。
ffff800010022b08:   d51b423f    msr daif, xzr
    // trace_exit标记位：
ffff800010022b0c:   aa1303e0    mov x0, x19
ffff800010022b10:   97ffd38e    bl  ffff800010017948 <syscall_trace_exit>
    // el0_svc_common函数结束处，恢复栈，ret
ffff800010022b14:   a94153f3    ldp x19, x20, [sp, #16]
ffff800010022b18:   a9425bf5    ldp x21, x22, [sp, #32]
ffff800010022b1c:   a8c37bfd    ldp x29, x30, [sp], #48
ffff800010022b20:   d50323bf    autiasp
ffff800010022b24:   d65f03c0    ret
==========================  el0_svc_common end  =====================
==========================  sve_user_disable =====================    
ffff800010022b28:   d000bde0    adrp    x0, ffff8000117e0000 <reset_devices>
ffff800010022b2c:   f9428000    ldr x0, [x0, #1280]
ffff800010022b30:   720a001f    tst w0, #0x400000
ffff800010022b34:   54fffaa0    b.eq    ffff800010022a88 <do_el0_svc+0x20>  // b.none
ffff800010022b38:   d5384100    mrs x0, sp_el0
ffff800010022b3c:   1400000b    b   ffff800010022b68 <do_el0_svc+0x100>
ffff800010022b40:   1400000a    b   ffff800010022b68 <do_el0_svc+0x100>
ffff800010022b44:   d2a01001    mov x1, #0x800000               // #8388608
ffff800010022b48:   f821101f    stclr   x1, [x0]
    // sve_user_disable，设置cpacr_el1.zen为el0 disable。
ffff800010022b4c:   d5381040    mrs x0, cpacr_el1
ffff800010022b50:   926ef801    and x1, x0, #0xfffffffffffdffff
ffff800010022b54:   368ff9a0    tbz w0, #17, ffff800010022a88 <do_el0_svc+0x20>
ffff800010022b58:   d5181041    msr cpacr_el1, x1
    // 跳回去继续执行 el0_svc_common
ffff800010022b5c:   17ffffcb    b   ffff800010022a88 <do_el0_svc+0x20>
==========================  sve_user_disable  end =====================  
    
ffff800010022b60:   9401cab4    bl  ffff800010095630 <sys_ni_syscall>
ffff800010022b64:   17ffffe1    b   ffff800010022ae8 <do_el0_svc+0x80>
ffff800010022b68:   d2a01001    mov x1, #0x800000               // #8388608
ffff800010022b6c:   14000021    b   ffff800010022bf0 <do_el0_svc+0x188>
ffff800010022b70:   17fffff7    b   ffff800010022b4c <do_el0_svc+0xe4>
ffff800010022b74:   310006bf    cmn w21, #0x1
ffff800010022b78:   54000061    b.ne    ffff800010022b84 <do_el0_svc+0x11c>  // b.any
ffff800010022b7c:   928004a0    mov x0, #0xffffffffffffffda     // #-38
ffff800010022b80:   f9000260    str x0, [x19]
ffff800010022b84:   aa1303e0    mov x0, x19
ffff800010022b88:   97ffd322    bl  ffff800010017810 <syscall_trace_enter>
ffff800010022b8c:   2a0003f4    mov w20, w0
ffff800010022b90:   3100041f    cmn w0, #0x1
ffff800010022b94:   54fff921    b.ne    ffff800010022ab8 <do_el0_svc+0x50>  // b.any
ffff800010022b98:   17ffffdd    b   ffff800010022b0c <do_el0_svc+0xa4>
ffff800010022b9c:   52800580    mov w0, #0x2c                   // #44
ffff800010022ba0:   97fff8a8    bl  ffff800010020e40 <this_cpu_has_cap>
ffff800010022ba4:   72001c1f    tst w0, #0xff
ffff800010022ba8:   54fff820    b.eq    ffff800010022aac <do_el0_svc+0x44>  // b.none
ffff800010022bac:   910003e3    mov x3, sp
ffff800010022bb0:   52800022    mov w2, #0x1                    // #1
ffff800010022bb4:   d538d081    mrs x1, tpidr_el1
ffff800010022bb8:   f0008e60    adrp    x0, ffff8000111f1000 <overflow_stack+0xd50>
ffff800010022bbc:   911aa000    add x0, x0, #0x6a8
ffff800010022bc0:   b8216802    str w2, [x0, x1]
ffff800010022bc4:   d5300241    mrs x1, mdscr_el1
ffff800010022bc8:   52840022    mov w2, #0x2001                 // #8193
ffff800010022bcc:   2a010042    orr w2, w2, w1
ffff800010022bd0:   d5100242    msr mdscr_el1, x2
ffff800010022bd4:   d50348ff    msr daifclr, #0x8
ffff800010022bd8:   d5033fdf    isb
ffff800010022bdc:   92407c21    and x1, x1, #0xffffffff
ffff800010022be0:   d5100241    msr mdscr_el1, x1
ffff800010022be4:   d538d081    mrs x1, tpidr_el1
ffff800010022be8:   b821681f    str wzr, [x0, x1]
ffff800010022bec:   17ffffb0    b   ffff800010022aac <do_el0_svc+0x44>
ffff800010022bf0:   f9800011    prfm    pstl1strm, [x0]
ffff800010022bf4:   c85f7c02    ldxr    x2, [x0]
ffff800010022bf8:   8a210042    bic x2, x2, x1
ffff800010022bfc:   c8037c02    stxr    w3, x2, [x0]
ffff800010022c00:   35ffffa3    cbnz    w3, ffff800010022bf4 <do_el0_svc+0x18c>
ffff800010022c04:   17ffffdb    b   ffff800010022b70 <do_el0_svc+0x108>
ffff800010022c08:   d53cd041    mrs x1, tpidr_el2
ffff800010022c0c:   d53cd041    mrs x1, tpidr_el2

```





# 参数调优

## 性能调优-单核

### AUDIT审计

详见多核部分。

### kpti

详见多核部分。

### pac
详见多核部分。



## 性能调优-多核

目前看来，影响性能的有2个地方，fput/fget中的锁变量，以及fget函数中的cas。

1. 前者是有count和lock在同一个128Byte导致rs转ru的问题，目前通过修改struct file和struct files_struct结构体将两个变量隔离在不同的128B，可以有效规避，但性能提升不到10%。

2. 后者是1620的一个已知bug，在开lse的情况下性能差。在规避掉1的问题后，热点集中到了cas这里。目前正在寻找对应内核代码，看看能不能修改cas。

另外，在关lse的时候，1620依然比1616差，1620上抓的exclusive失败率在70%，推测是由于1616的exclusive有snoop delay机制，可以有效降低抢锁失败率。

### 关AUDIT审计

audit是linux系统中用于记录用户底层调用情况的系统，如记录用户执行的open,exit等系统调用.
并会将记录写到日志文件中。

- 影响：
  - ​	开AUDIT后性能下降20%
  - ​	82#（Intel 6248）上测试，单核syscall在开audit时为997分，关audit为1370分（关turbo，4.19.81内核）。

- 开关方式：

  - 关AUDIT：

    - 内核：CONFIG_AUDIT = N， 或
    - OS： auditctl –a never,task    // 所有task的audit都不使能。

  - 开AUDIT

    - 内核： CONFIG_AUDIT = Y， 且

    - cmdline： audit=0 

    - OS：auditctl -l = No rules。

      // 恢复AUDIT的方式为： auditctl -D ，删除所有audit的规则。

  - 没有效果的方式：

    - 

- auditctl命令的使用方式：

  - 需要使能用户态的auditd服务。
    - 若不使能则保持默认规则（即没有规则），若内核开启了syscall_audit则会进行审计工作。
    - 若使能，则可以使用auditctl工具。
  - 添加规则： auditctl -a <l,a>，其中
    - l： list，包括task，exit，user，exclude
      - task： 创建任务时调用，fork时由父进程调用。
      - exit：退出syscall时使用。
      - user：添加规则到用户信息filter列表中。
      - exclude：排除掉的事件，不进行审计。
    - a：actions，包括always，never。

### 关kpti

kpti会影响syscall，在1620上实测单核syscall（getpid only），开kpti时160ns，关kpti就可以达到124ns。

cmdline中关闭kpti特性：kpti=off

### pac
内核编译时关掉ptrauth选项。

### 关lse

1620上实测测试多核性能：

已知1620在开lse时cmpxchg性能差，所以在1620上调优时，需要关掉lse测试。

 

在关lse时，1616有snoop delay的形式，会在抢锁的时候降低失败率；而1620没有这个机制，因此抢锁严重的情况下失败率就比较高了。

perf stat -e r6c -e r6d -e r6e -e r6f ./Run -c 96 syscall

其中r6e/r6f = 抢锁失败率。

 ### 跨P

在2p时，1620比intel差，因为有4个numa节点，抢锁时会模糊目录，同时跨片时延也长。



### 内核隔离f_path&f_mode

https://lkml.iu.edu/hypermail/linux/kernel/2104.1/01664.html



### 不影响的因素

- chickenbit
- 开关预取
- 



## 热点梳理

### 多核

从5个系统调用层面上看：统计上看`dup`和`close`两个系统调用占了绝大多数。火焰图上也可以check。

```c
$ perf script | grep " dup+" | wc -l
388143
$ perf script | grep "__close+" | wc -l
407340
$ perf script | grep "umask" | grep -v arm64 | wc -l
2800
$ perf script | grep "getpid" | grep -v arm64 | wc -l
1866
$ perf script | grep "getuid" | grep -v arm64 | wc -l
1823
```



在1620 openEuler 21.03(linux 5.10)上实测64c syscall的热点，主要在`dup`和`close`两个系统调用上。函数层级上来说，主要集中在在`__fget_files`函数中。

```shell
70.31%  [kernel]                  [k] __fget_files                          
 9.48%  [kernel]                  [k] filp_close                            
 7.48%  [kernel]                  [k] fput_many                             
 3.88%  [kernel]                  [k] page_counter_try_charge               
 3.07%  [kernel]                  [k] page_counter_cancel                   
```

![image-20220724163241982](D:\at_work\Documents\我的总结文档\images\image-20220724163241982.png)

针对`fget_files/fput_many`等函数的代码分析，详见<file结构体及各种文件操作.md>的dup和close一节。



## 多核性能差的锁操作梳理

涉及到struct file中的4个变量，关于struct file详见<file结构体及各种文件操作.md>

​	*f_inode  -  0x20  ： 文件inode信息， struct inode *指针

​    f_count    -  0x38  ： 文件引用数，atomic_long_t类型

​	*f_op       -  0x28 ： 文件操作类型，struct file_operations。

​	f_mode   -  0x44   :  uint类型， 文件权限管理，比如该文件是否可读、可写等，或者是指示是否为O_PATH。 



dup中： 详细源码见<file结构体及各种文件操作.md>

1. f_mode(0x44)的纯读操作(__fget())，若MASK=NULLL（这个case一定是NULL），则
2. f_count(0x38)的atomic_long_read读(file_count())     -> rs操作，非锁操作
3. f_count(0x38)的cas操作-while循环。
4. 若MASK ！= NULL，则没有上述操作。

   

close中：详细源码见<file结构体及各种文件操作.md>

1. f_count（0x38)的atomic read操作（L3看不见，filp_close）
2. f_mode（0x44)的纯读操作（filp_close）
3. f_op      （0x28)的纯读操作（filp_close）
4. f_inode（0x20)的纯读操作（dnotify_flush)
5. f_count（0x38)的ldadd原子减操作。

![image-20210823140318872](D:\at_work\Documents\我的总结文档\images\image-20210823140318872.png)
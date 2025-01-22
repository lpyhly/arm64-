arm64  内核调度/抢占/进程创建   原理源码详解
===========================

[TOC]

# task_struct

![image-20220818203632462](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20220818203632462.png)

# 进程创建

## init进程

系统所有进程的鼻祖，又称idle进程。在start_kernel()内核启动时静态创建，所有核心数据结构都预先静态赋值。

1. 在arch/arm/kernel/head-common.S，为汇编跳到C语言的入口处，在进入start_kernel函数之前，设置了SP指向8KB内核栈顶部区域。

   ![image-20220818203456521](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20220818203456521.png)

2. 上图左侧：thread_info和stack的初始化为静态的，在include/linux/init_task.c中进行初始化，通过INIT_THREAD_INFO宏来初始化thread_info，init_task通过INIT_TASK宏来初始化。它们都存放在.data..init_task段内存中。

thread_info的task指针指向init_task结构体（task_struct），init_task的stack指针反向指回来thread_info。

 

1. 获取当前进程task_struct数据架构的方式

用current常量，它通过SP寄存器获取当前内核栈地址，并对齐，返回thread_info->task成员。

 

1. 整个流程：

初始化init_task（task_struct型结构体） ->  初始化thread_info和8KB stack空间，并指向init_task -> 将上面的空间放到.data..init_task段内存 -> 将sp指向这段内存的栈顶 -> start_kernel().

 

内核启动后，0号进程为idle进程，徐通没有任务运行时，运行idle进程；

1号进程为初始化进程（通常为systemd），是所有用户态进程的父进程；

2号进程为kthreadd进程，负责产生其他内核进程。

![image-20220818203511261](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20220818203511261.png)

## fork

进程或线程都是用fork, vfork,clone等系统调用来建立，它们都调用同一个函数_do_fork()来实现。

　　fork的实现： 　do_fork(SIGCHLD,...)

　　clone的实现： do_fork(CLONE_VM|CLONE_FS|CLONE_FILES| SIGCHLD,...)

　　vfork的实现：  do_fork(CLONE_VFORK|CLONE_VM| SIGCHLD,...)

 

![计算机生成了可选文字: _do_fork(unsignedlongclone_flags, unsignedlongstack--start, unsignedlongstack--size, int_user*parent_tidptr, int_user*child_tidptr, unsignedlongtls) 门3 ,n O 一L](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\02\clip_image001.png)

clone_flags为标志位；见/include/uapi/linux/sched.h

![计算机生成了可选文字: #det1De #define #define #define #define #define #define #defiDe #defiDe #define #defiDe #define #defiDe #define #defiDe 日define #defiDe #defiDe #defiDe #define #defiDe #define #defiDe #define 仁51七NAL0x000000tt/*SlgnaLmaSKtobeSentatexlt*/ CLONE-－姻0x00000100/*setifVMsharedbet树eenProcesses*/ CLONE--FSox0000oZoo/*SetiffsinfoSharedbetweenproceSSeS*/ CLONE--FILES0x00000400/*setifopenfilessharedbet树eenProcesses*/ CLONEeeSIGHANO0x00000800/*setifsignalhandlersandblockedSigna1SShared*/ CLoNE--PTRAcE0x000o2000/*setif树ewanttolettracingcontinueonthechildtoo*/ cLoNE--vro队ox000e400o/*setiftheparentwantsthechildtowakeituponmm_release*/ CLoNE--PARENT0xo0008000/*setif树ewonttohovetheSOmeporentasthec10ner*/ CLONEweTHREAO0x00010000/*Samethreadgroup?*/ CLONE--NE树NS0x00020000/*Newmountnomespacegroup*/ CLONEeeSYSVSEM0x00040000/*sharesystemVSE性UN00semantics*/ CLONESETTLS0x00080000/*creote0ne树TLSforthechild*/ CLON几PARENT--SETTIO0xo0100000/*SettheTIDintheparent*/ CLONECHILOCLEARTID0x00200000/*cleartheTIOinthechild*/ CLONEeeDETACHED0x00400000/*Unused,ignored*/ CLONE--UNTRACED0x00800000/*setifthetracingprocesscan'tforceCLONE--PTRACEonthisclone*/ CLONECHILOSETTID0x01000000/*settheTIDinthechild*/ CLONE--N印CGROUP0x02000000/*NewCgrouPnomespace*/ CLONEeeNE树UTS0x04000000/*Newutsnamenamespace*/ CLONE--N印IPC0x08000000/*Ne树iPcnamespace*/ CLONEweNE树USER0x10000000/*Newusernamespace*/ CLONE--N印PIO0x20000000/*Ne树pidnamespace*/ CLONE--NE树NET0x40000000/*Newnet树orknamespace*/ CLONE100x80000000/*Clone10context*/](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\02\clip_image002.png)

stack_start为用户态栈的起始地址；

stack_size为用户态栈的大小，通常为0

parent_tidptr和child_tidptr:分别指向父子进程的pid。

do_fork 函数主要干的事情：

- copy_process() 最主要的函数
- [wake_up_new_task()](onenote:#进程的诞生&section-id={25C1D9CC-F81E-4403-A719-1FC61F5513C1}&page-id={5D84A607-D670-42D4-BE76-C95B6573955B}&object-id={7A3D1DD9-ABBE-4A1E-B697-AF71728FA597}&32&base-path=C:\Users\h00424781\Documents\OneNote Notebooks\Work Notebook\内核.one)准备唤醒新创建的进程，也就是把进程加入调度器里接受调度运行。
- 对vfork进程，扣留父进程，直到子进程结束。
- 返回进程的pid。

 

### copy_process 

copy_process函数创建一个新进程

1. - 标志位检查 clone_flags

   - dup_task_struct复制父进程的task_struct和thread_info结构体和证书，清除调度标志位，并进行自己的重新初始化。

   - [sched_fork](onenote:#进程的诞生&section-id={25C1D9CC-F81E-4403-A719-1FC61F5513C1}&page-id={5D84A607-D670-42D4-BE76-C95B6573955B}&object-id={48ACC69D-B463-4A46-B497-4E89805E649B}&30&base-path=C:\Users\h00424781\Documents\OneNote Notebooks\Work Notebook\内核.one) 初始化进程调度相关的数据结构

   - - 设置进程状态为TASK_RUNNING
     - get_cpu()关抢占，抢占计数，调度抢占，put_cpu()开抢占。

   - 内存空间、文件系统、信号系统、IO系统等核心内容的复制操作。

   - - copy_files 父进程打开的文件
     - copy_fs 父进程fs_struct结构
     - copy_signal 父进程的信号系统
     - copy_io 父进程IO相关
     - copy_mm 父进程的内存空间
     - copy_thread() 若新进程不是内核线程，则将父进程的寄存器值复制到子进程中。保存进程的上下文相关的通用寄存器，以及pc，sp等。

   - 为新进程重新分配pid数据结构，获取真正的pid，设置线程组group_leader，全局变量nr_threads++,total_forks++。

   - 返回新创建的进程的task_struct结构。

### copy_mm

![img](D:\at_work\Documents\我的总结文档\images\format,png)

copy_mm 复制父进程的内存空间到子进程

1. - copy_mm

   - - 若父进程没有自己的运行空间，则子进程也不创建

     - 若父进程有，但CLONE_VM=1， 则子进程直接使用父进程的mm_struct结构体

     - dup_mm 其他情况，子进程需要创建一个新的mm数据结构 

     - - mm_init 为新进程分配一个描述内存空间的mm_struct数据结构指针

       - - 先初始化mm_struct的部分成员
         - mm_alloc_pgd()分配PGD页表，不同体系结构实现方式不同。从init进程内核空间复制。若异常向量表为低端向量表，则还需要复制第一个页面和第二个页面（pte），因为包含向量表和相应信息。

       - dup_mmap 把父进程mm的内容复制到新进程的mm_struct数据结构。

       - - 遍历父进程中所有VMAs（virtual memory addr），然后创建新的VMA，复制父进程的pte页表项（不复制对应页面的内容）到子进程（具体实现由copy_page_range函数实现），并插入到子进程的mm中。
         - copy_page_range() 复制父进程VMA的页表到子进程页表，从PUD、PMD开始顺着页表方向循环到PTE页表。



代码详细流程如下：


``` c
// part 1，关于cow，这里用处不大，可以忽略。
int copy_page_range(dst_mm, src_mm, vma)
{
    ......
     // 如果有pte的权限降级，则需要flush掉相关的tlb，可能发生权限降级的情况只有cow场景。
     // 这里只和虚拟化有关。用在host无效了一个tlb时，通知使用该vma的guest这件事。从而让guest也无效化它自己的tlb（host和guest的tlb不同，因为guest的是含有vmid的）
     if (is_cow) {
         mmu_notifier_range_init(&range, MMU_NOTIFY_PROTECTION_PAGE,
                     0, vma, src_mm, addr, end);
         // 这里只是调用一个mmu_notifier，在此之前guest会注册一个notifier，这里会触发这个回调函数kvm_mmu_notifier_invalidate_range_start，然后发出tlbi vmall.
         // 用到的概率应该不大，因为需要host和guest共享同一个vma，应该是文件共享那种。
         mmu_notifier_invalidate_range_start(&range);
     }    
    ......
    if (is_cow) // unmap之后，再调用invaludate_range_end。
        mmu_notifier_invalidate_range_end(&range);
}
```

``` c
// part 2，关于copy_page
int copy_page_range(dst_mm, src_mm, vma)
{
    unsigned long addr = vma->vm_start;
    unsigned long end = vma->vm_end;
    ......
     dst_pgd = pgd_offset(dst_mm, addr); 
    	// 从dst_mm中获取pgd，就是vma->pgd（ttbr基地址）与addr（pa offset）的结合。
     src_pgd = pgd_offset(src_mm, addr); //
     do { // 遍历所有的pgd entry
         next = pgd_addr_end(addr, end);
         if (pgd_none_or_clear_bad(src_pgd))
             continue;
         if (unlikely(copy_p4d_range(dst_mm, src_mm, dst_pgd, src_pgd,
                         vma, addr, next))) {
             ret = -ENOMEM;
             break;
         }
     } while (dst_pgd++, src_pgd++, addr = next, addr != end);
	......
}
// 其中，copy_p4d_range中，先调用p4d_alloc，再对当前的pgd遍历所有的p4d（copy_pud_range）。
// arm64由于没有p4d的概念(p4d == pgd)，因此可以忽略。直接看p4d->pud（level 1）。
static inline p4d_t *p4d_alloc(struct mm_struct *mm, pgd_t *pgd,
        unsigned long address)
{ // arm64不存在p4d这一层，因此pgd_none固定返回0.
    return (unlikely(pgd_none(*pgd)) && __p4d_alloc(mm, pgd, address)) ?
        NULL : p4d_offset(pgd, address);
}
// copy_pud_range中，先调用pud_alloc来填充pgd的所有entry，再对当前的pgd遍历所有的pud。
static inline pud_t *pud_alloc(struct mm_struct *mm, p4d_t *p4d,
        unsigned long address)
{
    return (unlikely(p4d_none(*p4d)) && __pud_alloc(mm, p4d, address)) ?
        NULL : pud_offset(p4d, address);
    // pgd存在的情况下，不会真正alloc一段页表地址，只是返回计算出的p4d地址。
}

```



## exec

### libc到kernel的调用流程

运行一个新的程序会干掉所有旧的内存空间，并为新进程重新分配新的线性地址空间，从`sys_execve()`即exec的系统调用例程开始，调用流程依次是`sys_execve`-->`do_execve`-->`mm_alloc`-->`mm_init`-->`mm_alloc_pgd`-->`pgd_alloc`。最后这个`pgd_alloc`为这个进程分配了一个新的页全局目录（第一级页表）。

`sys_execve()`会在最后会调用这个可执行程序对应格式的`load_binary`函数，这个函数完成了这种格式的可执行程序的加载，其中最主要的过程就是多次调用`do_mmap`为该进程分配一系列的线性区并存放不同的内容，分配顺序是，栈段->代码段->数据段->bss->依赖库，堆是在运行过程中动态分配的，由内核中brk和mmap函数实现，C库将其封装成我们熟知的`malloc`函数。

# 内核调度

### 为什么要有调度？

​	因为多任务的出现。

​	有多个任务时，若全部串行去做，则性能绝对爆差无比，而且不符合实际。毕竟，操作系统的某些任务是必须要一直存在的。若多任务并行处理，则涉及到每个任务何时被执行，何时被退出的问题，这就是调度。

### 为什么有这么多调度器？

​	因为不同任务的需求不同。比如：

- cpu热拔  --  必须立刻停下来，毕竟cpu都要没了还管其他的东西作甚  ==>  stop型任务

- 屏幕刷新，必须在15ms内刷新一次，平时可以将优先级放低，但是当deadline快到了的时候，优先级立刻提高  ==>  deadline型任务

- 时延要求贼高的任务，收到立刻响应。 ==> realtime型任务

- 普通内核态/用户态进程，没有特别要求，只要完成任务就好  ==> normal型任务

- idle，其实不算是任务，只是用来标识当前cpu上没有任务。linux中用调度策略来实现，也就是将idle调度器放在最低优先级，调度时会按照优先级遍历所有task，当其他所有调度器上都没有任务了就会调度到idle调度器，而这里只有一个idle任务（pid 0），就调度进来了。idle进程的运行时间就是top中统计的cpu idle时间。

***

  据此，linux发展出了5种调度器：

`stop` 	

​	stop优先级最高，每个核上有一个migration进程，就是干这个的。可以在需要进程通信时，通过migration进程来把两个核上的任务都停下来。适用于: 任务切换，cpu热插拔，ftrace，clockevent等。

`deadline`

​	SCHED_DEADLINE，Deadline避免有些请求太长时间不能被处理。另外可以区分对待读操作和写操作。DEADLINE额外分别为读I/O和写I/O提供了FIFO队列。

`realtime`

​	使用优先级，有两种仅支持rt的算法：SCHED_FIFO，SCHED_RR（支持高优先级抢占）。

`CFS` 

​	SCHED_NORMAL SCHED_BATCH，下文详述。

`idle`

​	SCHED_IDLE

***

### 分配哪个调度器呢？

​	可以看出，任务的不同调度策略对应的是不同的紧急程度。那么，对于一个任务，应该分配怎样的调度器给它，就取决于它的紧急度，可以用优先级标识。在linux中，用0\~139表示进程优先级，值越小标识优先级越高。其中0\~99给实时进程（不讲道理的进程），100\~139给普通进程（讲道理的进程，包含用户态/内核态进程）

> - 对于普通进程
>
>   ​	这些是讲道理的，所以引入nice值（-20 ~ +19，对应的就是100 ~ 139），进程nice值越高，则越好说话，因此优先级就越低。
>
>   ​	由于它们都是讲道理的，因此高nice值的task只是表示其可以占用的cpu时间较长，但不会完全挤压低nice值的task。实际上，nice值增加1，则相对其他进程cpu时间将减少10%。即：原来占45%，nice+1后占45%\*0.9=39%。（为粗略计算值）。
>
>   ​	对于普通进程，会分配CFS调度器。
>
>   
>
> - 对于实时进程
>
>   ​	它们不讲道理，所以没有nice值的概念，只有优先级0-99. 对于实时进程，只要来了，就直接抢到普通进程前面去做事情。高优先级的进程没做完的话，低优先级的进程就别想进来。
>
>   ​	实时进程有2个调度器：RR调度器和FIFO调度器。区别在对同一个优先级的进程们，RT会平均分配时间，而FIFO是先到先做。对于不同优先级的进程们，二者处理方式是一致的，即高优先级的进程没做完的话，低优先级的进程就别想进来。
>
>   
>

​     这样其实有个问题，若实时进程一直占用，则普通进程就完全没机会了，同时还可能出现FIFO导致死锁问题。所以，后来内核引入了新的机制，linux中会让rt进程定期释放一段时间的cpu给普通进程。

```c
/proc/sys/kernel/sched_rt_period_us：可以设置rt周期，默认1s
/proc/sys/kernel/sched_rt_runtime_us：rt进程在一个周期内的最多的运行时间，默认0.95s
```

### 什么时候调度？

​		进程的时间片分配，与实际发生调度的点是分开的。前者发生在timer中断/scheduler()等的过程中，会对各个进程的时间片进行重新计算，有需要被调度出去的，会置位相关标志位，比如NEED_RESCHED。后者是发生在主动调度时、ret_to_user前夕、内核态中断返回前夕、preempt_enable()时，真正发生ctx switch任务调度。

​		ctx switch实际发生的时机，与中断、抢占密切相关。因此若不想发生

#### 内核开抢占（CONFIG_PREEMPT=y）

增加了两个调度点：内核态中断返回前夕、preempt_enable()时，注意irq_enable()不是调度点。

- 若唤醒动作发生在硬中断上下文中 ------
  		硬中断返回前夕检查（当前任务被中断后，中断返回当前任务前夕会检查是否有其他高优先级任务，若有，则去到高优先级任务，而不是返回当前任务，意味着发生了抢占。）

- 若唤醒动作发生在系统调用或异常处理上下文中（开中断，关抢占）------
  
  ​		此时已经关了抢占（毕竟是中断/异常上下文中），在返回点本该有一个调度点，但是发现此时还关着抢占（没有关中断，所以会有中断返回点这个调度点），因此无法立刻进行抢占，需要在下一次调用preempt_enable时检查再进行抢占。

show me the code：

​	若发生中断时在内核态，发生中断时会调用el1_irq，其中irq_handler处理中断，处理完毕后打算返回前，判断是否有任务来抢占，若有，则arm64_preempt_schedule_irq调用__schdule来进行任务切换，把自己换出去。

```c
SYM_CODE_START_LOCAL_NOALIGN(el1_irq)
    kernel_entry 1
    ......
    irq_handler     // 处理中断

#ifdef CONFIG_PREEMPTION
    ldr x24, [tsk, #TSK_TI_PREEMPT] // get preempt count
	mrs x0, daif
    orr x24, x24, x0
    cbnz    x24, 1f          
    bl  arm64_preempt_schedule_irq  // irq en/disable is done inside
#endif
```

​	若发生中断时在用户态，则调用el0_irq，处理完中断后，返回用户态前会去判断TIF_NEED_RESCHED决定是否发生调度。若时间片到时，会在timer中断处理过程中使能该位。

#### 内核不开抢占

  - 当前进程调用cond_resched()时检查
  - 主动调度调用schedule()
  - 系统调用或异常处理返回user时
  - 中断处理完成返回user时（与硬中断返回前夕不同，这里只是返回user空间时检查。用户态的进程总是可以被抢占的）

#### 什么时候可以抢占？

两个条件都满足才可以抢占：  

 	1. preempt_count计数值为0时，并且；
 	2. 开中断。

```
# define preemptible()   (preempt_count() == 0 && !irqs_disabled())
```

也就是说，若关中断，则内核一定不会发生抢占。不过，若某些人执意不用preeptible()来判断，而是使用preempt_count()来判断，就没办法了，理论上不会有这种人吧…… 所以，可以粗略认为，关中断后就默认关抢占了。

可以反过来记忆，去check什么时候不能抢占：

1. 硬中断/软中断处理中  -- 会调用preempt_count_add()来增加计数，包好中断上下文，以及bottom half。
2. spinlock等持锁过程中   -- 增加计数
3. get_percpu（若被调度到别的核上，应该获取的percpu变量就不对了）   -- 增加计数
4. schedule（）过程中。 不可被抢占。

#### 中断抢占的考虑

若开抢占，需要保证已经做好了同步相关的保护。

若开中断，需要考虑是否允许自己运行中被打断，以及中断上下文对自己的影响，比如持锁时被中断了。

- 开中断 + 开抢占： 已做好同步

- 只开中断： 允许timer等硬件中断的发生，但是不允许其他任务抢占自己的运行，适用于持锁期间等。

- 只开抢占：内核中不存在这种情况。因为 preemptible()的判断中，若关了中断则直接返回N。

- 两个都关：只要不自己作死去主动调度，就一定保证执行完毕到开中断 or 开抢占。至于作死行为，内核中有对应CONFIG会去做死锁检查。



### 持锁时的调度处理

某些时刻是不想被调度的，比如持锁时；但考虑到可能发生长时间抢不到锁的情况，又需要允许中断的发生。这种矛盾该怎么办？原则就是在保证不死锁的前提下尽可能允许中断抢占的发生。那么，就可以讨论下可能发生死锁的场景：

1.  进程1持锁A，等锁B；进程2持锁B，等锁A的场景下，开抢占就会造成死锁，这说明进程间同步没有做好。

   ​		解决办法是，spinlock抢到锁后，一律关抢占，那么即使发生了中断，在中断返回时也不会被调度到另外的进程中。

2. 进程1持锁A时， 中断来了， 中断上下文去抢同一把锁，但一直等不到，于是死锁了。

   ​		那么就需要保证，中断上下文中用到的锁，一定是spinlock_irqsave关中断的（不会睡眠主动调度，不会抢占，也不会中断，一定等到软件释放锁才会执行其他东西），这样就不会发生持锁时被中断的场景了。

3. 进程1持锁A，中断来了，在中断后半部去抢锁A，但一直等不到，于是死锁了。

   ​		解决方案是，禁止中断bottom half的执行。（spin_lock_bh）

   综上可知：抢锁类型可以分为：

   | 接口API           | 作用                                                         |
   | ----------------- | ------------------------------------------------------------ |
   | spin_lock         | 关抢占，允许中断                                             |
   | spin_lock_irq     | 关抢占，关中断                                               |
   | spin_lock_irqsave | 关抢占，关中断，同时保存本cpu的irq状态（这样在unlock的时候，是恢复当前的irq状态，而不是直接开中断。） |
   | spin_lock_bh      | 关抢占，开中断，禁止bottom half                              |

​	详见： http://www.wowotech.net/kernel_synchronization/spinlock.html/comment-page-2



### linux中的调度

通过CONFIG_HZ配置gic的timer上报，每次timer中断处理过程中都会更新se们的runtime/load等信息；而是否会进行任务切换则需要参考下述linux配置的参数。

#### 判断时机  -- timer中断时

timer中断处理函数为tick_periodic ，在这里会判断irq是从user还是kernel过来的。

  - 调用update_process_timer-\>scheduler_tick，在这里先update_rq_clock更新rq中pelt策略的计数值。

  - 调用task_tick-\>entity_tick来进行调度方面的更新，包括

    - update_load_avg+update_cfs_group来更新runnable任务的load

    - 若当前核有多个任务，则进行任务抢占的判断check_preempt_tick

#### 判断条件 -- sched调度参数

  - /proc/sys/kernel/sched_latency_ns： 延迟周期，保证在这个周期内所有任务都至少运行一次；
  - sched_nr_latency：用来限制一个延迟周期内能处理最大的活动进程数，默认值为8，如果活动进程超出该上限，则延迟周期则会线性比例的扩展，所以延迟周期还通过sysctl_sched_min_granularity间接的控制线性比例的大小。
  - /proc/sys/kernel/sched_min_granularity_ns：默认值4ms；如果当前runqueue中的就绪作业数超过sched_nr_latency时，就根据让slice = sysctl_sched_min_granularity \* cfs_rq-\>nr_running；而每个调度实体的时间片则为当前调度实体在整个cfs_rq中所占的比例乘以延迟周期。

show me the code：

***

1.  sysctl_latency_ns：-\>sched_slice-\>__sched_period获取周期值。


 ```
 static u64 __sched_period(unsigned long nr_running)
 {
     if (unlikely(nr_running > sched_nr_latency))
         return nr_running * sysctl_sched_min_granularity;
     else
         return sysctl_sched_latency;
 }
 ```

2.  sysctl_sched_min_granularity，若当前任务实际运行时间小于这个值则不会进行调度。

```c
if (delta_exec < sysctl_sched_min_granularity)
    return;
```
***

举个栗子： 

​		若CONFIG_HZ=1000，sysctl_latency_ns = 24000000（周期24ms），min_granurity_ns = 3000000（最小运行3ms），当前核有两个任务A/B，A.share = B.share =1024，则每1ms上报一次timer中断，处理函数中判断当前任务运行时间是否\<3ms，若小于则不会调度，否则判断是否需要被调度出去（这里需要综合考虑任务A/B的vruntime，包含已运行时间，任务优先级等，若计算下来发现B比A该做，就调度，否则还是运行A，因此一个周期内的调度次数不一定）。每次执中断处理还需要看一下时间是否已经超过24/2=12ms，若超过则即使计算下来B优先级比A高也不会在当前的24ms内调度它。 



###调度源码解析

在linux中，函数sched_setscheduler()用来设定用户进程的调度策略。

***

####weight的计算

  - 获得的cpu时间比例为(自己的weight/所有进程的weight）

  - inv_weight = 2^32/weight，为了计算vruntime虚拟运行时间。

  - vruntime=实际运行时间 \* nice_0_weight/weight.
    优先级越高，vruntime比实际运行时间更小。会调度vruntime小的进程。
    
    

#### struct task_struct

​	与调度相关的成员有：

```c
- int static_prio 不会随时间变化，可通过nice值或sched_setscheduler()来修改.
- int normal_prio 基于static_prio和调度策略计算出来的优先级。
    对普通进程，=static_prio；对实时进程，与rt_priority相关。
- unsigned int rt_priority 实时进程的优先级
- int prio：动态优先级，是调度类考虑的。
- struct sched_entity* 调度实体
```



####struct sched_entity

```c
- struct load_weight 结构体，记录weight信息
	weight 根据prio_to_weight查表获得
	inv_weight 根据prio_to_wmult查表获得
- struct sched_avg 描述进程的负载
    - last_runnable_update 上次计算的时间点，单位 ns，用于计算时间间隔
    - runnable_avg_sum  调度实体在就绪队列里（调用enqueue_entity()，se->on_rq=1；若调用dequeue_entity(),se->on_rq=0）runnable的平均时间。包括两部分：running时间，在就绪队列等待的时间。
    - runnable_avg_period 调度实体在系统中的时间，自进程被fork出来开始算。
        （上面两个是avg时间，单位 us， 采用衰减系数计算。两个值越接近，则平均负载越大，表示该调度实体一直在占用cpu）
    - decay_count
    - load_avg_contrib 进程平均负载的贡献度。
      - load_avg_contrib = runnable_avg_sum * weight / runnable_avg_period
```



####负载计算算法

PELT（pre-entity load tracking）,考虑<span class="underline">权重</span>和<span class="underline">调度实体的负载</span>情况。由sched_avg描述进程的负载。

负载计算

  - 一个周期(period，简称PI)=1ms=1024us.是一个点，当时间点跨过1ms的倍数时间点时，就会进行衰减计算。

  - decay_load(val, n) 计算val \* y^n，第n个周期的衰减值。
      - 使用runnable_avg_yN_inv\[\] 存储衰减因子y的值{y,y^2,y^3,……}
    
- __compute_runnable_contrib(n)可以计算连续n个PI周期的负载累计贡献值。
    - 使用runnable_avg_yN_sum\[\] =
      1024\*(y+y^2+……+y^n)，即对于runnable_avg_yN_inv\[\]的累加。1024为一个周期period=1024us.
  
- __update_entity_runnable_avg(now, sched_avg \*sa,
  runnable)
    - 从last_runnable_update到now这段时间的runnable_avg_sum和runnable_avg_period的计算。
    - 先计算这段时间跨越了n个PI点

    - 然后把之前的runnable_avg值乘以y^n --decay_load

    - 加上计算这n个周期内的的负载累计值 -- __compute_runnable_contrib

    - 加上最后不足一个period的负载值（不需要衰减的） --- + delta。

> ![计算机生成了可选文字: runnablUvg_period del怕＿p de\!t日W deltaw' 石 delt日’ .n材．
> 〔尸 6 7 ，山 n尸 亡J 气乙 p 4 『j 〔尸 O 1 2 p4 哎J n尸 户O 〔阿 delt日 n0w
> last--runnable--update](media/image3.png)

  - update_entity_load_avg 计算最终的负载贡献度load_avg_contrib。
      - 计算出now的时间点
      - 调用__update_entity_runnable_avg来计算两个avg值。
      
      - 调用__update_task_entity_contrib，计算load_avg_contrib =
        runnable_avg_sum \* weight / runnable_avg_period
      
      - 更新cfs_rq-\>runnable_load_avg为原来的值加上该se的load_avg_contrib.



### fork: 新进程的调度策略

在copy_process中，调用了__sched_fork（clone_flags, task_struct\*p)。

  - __sched_fork
    
      - get_cpu()
      
          - 关抢断，preempt_disable()
          
          - 获取当前cpu的id，通过current_thread_info()-\>cpu。
        
      - __sched_fork初始化调度相关的数据结构，其实就是清零。
          - on_rq=0
          - 内部的se调度实体的各数据清零
        
      - 设置p-\>state为TASK_NEW（书上为TASK_RUNNING）
      
        
      
          
      
      ​	进程还没加入rq，外部的事件不能唤醒它。
      
      - 设置优先级prio，先用父进程的normal_prio
      
      - 判断p-\>sched_reset_on_fork位，若为1，则fork子进程时会让子进程恢复到默认的调度策略和优先级。
      
      - 选择调度策略(rt/fair/deadline)，在sched_class成员中标识。
      
      - 对于fair_sched_class,定义了一些该调度策略的钩子函数，包括：
        
          - .enqueue_task
      - .dequeue_task
      
      - .task_fork
      - .update_curr
      - ……
      
      - __set_task_cpu
      
      
    设置task的cpu为当前的cpu，并设置唤醒cpu wake_cpu。
          
    - 调用相应sched_class的task_fork作初始化
      
      - 其中，若sched_class = fair_sched_class,则，task_fork实际实现者为**<u>task\_fork\_fair</u>**
      
    - 初始化preempt_count计数
    
      ​	令preempt_count = 1, 首先要禁止抢占。当preempt_count=0,则可以抢占了。每次调用preempt_disable()会令该变量加一。
    
      - put_cpu()。



  - task_fork_fair 确定新进程的vruntime值。

      - 获取当前task对应的cfs_rq队列（通过task_cfs_rq函数,获取当前task所在的cpu的rq的cfs_rq）。

        - 每个cpu有一个runqueue，用struct
        rq结构体来描述该队列，是CPU的通用runqueue，内部记录了cfs_rq,rt_rq,dl_rq等具体的调度策略的实体，其中，cfs_rq包含load_weight, min_vruntime, runnable_load_avg等。

      - 通过curr指针获取当前task，然后**<u>update\_curr()</u>** 更新当前task的runtime的统计信息。

        

  - update_curr()

      - rq_clock_task() 获取当前rq的clock_task值，每次时钟tick这个值都会更新。

      - 计算上次调用update_curr()到当前时间之间的时间差，然后计算该进程的exec_runtime和vruntime。

      - 更新min_vruntime，可能本次task的vruntime最小呢。

        

  - place_entity() 计算新task的vruntime

      - 其中，参数中，cfs_rq为task所在的rq队列，因为是从复制来的，所以是父进程的cfs_rq,而se为task_struct的调度实体，就是新进程的调度实体，initial=1，表示这是新的进程，需要对其vruntime进行一些惩罚，因为新进程的加入导致cfs队列的权重发生了变化。

      - sched_vslice 计算惩罚值。

        - __sched_period 计算cfs队列中一个调度周期的长度，与队列中的任务数正相关，且只与任务数相关；
      - sched_slice 根据当前进程的权重来计算在CFS队列总权重中可以瓜分到的调度时间。

      - calc_delta_fair 调用__calc_delta来计算vruntime。

      - sched_vslice调用calc_delta_fair.

    - 新task的vruntime为max{计算出的vruntime，min_vruntime）。

    - 令调度实体的vruntime-=min_vruntime，后面会再加回来的。

      

- 再看wake_up_new_task()，在do_fork()中，process_task()后。作用是：将新task加入到调度器中。

    - 设置任务状态为TASK_RUNNING

    - 设置task的cpu，通过select_task_rq来选择一个调度域中最悠闲的CPU。

    - activate_task

      - 调用enqueue_task，调用sched_class-\>enqueue_task钩子函数，到enqueue_task_fair。

    - enqueue_entity把调度实体添加到cfs_rq队列中
          - vruntime+=min_vruntime，之前减掉过的。因为min_vruntime会变，所以使用真正进队列时的值，才能保证新进程的vruntime一定大于此时的min_vruntime.
      
          - update_curr更新当前task的vruntime和min_vruntime.
            
          - 计算se的Load_avg_contrib,添加到整个cfs就绪队列的总平均负载cfs_rq-\>runnable_load_avg中。
            
          - 处理刚被唤醒的进程，place_entity对其vruntime进行一定补偿，即增加vruntime值。
            
          - __enqueue_entity 将se添加到cfs队列的红黑树中。
            
          - on_rq=1.

      - update_rq_runnable_avg(se,1)
        更新该调度实体的负载load_avg_contrib和rq的负载runnable_load_avg.

### wake: 进程唤醒的WAKE_AFFINE机制

https://oskernellab.com/2020/05/24/2020/0524-0001-linux_cfs_scheduler_31_wake_affine/

wake_affine: 倾向于将被唤醒的进程安排到waking cpu上。

通过`select_task_rq`函数来选择一个CPU，从而将待调度的任务放过去。fork，exec，切换三种场景都会调用这个函数来进行抉择。其中，fork和exec不会考虑WAKE_AFFINE机制，只是在idlest的group中找寻一个idlest的core，供新进程来运行。而进程切换较为复杂，这里区分这3种场景的是sd_flag。如下。

``` c
#define SD_BALANCE_EXEC     0x0004  /* Balance on exec */
#define SD_BALANCE_FORK     0x0008  /* Balance on fork, clone */
#define SD_BALANCE_WAKE     0x0010  /* Balance on wakeup */

static int select_task_rq_fair(struct task_struct *p, int prev_cpu, int sd_flag, int wake_flags) 
{
        if (sd_flag & SD_BALANCE_WAKE) {
        record_wakee(p);
        want_affine = !wake_wide(p) && cpumask_test_cpu(cpu, &p->cpus_allowed); 
    }
}
```

对于进程切换的新进程应该调度到哪个core，有2种策略： 

1. 就近调度，可以保持hot cache，同时降功耗；
2. wide策略： 选择idlest的core，可以充分利用核的能力，高负载下降低时延。

对比这两种策略，策略1适合pipe类型的通信；策略2适合1个waker唤醒多个wakee的场景，此时同一调度域中的wakee过多可能会挤压waker的cpu时间从而增大时延。

`want_affine`为1则使用策略1，为0则使用策略2.区分两种策略场景是通过`wake_wide`函数，原理如下：

``` c
static int wake_wide(struct task_struct *p)
{
    unsigned int master = current->wakee_flips;
    unsigned int slave = p->wakee_flips;
    int factor = __this_cpu_read(sd_llc_size);

    if (master < slave)
        swap(master, slave);
    if (slave < factor || master < slave * factor)
        return 0;
    return 1;
}
```

> - 对于A进程，使用task_struct->weekee_flips变量，近似计数一段时间内A唤醒的进程数，每隔1s会进行除2的衰减。具体原理是：
> - 过往的唤醒中： 当进程A唤醒进程B（A称作waker，B乘坐wakee），则判断A上一次唤醒的是哪个进程，若不是B，则令A->wakee_flips++，若仍是B，则不变。
> - 本次唤醒过程中，先统计A->wakee_flips和B->wakee_flips，只要waker和wakee中有一个的wakee_flips<llc_size，则不需要使用wide策略（预计二者有一个ctx不会太多）；或者二者的wakee_flips数量级相差不大，则也不需要wide策略（破罐子破摔？反正都需要大量ctx），就近就好了。（llc_size：同LLC的cpu数量，即一个numa的cpu个数）
> - 将 wakee_flips 值大的 wakee 唤醒到临近的 CPU, 可能有利于系统其他进程的唤醒, 同样这也意味着, waker 将面临残酷的竞争。此外, 如果 waker 也有一个很高的 wakee_flips, 那意味着多个任务依赖它去唤醒, 然后 1 中造成的 waker 的更高延迟会对这些唤醒造成负面影响, 因此一个高 wakee_flips 的 waker 再去将另外一个高 wakee_flips 的 wakee 唤醒到本地的 CPU 上, 是非常不明智的决策. 因此, 当 waker-> wakee_flips / wakee-> wakee_flips 变得越来越高时, 进行 wake_affine 操作的成本会很高.

### wake_affine 对 select_task_rq_fair 的影响

- 首先 sd_flag 必须配置 SD_BALANCE_WAKE 才会去做 wake_affine, 如果是 energy aware, EAS 会先通过 find_energy_efficient_cpu 选核,.
- 先 record_wakee 更新 wake affine 统计信息, 接着通过 wake_cap 和 wake_wide 看这次对进程的唤醒是不是 want_affine 的.

1. 如果是 want_affine, 则先通过 wake_affine 在当前调度域 tmp 中是从 `prev_cpu` 和 `waker CPU` 以及`上次的 waker CPU( recent_used_cpu) `中优选一个合适的 new CPU, 待会选核的时候, 就会从走快速路径 select_idle_sibling 中从 prev_cpu 和 new cpu 中优选一个 CPU. 同时设置 recent_used_cpu 为当前 waker CPU

2. 否则, 如果是 want_affine, 但是 tmp 中没找到满足要求的 CPU, 则最终循环结束条件为 !(tmp->flag & SD_LOAD_BALANCE), 同样如果不是 want_affine 的, 则最终循环结束条件为 !(tmp->flag & SD_LOAD_BALANCE) 或者 !tmp->flags & sd_flag,则 sd 指向了配置了 SD_LOAD_BALANCE 和 sd_flag 的**最高层次**的 sd, 这个时候会通过 find_idlest_cpu 慢速路径选择, 从这个 sd 中选择一个 idle 或者负载最小的 CPU.

   只有 wakeup 的时候通过 wake_affine, 然后通过 select_idle_sibling 来选核。其他情况下, 都是找到满足 sd_flags 的最高层次 sd, 然后通过 find_idlest_cpu 在这个调度域 sd 中去选择一个最空闲的 CPU.

``` c
static int
select_task_rq_fair(struct task_struct *p, int prev_cpu, int wake_flags)
{
    int sync = (wake_flags & WF_SYNC) && !(current->flags & PF_EXITING);
    struct sched_domain *tmp, *sd = NULL;
    int cpu = smp_processor_id();
    int new_cpu = prev_cpu;
    int want_affine = 0;
    /* SD_flags and WF_flags share the first nibble */
    int sd_flag = wake_flags & 0xF;

    if (wake_flags & WF_TTWU) {
        record_wakee(p);  // 更新wakee_flips的数值，只有wake_wide函数才会用到。

        if (sched_energy_enabled()) { // 功耗模式，只有开了CONFIG_ENERGY_MODEL & CONFIG_GOV_CPUFREQ_SCHEDUTIL时才会进入这里。正常情况不进入。
            new_cpu = find_energy_efficient_cpu(p, prev_cpu);
            if (new_cpu >= 0)
                return new_cpu;
            new_cpu = prev_cpu;
        }

        want_affine = !wake_wide(p) && cpumask_test_cpu(cpu, p->cpus_ptr);
        // 调用wake_wide函数。
    }

    rcu_read_lock();
    for_each_domain(cpu, tmp) {
        /*
         * If both 'cpu' and 'prev_cpu' are part of this domain,
         * cpu is a valid SD_WAKE_AFFINE target.
         */
        if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
            cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
        // 若want_affine，则需要调用wake_affine，从当前的cpu，wakee之前运行的cpu，以及最近使用的cpu中选择最idle的一个，作为target
            if (cpu != prev_cpu)
                new_cpu = wake_affine(tmp, p, cpu, prev_cpu, sync);

            sd = NULL; /* Prefer wake_affine over balance flags */
            break;
        }

        if (tmp->flags & sd_flag) 
        // 只要sched_domain支持EXEC,FORK,WAKE，则走这个分支，一般都支持。
        // 那么这个分支跳出时，sd就是最高级别的sched_domain，也就是整芯片全核了。
            sd = tmp;
        else if (!want_affine)
            break;
    }

    if (unlikely(sd)) { // 从整芯片范围内选择一个cpu，当然当前cpu和prev_cpu依然是最优选择。
        /* Slow path */
        new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);
    } else if (wake_flags & WF_TTWU) { /* XXX always ? */
        /* Fast path */
        new_cpu = select_idle_sibling(p, prev_cpu, new_cpu);

        if (want_affine)
            current->recent_used_cpu = cpu;
    }
    rcu_read_unlock();

    return new_cpu;
}
```



**两种策略调度范围的差异，由sched_domain决定。策略1表示sched_domain内调度，策略2表示会跨sched_domain调度。**

- 对应1620就是是否跨die，至于cluster的概念，这里没有体现。
- 具体原因见下一节。
- 

### 调度域 sched_domain

通过sysctl | grep sched_domain可以查看到。调度域是分层级的，最亲近的为SMT级别，表示为同一个物理核；其次是MC级别，再次是DIE，最上面是numa级别。

- 在1620上，调度域有4个，分别对应着4个numa节点，其中domain0为MC，domain1-3为NUMA。
- 在6248上，有3个domain（只有两个numa节点时），分别对应着SMT->MC->NUMA，第一个调度域包含2个smt线程，第二个调度域为本地numa节点，第三个调度域为跨numa节点。
- 在amd 7742上，关smt时有3个domain，分别对应着MC->DIE->NUMA，MC包含4个core，DIE包含64个core，numa包含128个core。



## 进程调度

### mm_struct

​	地址空间（address space）用`mm_struct`结构体来进行承载，进程切换时，需要更改到目的进程的地址空间。但考虑到有时不需要切换，包括但不限于用户态与内核态的相互切换。因为内核态是不需要使用用户态地址空间的，也就没有必要进行切换。因此，进行了real & annoymous两种的区分，前者用`tsk->mm`表示，后者用`tsk->active_mm`表示，比如内核态下，`tsk->mm = NULL`， `tsk->active_mm = previous user mm`。在用户态场景下，`tsk->active_mm = tsk->mm`.

​	为了区分real & annoymous，在mm引用数方面使用了两个变量：`mm_users` & `mm_count`。 `mm_user`用来计数real users的数量；而`mm_count = annoymous的用户数量+ 1(if any real user exists)`；当`mm_count = 0`时才会真正释放地址空间。在linux 6.4中，引入了变量 `CONFIG_MMU_LAZY_TLB_REFCOUNT`，默认使能，当不使能时，则`mm_count` 不会在ctx时记录lazy user了。引入原因是，希望硬件在没人使用mm的时候可以自己去做这种cleanup。

```C
// mm_count的操作, annoymous切换：
mmgrab(); // 加1；
mmdrop(); // 减1；

// mm_users的操作， real切换：
mmget(); // 加1；
mmput(); // 减1；
    
```

​	当real mm的进程被调度到其他核，而借用它的annoymous进程还活着的时候，就会发生mm_count只有lazy

 user的场景。 





> __schedule是调度器的核心函数。

1.  #### 调度时机：
  
      - > 阻塞操作：mutex,semaphore, waitqueue.
      
      - > 中断返回前，系统调用返回user空间前，检查TIF_NEED_RESCHED
      
      - > wakeups进程会进入CFS队列中，并设置TIF_NEED_RESCHED位，具体被调度时机为：

> __schedule函数：

  - > 让进程调度器从就绪队列中选择一个最合适的进程next，然后切换到next进程运行。

> 必须在关抢占时调用该函数。即，
> 
> ![计算机生成了可选文字: 尸asmlinkage l{ VISibleVoid schedschedule(void)
> StFUCttaskStFUCt\*tsk二CUFFent submitwork(tsk); pree巴吕 _SCn日
> tdisable(); dule(false); 1SChed一reelnpt_enable_no_resched();
> ;}while(need_resched()); 尸｝
> 1EXPORTSYMBOL(schedule);](media/image21.png)
> 
>  

  - > ![计算机生成了可选文字: pick_next task schedule context switch prepare
    > task-switch switch m m Irqs Off fire sched out-preempt_notlfiers
    > notlfier-s ops.sched Out asid generation check and switch context
    > local flush tlb Cpu switch m m kvm sched out CPu switch to switch
    > to fire sched_in_preempt_notifiers notifier- N)psosched in
    > finish task_switch mmdrop kvm sched In ](media/image22.png)

>  

  - > 判断当前队列中的进程状态
    
      - > 如果设置了PREEMPT_ACTIVE，说明该进程是由于内核抢占而被调离CPU的，这时不把它从Run
        > Queue里移除；如果PREEMPT_ACTIVE没被设置（进程不是由于内核抢占而被调离），还要看一下它有没有未处理的信号，如果有的话，也不把它从Run
        > Queue移除。
    
          - > 总之，只要不是主动呼叫schedule()，而是因被抢占而调离CPU的，进程就还在运行队列中，还有机会运行。
        
          - > 内核**抢占**过程中不能直接呼叫schedule()调度器，而是呼叫preempt_schedule()，再通过它来调用schedule()，preempt_schedule()会在调用schedule()之前设置PREEMPT_ACTIVE标志，调用之后再清除这个标志。而schedule()会检查这个标志，如果设置了PREEMPT_ACTIVE标志，意味着这是从抢占过程中进入schedule()，对于不是TASK_RUNNING(state
            > \!= 0)的进程，就不会调用deactivate_task()把进程从Run Queue移除。
        
          - > 详见《PREEMPT_ACTIVE标志在内核抢占中的作用》

> 来自 \<<http://linuxperf.com/?p=38>\>

  - > 当前的代码不用PREEMPT_ACTIVE位，而是用__schedule(bool
    > preempt)中的参数来看。对于preempt_schedule，则preempt=1，表示这是抢占来的，之前的任务不能移除；若preempt=0，表示主动调度出去的，则之前的任务可以移除。

> ![计算机生成了可选文字: 币／ 目taticvoid_ SChednotFaCe
> schedule(boolpreempt](media/image23.png)

  - > pick_next_task(rq,prev)
    
      - > 若调度策略为cfs，且CPU的rq中的进程数量=cfd队列中的进程数量，说明cpu中只有普通进程，因此直接调度cfs的pick_next_task_fair，否则需要遍历整个调度类，调度类的优先级为：stop_sched_class
        > -\>dl-\>rt-\>fair-\>idle。
    
      - > pick_next_task_fair
        
          - > 若cfs_rq-\>nr_running = 0, 则选择idle进程，否则调用
        
          - > pick_next_entity选择CFS队列中红黑树最左边的进程。

<!-- end list -->

  - > context_switch（rq,prev,next）切换到next进程。其中，prev是要被换出的进程，next是要被换入的进程，rq为进程切换所在的就绪队列。
    
      - > prepare_task_switch设置next进程的task_struct的on_cpu=1
    
      - > mm=next-\>mm, oldmm=prev-\>active_mm
        
          - > 内核进程的mm=null，active_mm是用来进行进程调度的，是借用的一个地址空间；
        
          - > 普通进程的mm和active_mm都指向mm_struct.
      
      - > 若next为内核线程，则
        
          - > 借用prev的active_mm作为自己的active_mm；
        
          - > 增加prev-\>active_mm的mm_count计数，防止别人释放；
      
      - > 若next不是内核线程，则
        
          - > switch_mm()，进行一些进程地址空间的转换，把新进程的页表基地址设置到页目录表基地址寄存器。详见[switch_mm()
            > 函数](onenote:##CFS调度器&section-id={25C1D9CC-F81E-4403-A719-1FC61F5513C1}&page-id={B866BA3D-6014-42B3-AE84-8B6B522C3914}&object-id={0B83DD14-C9E2-4735-A05B-078F17B487C0}&C7&base-path=C:\\Users\\h00424781\\Documents\\OneNote%20Notebooks\\Work%20Notebook\\内核.one)。
      
      - > 若prev为内核线程，则
        
          - > 设置prev-\>active_mm=NULL
        
          - > rq-\>prev_mm设置成prev之前的active-\>mm。
      
      - > switch_to()切换进程，完成后cpu运行next进程，prev睡眠。详见[switch_to()函数](onenote:##CFS调度器&section-id={25C1D9CC-F81E-4403-A719-1FC61F5513C1}&page-id={B866BA3D-6014-42B3-AE84-8B6B522C3914}&object-id={E54F5E5F-BE66-4610-9F5C-FDD6389AAB80}&3E&base-path=C:\\Users\\h00424781\\Documents\\OneNote%20Notebooks\\Work%20Notebook\\内核.one)。
        
          - > 该函数是新旧进程的切换点，所有进程在收到调度时的切入点都在这里。
        
          - > 新创建的进程由switch_to（）到copy_thread()函数中的ret_from_fork开始执行。
      
      - > barrier()
      
      - > finish_task_switch
        
          - > mm_count计数减一；
        
          - > prev的on_cpu=0；

<!-- end list -->

  - > switch_mm() 函数
    
      - > asid与tlbi
      
          - > 以前，切换进程时都会invalidate整个tlb，因为使用的页表不同。但这样就会使得刚开始出现严重的TLB
            > miss。
        
          - > 将tlb划分为global类型（内核空间都是global的，因为所有进程共享）和process-specific类型（用户地址空间是每个进程独立的）。对于后者，引入asid，让每个TLB
            > entry包含一个asid号。这样，tlb命中=va命中 & asid命中，进程切换也就不需要TLBi了。
        
          - > 页表PTE enty的第11个bit
            > nG=1时，表示该页是process-specific的，切换时不需要TLBi。
        
          - > 硬件asid号最大支持256个（8
            > bit），若超过则溢出，需要把全部TLB冲刷掉，重新分配；每次硬件asid号溢出，则软件的低9
            > bit加一，可以看成是用来标识TLB的版本号。
    
      - > switch_mm()先将当前cpu设置到下一个进程的cpumask位图中，然后调用
    
      - > check_and_switch_context()来完成硬件设置。
        
          - > 读取软件asid，若除低8bit外的值相同，说明还属于同一批次，不需要tlbi，直接跳到cpu_switch_mm，同时将asid设置到per-cpu变量active_asids中。
        
          - > 若不相同，则不属于同一批次，需要调用new_context()分配一个新的软件asid，同时invalidate所有本地tlb。
            
              - > new_context() 分配一个新的软件asid，并设置到mm-\>context.id中
                
                  - > 检查mm-\>context.id，若不为0，说明之前分配过asid，只需要加上新的generation值就可以组成一个新的软件asid；
                
                  - > 若为0，说明是一个新的进程，需要从asid_map位图中查找第一个空闲的比特位用作这次的硬件asid。
                  
                  - > 若找不到空闲的比特位，说明硬件asid溢出，需要提升generation值，并flush_context()冲刷掉所有tlb，同时清空asid_map。
      
          - > 若发生硬件asid溢出，则需要flush掉本地的tlb。
            
          - > 将asid设置到per-cpu变量active_asids中
            
          - > cpu_switch_mm
            > 钩子函数，设置页表基地址TTB寄存器，设置硬件asid（把mm-\>context.id设置到CONTEXTIDR寄存器的低8位）。
            
              - > 汇编实现。
    
    1.  ###### switch_to()函数
    
      - > 调用__switch_to()汇编函数，有三个参数：prev进程的task_struct结构，prev进程的thread_info结构，next进程的thread_info结构。
    
      - > 将prev进程的相关寄存器上下文保存到该进程的thread_info-\>cpu_context结构体中，把next进程的thread_info-\>cpu_context结构体中的值设置到相关寄存器中，从而实现进程的堆栈切换。

 

## scheduler tick

- > scheduler_tick(void) 该函数由timer定时调用（HZ一次）。
  
    - > 获取当前cpu_id,由cpu_id -\> rq -\> curr.
  
    - > update_rq_clock(rq)
      
        - > 更新当前rq的时钟计数clock和clock_task。
  
    - > task_tick() 处理时钟tick到来时与调度器相关的事情
      
        - > entity_tick检查是否需要调度
          
            - > update_curr 更新当前进程的vruntime和min_vruntime
          
            - > update_entity_load_avg更新该se的平均负载和该rq的平均负载。
            
            - > check_preempt_tick检查当前进程是否需要被调度出去。
              
                - > sched_slice 计算理论运行时间，即按权重计算出的时间
              
                - > 统计实际运行时间
              
                - > 若实际超过理论，则需要调度出去resched_curr，设置TIF_NEED_RESCHED位
              
                - > 若实际运行时间\<定义的进程最小运行时间sysctl_sched_min_granularity，默认为0.75ms，不需调度；
              
                - > 对比该进程的vruntime和rq中红黑树最左边的se的vruntime，若小于，则不调度；若大于理论运行时间，则调度resched_curr。
      
        - > update_rq_runnable_avg更新rq的统计信息
  
    - > cpu_load_update_active(rq)更新rq中的cpu_load\[\]

7.  ## 组调度

> ![C:\\3239CEA5\\B1EF8696-DCBD-41B5-AE87-6D30E577B224.files\\image021.jpg](media/image24.jpg)
> 
>  
> 
>  
> 
>  
> 
> ![C:\\3239CEA5\\B1EF8696-DCBD-41B5-AE87-6D30E577B224.files\\image022.gif](media/image25.gif)
> 
>  

  - > 调度粒度是用户组，不是进程，从而保障多个用户平均分配cpu时间。

  - > struct task_group，需打开CONFIG_CGROUP_SCHED,
    > CONFIG_FAIR_GROUP_SCHED.

  - > sched_create_group(struct task_group \*parent) 创建和组织一个组调度
    
      - > task_group \*tg 创建一个结构体
    
      - > alloc_fair_sched_group CFS需要的。
        
          - > kzalloc tg-\>cfs_rq，se，共nr_cpu_ids个。
        
          - > tg-\>share初始化为nice 0，表示该组的权重。
        
          - > init_cfs_bandwidth 初始化cfs带宽控制相关信息
        
          - > 遍历所有cpu，为每个cpu分配一个cfs_rq和se，对每个cpu：
            
              - > init_cfs_rq 初始化cfs_rq中的time信息。
            
              - > init_tg_cfs_entry 构建组调度结构的关键函数
                
                  - > 将cfs_rq和se与自己的task_group连接起来；同时，se-\>my_q=cfs_rq，只有组调度有my_q，普通进程没有自己的cfs_rq。进程se-\>cfs_rq指针是指向它所在的父节点的队列，与这个不同。
                
                  - > 将cfs_rq和se与parent连接起来。

>  

  - > alloc_rt_sched_group RT调度器需要的。

<!-- end list -->

  - > cpu_cgroup_attach(cgroup_subsys_state, cgroup_taskset)
    > 进程加入组调度
    
      - > 遍历tset的进程链表，对每个一个task调用
        
          - > sched_move_task(task)
          
              - > task_current， task_on_rq_queued
                > 判断当前进程是否正在运行/在就绪队列中，队列中的需要退出，running的则先退出，put_prev_task将当前进程，也就是被切换出去的进程重新插入到各自的运行队列中。
            
              - > task_move_group
                
                  - > set_task_rq 设置该进程的cfs_rq和parent为调度组自身的cfs队列和se。
            
              - > 若之前on_rq，则enqueue_task重新enqueue这个task到组调度的队列中，以及它的parent到该到的队列中。若之前running，则set_curr_task将该进程插队进来。

>  
> 
> 《linux组调度浅析》 包含加入组调度的方法。
> 
> <https://blog.csdn.net/ctthuangcheng/article/details/8914825>

![C:\\3239CEA5\\B1EF8696-DCBD-41B5-AE87-6D30E577B224.files\\image023.gif](media/image26.gif)

 ```

 ```

# 进程切换

## Linux 6.5特性-lazy tlb

```
https://lore.kernel.org/linux-mm/20230203071837.1136453-6-npiggin@gmail.com/T/#re70a405484bc9db2b349496142aebe101c5cb413
```

目的： 在lazy tlb模式下，用户态进程<->idle进程（也会借用用户态init_mm)频繁切换的场景下，会发生大量`struct_mm`的原子操作。该patch会减少这种场景的发生。方式是，在lazy tlb的时候，可以不在发生借用mm的进程中对mm进行`mmgrab/mmdrop`，也就是不再进行annoymous场景的refcount记录。在mm_count计数真正降低到0要释放时，增加一个判断逻辑，去广播到所有使用了该mm的cpu上，问一下它们是否有该mm的annoymous引用。若有，则不能直接释放。若没有，则可以正常进行释放了。



具体实现：

1. 引入`mmgrab_lazy_tlb`()来替换`mmgrab()`函数，当`CONFIG_MMU_LAZY_TLB_REFCOUNT`使能时就执行`mmgrab`，若不使能则直接返回。 `mmdrop_lazy_tlb()`同理替换，使能时为`mmdrop`，不使能时则为 `smp_mb()`。

   ```c
   static inline void mmgrab(struct mm_struct *mm)                        
   {                                                                      
           atomic_inc(&mm->mm_count);                                     
   }                            
   static inline void mmdrop(struct mm_struct *mm)                                    
   {                                                                                  
           /*                                                                         
            * The implicit full barrier implied by atomic_dec_and_test() is           
            * required by the membarrier system call before returning to              
            * user-space, after storing to rq->curr.                                  
            */                                                                        
           if (unlikely(atomic_dec_and_test(&mm->mm_count)))                          
                   __mmdrop(mm);   // active/lazy都没人使用这个mm了，则进行地址空间释放。 
   }                                                                                   
   ```

   

2. `kthread_use_mm()`函数中，不再对`active_mm == mm`的特殊情况进行特殊处理（`mm_count`不增加），因为这里不会成为瓶颈，因此可以简化一下操作。 这个函数，在内核中只有IO驱动会用到，正常的ctx不涉及。应该是内核态切内核态之类的场景。

   

3. 部分函数替换：

   1. 函数`exec_mmap(mm_struct *mm)`作用是将指定的`mm_struct`映射到当前`task struct`上，并释放之前映射的`mm_struct`。这里的`mmdrop`更换为`mmdrop_lazy_tlb`。

   2. 函数`finish_cpu()`中的drop替换；

   3. 函数`exit_mm()`中的`mmgrab()`替换

   4. 函数`finish_task_switch`中的mmdrop替换；

   5. 函数`sched_init`中的grab替换；

   6. 函数`context_switch`中的mmgrab替换；

      

4. 进程切换的逻辑变更为：

   ```shell
   kernel -> kernel   lazy + transfer active        # 之前内核借用的active_mm直接传递给新的内核进程，之前进程的active_mm置为Null。
     user -> kernel   lazy + mmgrab_lazy_tlb() active   # 之前用户态进程的active_mm++，因为当前内核进程借用它了。
   kernel ->   user   switch + mmdrop_lazy_tlb() active # 执行mm地址空间的切换，同时active_mm--
     user ->   user   switch            # 直接切换，啥优化都不做。                
   ```

   

5. 在`mmdrop`中，若发现引用数被降至0了，则通过IPI的方式去通过所有引用过当前`mm`的核，看看它们当前的进程是否借用了这个`mm`，若有借用，则替换成`init_mm`，从而让当前的`mm`可以顺利被释放。

```c
static void do_shoot_lazy_tlb(void *arg)
{
	struct mm_struct *mm = arg;

	if (current->active_mm == mm) {
		WARN_ON_ONCE(current->mm);
		current->active_mm = &init_mm;
		switch_mm(mm, &init_mm, current);
	}
}

static void cleanup_lazy_tlbs(struct mm_struct *mm)
{
	if (!IS_ENABLED(CONFIG_MMU_LAZY_TLB_SHOOTDOWN)) {

		return;
	}
	on_each_cpu_mask(mm_cpumask(mm), do_shoot_lazy_tlb, (void *)mm, 1);
	if (IS_ENABLED(CONFIG_DEBUG_VM_SHOOT_LAZIES))
		on_each_cpu(do_check_lazy_tlb, (void *)mm, 1);
}
```

关键点：

1. 只用`mm & active_mm`来指示内核态是否借用了用户态的`mm`，从而释放`mm_count`变量，起到降低`mm_count`被多核竞争atomic操作的作用。
2. 原来`mm_count`的作用，通过mm记录所有cpu的方式，在最后通过IPI方式去通知每个core。


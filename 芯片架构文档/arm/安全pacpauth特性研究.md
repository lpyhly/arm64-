# 安全pac/pauth特性研究

[TOC]

# pauth简介

### CFI/DFI

​		很多的安全问题通过攻击者人为制造的恶意指针，然后处理器解释恶意指针为代码地址，然后执行恶意指针所指的代码，这里的代码恰恰就是攻击者预先准备的恶意代码。所以对于指针的合法性问题一直是安全防御的重点。

​	CFI/DFI（控制流/数据流完整性保护技术），是**内存**安全防护的一种缓解技术。针对的主要攻击方式有：

1. 函数返回值指令为指针，可以篡改；
2. 间接调用或跳转，可以篡改其指针；
3. 使用篡改过的数据变量

### PAC/BTI硬件辅助方式实现CFI/DFI

​		在2016年的时候ARMv8架构里增加了ARMv8.3-A，这个版本里增加了Pointer Authentication 指令：强化指针安全的一种机制。硬件级别的安全防御可以预见将会是趋势，因为不论是从代码执行效率代码的清晰度还是加固的彻底程度来说都明显优于软件级别的实现。
​		PAC：指针认证码(Pointer authentication code)。64位VA虚拟地址的有效范围由虚拟地址空间大小和MMU配置决定。VA的高bit位（16位或12位）不含有效的地址信息，用于保存地址标签和（或）PAC。
​		ARMv8.3-PAuth特性增加功能：支持在寄存器中内容作为目的地址用于间接分支跳转或加载内存指令前，对地址进行认证。该特性仅在AArch64下支持，在ARMv8.3中必选实现。

包含forward和backwards两类，前者是在保护call的地址，不会跳转到非法地址去；后者是保护返回地址，不会跳回时到非法地址去。

pac：通过栈指针SP、返回地址、ia key三者来进行加密操作，算出来的校验值放到x30返回地址的高位。

aut：反向校验。

![image-20210517203850133](D:\at_work\Documents\我的总结文档\images\image-20210517203850133.png)

## 软件规避方案

###　CONFIG_STACK_PROTECTION

函数返回时，返回值会从栈中取出放到x30寄存器，因此若栈被篡改了，则函数返回值就会被修改。这里提出的方法是用来保护栈的，不让栈内容被篡改，也就间接保护了函数返回值。

![img](D:\at_work\Documents\我的总结文档\images\20141204180250247)

实现思路是：

​	地址从低到高分别为： text -> data -> bss -> heap -> stack ->命令行参数和环境变量。stack从高地址往低地址生长，在栈底指针EBP的上面存放着返回地址，在EBP下面存放着函数的局部变量（入栈）。而局部变量的生长方向是自低往高，如果函数越界则可能会污染到函数返回值。

![img](http://hi3ms-image.huawei.com/hi/showimage-12473913-212221-4f9e3cbcb889ba0e9fa61dfc7535c5ab.jpg)

​	STACK-PROTECTION机制就是通过在栈底插入一个canaries，里面的值可以是固定也可以是一个随机值。当出栈时进行检测，若canaries的值符合预期，则表示没有被修改，那么认为EBP上面的返回地址也没有被修改，可以正常出栈操作，若canaries的值不符合预期，则会跳转到`__stack_chk_fail`进行错误处理。

|                   | **No stack protection**                                      | **Software** **Stack protection**                            | **With Pointer** **Authentication**                          |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Function Prologue | SUB sp, sp,  #0x40  <br>STP  x29, x30, [sp,#0x30] <br> ADD  x29, sp,  #0x30  … | SUB   sp, sp, #0x50<br>  STP x29, x30, [sp,#0x40] <br> ADD x29, sp, #0x40  <br/>ADRP x3,{pc}  <br/>LDR   x4 ,[x3, #SSP]  <br/>STR   x4,[sp, #0x38]  <br/>… | PACIASP<br/>  SUB sp, sp, #0x40  <br/>STP x29,  x30, [sp,#0x30]  <br/>ADD  x29, sp,  #0x30 <br/> … |
| Function Epilogue | …<br>  LDP x29, x30, [sp,#0x30] <br> ADD  sp,sp,#0x40  <br>RET | …<br/>  LDR x1,[x3, #SSP] <br/> LDR x2,[sp, #0x38]  <br/>CMP x1,x2  B.NE _stack_chk_fail  <br/>LDP x29,x30, [sp, #0x40]  <br/>ADD sp,sp #0x50 <br/> RET | …  <br/>LDP x29, x30, [sp,  #0x30]  <br/>ADD sp, sp, #0x40  <br/>AUTIASP  RET |

若开了pac，则不用再开STACKPROTECTOR了。

``` c
commit 74afda4016a7437e6e425c3370e4b93b47be8ddf
Author: Kristina Martsenko <kristina.martsenko@arm.com>
Date:   Fri Mar 13 14:35:03 2020 +0530

	As ptrauth protects the return address, it
    can also serve as a replacement for CONFIG_STACKPROTECTOR, although note
    that it does not protect other parts of the stack.
```

### CONFIG_SHADOW_CALL_STACK - clang编译器专属

只有clang编译器支持，gcc暂时未看到。内核中有相应代码，`scs_***`函数，判断宏定义若未实现则不会编译。

属于加强版`-fstack-protector`，需要在编译选项中添加`-fsanitize=shadow-call-stack`来使能。作用是，在函数调用前后，先将`x30`寄存器的值放到`x18`寄存器指定的某块内存中，最后在函数返回前再从这块内存中取出来。

``` c
	// normal
stp     x29, x30, [sp, #-16]!
mov     x29, sp
bl      bar
add     w0, w0, #1
ldp     x29, x30, [sp], #16
ret
    // after -fsanitize=shadow-call-stack
str     x30, [x18], #8     // 保存x30
stp     x29, x30, [sp, #-16]!
mov     x29, sp
bl      bar
add     w0, w0, #1
ldp     x29, x30, [sp], #16
ldr     x30, [x18, #-8]!    // 恢复x30
ret
```



## ptr_auth方式

函数开头添加： PACIASP， RET返回前添加AUTIASP。

若不使能PAC功能，则编译为两个nop指令。

当AUT验证失败，则报instruction abort，内核进入oops并kill当前任务。



### Linx核的具体实现方式

aut指令划分为2部分，可以看做是2uops：1uops为计算部分，1uops为mov部分。

- 计算：
  - 在ALU中进行计算，可以并发执行
  - 耗时5 cycle
- mov：
  - 将计算结果mov到x30寄存器中。
  - 1-cycle uops。

性能瓶颈点：

​	pac+aut背靠背场景，因为nonflush特性。



# pac配置

## pac分类

分为两类，

- generic  ：两种实现二选一，要么GPI要么GPA。

 - address :    两种实现二选一，要么API要么APA。

key有5组：APIA, APIB, APDA, APDB, APGA, 分别对应以下5组指令：

- pacia/autia
- pacib/autib
- pacda/autda
- pacdb/autdb
- pacga/autga

其中，前4组对应的是address类型的pac校验，最后一组对应的是generic的pac校验。



## 硬件ID寄存器特性标识

寄存器ID_AA64ISAR1_EL1.{GPI, GPA, API, APA}标识特性开关。



**GPI, bits [31:28]**

ARMv8.3及以上版本：

​     1）0b0000：通用认证未实现使用微架构定义算法；

​     2）0b0001：通用认证使用微架构定义算法实现；包括了PACGA指令。

​     3）该版本下，只允许取0b0000和0b0001，其他取值保留。

​     4）如果ID_AA64ISAR1_EL1.GPA非零，那么当前字段必须为0b0000。

其他版本，字段保留，置0。

**GPA, bits [27:24]**

ARMv8.3及以上版本：

​     1）0b0000：通用认证未实现使用架构定义算法；

​     2）0b0001：通用认证使用架构定义算法QARMA实现；包括了PACGA指令。

​     3）该版本下，只允许取0b0000和0b0001，其他取值保留。

​     4）如果ID_AA64ISAR1_EL1.GPI非零，那么当前字段必须为0b0000。

其他版本，字段保留，置0。

**API, bits [11:8]**

ARMv8.3及以上版本：

​     1）0b0000：地址认证未实现使用微架构定义算法；

​     2）0b0001：地址认证使用微架构定义算法实现；包括了除PACGA指令外的所有指针认证指令；

​     3）0b0010：地址认证使用微架构定义算法实现，EnhancedPAC()函数返回值TURE；适用于除PACGA指令外的所有指针认证指令；

​     4）该版本下，只允许取0b0000和0b0001（怀疑文档少写了0b0010），其他取值保留。

​     5）如果ID_AA64ISAR1_EL1.APA非零，那么当前字段必须为0b0000。

其他版本，字段保留，置0。

**APA, bits [7:4]**

ARMv8.3及以上版本：

​     1）0b0000：地址认证未实现使用架构定义算法；

2）0b0001：地址认证使用架构定义算法QARMA实现，EnhancedPAC()函数返回FALSE；适用于除PACGA指令外的所有指针认证指令；

​     3）0b0010：地址认证使用架构定义算法QARMA实现，EnhancedPAC()函数返回TURE；适用于除PACGA指令外的所有指针认证指令；

​     4）该版本下，只允许取0b0000和0b0001（怀疑文档少写了0b0010），其他取值保留。

​     5）如果ID_AA64ISAR1_EL1.API非零，那么当前字段必须为0b0000。

其他版本，字段保留，置0。

# GCC

gcc编译器使能：

``` c
-msign-return-address=scope 
// gcc7/8有这个，gcc9以上就不用了，被branch-protection替代
	Select the function scope on which return address signing will be applied. Permissible values are ‘none’, which disables return address signing, ‘non-leaf’, which enables pointer signing for functions which are not leaf functions, and ‘all’, which enables pointer signing for all functions. The default value is ‘none’. This option has been deprecated by -mbranch-protection.
			
-mbranch-protection=none|standard|pac-ret[+leaf]|bti
			Select the branch protection features to use. ‘none’ is the default and turns off all types of branch protection. ‘standard’ turns on all types of branch protection features. If a feature has additional tuning options, then ‘standard’ sets it to its standard level. ‘pac-ret[+leaf]’ turns on return address signing to its standard level: signing functions that save the return address to memory (non-leaf functions will practically always do this) using the a-key. The optional argument ‘leaf’ can be used to extend the signing to include leaf functions. ‘bti’ turns on branch target identification mechanism.
// leaf function： 没有调用其他函数的函数
                
```



# 内核

## QA

### 有哪些相关的config？

内核中与pac相关的有4个config选项，第一个是可选的，后面3个是标识编译器功能的。

```c
CONFIG_ARM64_PTR_AUTH=y    // 用户可选，是否使能pac特性
CONFIG_CC_HAS_BRANCH_PROT_PAC_RET=y     // gcc使能支持-mbranch-protection的编译选项。
CONFIG_CC_HAS_SIGN_RETURN_ADDRESS=y     // gcc是否支持-msign-return-address的编译选项，功能同上，在gcc7/8时使用，目前已经废弃。
CONFIG_AS_HAS_PAC=y                    // assembler是否支持pac功能
```
在5.1x版本（5.10没有，5.14已经有了）中，新增CONFIG_ARM64_PTR_AUTH_KERNEL选项。

``` c
CONFIG_ARM64_PTR_AUTH=y        // 支持用户态的pac校验，即切换到用户态时需要msr修改所有key。
CONFIG_ARM64_PTR_AUTH_KERNEL=y // 支持内核态的pac校验，依赖CONFIG_ARM64_PTR_AUTH使能，编译内核时是否指定pac的编译选项，同时控制切换到内核时，是否msr切换到内核的key。
```

其他依赖详见下面的<内核Makefile>章节。

### ARM64_HAS_ADDRESS_AUTH和ARM64_HAS_GENERIC_AUTH是怎么判断的？

当`CONFIG_ARM64_PTR_AUTH=y`时，会根据ID寄存器的值进行如下判断：("arch/arm64/kernel/cpufeature.c")

1. 若上述的APA使能，则内核启动时的hwcap判断`ARM64_HAS_ADDRESS_AUTH_ARCH`这个capability使能；
2. 若上述的GPA使能，则内核启动时的hwcap判断`ARM64_HAS_GENERIC_AUTH_ARCH`这个capability使能；

3. 若上述的API使能，则内核启动时的hwcap判断`ARM64_HAS_ADDRESS_AUTH_IMP_DEF`这个capability使能；

4. 若上述的GPI使能，则内核启动时的hwcap判断`ARM64_HAS_GENERIC_AUTH_IMP_DEF`这个capability使能；

然后将上述4个hwcap特性两两组合，共后续逻辑判断。

1. 若APA || API至少有一个使能，则`ARM64_HAS_ADDRESS_AUTH=y`
2. 若GPA || GPI至少有一个使能，则`ARM64_HAS_GENERIC_AUTH=y`

### 为什么不能使能内核pac？

``` c
Symbol: ARM64_PTR_AUTH [=y]                                                           Type  : bool                                                                           Defined at arch/arm64/Kconfig:1506                                                       Prompt: Enable support for pointer authentication                                     Depends on: (CC_HAS_SIGN_RETURN_ADDRESS [=y] || CC_HAS_BRANCH_PROT_PAC_RET [=y]) && AS_HAS_PAC [=y] && (LD_IS_LLD [=n] || LD_VERSION [=236010000]>=\    
233010000 || CC_IS_GCC [=y] && GCC_VERSION [=100301]<90100) && (!CC_IS_CLANG [=n] || AS_HAS_CFI_NEGATE_RA_STATE [=y]) && (!FUNCTION_GRAPH_TRACER [=n] || \
DYNAMIC_FTRACE_WITH_REGS [=n])                                                           Location:                                                                               -> Kernel Features                                                                       -> ARMv8.3 architectural features                                                                                                                   
```

从内核Kconfig中可以看出，需要以下条件全部满足：

- 内核若开了graph tracer，则必须开启dynamic_ftrace_with_regs才可以。（否则ftrace无法通过栈指针获取函数调用关系）
- 编译器支持sign_return_address功能或者branch_prot功能：通过-msign-return-address=all或者-mbranch-protection=pac-ret+leaf来check。
- AS支持pac功能： 通过-march=armv8.3-a来check是否支持。
- LD版本>2.33

### pac功能有哪些相关的内核函数？

上层接口函数：

``` c
// 最上层的init函数，只要新建key（写到task_struct中），不需要写入生效。
ptrauth_thread_init_kernel(tsk)    // 参数是task_struct
    -> ptrauth_keys_init_kernel(&(tsk)->thread.keys_kernel)
ptrauth_thread_init_user   // 先新建key，再根据prctl的配置去设置sctlr相关域段。
    -> ptrauth_keys_init_user 
    
// 最上层的switch函数，不用新建key，只需要切换key。
ptrauth_thread_switch_user(tsk)
    -> ptrauth_keys_install_user
ptrauth_thread_switch_kernel(tsk)
    -> ptrauth_keys_install_kernel
    
// ptrauth_keys_init_cpu函数，会打开sctlr，同时msr key
ptrauth_keys_init_cpu

// install 函数，会切换key，与switch函数类似。
ptrauth_keys_install_kernel() //  msr后需要isb
ptrauth_keys_install_kernel_nosync() //  msr后不需要isb
__ptrauth_keys_install_user() // 仅修改APIA的key。
```



基础函数：

``` c
// asm_pointer_auth.h  都是汇编实现的
#ifdef CONFIG_ARM64_PTR_AUTH_KERNEL 
__ptrauth_keys_install_kernel_nosync // 基本实现函数，只msr IA key
  <--  ptrauth_keys_install_kernel_nosync // msr后不需要isb
  <--  ptrauth_keys_install_kernel // msr后需要isb
#endif
#ifdef CONFIG_ARM64_PTR_AUTH    
__ptrauth_keys_install_user // 基本实现函数，只msr IA key    ???
__ptrauth_keys_init_cpu // 会打开sctlr，同时调用kernel_nosync函数。
    <-- ptrauth_keys_init_cpu // 若支持API或APA，则调用init_cpu函数打开pac功能。
#endif

// asm_pointer_auth.h  都是C实现的
__ptrauth_key_install_nosync // 基本实现函数，msr key
#ifdef CONFIG_ARM64_PTR_AUTH_KERNEL
ptrauth_keys_init_kernel // 初始化kernel的apia key，随机获取。
ptrauth_keys_switch_kernel // msr APIA key

#endif
ptrauth_keys_install_user // msr APIB/APDA/APDB/APGA key
    <-- ptrauth_keys_init_user //  初始化user的apia/APIB/APDA/APDB/APGA key，随机获取,再msr进去。
```

### key是怎么来的？什么时候传给user和kernel？

无论是用户态还是内核态，key都是存放在task_struct-> thread_struct中。

内核态key：	&(tsk)->thread.keys_kernel

用户态key：   &(tsk)->thread.keys_user

``` c
struct thread_struct {
......
#ifdef CONFIG_ARM64_PTR_AUTH
    struct ptrauth_keys_user    keys_user;
#ifdef CONFIG_ARM64_PTR_AUTH_KERNEL
    struct ptrauth_keys_kernel  keys_kernel;
#endif
#endif
...
    u64         sctlr_user;
};
// 其中，key的结构体为：
struct ptrauth_keys_user {
    struct ptrauth_key apia;
    struct ptrauth_key apib;
    struct ptrauth_key apda;
    struct ptrauth_key apdb;
    struct ptrauth_key apga;
};
struct ptrauth_keys_kernel {
    struct ptrauth_key apia;
};
```

 ### key的创建和切换时机

``` c
// 创建新进程：新建专属该进程的kernel的key。
copy_process
    -> copy_thread
    	-> ptrauth_thread_init_kernel
    
// 执行elf程序：
do_execve -> do_execve_common // copy文件名、命令行参数、环境变量，填充binprm结构体
    -> exec_binprm -> search_binary_handler // 扫描寻找合适的handler
    	->load_elf_binary // 处理elf格式的，若a.out则执行load_aout_binary，e.t.
    		-> setup_new_exec -> arch_setup_new_exec ->  ptrauth_thread_init_user
    
// 进程切换时： 切换kernel/user的key
__schedule
    -> swith_to -> __switch_to
    	-> ptrauth_thread_switch_user(next) // 切换到下一个user的key们
	    -> cpu_switch_to  // 真正的切换：保存caller的寄存器们(仅x19-x30,sp_el0)，恢复next的寄存器们(仅x19-x30,sp_el0)。
     		-> ptrauth_keys_install_kernel(next) //这里才修改内核的key，因为已经切换完毕了。

// linux启动时：每个核都使能pac功能，这里的key是事先初始化过的。
secondary_startup
    -> __secondary_switched
    	-> ptrauth_keys_init_cpu
    
// 用户态进入内核态时：
kernel_entry
    -> __ptrauth_keys_install_kernel_nosync
    
// 返回用户态时：
kernel_exit    
    -> __ptrauth_keys_install_user
```



## 内核的优化之路

### install_user的优化

https://www.spinics.net/lists/linux-api/msg46690.html

既然内核只用到了APIA key，那么就不需要在退出内核态时将user的所有key都重新msr，只需要动APIA寄存器就可以。这样，5组寄存器就变成了1组。在Apple M1上实测:

``` c
On an Apple M1 under a hypervisor, the overhead of kernel entry/exit
has been measured to be reduced by 15.6ns in the case where IA is
enabled, and 31.9ns in the case where IA is disabled.
```

### prctl的引入

https://www.spinics.net/lists/linux-api/msg47349.html

内核5.12版本新增，用户态可通过prctl机制来控制EL0程序的pac是否使能，实际上是内核通过控制sctlr的EnIA/EnIB/EnDA/EnDB实现的。

当进入内核时： 

​	若enable pacia，则和当前处理流程一样；

​	若disable pacia，则说明用户态没有修改过APIAKEY，那么，就不需要msr apiakey寄存器，但要修改sctlr为enable；

退出内核时：

​	若enable pacia，则和当前处理流程一样；

​	若disable pacia，则说明用户态不会修改APIAKEY，同样不需要msr apia寄存器。





## 内核具体实现

![image-20210517204239511](D:\at_work\Documents\我的总结文档\images\image-20210517204239511.png)



## example： svc中的pac使用

|      |             |                        |
| ---- | ----------- | ---------------------- |
| 1    | paciasp     | el0_sync_handler       |
| 2    | paciasp     | el0_svc                |
| 3    | paciasp     | enter_from_user_mode   |
| 4    | ==  autiasp | enter_from_user_mode   |
| 5    | paciasp     | do_el0_svc             |
| 6    | paciasp     | __arm64_sys_getppid    |
| 7    | paciasp     | __task_pid_nr_ns       |
| 8    | paciasp     | rcu_read_unlock_strict |
| 9    | ==  autiasp | rcu_read_unlock_strict |
| 10   | ==  autiasp | __task_pid_nr_ns       |
| 11   | paciasp     | rcu_read_unlock_strict |
| 12   | ==  autiasp | rcu_read_unlock_strict |
| 13   | ==  autiasp | __arm64_sys_getpid     |
| 14   | ==  autiasp | do_el0_svc             |
| 15   | ==  autiasp | el0_svc                |
| 16   | ==  autiasp | el0_sync_handler       |
|      |             |                        |
|      |             |                        |



``` c
svc指令
 -> 内核的el0_sync(无pac/aut)
	-> kernel_entry 0  // 相关寄存器
			-> __ptrauth_keys_install_kernel_nosync  // 这里会msr pac的各个key寄存器
    -> el0_sync_handler（paciasp)
    	-> el0_svc （paciasp)
    		-> enter_from_user_mode(pac+aut空函数)
    		-> do_el0_svc(pac)
    			-> sve_user_discard(static函数，不单独编)
				-> el0_svc_common（static函数，不单独编）
                    -> ret_to_user
                        -> kernel_exit 0
                            -> __ptrauth_keys_install_user  //  这里也会msr pac的各个key寄存器
-> 返回用户态
```


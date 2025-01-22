# ssbs

[TOC]

# ssbs介绍

Speculative Store Bypassing Safe

SSB的出现是由于CPU优化所致，它允许：有store指令在前时，以投机方式执行潜在依赖于store结果的load指令。具体地，如果预测load不依赖于先前store，则可以在store之前投机地执行该load。如果预测不正确，则可能导致load读取了陈旧数据，并有可能在投机期间将该数据转发到其他相关的指令上。这可能会引起投机执行侧通道和敏感信息的泄露。

简化示例场景：

![image-20210426161935817](D:\at_work\Documents\我的总结文档\images\image-20210426161850381.png)

如果RDI和RSI指向的是同一个地址，则假设第1行中的MOV指令在特殊情况下可能需要额外的时间来执行（如果计算RDI+RCX的地址表达式正在等待先前的指令执行）。在这种情况下，CPU可能会预测MOVZX不依赖于MOV，并且可以在执行保存AL数据的MOV之前进行预测执行。这可能会导致位于RSI+RCX的内存中的旧数据被加载到R8中，从而导致第四行代码使用了错误的数据。如果R8中的值是敏感（重要）的，则可以通过利用高速缓存通过边通道观察到它基于公开的源码，例如FLUSH + RELOAD（如果RDX指向共享内存）或PRIME + PROBE。CPU最终将检测到错误预测并丢弃计算出的状态，但是到此为止，在投机期间访问的数据可能已在缓存中产生了残留的副作用，然后可以对其进行测量以推断装入R8的值。

## ssbs on 鲲鹏

### ssbs on linx

linx核的具体实现是：等所有的store address都确定了，确定没有alias，LSU才会将load的request往外发。

1620&1630实现方式差不多，1650会进行优化。

可能会出现开ssbs反而提高了性能的场景：

​	如果投机的load拿到的数据大多都是stale的，也就是投机出错，会回滚到出错这笔load执行，并且在这笔load之后与这个load有数据依赖的指令也都需要回滚。如果这种场景多，那开ssbs能提高性能

### ssbs on 1620

1620上默认配置硬件不支持ssbs功能，即相应ID寄存器配置为0； 但是firmware默认支持spectre v4的防护。因此默认情况下，是使用了firmware去配置spectre v4的使能，实现的功能是一样的。

同时，1620上存在chickenbit来控制是否可以修改pstate.ssbs域段，如果不使能的话，软件也无法修改pstate.ssbs域段。cpuactlr1_el1这个寄存器的修改是需要actlr_el2控制其读写，但是actlr_e2也没法修改。只有修改actlr_el3才可以配置actlr_el2。因此配置链为：

​	actlr_el3=0xff -> actlr_el2=0xff -> cpuactlr1_el1.ssbs=1 -> pstate.ssbs=0

![img](D:\at_work\Documents\我的总结文档\images\6765B494-38AD-49A8-AAC8-DBA2FE074050.png)

### ssbs在1620上进行配置的方式

1. 修改bios：actlr_el3 = 0xff, actlr_el2 = 0xff（从而可以配置cpuactlr1_el1.ssbs）
2. 修改内核：
   1. 因为ID寄存器显示的是不支持，因此需要修改内核的判断逻辑，令ID寄存器即使读出来为全0，也认为其支持硬件的ssbs功能，从而令内核在hwcap中显示支持ssbs，方便后面的逻辑，同时可以挂上回调函数读出来ssbs的值。
   2. 配置cpuactlr1_el1.ssbs=1，真正令硬件使能ssbs功能。
3. 启动时，ssbd=force-on/force-off来配置ssbs的功能。



``` c
// 内核修改patch。
From 3ca40ec8ce885ce119e2de44ae4af34379337e79 Mon Sep 17 00:00:00 2001
From: Lingyan Huang <huanglingyan2@huawei.com>
Date: Tue, 27 Apr 2021 21:15:45 +0800
Subject: [PATCH] ssbs_on_1620_modify_kernel

Signed-off-by: Lingyan Huang <huanglingyan2@huawei.com>
---
 arch/arm64/kernel/cpufeature.c  | 9 +++++----
 arch/arm64/kernel/proton-pack.c | 8 ++++++++
 2 files changed, 13 insertions(+), 4 deletions(-)

diff --git a/arch/arm64/kernel/cpufeature.c b/arch/arm64/kernel/cpufeature.c
index 65a522f..f73986e 100644
--- a/arch/arm64/kernel/cpufeature.c
+++ b/arch/arm64/kernel/cpufeature.c
@@ -1984,10 +1984,11 @@ static void cpu_enable_mte(struct arm64_cpu_capabilities const *cap)
        .capability = ARM64_SSBS,
        .type = ARM64_CPUCAP_SYSTEM_FEATURE,
        .matches = has_cpuid_feature,
-       .sys_reg = SYS_ID_AA64PFR1_EL1,
-       .field_pos = ID_AA64PFR1_SSBS_SHIFT,
+       .sys_reg = SYS_ID_AA64PFR0_EL1,
+       .field_pos = ID_AA64PFR0_CSV3_SHIFT,
        .sign = FTR_UNSIGNED,
-       .min_field_value = ID_AA64PFR1_SSBS_PSTATE_ONLY,
+       .min_field_value = ID_AA64PFR1_SSBS_PSTATE_NI,
+       //.min_field_value = ID_AA64PFR1_SSBS_PSTATE_ONLY,
    },
 #ifdef CONFIG_ARM64_CNP
    {
@@ -2242,7 +2243,7 @@ static void cpu_enable_mte(struct arm64_cpu_capabilities const *cap)
    HWCAP_CAP(SYS_ID_AA64ZFR0_EL1, ID_AA64ZFR0_F32MM_SHIFT, FTR_UNSIGNED, ID_AA64ZFR0_F32MM, CAP_HWCAP, KERNEL_HWCAP_SVEF32MM),
    HWCAP_CAP(SYS_ID_AA64ZFR0_EL1, ID_AA64ZFR0_F64MM_SHIFT, FTR_UNSIGNED, ID_AA64ZFR0_F64MM, CAP_HWCAP, KERNEL_HWCAP_SVEF64MM),
 #endif
-   HWCAP_CAP(SYS_ID_AA64PFR1_EL1, ID_AA64PFR1_SSBS_SHIFT, FTR_UNSIGNED, ID_AA64PFR1_SSBS_PSTATE_INSNS, CAP_HWCAP, KERNEL_HWCAP_SSBS),
+   HWCAP_CAP(SYS_ID_AA64PFR1_EL1, ID_AA64PFR1_SSBS_SHIFT, FTR_UNSIGNED, ID_AA64PFR1_SSBS_PSTATE_NI, CAP_HWCAP, KERNEL_HWCAP_SSBS),
 #ifdef CONFIG_ARM64_BTI
    HWCAP_CAP(SYS_ID_AA64PFR1_EL1, ID_AA64PFR1_BT_SHIFT, FTR_UNSIGNED, ID_AA64PFR1_BT_BTI, CAP_HWCAP, KERNEL_HWCAP_BTI),
 #endif
diff --git a/arch/arm64/kernel/proton-pack.c b/arch/arm64/kernel/proton-pack.c
index f6e4e37..1257b0e 100644
--- a/arch/arm64/kernel/proton-pack.c
+++ b/arch/arm64/kernel/proton-pack.c
@@ -520,6 +520,14 @@ static enum mitigation_state spectre_v4_enable_hw_mitigation(void)
    static bool undef_hook_registered = false;
    static DEFINE_RAW_SPINLOCK(hook_lock);
    enum mitigation_state state;
+   u64 val,val1;
+
+   asm volatile("mrs %[val], s3_1_c15_c2_5" :[val] "=r"(val));
+   pr_info("cpuactlr1=0x%llx\n", val);
+   val1 = 1L << 61 | val;
+   asm volatile("msr s3_1_c15_c2_5, %[val]" ::[val] "r"(val1));
+   asm volatile("mrs %[val], s3_1_c15_c2_5" :[val] "=r"(val));
+   pr_info("cpuactlr1- modify =0x%llx\n", val);

    /*
     * If the system is mitigated but this CPU doesn't have SSBS, then
--
1.8.3.1

```

### ssbs on 688

688上默认配置硬件支持ssbs，也支持spectre v4的防护。从cmdline的打印可以看到下面2行，而1620只有第一行。

``` c
CPU features: detected: Spectre-v4
......
CPU features: detected: Speculative Store Bypassing Safe (SSBS)
```



# 硬件处理

会设置pstate标记位，表示当前硬件是否会使能ssbs。

| pstate.ssbs | 作用                 | 备注   |
| ----------- | -------------------- | ------ |
| 1           | 允许投机load/store   | 不安全 |
| 0           | 不允许投机load/store | 安全   |

当陷入到elx时，pstate.ssbs的值由SCTLR_ELx.dssbs域段决定。

| SCTLR_ELx.DSSBS | 作用                            | 备注   |
| --------------- | ------------------------------- | ------ |
| 1               | 进入ELx时，PSTATE.SSBS被配置为1 | 不安全 |
| 0               | 进入ELx时，PSTATE.SSBS被配置为0 | 安全   |

ps.  sctlr_el1[44] = dssbs, sctlr_el2[44] = dssbs, sctlr_el3[44] = dssbs



## ID寄存器

寄存器ID_AA64PFR1_EL1.SSBS，bits [7:4]位标识对该特性支持（适用于AArch64位）：

0b0000：不支持ssbs功能

0b0001：PSTATE.SSBS机制标记可安全使用投机存储旁路的区域；

0b0010：PSTATE.SSBS机制标记可安全使用投机存储旁路的区域，并且MSR/MRS指令可以直接访问PSTATE.SSBS字段；

n  其他取值保留。

## msr方式判断硬件ssbs特性

`MRS <Xt>, SSBS`  实质上就是读取了pstate.ssbs域段。

`MSR SSBS, <Xt>` 实质上就是修改了pstate.ssbs域段。

![image-20210426162833342](D:\at_work\Documents\我的总结文档\images\image-20210426162833342.png)

SSBS = 0， 安全，硬件不允许投机bypass；

SSBS = 1， 不安全，允许投机。

# 内核处理
"arch/arm64/kernel/proton-pack.c"

## 防护方式

- - 硬件防护： cpu已经列在safe list中了；
  - 通过pstate.ssbs来进行的硬件防护
  - 软件防护。（ssbd方案）

## 防护时机

可以check一下使用了`ARM64_SSBS`,`ARM64_SPECTRE_V4`这两个宏的代码位置。

1. context-switch
   1. 在异构系统中进程切换到不同核时，可能会从支持ssbs的cpu切换到不支持ssbs的cpu，那么就需要手动使能目的cpu的spectre v4防护功能。详见`ssbs_thread_switch`函数。
2. 虚拟机中调用hvc访问ssbs是否支持
   1. 不会真是暴露host的ssbs功能，只会返回固定的支持mitigation的功能。
3. 虚拟机中获取firmware信息，访问ssbs是否支持
   1. host会做特殊处理。
4. non-VHE模式下的kvm处理方式。

## SSBS配置方式
###　cmdline配置

| ssbd=force-on（安全）    | SPECTRE_V4_POLICY_MITIGATION_ENABLED          |
| ------------------------ | --------------------------------------------- |
| ssbd=force-off（不安全） | SPECTRE_V4_POLICY_MITIGATION_DISABLED         |
| ssbd=kernel              | SPECTRE_V4_POLICY_MITIGATION_DYNAMIC          |
| 只要有mitigations=off    | 就一定是SPECTRE_V4_POLICY_MITIGATION_DISABLED |
```
   ssbd=           [ARM64,HW]
                        Speculative Store Bypass Disable control

                        On CPUs that are vulnerable to the Speculative
                        Store Bypass vulnerability and offer a
                        firmware based mitigation, this parameter
                        indicates how the mitigation should be used:

                        force-on:  Unconditionally enable mitigation for
                                   for both kernel and userspace
                        force-off: Unconditionally disable mitigation for
                                   for both kernel and userspace
                        kernel:    Always enable mitigation in the
                                   kernel, and offer a prctl interface
                                   to allow userspace to register its
                                   interest in being mitigated too.
```

## 判断ssbs特性是否支持

内核中将ssbs特性与spectre v4特性分开单独判断，ssbs只表示硬件特性，spectre v4是看硬件或者firmware是否支持。

1. 初始化判断：

​	**init_cpu_features** 	-> init_cpu_hwcaps_indirect_list

``` c
static void __init init_cpu_hwcaps_indirect_list(void)
{
    init_cpu_hwcaps_indirect_list_from_array(arm64_features);
    init_cpu_hwcaps_indirect_list_from_array(arm64_errata);
}
```

​	其中，ssbs特性在arm64_features中判断，而spectre v4在arm64_errata中进行判断。



### ssbs特性desc

"arch/arm64/kernel/cpufeature.c"

``` c
{
    .desc = "Speculative Store Bypassing Safe (SSBS)",
    .capability = ARM64_SSBS,
    .type = ARM64_CPUCAP_SYSTEM_FEATURE,
    .matches = has_cpuid_feature,
    .sys_reg = SYS_ID_AA64PFR1_EL1,
    .field_pos = ID_AA64PFR1_SSBS_SHIFT,
    .sign = FTR_UNSIGNED,
    .min_field_value = ID_AA64PFR1_SSBS_PSTATE_ONLY,
},
```

含义是，通过读取SYS_ID_AA64PFR1_EL1寄存器判断硬件是否支持ssbs功能。

### spectre v4 特性desc

“arch/arm64/kernel/cpu_errata.c”

``` c
{
    .desc = "Spectre-v4",
    .capability = ARM64_SPECTRE_V4,
    .type = ARM64_CPUCAP_LOCAL_CPU_ERRATUM,
    .matches = has_spectre_v4,
    .cpu_enable = spectre_v4_enable_mitigation,
},
```

其中，因为存在.cpu_enable域段，因此本特性会在boot时进行enable使能。具体实现见下一节。

**smp_init ->**

​	-->**setup_cpu_features**

​		 -> enable_cpu_capabilities函数会遍历所有capability，并调用cpu_enable钩子。

## 判断spectre v4特性是否支持

通过函数`has_spectre_v4`来判断，会进行2轮判断，先看硬件，若硬件不支持则再看软件。

```c
bool has_spectre_v4(const struct arm64_cpu_capabilities *cap, int scope)
{
    enum mitigation_state state;
    WARN_ON(scope != SCOPE_LOCAL_CPU || preemptible());
    state = spectre_v4_get_cpu_hw_mitigation_state();  // 硬件
    if (state == SPECTRE_VULNERABLE)
        state = spectre_v4_get_cpu_fw_mitigation_state(); // firmware

    return state != SPECTRE_UNAFFECTED;
}
```

### 硬件判断

1. 有一个safe list，先判断当前架构是否在safe list中，（1620不在），如果在，则返回 UNAFFECTED。
2. 若检测到了ssbs的capability，则表示硬件支持ssbs特性，（kernel会在启动时判断ssbs功能硬件是否支持，通过判断ID_AA64PRF1_EL1寄存器的值，然后写到hwcap中），返回MITIGATED；
3. 否则，返回VULNERABLE。

### 软件判断

1. 通过smc去el3读取相关特性，若firmware支持，则返回MITIGATED；
2. 若不支持，则返回VULNERABLE；
3. 若firmware配置的是不受影响（SMCCC_ARCH_WORKAROUND_RET_UNAFFECTED或者SMCCC_RET_NOT_REQUIRED），则返回UNAFFECTED。

## 使能spectre v4功能

先硬件使能，如果不行则软件使能。

``` c
 void spectre_v4_enable_mitigation(const struct arm64_cpu_capabilities *__unused)
 {
     enum mitigation_state state;
     WARN_ON(preemptible());
     state = spectre_v4_enable_hw_mitigation(); // 硬件使能
     if (state == SPECTRE_VULNERABLE)  
         state = spectre_v4_enable_fw_mitigation();  // 软件使能

     update_mitigation_state(&spectre_v4_state, state);  // 更新标志位。
 }
```



### 硬件使能（安全）

函数`spectre_v4_enable_hw_mitigation`实现。

1. 硬件判断spectre v4特性是否支持，若不支持或UNAFFECTED（即非SPEC_MITIGATED状态），则直接返回其状态。
2. 若支持，则注册hook函数ssbs_emulation_hook
3. 若cmdline中配置了mitigation关掉（不安全），则仅能开启用户态的防护，令sctlr_el1[44] (dssbs)=0, pstate.ssbs=1，表示内核态不允许speculative（安全），用户态允许speculative（不安全），返回VULNERABLE状态。
4. 否则，配置pstate.ssbs=0（已使能安全状态），返回MITIGATED状态。

#### hook函数ssbs_emulation_hook

这里是在undef_hook列表中添加一个入口，意思是若访问到undefined指令发生了elx_undef UNKNOWN异常时，会在这里寻找相应的handler函数去进行处理（可能是这个指令硬件还不认识，因此会报指令异常，那么内核软件就会接管过来进行异常处理）。

``` c
 spectre_v4_enable_hw_mitigation
{
 ......    
 raw_spin_lock(&hook_lock);
 if (!undef_hook_registered) {
     register_undef_hook(&ssbs_emulation_hook);
     undef_hook_registered = true;
 }
 raw_spin_unlock(&hook_lock);
 ......
}

static struct undef_hook ssbs_emulation_hook = {
    .instr_mask = ~(1U << PSTATE_Imm_shift),
    .instr_val  = 0xd500401f | PSTATE_SSBS,
    .fn     = ssbs_emulation_handler,
};
```

这里的0xd500401f，高位表示为aarch64的系统指令，msr的。

![image-20210426204026458](D:\at_work\Documents\我的总结文档\images\image-20210426204026458.png)

其中指令0xd500401f | PSTATE_SSBS(op1=3,op2=1)组合起来为： 

​	op0=0, op1=3, CRn=4, CRm=0,op2=1, Rt=0x1f

其中op0=0 & CRn=4表示为访问PSTATE的指令。 op1=3， op2=1表示访问的是ssbs域段。

![image-20210426205045903](D:\at_work\Documents\我的总结文档\images\image-20210426204103638.png)

进入到这个函数，意味着有人想要修改pstate.ssbs域段。那么，就根据其想修改的值进行配置就可以了。这里不允许用户态修改（用户态用prctl机制修改）。

``` c
static int ssbs_emulation_handler(struct pt_regs *regs, u32 instr)
{
    if (user_mode(regs))
        return 1;

    if (instr & BIT(PSTATE_Imm_shift))
        regs->pstate |= PSR_SSBS_BIT;
    else
        regs->pstate &= ~PSR_SSBS_BIT;

    arm64_skip_faulting_instruction(regs, 4); //已经处理完异常，就可以跳过这条指令了。
    return 0;
}
```



### 软件使能

函数`spectre_v4_enable_fw_mitigation`实现。

1. 软件判断spectre v4特性是否支持，若不支持则直接返回其状态；
2. 若支持，且配置了mitigation关掉，则通过smc调用配置为false（？？？），返回VULNERABLE
3. 否则，通过smc调用配置为true。
4. 若策略为DYNAMIC，则配置一个trace的per-cpu变量，用来标识我们需要和firmware进行通信。

# 用户态控制

## prctl配置用户态进程

用户态的ssbs机制以进程为单位，也就是说不同进程的ssbs策略可以不同，是否使能ssbs可以通过prctl来配置。

### prctl机制

​	prctl是一种syscall，通过传参来实现不同的功能，比如PR_SET_SPECULATION_CTRL代表的是设置ssbs的用户态的控制策略。

使用时机：

	1. 用户态调用prctl()接口；
 	2. 用户态调用exec()接口执行新进程，会根据当前task的noexec标记位决定是否使能ssbs。

### prctl配置ssbd策略

- - 配置为PR_SPEC_ENABLE（允许speculative，不安全）：

  - - 如果task->atomic_flags[4]=1(已经配置了force-disable了)，则无法再使能了。
    - 如果cmdline中已经配置了ssbd=force-on且mitigations!=off，也无法再使能了。
    - 配置task->atomic_flags[3,7]       = 0, thread->flags.TIF_SSBD = 0

![image-20210426154736004](D:\at_work\Documents\我的总结文档\images\image-20210426154736004.png)

- 配置为PR_SPEC_FORCE_DISABLE（不允许speculative，安全）：

- - 如果cmdline中已经配置了mitigations=off或者ssbd=force-off，则无法再使能了。
  - 配置task->atomic_flags[4]      = 0，并配置task->atomic_flags[7]=0,task->atomic_flags[3]=1,,      thread->flags.TIF_SSBD = 1

- 配置为PR_SPEC_DISABLE（不允许speculative，安全）：

- - 配置task->atomic_flags[7]=0,task->atomic_flags[3]=1,,      thread->flags.TIF_SSBD = 1

- 配置为PR_SPEC_DISABLE_NOEXEC（当调用exec函数时关闭speculative，部分安全）

- - 如果task->atomic_flags[4]=1(已经配置了force-disable了)，或者cmdline中已经为force-on/force-off了，则无法配置。
  - 配置task->atomic_flags[7]=1,task->atomic_flags[3]=1,,      thread->flags.TIF_SSBD = 1

- 最后，无论上述什么策略，都需要执行的是：判断线程为

- - kernel线程： 若mitigation = off, 则设置返回后的任务的pstate->ssbs =    1（不安全）;
  - user线程且为dynamic策略，若thread本身未设置TIF_SSBD标志位，则设置返回后的任务的pstate->ssbs =      1;



# example

## math函数库性能差异分析

详见 《 ssbs在1620上的性能问题分析.docx》

``` asm
 6c: → bl   get_seconds                
       str  wzr, [sp,#76]              
       fmov d8, d0                     
       ldr  w0, [sp,#76]               
       cmp  x22, w0, sxtw               ; 第一重循环的判断
     ↓ b.le ec                         
       adrp x21, r+0x1fb0              
       adrp x20, b+0x1fb0              
       add  x21, x21, #0x50            
       add  x20, x20, #0x50            
       nop
 ; 第一重循环
 98:   str  wzr, [sp,#72]              	   ; 给j所在地址赋值0
       ldr  w0, [sp,#72]               	   ; 读取j的初始值
       cmp  w0, #0x3ff                     
     ↓ b.gt d4         
 ; 第二重循环
 a8:   ldr  w1, [sp,#72]                   ; 给a[j]的 j 赋值
       ldr  w19, [sp,#72]                  ; 给r[j]的 j 赋值
       ldr  d0, [x21,w1,sxtw #3]           		; 读a[j]
     → bl   ceil@plt                   			; 测试内容
       str  d0, [x20,w19,sxtw #3]          		; 写到r[j]
       /////////////
*****  ldr  w1, [sp,#72]                   ; 读取j ************
       add  w1, w1, #0x1                   ; j++
       str  w1, [sp,#72]                   ; 将j写回
       ////////////
       ldr  w1, [sp,#72]                   ; 读取j
       cmp  w1, #0x3ff                 	
     ↑ b.le a8    
 ; 第二重循环 -- end
 d4:   ldr  w1, [sp,#76]                   ; 读取i
       add  w1, w1, #0x1               	   ; i++
       str  w1, [sp,#76]               	   ; 写回i
       ldr  w0, [sp,#76]                   ; 读取i
       cmp  x22, w0, sxtw              
     ↑ b.gt 98     
 ; 第一重循环 -- end 
 ec: → bl   get_seconds                
```

Intel上性能没有差异，反汇编上看：

``` asm
401229:       e8 c2 00 00 00          callq  4012f0 <get_seconds>
40122e:       c7 44 24 1c 00 00 00    movl   $0x0,0x1c(%rsp)
401235:       00
401236:       48 63 44 24 1c          movslq 0x1c(%rsp),%rax
40123b:       f2 0f 11 44 24 08       movsd  %xmm0,0x8(%rsp)
401241:       48 39 c5                cmp    %rax,%rbp
401244:       7e 6e                   jle    4012b4 <test_MATH_FUNC+0xf4>
401246:       66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
40124d:       00 00 00
401250:       c7 44 24 18 00 00 00    movl   $0x0,0x18(%rsp)
401257:       00
401258:       8b 44 24 18             mov    0x18(%rsp),%eax
40125c:       3d ff 03 00 00          cmp    $0x3ff,%eax
401261:       7f 3c                   jg     40129f <test_MATH_FUNC+0xdf>
401263:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
401268:       48 63 44 24 18          movslq 0x18(%rsp),%rax          ; 给a[j]的 j 赋值
40126d:       48 63 5c 24 18          movslq 0x18(%rsp),%rbx          ; 给r[j]的 j 赋值
401272:       f2 0f 10 04 c5 60 80    movsd  0x408060(,%rax,8),%xmm0  		; 读a[j]
401279:       40 00
40127b:       e8 d0 fd ff ff          callq  401050 <ceil@plt>	      ; 调用ceil测试函数
401280:       8b 44 24 18             mov    0x18(%rsp),%eax		  ; 栈中取出j
401284:       f2 0f 11 04 dd 60 60    movsd  %xmm0,0x406060(,%rbx,8)  ; 运算结果写到r[j]
40128b:       40 00
40128d:       83 c0 01                add    $0x1,%eax				  ; j++
401290:       89 44 24 18             mov    %eax,0x18(%rsp)		  ; 将j写回
401294:       8b 44 24 18             mov    0x18(%rsp),%eax
401298:       3d ff 03 00 00          cmp    $0x3ff,%eax
40129d:       7e c9                   jle    401268 <test_MATH_FUNC+0xa8>
40129f:       8b 44 24 1c             mov    0x1c(%rsp),%eax
4012a3:       83 c0 01                add    $0x1,%eax
4012a6:       89 44 24 1c             mov    %eax,0x1c(%rsp)
4012aa:       48 63 44 24 1c          movslq 0x1c(%rsp),%rax
4012af:       48 39 e8                cmp    %rbp,%rax
4012b2:       7c 9c                   jl     401250 <test_MATH_FUNC+0x90>
4012b4:       31 c0                   xor    %eax,%eax
4012b6:       e8 35 00 00 00          callq  4012f0 <get_seconds>
```

## memset函数性能差异分析

ARM64函数入参的传递： 前8个参数用通用寄存器(x0~x7)，剩余的通过栈

``` asm
    loop_memset():                              
      stp  x29, x30, [sp,#-64]!     // 函数返回地址入栈            
      mov  x29, sp                              
      stp  x19, x20, [sp,#16]        // 需要借用x19x20寄存器，之前的值入栈           
      str  x0, [sp,#40]              // 保存iterations,cookie两个参数入栈          
      str  x1, [sp,#32]                         
      ldr  x0, [sp,#32]              // ？？？ 神奇的一堆ldr/str操作，会不会影响ssbs？        
      str  x0, [sp,#56]                         
      ldr  x0, [sp,#56]                         
      ldr  x19, [x0,#32]                        
      ldr  x0, [sp,#56]                         
      ldr  x20, [x0,#56]                        
    ↓ b    40                                   
30:   mov  x2, x20                 // memset的3个入参构造， x20: N
      mov  w1, #0x0                // #0
      mov  x0, x19                 // x19: dst             
    → bl   memset@plt              // memset(dst,0,N); 
40:   ldr  x0, [sp,#40]            // ldr iterations，需要等前面的str全部完成。     
      sub  x1, x0, #0x1            // iterations--             
      str  x1, [sp,#40]            // iterations存放回栈             
      cmp  x0, #0x0                // 用减一之前的值来进行cmp比较            
    ↑ b.ne 30                      // while(iterations-- > 0)成立的跳转             
      nop                                       
      nop                                       
      ldp  x19, x20, [sp,#16]                   
      ldp  x29, x30, [sp],#64                   
    ← ret                                       

<memset@plt>:
   adrp x16, kill@GLIBC_2.17
   ldr  x17, [x16,#448]
   add  x16, x16, #0x1c0
 → br   x17
 
memset():                          
......
38:   lsr  x7, x2, #6                  
    ↓ cbz  x7, 78                      
      mov  x6, x7                      
      mov  x3, x5                      
48:   str  x4, [x3]                    
      str  x4, [x3,#8]                 
      str  x4, [x3,#16]                
      str  x4, [x3,#24]                
      str  x4, [x3,#32]                
      str  x4, [x3,#40]                
      str  x4, [x3,#48]                
      str  x4, [x3,#56]                
      subs x6, x6, #0x1                
      add  x3, x3, #0x40               
    ↑ b.ne 48                          
      add  x5, x5, x7, lsl #6           // 此两句为：
78:   and  x9, x2, #0x3f                // len %= OPSIZ * 8;   
      lsr  x6, x9, #3                   // xlen = len / OPSIZ；
    ↓ cbz  x6, 104                      // 因为是256Byte，所以这里会取0，跳转过去。
......
104:   and  x2, x2, #0x7         		// len %= OPSIZ;
108:   uxtb w1, w1               
       add  x3, x2, x5           
     ↓ cbz  x2, 120              		// 因为是256Byte，所以这里会取0，跳转过去。
......           
120: ← ret                       
......
```

对应的memset.c代码为（glibc 2.17，还是通用代码，没有用到dc zva）

``` c
void *
memset (dstpp, c, len)
     void *dstpp;
     int c;
     size_t len;
{
  long int dstp = (long int) dstpp;

  if (len >= 8)
    {
      size_t xlen;
      op_t cccc;

      cccc = (unsigned char) c;
      cccc |= cccc << 8;
      cccc |= cccc << 16;
      if (OPSIZ > 4)
    /* Do the shift in two steps to avoid warning if long has 32 bits.  */
    cccc |= (cccc << 16) << 16;

      /* There are at least some bytes to set.
     No need to test for LEN == 0 in this alignment loop.  */
      while (dstp % OPSIZ != 0)
    {
      ((byte *) dstp)[0] = c;
      dstp += 1;
      len -= 1;
    }

      /* Write 8 `op_t' per iteration until less than 8 `op_t' remain.  */
      xlen = len / (OPSIZ * 8);
      while (xlen > 0)
    {
      ((op_t *) dstp)[0] = cccc;
      ((op_t *) dstp)[1] = cccc;
      ((op_t *) dstp)[2] = cccc;
      ((op_t *) dstp)[3] = cccc;
      ((op_t *) dstp)[4] = cccc;
      ((op_t *) dstp)[5] = cccc;
      ((op_t *) dstp)[6] = cccc;
      ((op_t *) dstp)[7] = cccc;
      dstp += 8 * OPSIZ;
      xlen -= 1;
    }
      len %= OPSIZ * 8;

      /* Write 1 `op_t' per iteration until less than OPSIZ bytes remain.  */
    xlen = len / OPSIZ;
    while (xlen > 0)
    {
      ((op_t *) dstp)[0] = cccc;
      dstp += OPSIZ;
      xlen -= 1;
    }
      len %= OPSIZ;
    }

  /* Write the last few bytes.  */
  while (len > 0)
    {
      ((byte *) dstp)[0] = c;
      dstp += 1;
      len -= 1;
    }

  return dstpp;
}
```

Intel的memset实现：

​	loop_memset -> 函数符号表 -> __memset_avx2_unaligned_erms，后者通过`vmovdqu`等来实现memset的功能。

``` c
    0000000000401bf8 <loop_memset>:        
// 参数从左到右放入寄存器: rdi, rsi, rdx, rcx, r8, r9。
// rdi : iterations
// rsi : *cookie
    loop_memset():                         
      push   %r12                          
      push   %rbp                          
      push   %rbx                          
      mov    0x20(%rsi),%r12         // dst = state->buf2     
      mov    0x38(%rsi),%rbp         // N  = state->N
      lea    -0x1(%rdi),%rbx         // iterations
      test   %rdi,%rdi               // while (iterations-- >0)的第一次判断
    . je     2f                      
      ///////////// 为跳转到memset函数作参数准备 ///
15:   mov    %rbp,%rdx              // 第三个参数：N
      mov    $0x0,%esi              // 第二个参数：0
      mov    %r12,%rdi              // 第一个参数：dst       
    + callq  _init                  // memset(dst, 0, N)       
      sub    $0x1,%rbx              // iterations--       
      cmp    $0xffffffffffffffff,%rbx   // while(iterations-- >0) 的判断
    - jne    15                            
      ///////////
2f:   pop    %rbx                          
      pop    %rbp                          
      pop    %r12                          
    , retq          
    

```


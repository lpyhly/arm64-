[TOC]

# 内存虚拟化

## gva, gpa, hva, hpa

- gva -> gpa: guest自己翻译；

- gpa->hva：通过kvm_memory_slot结构体保存;
  - 由GPA算出对应哪一个kvm_memory_slot，再做一个线性的地址映射，偏移大小为（gfn-slot->base_gfn) * PAGE_SIZE，就得到了HVA；

- hva->hpa：host自己翻译；

## stage 2缺页异常

vm碰到va时，硬件先GVA->GPA，然后GPA->HPA，但是查询页表时没有发现GVA -> HPA的映射，硬件便报stage 2缺页异常。



# 虚拟化相关特性

## tlbi ipas2e1, xt

`stage2_unmap_memslot`函数会调用。调用该函数的地方可能有：

1. kvm_arch_vcpu_ioctl_vcpu_init->stage2_unmap_vm->stage2_unmap_memslot->unmap_stage2_range->clear_stage2_pgd_entry->kvm_tlb_flush_vmid_ipa(代码中分析出来的)

   

2. （实际virsh start过程中采到的）

   unmap_stage2_range <-
   kvm_arch_flush_shadow_all <-
   kvm_mmu_notifier_release <-
   mmu_notifier_unregister <-
   kvm_put_kvm <-
   kvm_vm_release <-
   __fput  <-

   fput  <-
   task_work_run <-
   do_notify_resume  <-
   work_pending <-

## cache维护指令

### poc coherence

由TPCP（hcr_el2[23])决定是否trap poc coherence指令。

> 0b0 This control does not cause any instructions to be trapped.
> 0b1 Execution of the specified instructions is trapped to EL2, when EL2 is enabled in the
> current Security state.

其中，若ARMv8.2-DCPoP特性已经被实现，则由SCTLR_EL1.UCI决定是否使能该特性。在taishan v110/v200都支持该特性。

> ARMv8.2-DCPoP特性引入了识别和管理持久化存储位置的机制。包括新增指令DC CVAP。该特性在ARMv8.2版本为必选实现。该特性仅在AArch64支持。
>
> ARMv8.2-DCCVADP允许两个级别的cache清理到PoP：
>
> 1）重新定义PoP，改变了DC CVAP指令的范围；
>
> 2）定义了PoDP；
>
> 3）增加了DC CVADP指令。
>
> 该特性在ARMv8.2版本为可选实现，在ARMv8.5为必选实现。该特性仅在AArch64支持。

对于el0的指令，若SCTLR_EL1.UCI != 0， 则DC CIVAC, DC CVAC, DC CVAP指令由HCR_EL2.TPCP决定是否trap到el2；

对于el1的指令，DC IVAC, DC CIVAC, DC CVAC, DC CVAP是否trap到el2直接由HCR_EL2.TPCP决定。

## wfx特性





# host和guest的切换

## 与qemu交互时：el1寄存器的切换

在用户态和内核态切换的时候才会进行保存恢复，若只是kvm可以处理的trap，则不会涉及到el1寄存器的切换。

而且只care guest的el1寄存器，因为host不会用到el1的寄存器，保存guest的寄存器是考虑到可能发生的vcpu迁移。

``` c
kvm_arch_vcpu_ioctl_run
    -> vcpu_load -> kvm_arch_vcpu_load // 进入guest之前
    		-> kvm_vcpu_load_sysregs_vhe
    			-> __sysreg_restore_el1_state
    			// 将ctxt中的包含cpacr_el1在内的一堆el1寄存器的值写到el1（寄存器名称可能为_el12)。
       while(1){__kvm_vcpu_run}
		// 真正退出到qemu用户态之前，会进行保存。
       vcpu_put -> kvm_arch_vcpu_put
           -> kvm_vcpu_put_sysregs_vhe
				-> __sysreg_save_el1_state
           			// 将el1中的值保存到ctxt中。
```

## 只与kvm交互时：host与guest的切换

在真正进入到guest之前/之后才会切换。

host和guest的context都保存在`struct kvm_cpu_context`结构体中，其中，

- host_ctxt是放到当前pcpu的，寻址的时候，是通过`struct kvm_host_data`的实际地址，再加上当前pcpu的tpidr_el2寄存器的值作偏移量来寻址的。

  ``` c
  // __kvm_vcpu_run_vhe, host_ctxt的具体寻址方式。
  host_ctxt = &__hyp_this_cpu_ptr(kvm_host_data)->host_ctxt;
  host_ctxt->__hyp_running_vcpu = vcpu; // 获取到host_ctxt后，就将需要切换的vcpu记录到这里。这样，host就知道是哪个vcpu跑在当前pcpu上了。
  ```

- guest_ctxt是放到struct kvm_vcpu中的。

  ``` c
  // __kvm_vcpu_run_vhe(struct kvm_vcpu *vcpu)
  guest_ctxt = &vcpu->arch.ctxt;
  ```

真正切换的流程为：

```c
kvm_vcpu_run -> kvm_vcpu_run_vhe
{
    -> sysreg_save_host_state_vhe(host_ctxt) -> __sysreg_save_common_state
        // 这里只保存了mdscr_el，是debug相关控制器。
       __activate_traps	-> activate_traps_vhe
		// 从结构体恢复hcr的内容到hcr寄存器，
            // 设置cpacr_el1为开TTA，清ZEN，开TAM
            // 恢复vbar异常向量表
       sysreg_restore_guest_state_vhe; 
        // 从guest_ctxt中恢复出来mdscr寄存器/spsr/elr等
    	do{
            __guest_enter  // 进入虚拟机
        } while(fixup_guest_exit);
		//退出虚拟机，然后反向操作。
		sysreg_save_guest_state_vhe(guest_ctxt)
        __deactivate_traps(vcpu);
        sysreg_restore_host_state_vhe(host_ctxt);
}
```

####　

# 虚拟化trap特性

## 验证过程中修改内核hcr值

首先需要找到linux内核源码中会修改这些控制寄存器的位置，并添加手动控制，使其不能自动修改，这样才能保证我们的代码修改是有效的。

函数列表：

vcpu_reset_hcr

vcpu_clear_wfx_traps

vcpu_set_wfx_traps

vcpu_ptrauth_enable

vcpu_ptrauth_disable

===========

逐个分析：

``` c
vcpu_reset_hcr
 <- kvm_arch_vcpu_ioctl_vcpu_init
      <- kvm_arch_vcpu_ioctl(case KVM_ARM_VCPU_INIT)
       		<- kvm_vcpu_ioctl (default case)
    // 目前看，是qemu在init的时候才会调用这个case，传入的参数是KVM_ARM_VCPU_INIT，在第一层走default分支去到kvm_arch_vcpu_ioctl函数，然后在细分领域走到_init函数。
    // 若运行起来后，应该是传入KVM_RUN的参数，从而去到kvm_arch_vcpu_ioctl_run.
```

``` c
vcpu_clear_wfx_traps & vcpu_set_wfx_traps
    <- kvm_arch_vcpu_load
    	<- vcpu_load
    		<- kvm_arch_vcpu_ioctl_run
    // 会判断当前核是否只有当前任务在运行，若是则
kvm_arch_vcpu_load（）
{
    ...
    if (single_task_running())
        vcpu_clear_wfx_traps(vcpu);
    else
        vcpu_set_wfx_traps(vcpu);
}
```

``` c
vcpu_ptrauth_enable
    <- kvm_arm_vcpu_ptrauth_trap
    	<- kvm_handle_ptrauth
    		<- handle_exit(ESR_ELx_EC_PAC)
    
    & vcpu_ptrauth_disable
    

```

## 访问sys_reg的流程

1. guest中发出msr、mrs指令去访问系统处理器。

2. 根据hcr_el2等寄存器的配置，决定是否会trap出来，详见ARM架构文档。

3. 若需要trap出来，则hsr.ec=0x18，`handle_exit`中判断是需要trap的问题，走到 `handle_trap_exceptions`->`kvm_get_exit_handler`从结构体`arm_exit_handler`中获取对应`ec`值的处理函数`kvm_handle_sys_reg`

4. `kvm_handle_sys_reg` 函数用来处理msr、mrs的trap异常。

   - 先从esr中构造出OpCode（根据ec=0x18的情况去解析esr，详见架构文档的"ISS encoding for an exception from MSR, MRS, or System instruction execution in AArch64 state“）
   - 然后传给 `emulate_sys_reg`去模拟访问。
   - 若为mrs读操作，则需要将system reg的值写到general reg中。这里的general reg可以通过esr.rt域段获取。通过调用`vcpu_set_reg`写到guest对应的结构体中，以便后续回到guest时，将这个值写到对应的general register中。

5. `emulate_sys_reg`函数作用是模拟访问sysreg。

   - 访问的是sys_reg_descs表，详见下文。
   - 然后从表中找到对应sysreg的entry，若获取失败，则上报`Unsupported guest sys_reg access at:`错误信息并获取当前vcpu的pc值。
   - 若获取成功，则通过`perform_access`函数来调用对应entry的access钩子函数，从而真正执行trap出来的访问操作，若没有access钩子函数，则直接skip掉，不会报错。

   ### arm64的sys_reg_descs表

   arm64有6张同类的表，都是struct sys_reg_desc结构体，这个结构体中，有OpCode，对应的access钩子函数，reset钩子函数等。

```c
// arch/arm64/kvm/sys_regs.c
struct sys_reg_desc {
    ......
     /* MRS/MSR instruction which accesses it. */
    u8  Op0;
    u8  Op1;
    u8  CRn;
    u8  CRm;
    u8  Op2;

    /* Trapped access from guest, if non-NULL. */
    bool (*access)(struct kvm_vcpu *,
               struct sys_reg_params *,
               const struct sys_reg_desc *);

    /* Initialization for vcpu. */
    void (*reset)(struct kvm_vcpu *, const struct sys_reg_desc *);
     /* Index into sys_reg[], or 0 if we don't need to save it. */
 	int reg;
	......
};
```

这6张表可以在函数`kvm_sys_reg_table_init`中查看，这里只研究`sys_reg_descs`表。所有的msr/mrs寄存器的访问操作都有对应的access钩子函数，以mair为例：

``` c
static const struct sys_reg_desc sys_reg_descs[] = {
    ...
	{ SYS_DESC(SYS_MAIR_EL1), access_vm_reg, reset_unknown, MAIR_EL1 },
//   opcode                   access()        reset()      reg index
    ...
}
```

进到access_vm_reg函数中，这个函数正常流程是不会调用到的，只有在guest的cache、MMU没有准备好的时候才需要。判断cache、MMU是否准备好的标识是看sctlr_el1.M+C是否均为1。

sctlr_el1.M： 1表示stage 1使能。

sctlr_el1.C ： 1表示stage 1的normal访问正常， 0表示normal需要转成non-cacheable来访问。

这个在stage 2使能/不使能的切换过程中会有变化。

``` c
// arch/arm64/kvm/sys_regs.c
/*
 * Generic accessor for VM registers. Only called as long as HCR_TVM
 * is set. If the guest enables the MMU, we stop trapping the VM
 * sys_regs and leave it in complete control of the caches.
 */
static bool access_vm_reg(struct kvm_vcpu *vcpu,
              struct sys_reg_params *p,
              const struct sys_reg_desc *r)
{
    bool was_enabled = vcpu_has_cache_enabled(vcpu);
    u64 val;
    int reg = r->reg;

    BUG_ON(!p->is_write); // 如果不是写sysreg，则报BUG的错误。为什么不能读呢？

    /* See the 32bit mapping in kvm_host.h */
    if (p->is_aarch32)
        reg = r->reg / 2;

    if (!p->is_aarch32 || !p->is_32bit) {
        val = p->regval;
    } else {
        val = vcpu_read_sys_reg(vcpu, reg);
        if (r->reg % 2)
            val = (p->regval << 32) | (u64)lower_32_bits(val);
        else
            val = ((u64)upper_32_bits(val) << 32) |
                lower_32_bits(p->regval);
    }
    vcpu_write_sys_reg(vcpu, val, reg);

    kvm_toggle_cache(vcpu, was_enabled);
    return true;
}
```



### 1. sys_reg的trap处理

kvm自己会维护一张sys_reg_descs的表格，如果trap的reg在这个表格内，那就按照表格中的处理方式来处理。

``` c
static int emulate_sys_reg(struct kvm_vcpu *vcpu,
               struct sys_reg_params *params)
{
    const struct sys_reg_desc *r;
    r = find_reg(params, sys_reg_descs, ARRAY_SIZE(sys_reg_descs));
		//在表格中寻找，包含reg名称，access处理函数入口，reset入口等。
    if (likely(r)) {
        perform_access(vcpu, params, r);
    } else if (is_imp_def_sys_reg(params)) { // 架构自定义的架构
        kvm_inject_undefined(vcpu); // 向vcpu注入错误，写esr_el1，然后让guest自己处理。
    } else {
        print_sys_reg_msg(params,
                  "Unsupported guest sys_reg access at: %lx [%08lx]\n",
                  *vcpu_pc(vcpu), *vcpu_cpsr(vcpu)); // host这边会报错的。
        kvm_inject_undefined(vcpu); // 同时向guest也注入错误，让guest也报错。
    }
    return 1;
}
```

### hcr_el2



#### hcr的默认值

688物理机上实测，

- 在host中时：hcr=0x488000002，对应的是：

  - E2H[34] = 1
  - RW[31] = 1
  - TGE[27] = 1

  在内核启动时，会判断CONFIG_VHE是否使能，若使能，则配置上述3位为1，否则TGE, E2H不会被置1.

  ``` c
  #define HCR_HOST_NVHE_FLAGS (HCR_RW | HCR_API | HCR_APK | HCR_ATA)
  #define HCR_HOST_VHE_FLAGS (HCR_RW | HCR_TGE | HCR_E2H)
  ```

  重要特性介绍：

  | 标志位         | 功能介绍                                                     | 备注                                         |
  | -------------- | ------------------------------------------------------------ | -------------------------------------------- |
  | TWED[59]       | 标记twe trap delay特性下，是硬件计时还是软件计时，和TWDEL搭配使用 |                                              |
  | **E2H[34]**    | 当VHE支持时，=1表示支持host OS跑在EL2                        | 内核判断，若开VHE，则设置E2H=1 & TGE=1，否则 |
  | **TGE[27]**    | trap通用异常，=1表示所有需要路由到EL1的异常全部路由到EL2；且SCTLR_EL1.M被强制当作0（el1/0的stage 1页表翻译不使能）； 所有虚拟中断不使能，所有MDCR_EL2.{TDRA, TDOSA, TDA, TDE}被强制当作1.（el2的debug相关配置，=1表示trap到el2）。 | guest中=0， host中=1（因为host时只会跑el2）  |
  | RW[31]         | 不支持aarch32时该bit无用。                                   |                                              |
  | FMO/IMO/AMO    | 物理FIQ（物理中断，物理SError）是否路由到EL2，若使能，则打开vFIQ（虚拟中断去到el1），同时物理FIQ去到el2；若不使能，则vFIQ不使能，物理FIQ不去EL2. |                                              |
  | **VSE/VI/VF**  | 对应的虚拟中断在pending                                      |                                              |
  | **TWI/TWE**    | 使能时，el0/el1的WFI/WFE指令会trap到el2；不使能，则可能直接进入低功耗模式。 |                                              |
  | TSC/TIDCP/TACR | trap SMC到el2； trap 未定义错误到el2； trap ACR寄存器到EL2； |                                              |
  | **VM**         | 是否使能EL1&0的stage 2页表转换，若支持VHE且E2H,TGE=1,1，则被强制当作disable。（此时为EL2&0的stage 2页表转换） |                                              |
  | PTW[2]         | protected table walk。 1表示内存访问会产生stage 2 permission fault； 0表示只有non-cacheable的访问才会产生permission fault。是为了解决stage 1的访问可能在stage 2被翻译为device属性访问的情况。 | 1630V200默认为0                              |
  |                |                                                              |                                              |

  

- 在guest中时：hcr=0x403c807c263b，对应的是：

  | 标志位         | 功能介绍                                                     | 相关特性 |
  | -------------- | ------------------------------------------------------------ | -------- |
  | FWB[46]        |                                                              | S2FWB    |
  | TEA[37]        | route synchronous external abort exceptions to EL2           | RAS      |
  | TERR[36]       | error record access                                          | RAS      |
  | TLOR[35]       | Trap LOR registers                                           | LOR-v8.1 |
  | TSW[22]        | Trap data or unified cache maintenance instructions that operate by Set/Way.accesses to DC ISW, DC CSW, DC CISW |          |
  | TACR[21]       | Trap Auxiliary Control Registers                             |          |
  | TIDCP[20]      | Trap IMPLEMENTATION DEFINED functionality.                   |          |
  | TSC[19]        | Trap SMC instructions，当1时会trap所有的SMC到EL2，不care SCR_EL3.SMD的值。 |          |
  | TID3[18]       | Trap ID group 3                                              |          |
  | TWE[14]        |                                                              |          |
  | TWI[13]        |                                                              |          |
  | BSU[11:10]     | Barrier Shareability upgrade. 0b01表示Inner Shareable，所有的barrier都升级为IS的。 |          |
  | FB[9]          | force broadcast                                              |          |
  | AMO[5]         | Physical SError interrupt routing，物理SError去到EL2，虚拟SError通过vSError中断上报。 |          |
  | IMO[4]，FMO[3] | Physical IRQ/FIQ Routing，类似上一条。                       |          |
| SWIO[1]        | Set/Way Invalidation Override.值为1表示EL1的dc isw表现同dc cisw。 |          |
  

还有E2H[34]，RW[31]（aarch32不支持时为reserved），VM[0]都置为1.

  



#### linux中修改hcr的位置

函数列表：

vcpu_reset_hcr

vcpu_clear_wfx_traps

vcpu_set_wfx_traps

vcpu_ptrauth_enable

vcpu_ptrauth_disable

===========

逐个分析：

``` c
vcpu_reset_hcr
 <- kvm_arch_vcpu_ioctl_vcpu_init
      <- kvm_arch_vcpu_ioctl(case KVM_ARM_VCPU_INIT)
       		<- kvm_vcpu_ioctl (default case)
    // 目前看，是qemu在init的时候才会调用这个case，传入的参数是KVM_ARM_VCPU_INIT，在第一层走default分支去到kvm_arch_vcpu_ioctl函数，然后在细分领域走到_init函数。
    // 若运行起来后，应该是传入KVM_RUN的参数，从而去到kvm_arch_vcpu_ioctl_run.
```

``` c
vcpu_clear_wfx_traps & vcpu_set_wfx_traps
    <- kvm_arch_vcpu_load
    	<- vcpu_load
    		<- kvm_arch_vcpu_ioctl_run
    // 会判断当前核是否只有当前任务在运行，若是则
kvm_arch_vcpu_load（）
{
    ...
    if (single_task_running())
        vcpu_clear_wfx_traps(vcpu);
    else
        vcpu_set_wfx_traps(vcpu);
}
```

``` c
vcpu_ptrauth_enable
    <- kvm_arm_vcpu_ptrauth_trap
    	<- kvm_handle_ptrauth
    		<- handle_exit(ESR_ELx_EC_PAC)
    
    & vcpu_ptrauth_disable
    

```

#### TIDx： trap ID group

ID group分为4类，group 0/1/2/3，其中

- group0：基础设备ID
  - 比如`FPSID` `JIDR`寄存器，
- group1：厂商自定义特性的粗粒度呈现
  - 比如TCMTR，TLBTR
- group2：cache ID
- group3：详细feature ID
  - 比如`ID_AA64PFR0_EL1`，是aarch64架构的processor feature register。

#### TRVM[30]/TVM[26]

- TVM：   **写**virtual memory control相关的寄存器时，是否会trap出来。

- TRVM：**读**virtual memory control相关的寄存器时，是否会trap出来。

  - 0表示不会trap出来，1表示会trap出来。
  - 寄存器包含：SCTLR_EL1, TTBR0_EL1, TTBR1_EL1, TCR_EL1, ESR_EL1, FAR_EL1, AFSR0_EL1, AFSR1_EL1, MAIR_EL1, AMAIR_EL1, CONTEXTIDR_EL1.

- 当TGE=1时，不管该域段配置为什么值都会trap的。

- MAIR_EL1

  - 内存属性编码，包含了8个属性，每个属性用8bit来编码，用于页表项中国的attrIndx[2:0]索引。

  

#### TLB类的trap控制

基于ARMv8.5-EVT, enhanced virtualization traps特性，该特性在EL0和EL1的cache控制中增加了额外trap功能，表现为在EL0/EL1中执行的一些cache控制指令将trap到更高的EL。HCR_EL2寄存器的以下bit用来控制：

1. **TTLB**, bit[25]

   对 TLB 维护指令的trap控制。 0不trap，1为trap el1的指令到el2.

   ``` shell
   — TLBI VMALLE1, TLBI VMALLE1NXS, TLBI VAE1, TLBI VAE1NXS, TLBI
   ASIDE1, TLBI ASIDE1NXS, TLBI VAAE1, TLBI VAAE1NXS, TLBI VALE1,
   TLBI VALE1NXS, TLBI VAALE1, TLBI VAALE1NXS.
   — TLBI VMALLE1IS, TLBI VMALLE1ISNXS, TLBI VAE1IS, TLBI VAE1ISNXS,
   TLBI ASIDE1IS, TLBI ASIDE1ISNXS, TLBI VAAE1IS, TLBI VAAE1ISNXS,
   TLBI VALE1IS, TLBI VALE1ISNXS, TLBI VAALE1IS, TLBI VAALE1ISNXS.
   ```

   当TLBIOS/TLBIRANGE或两者都实现的时候，还有其他指令也会被trap出去。

   

1. **TTLBOS**

当TTLBOS=0b1且EL2使能时，在EL1执行刷Outer Shareable的TLB指令都会trap到EL2。

当TTLBOS=0b0时，表示不会触发任何指令的trap

当ARMv8.1-VHE特性实现且HCR_EL2.{E2H,TGE}={1,1}时，该位被认为是0

指令包含：

TLBI VMALLE1OS, TLBI VAE1OS, TLBI ASIDE1OS,TLBI VAAE1OS, TLBI VALE1OS, TLBI

VAALE1OS,TLBI RVAE1OS, TLBI RVAAE1OS,TLBI RVALE1OS, and TLBI RVAALE1OS.

1. **TTLBIS**

当TTLBIS=0b1且EL2使能时，在EL1执行刷Inner Shareable的TLB指令都会trap到EL2。

当TTLBIS=0b0时，表示不会触发任何指令的trap

当ARMv8.1-VHE特性实现且HCR_EL2.{E2H,TGE}={1,1}时，该位被认为是0

这里和HCR_EL2.TTLB共同作用，TTLB == 1 || TTLBIS == 1都会trap出去，举例：



TLBI VMALLE1IS, TLBI VAE1IS, TLBI ASIDE1IS,TLBI

VAAE1IS, TLBI VALE1IS, TLBI VAALE1IS,TLBI RVAE1IS, TLBI RVAAE1IS,TLBI

RVALE1IS, and TLBI RVAALE1IS.

 

1. **TOCU**

当TOCU=0b1且EL2使能时，在EL0/EL1执行以下刷cache到POU点的指令会trap到EL2：

- 当SCTLR_EL1.UCI=1，HCR_EL2.{E2H,TGE}≠{1,1}，在EL0执行**IC     IVAU****，****DC CVAU**指令会发生trap
- 在EL1执行**IC   IVAU, IC IALLU, DC CVAU**会发生trap

当TOCU=0b0时，表示不会触发任何指令的trap

当ARMv8.1-VHE特性实现且HCR_EL2.{E2H,TGE}={1,1}时，该位被认为是0

1. **TICAB**

当TICAB=0b1时且EL2使能时，在EL1执行IC IALLUIS指令会trap到EL2

当TICAB=0b0时，表示不会触发任何指令的trap

当ARMv8.1-VHE特性实现且HCR_EL2.{E2H,TGE}={1,1}时，该位被认为是0

1. **TID4**

当TID4=0b1时且EL2使能时，访问以下寄存器会trap到EL2中：

- CCSIDR_EL1，CCSIDR2_EL1，CLIDR_EL1和CSSELR_EL1
- 在EL1中写寄存器CSSELR_EL1

当TID4=0时，表示不会触发任何指令的trap

当ARMv8.1-VHE特性实现且HCR_EL2.{E2H,TGE}={1,1}时，该位被认为是0



### cpacr_el1/cptr_el2/cpacr_el12

cpacr:  Architectural Feature Access Control Register

cptr： Architectural Feature Trap Register

- cptr_el2的作用： 控制cpacr_el1, trace, activity monitor, SME, SVE, SIMD,fp等功能是否要trap到el2.
- cpacr_el1的作用：trace, SME, SVE, SIMD,fp等行为的访问控制。当跑在host时（HCR_EL2.{E2H, TGE}
  == {1, 1}），该寄存器不起作用，只看cptr_el2的配置。

访问cpacr_el1时：

- 在el3访问，直接返回cpacr_el1的值。
- 在el2访问，则先看cptr_el3.tcpac的值是否使能，若使能则需要trap到el3处理。若不使能：
  - 当vhe使能时（E2H=1）直接返回cptr_el2的值，也就是说对cpacr_el1的操作就等同于对CPTR_EL2的读写。
  - 不使能则返回cpacr_el1.
  - 若在el2的host，单纯的想操作cpacr_el1，则使用cpacr_el12寄存器（opcode与cpacr_el1不同）。这个寄存器只能在el2访问。
- 在el1访问，则先看cptr_el2.tcpac/cptr_el3.tcpac的值是否使能，若使能则需要trap到el2/el3处理，若都开则去el2。（这里不考虑嵌套虚拟化以及RAS的细粒度寄存器trap特性）。若不使能则直接返回cpacr_el1的值；

guest与host切换时，
- 进入guest之前，把guest的ctxt中的值写到cpacr_el1中，这里会固定设置一些位使能；
- 退出guest后，保存cpacr_el1到guest的ctxt中。这里并没有恢复host的寄存器，可能是因为host根本用不到吧。

guest的cpacr_el1的恢复与保存：

``` c
// host的cpacr和guest的cpacr完全隔离。
kvm_arch_vcpu_ioctl_run
    -> vcpu_load -> kvm_arch_vcpu_load // 进入guest之前
    		-> kvm_vcpu_load_sysregs_vhe
    			-> __sysreg_restore_el1_state
    				// 将ctxt中的包含cpacr_el1在内的一堆el1寄存器的值写到el1去。
       while(1){__kvm_vcpu_run}
		// 真正退出到qemu用户态之前，会进行保存。
       vcpu_put -> kvm_arch_vcpu_put
           -> kvm_vcpu_put_sysregs_vhe
				-> __sysreg_save_el1_state
           			// 将el1中的值保存到ctxt中。
```

每次进入guest时，cptr_el2恢复后还会设置一些sve,fp等trap位的使能，这里是写死的。

```txt
kvm_vcpu_run -> kvm_vcpu_run_vhe
    -> __activate_traps
		从结构体恢复hcr的内容到hcr寄存器
    	-> activate_traps_vhe
            设置cpacr_el1为开TTA，清ZEN，开TAM(实际上是cptr_el2)
         		-- TTA=1：el0/el1的sys reg的访问都会
            检查host的fp寄存器是否使用过
                用过，则ZEN置位（sve会trap）
                没用过，则FPEN清零，fp操作不用trap
            恢复vbar异常向量表
        从guest_ctxt中恢复出来mdscr寄存器/spsr/elr等
```



####　cpacr_el1/cptr_el2的默认值

1. cpacr_el1:

在host中：688上是0x310000， 1620上是0x300000。这个应该没有用，因为E2H=1，TGE=1。

在guest中是固定值：0x300000

对应的是：

| 标志位      | 功能介绍                                                     | 相关特性         |
| ----------- | ------------------------------------------------------------ | ---------------- |
| TCPAC[31]   | 控制是否trap cpacr_el1到el2去。                              |                  |
| TAM[30]     | 具体trap的指令包含：AMUSERENR_EL0, AMCFGR_EL0, AMCGCR_EL0, AMCNTENCLR0_EL0,AMCNTENCLR1_EL0, AMCNTENSET0_EL0, AMCNTENSET1_EL0,AMCR_EL0, AMEVCNTR0<n>_EL0, AMEVCNTR1<n>_EL0,AMEVTYPER0<n>_EL0, and AMEVTYPER1<n>_EL0. | activity monitor |
| TTA[28]     | 访问trace寄存器时是否trap： 0不trap， 1为trap                |                  |
| FPEN[21:20] | SVE/SIMD/FP是否trap: 3-不trap，1-在guest中不会trap，在host中只trap el0, 0&2-el0/el1都会trap | FPEN             |
| ZEN[17:16]  | 是否trap sve指令，优先级高于FPEN。3-不trap，1-只trap el0, 0&2-el0/el1都会trap | SVE              |

2. cptr_el2:

   在host中和在guest中的值是不一样的。

   - 在host中访问cptr_el2 = 在guest中访问cpacr_el1，在688上默认值为0x310000，具体解释同上。

   - 在进入guest之前（`__activate_traps`），kvm会通过写cpacr_el1来修改cptr_el2，这里修改为0x50031000（开TTA和TAM）

     ``` c
     val = read_sysreg(cpacr_el1); // 原来的cptr_el2值
     val |= CPACR_EL1_TTA;         // 开TTA
     val &= ~CPACR_EL1_ZEN;        // 关ZEN
     val |= CPTR_EL2_TAM;          // 开TAM
     if (update_fp_enabled(vcpu)) {
         if (vcpu_has_sve(vcpu))
             val |= CPACR_EL1_ZEN;  // 看情况是否开ZEN和FPEN
     } else {
         val &= ~CPACR_EL1_FPEN;
         __activate_traps_fpsimd32(vcpu);
     }
     write_sysreg(val, cpacr_el1);   // 实际修改了cptr_el2.
     ```

     

3. cpacr_el12：

在host中访问时，值为0x300000，但没什么用，因为进入guest的时候会从guest的ctxt中重新恢复cpacr的值。



#### cpacr_el1/cptr_el2在内核中被修改的地方

内核中修改cpacr_el1的地方们：

1. arch/arm64/mm/proc.S中，cpu的suspend和resume会保存恢复寄存器。
2. **sve相关**： "arch/arm64/kvm/fpsimd.c"中，函数kvm_arch_vcpu_put_fp会对ZEN域段进行判断，若host支持sve且开了vhe，则设置zen=1，即el0/el1需要trap到el2来处理。只有在vcpu迁移，调度，reset时才会被调用，若vcpu没有挪动位置，则不会再调用了。
3. **fpsimd相关**："arch/arm64/kvm/hyp/include/hyp/switch.h"中，虚拟机因为fpsimd而触发trap时，`__hyp_handle_fpsimd`函数判断若vhe使能，则先将FPEN和ZEN都使能（无需trap），然后再进行trap的处理工作。原因还没详细分析，可能是要后面在el0/el1去做fpsimd的相关运算？
4. **虚拟机相关**：在进入虚拟机之前，`__activate_traps`会修改cpacr_el1的值，直接在host的cpacr_el1基础上，开TTA，关zen（fpsimd都要trap），开TAM，zen和fpen是否使能要看host是否用过sve/fp寄存器（`update_fp_enabled`）；
5. **虚拟机相关**：在退出虚拟机之后，`__deactivate_traps`直接将`CPACR_EL1_DEFAULT`写入到cpacr_el1中。

内核中修改cptr的地方们：

1. arch/arm64/kernel/head.S中，函数`el2_setup` -> `install_el2_stub`，会写cptr_el2的值为固定的0x33ff。表示进制CoProcessor的trap操作。
2. arch/arm64/kvm/hyp/switch.c中，只有nvhe模式下，才会操作cptr_el2的值。



### mdcr_el2

Monitor Debug Configuration Register.提供自托管的debug功能，以及性能监控扩展（PME）

#### mdcr的默认值

688上check:

- 在host的时候，mdcr_el2的默认值为0x4000。

- 而当即将进入guest的时候，默认值为0x4e60（可以通过debugfs的kvm_arm_set_dreg32来查看）。

| 标志位   | 功能介绍                                                     |            |
| -------- | ------------------------------------------------------------ | ---------- |
| TTRF[19] | **trace类型的trap**<br/>• Access to TRFCR_EL1 is trapped to EL2, reported using EC syndrome value 0x18.<br/> | 默认不trap |
| [14]     |                                                              | 默认trap   |
| [11]     |                                                              | 默认trap   |
| [10]     |                                                              | 默认trap   |
| TDA[9]   | **debug类型的trap**.Trap Debug Access. Traps EL0 and EL1 System register accesses to debug System registers that are<br/>not trapped by MDCR_EL2.TDRA or MDCR_EL2.TDOSA, as follows:— MDCCSR_EL0, MDCCINT_EL1, OSDTRRX_EL1, MDSCR_EL1,<br/>OSDTRTX_EL1, OSECCR_EL1, DBGBVR<n>_EL1, DBGBCR<n>_EL1,<br/>DBGWVR<n>_EL1, DBGWCR<n>_EL1, DBGCLAIMSET_EL1,<br/>DBGCLAIMCLR_EL1, DBGAUTHSTATUS_EL1. | 默认trap   |
| TPM[6]   | **Performance Monitor**类型的trap。 — PMCR_EL0, PMCNTENSET_EL0, PMCNTENCLR_EL0, PMOVSCLR_EL0, PMSWINC_EL0, PMSELR_EL0, PMCEID0_EL0, PMCEID1_EL0, PMCCNTR_EL0, PMXEVTYPER_EL0, PMXEVCNTR_EL0, PMUSERENR_EL0, PMINTENSET_EL1, PMINTENCLR_EL1, PMOVSSET_EL0, PMEVCNTR<n>_EL0, PMEVTYPER<n>_EL0, PMCCFILTR_EL0. — If ARMv8.4-PMU is implemented, PMMIR_EL1 | 默认trap   |
| TPMCR[5] |                                                              | 默认trap   |
|          |                                                              |            |



#### 内核中修改mdcr_el2的地方们：

1. **初始化：**`arch/arm64/kernel/head.S `    设置SPE相关功能，不允许EL1使用spe，需要trap到el2处理。
3. **nonVHE模式**：`arch/arm64/kvm/hyp/switch.c`  `__deactivate_traps_nvhe`函数.
4. **退出到qemu前后**：`arch/arm64/kvm/hyp/switch.c`   从qemu回到kvm时，函数`vcpu_load -> __activate_traps_common`中，将vcpu->arch.mdcr_el2写回到mdcr_el2寄存器中。退出到qemu后，函数`vcpu_put -> kvm_arch_vcpu_put-> kvm_vcpu_put_sysregs -> deactivate_traps_vhe_put`，读mdcr_el2，将HPMN_MASK，E2PB，TPMS清零，然后再写回。
5. **虚拟机debug功能使能**：

是在kvm_arm_setup_debug()函数中，这个函数在每次进入/退出虚拟机时都会调用。

``` c
kvm_arch_vpu_ioctl_run
{
    kvm_arm_setup_debug(vcpu);  // 进入guest之前设置debug的trap位

    /**************************************************************
     * Enter the guest
     */
    trace_kvm_entry(*vcpu_pc(vcpu));
    guest_enter_irqoff();

    ret = kvm_call_hyp_ret(__kvm_vcpu_run, vcpu);

    vcpu->mode = OUTSIDE_GUEST_MODE;
    vcpu->stat.exits++;
    /*
     * Back from guest
     *************************************************************/

    kvm_arm_clear_debug(vcpu); // 退出guest之后清除debug的trap位。
}   
    
```

``` c
 #define MDCR_EL2_HPMN_MASK  (0x1F)
void kvm_arm_setup_debug(struct kvm_vcpu *vcpu)
{
    bool trap_debug = !(vcpu->arch.flags & KVM_ARM64_DEBUG_DIRTY);
    unsigned long mdscr, orig_mdcr_el2 = vcpu->arch.mdcr_el2;

    trace_kvm_arm_setup_debug(vcpu, vcpu->guest_debug);

    vcpu->arch.mdcr_el2 = __this_cpu_read(mdcr_el2) & MDCR_EL2_HPMN_MASK;
    	// MASK的存在表示只能设置bit 4:0，其他位都是函数中写死的。
    vcpu->arch.mdcr_el2 |= (MDCR_EL2_TPM |
                MDCR_EL2_TPMS |
                MDCR_EL2_TPMCR |
                MDCR_EL2_TDRA |
                MDCR_EL2_TDOSA);
    ...
    /* Write mdcr_el2 changes since vcpu_load on VHE systems */
    if (has_vhe() && orig_mdcr_el2 != vcpu->arch.mdcr_el2)
        write_sysreg(vcpu->arch.mdcr_el2, mdcr_el2); // 写回到mdcr_el2寄存器。

    trace_kvm_arm_set_dreg32("MDCR_EL2", vcpu->arch.mdcr_el2); // 可以通过trace来追踪。
    trace_kvm_arm_set_dreg32("MDSCR_EL1", vcpu_read_sys_reg(vcpu, MDSCR_EL1));
}
```

#### mdcr控制的trap类型

1. 

##### debug类型的trap

MDCR_EL2. TDA, bit [9]控制。

Trap Debug Access. Traps EL0 and EL1 System register accesses to debug System registers that are
not trapped by MDCR_EL2.TDRA or MDCR_EL2.TDOSA, as follows:

1. In AArch64 state, accesses to the following registers are trapped to EL2 reported using EC
   syndrome value 0x18:
   — MDCCSR_EL0, MDCCINT_EL1, OSDTRRX_EL1, MDSCR_EL1,
   OSDTRTX_EL1, OSECCR_EL1, DBGBVR<n>_EL1, DBGBCR<n>_EL1,
   DBGWVR<n>_EL1, DBGWCR<n>_EL1, DBGCLAIMSET_EL1,
   DBGCLAIMCLR_EL1, DBGAUTHSTATUS_EL1.
   — When not in Debug state, DBGDTR_EL0, DBGDTRRX_EL0, DBGDTRTX_EL0.

##### trace类型的trap

MDCR_EL2.TTRF, bit [19]控制。
When ARMv8.4-Trace is implemented:
Traps use of the Trace Filter Control registers at EL1 to EL2, as follows:
• Access to TRFCR_EL1 is trapped to EL2, reported using EC syndrome value 0x18.
• Access to TRFCR is trapped to EL2, reported using EC syndrome value 0x03.

### cnthctl_el2

Counter-timer Hypervisor Control register 控制el1/el0访问物理counter和timer时是否需要trap。

####　cnthctl_el2的默认值

在host中： 0xcd6， 对应的是bit[11,10,7,6,4,2,1] = 1.

当VHE使能并且HCR_EL2.E2H=1时，对应的是：

| 标志位       | 功能介绍                                                     | 备注                |
| ------------ | ------------------------------------------------------------ | ------------------- |
| EL1PTEN[11]  | **guest访问physical timer** 当HCR_EL2.TGE=0时，控制从EL0和EL1访问CNTP_CTL_EL0, CNTP_CVAL_EL0, CNTP_TVAL_EL0是否trap。 | 注意，1表示不trap。 |
| EL1PCTEN[10] | **guest访问cntpct** 当HCR_EL2.TGE=0时，控制el0和el1访问cntpct_el0是否trap。 | 注意，1表示不trap。 |
| EL0PTEN[9]   | **host访问physical timer** 当HCR_EL2.TGE=1时，控制从EL0访问CNTP_CTL_EL0, CNTP_CVAL_EL0, CNTP_TVAL_EL0 是否trap。 | 注意，1表示不trap。 |
| EL0VTEN[8]   | **host访问virtual timer**   当HCR_EL2.TGE=1时，0表示从EL0访问CNTV_CTL_EL0, CNTV_CVAL_EL0, CNTV_TVAL_EL0 时会trap到el2。 | 注意，1表示不trap。 |
| EL0VCTEN[1]  | **frequency & vcnt ** 当HCR_EL2.TGE=1时，0表示el0访问CNTVCT_EL0时会trap到el2；当el0vcten==0 && el1pcten==0时，el0访问CNTFRQ_EL0才会trap到el2。当HCR_EL2.TGE=0时，什么值都不会trap出去。 | 注意，1表示不trap。 |
| EL0PCTEN[0]  | **frequency & pcnt**  当HCR_EL2.TGE=1时，0表示el0访问CNTPCT_EL0时会trap到el2；当el0vcten==0 && el1pcten==0时，el0访问CNTFRQ_EL0才会trap到el2。当HCR_EL2.TGE=0时，什么值都不会trap出去。 | 注意，1表示不trap。 |

只会在创建虚拟机的时候配置，后面就不会再改变了。

内核中修改cnthctl_el2的地方们：

1. **初始化：**`arch/arm64/kernel/head.S `    设置初始值。
2. **nonVHE模式**：`virt/kvm/arm/hyp/timer-sr.c`  
3. **虚拟机内设置trap**：`"virt/kvm/arm/arch_timer.c"`中，`kvm_init-> kvm_arch_init -> init_subsystems-> _kvm_arch_hardware_enable->cpu_hyp_reinit -> kvm_timer_init_vhe`，将cnthctl的EL1PCEN=0，EL1PCTEN=1.

**注意**，当`HCR_EL2.E2H = 1`时， 在EL1对`CNTKCTL_EL1`的访问会转化为对`CNTHCTL_EL2`的访问，包含msr/mrs操作。



​	

### vtcr_el2

virtual translation control register，控制Stage 2页表翻译相关的东西。 注意，stage 1页表翻译相关寄存器是tcr_el1.



kvm修改vtcr_el2的地方：

注意，2020.1的patch（kvm: arm64: Configure VTCR_EL2 per VM）刚刚修改成下面这样，之前的做法是，在创建vm的时候固定了该vm的所有vcpu为相同的vtcr_el2的值，然后就不会再进行修改了。



1. 在创建虚拟机的时候，qemu会触发内核的`kvm_dev_ioctl_create_vm`函数，然后->`kvm_create_vm`，其中的`kvm_arch_init_vm`是用来初始化vm data结构体的，调用`kvm_arm_setup_stage2`函数来进行kvm->arch->vtcr成员变量的修改。（注意，这里并没有真正修改vtcr_el2)

```c
kvm_arm_setup_stage2函数中，
    会设置kvm->arch->vtcr结构体的值：
	// 固定设置inner shareable/outer shareable的cache属性；
	// 根据所需设置s2的pagesize
	// 根据所需设置ipa和pa的地址范围
    // 固定设置hardware Access Flag置位，对于不支持这个功能的，该bit不会被修改成功。
    // 设置vmid的bit数是16bit还是8bit
```

2. 虚拟机每次需要切换到guest之前，都会重新将arch->vtcr的值写入vtcr_el2；在切换回host时也不会重新保存arch->vtcr。因此，用的都是同样的值。

``` c
__kvm_vcpu_run_vhe
    -> __activate_vm
        -> __load_guest_stage2 // 将arch->vtcr的值写入到vtcr_el2寄存器。
    
__kvm_tlb_flush_vmid/__kvm_tlb_flush_vmid_ipa/__kvm_tlb_flush_local_vmid
	-> __tlb_switch_to_guest
 	   -> __load_guest_stage2
```

### midr/mpidr/pmcr



## cache维护指令

### poc coherence

由TPCP（hcr_el2[23])决定是否trap poc coherence指令。

> 0b0 This control does not cause any instructions to be trapped.
> 0b1 Execution of the specified instructions is trapped to EL2, when EL2 is enabled in the
> current Security state.

其中，若ARMv8.2-DCPoP特性已经被实现，则由SCTLR_EL1.UCI决定是否使能该特性。在taishan v110/v200都支持该特性。

> ARMv8.2-DCPoP特性引入了识别和管理持久化存储位置的机制。包括新增指令DC CVAP。该特性在ARMv8.2版本为必选实现。该特性仅在AArch64支持。
>
> ARMv8.2-DCCVADP允许两个级别的cache清理到PoP：
>
> 1）重新定义PoP，改变了DC CVAP指令的范围；
>
> 2）定义了PoDP；
>
> 3）增加了DC CVADP指令。
>
> 该特性在ARMv8.2版本为可选实现，在ARMv8.5为必选实现。该特性仅在AArch64支持。

对于el0的指令，若SCTLR_EL1.UCI != 0， 则DC CIVAC, DC CVAC, DC CVAP指令由HCR_EL2.TPCP决定是否trap到el2；

对于el1的指令，DC IVAC, DC CIVAC, DC CVAC, DC CVAP是否trap到el2直接由HCR_EL2.TPCP决定。

# ARM V8新特性

## WFx Timeout

ARMv8.7/v9.2-WFET, wfe with cycle time-out

### 原理与功能

使用方式： WFET Xd， xd保存了32-bit的cycle计数值。

使用CNTVCT_EL0作为计数时钟，当wfet xd中的计数值 > cntvct_el0，则在当前核产生一个event唤醒wfe。

- 仅对当前核有效；计数值为绝对值比较；

- cntvct而不是pct，表示可以通过scale/offset调整。

 详细文档见《2020 Architecture Extensions》![img](D:\at_work\Documents\我的总结文档\images\clip_image001.png)

- The WFET instruction is subject to the same traps and exceptions as WFE, and is completed by the same events as a WFE. WFET is also woken by the local timeout event.
- The WFIT instruction is subject to the same traps and exceptions as WFI, and is completed by the same events as a WFI. WFIT is also woken by the local timeout event.

trap时对应的EC值：

- When WFIT instruction is trapped, it is reported using EC==0x01, and ISS field of 0x1E00002
- When WFET instruction is trapped, it is reported using EC==0x01, and ISS field of 0x1E00003

## 配置使能方式

注意，在Hi1630V200中默认是disable的，需要通过chickenbit进行配置。

chickenbit配置方式：

![img](D:\at_work\Documents\我的总结文档\images\397215B7-9AE8-45DE-AA87-113129DEC7BD.png)

然后，就可以在ID_AA64ISAR2_EL1[3:0]中得到体现了。

- ID_AA64ISAR2_EL1（Op0=11, op1=0, CRn=0000, CRm=0110, Op2=010）
- WFxT:
  - 0000: WFET and WFIT are not supported
  - 0001: WFET and WFIT are supported


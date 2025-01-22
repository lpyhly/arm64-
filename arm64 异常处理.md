# arm64 异常处理

[TOC]



# 1 异常分类

## 1.1 SError

异步的，外部的，不可恢复的异常。

​    异步的：发生异常时硬件不能提供有效信息用于定位；

​    外部的：通常时外部存储系统导致的。

​    

产生原因：

​    异步数据异常： CPU读写内存数据时的异常。

​    外部引脚触发： ECC校验出错，SOC自定义。

​    

处理：

​    内核不知道怎么处理，所以：

​    如果是用户态产生的，则将出错信息传递给用户态；

​    如果是内核态产生的，直接panic。

## 1.2 Irq

GIC上报的中断，可能是SPI/PPI/SGI/LPI等。

## 1.3 Fiq

GIC上报的中断，可能是SPI/PPI/SGI/LPI等，优先级比irq更高。

## 1.4 Sync

同步错误，指令触发的，可以和时钟对应上的那种。

# 1 内核异常处理流程

## 1.1 异常向量表

异常向量表的基地址存放在vbar_elx中，以目的EL划分，不同的elx有不同的向量表，所以可以有3张(x = 1,2,3)。 在linux启动过程中会配置vbar_elx的值，其中

- vbar_el1

  - host的vbar_el1好像没什么用，因为开了vhe时不会进入到这里。
  - guest的vbar_el1指向arch/arm64/kvm/hyp/entry.S的__guest_enter函数，会经过一层判断，决定哪些异常可以直接在el1处理，哪些需要trap到el2去处理。

- vbar_el2的配置会有变化。 

  - host中vbar_el2指向arch/arm64/kernel/entry.S中的vector域段，使能了vhe（hcr_el2.e2h=1，linux跑在el2中）,且hcr_el2.tge = 1, 所以访问vbar_el1的操作会被路由到vbar_el2。表中所有的el1的寄存器访问操作同样也会被路由到el2.；若有hvc这种本来就需要访问el2的指令，也可以在同一张表中进行处理。

  - host中若没有使能vhe，则会用到arch/arm64/kernel/hyp_stub.S中的`__hyp_vector`域段，作用就是拦截el2路由配置，并且设置新的el2路由表。

  - 而切换到guest时，在切换的时候（load_vcpu），会将arch/arm64/kvm/hyp/hyp-entry.S中的异常向量表`__kvm_hyp_vector`放到`vbar_el2`。在切换回host时，会再切回原来的vector表。这样，在guest运行时，若有异常会trap到el2，则根据`__kvm_hyp_vector`的操作进行处理。这里的操作大部分会走去__kvm_exit处理。

     ``` c
  kvm_vcpu_run_vhe -> __activate_traps -> activate_traps_vhe
    // 中间是while循环进入guest。
  kvm_vcpu_run_vhe -> deactivate_traps -> deactivate_traps_vhe。
    
   ```
    
  `__vbar_el2`指向__hyp_stub_vectors，向量表在arch/arm64/kernel/hyp-stub.S，这里仅处理hvc的指令，普通的el0->el2的trap操作软件查的是vbar_el1，然后硬件来路由到vbar_el2。

- vbar_el3没找到对应表。

对于每个异常向量表，也有不同的offset，通过使用哪个el的栈指针sp/同级跳转还是不同级跳转/aarch64还是aarch32分为4类：

a)     同级跳转(非EL0)，使用el0的栈指针。（全部invalid，该场景可能出现在el0->elx的中断上下文处理中，但那时是关中断的，不应该有中断嵌套，所以正常不会发生该场景）

b)    同级跳转(非EL0)，使用elx的栈指针。（elx中发生了中断）

c)     不同级跳转，使用el0的栈指针。（用户态跳转到内核态的场景）

d)    同上，但是在aarch32架构。

每一类有不同的offset，会去往不同的vbar_elx + offset。offset共有16个，4类级别 * 4种中断类型（sync，irq，fiq，serror），每个offset为0x80 Byte。去往哪个entry由硬件进行计算，可以通过pstate的nRW（64bit or 32bit）， SP（当前使用的sp是el0还是elx的），EL（当前el等级）来计算。

> if nRW == 32bit:   to 第四类的对应中断；
>
> else if EL == 0: to 第三类；
>
> else if SP ==  0: to 第一类；
>
> else： to 第二类。

``` c
//arch/arm64/kernel/entry.S
SYM_CODE_START(vectors)
    kernel_ventry   1, t, 64, sync      // Synchronous EL1t
    kernel_ventry   1, t, 64, irq       // IRQ EL1t
    kernel_ventry   1, t, 64, fiq       // FIQ EL1h
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



## 1.2 触发异常-硬件操作

硬件发现触发了异常的时候，会做如下操作：

1. 判断需要到哪个elx去处理异常，这里不同场景下有不同的方式。比如虚拟化场景下，若当前异常对应的trap开了，则去往el2，否则去往el1.不同el的异常向量表是不同的。这里x=1，2，3，不可能出现0。

2. 保存当前pc到`elr_elx`，保存当前PSTATE到`spsr_elx`

3. 若为sync异常，则将对应的错误信息写到`esr_elx`中

4. 将vbar_elx + offset地址放到pc中，PSTATE清零（直接关中断）

5. 发生跳转。





## 1.3 触发异常-软件操作

到了异常向量表后，软件跳转到对应的入口去处理。

### 1.3.1 sync异常处理

如果是sync异常，则需要读取esr_elx进行判断。

用户态过来的sync异常走el0_sync， 内核态过来的sync异常走el1_sync.

其中， -> kernel_entry 1

### 1.3.2 serror异常处理

#### 1.3.2.1 虚拟化中的serror异常处理

在虚拟机运行时，异常向量表为__kvm_hyp_vector。

##### 1.3.2.1.1 虚拟化中EL1的serror异常处理

serror触发后，到el1_error。

``` c
el1_error:
    get_vcpu_ptr    x1, x0 // vcpu的x0寄存器保存到stack上。（？不太确定）
    mov x0, #ARM_EXCEPTION_EL1_SERROR // 将x0的bit 1 = 0，此时x0=0x2
    b   __guest_exit
```

在`__guest_exit`中，会判断是否支持RAS，

- 若支持，则将返回值的ARM_EXIT_WITH_SERROR_BIT （bit 31） 置位，同时可以少处理一些东西。这时，ARM_EXCEPTION_EL1_SERROR 和ARM_EXIT_WITH_SERROR_BIT同时都置位了，后者是在传入时就已经置位了。此时返回x0=0x80000002.
- 若不支持，则只返回x0=0x2。

在`__guest_exit`中有修改lr，所以ret后会回到函数`__kvm_vcpu_run_vhe`的`__guest_enter`后，然后将x0的值返回。

``` c
 static int __kvm_vcpu_run_vhe(struct kvm_vcpu *vcpu)
 {
    do {
     /* Jump in the fire! */
     exit_code = __guest_enter(vcpu, host_ctxt);
     /* And we're baaack! */
 	} while (fixup_guest_exit(vcpu, &exit_code));
    return exit_code;
 }
```

然后，返回值一路通过`__kvm_vcpu_run` ->`kvm_arch_vcpu_ioctl_run`带回，然后作为`handle_exit_early` 的参数先传入处理，处理完毕后再开抢占，最后通过`handle_exit`的参数`exception_index`传入处理。后者实际上啥也没干。

``` c
/*
 * Return > 0 to return to guest, < 0 on error, 0 (and set exit_reason) on
 * proper exit to userspace.
 */
int handle_exit(struct kvm_vcpu *vcpu, int exception_index)
{
    struct kvm_run *run = vcpu->run;

    if (ARM_SERROR_PENDING(exception_index)) { 
        // 这里判断的是ARM_EXIT_WITH_SERROR_BIT，是针对RAS的场景，
        // 因为RAS处理过了，所以直接返回。
        u8 esr_ec = ESR_ELx_EC(kvm_vcpu_get_esr(vcpu));
        return 1;
    }
    exception_index = ARM_EXCEPTION_CODE(exception_index); // 将bit 31清掉。
		
    switch (exception_index) {
    case ARM_EXCEPTION_IRQ:
        return 1;
    case ARM_EXCEPTION_EL1_SERROR: 
            //如果是EL1的serror，不作任何处理，直接返回到guest中。
        return 1;
}
```

然后看前者，这里处理的是必须在被抢占之前完成的任务，比如SERROR。

``` c
 /* For exit types that need handling before we can be preempted */
 void handle_exit_early(struct kvm_vcpu *vcpu, int exception_index)
 {
     if (ARM_SERROR_PENDING(exception_index)) {
     // 这里判断的是ARM_EXIT_WITH_SERROR_BIT，是针对RAS的场景.
         if (this_cpu_has_cap(ARM64_HAS_RAS_EXTN)) {
             u64 disr = kvm_vcpu_get_disr(vcpu);

             kvm_handle_guest_serror(vcpu, disr_to_esr(disr));
         } else {
             kvm_inject_vabt(vcpu);
         }

         return;
     }

     exception_index = ARM_EXCEPTION_CODE(exception_index);

     if (exception_index == ARM_EXCEPTION_EL1_SERROR)
         kvm_handle_guest_serror(vcpu, kvm_vcpu_get_esr(vcpu));
     	// 这里是没有RAS的场景下，发生EL1的serror的情况。
 }
```







##### 1.3.2.1.2 虚拟化中EL2的serror异常处理

虚拟机运行时，若host上发生了serror，则触发后到el2_error。保存寄存器后，就直接跳转到 kvm_unexpected_el2_exception -> ____kvm_unexpected_el2_exception。

``` c
static inline void __kvm_unexpected_el2_exception(void)
{
    unsigned long addr, fixup;
    struct kvm_cpu_context *host_ctxt;
    struct exception_table_entry *entry, *end;
    unsigned long elr_el2 = read_sysreg(elr_el2);

    entry = hyp_symbol_addr(__start___kvm_ex_table);
    end = hyp_symbol_addr(__stop___kvm_ex_table);
    host_ctxt = &__hyp_this_cpu_ptr(kvm_host_data)->host_ctxt; 

    while (entry < end) {
        addr = (unsigned long)&entry->insn + entry->insn;
        fixup = (unsigned long)&entry->fixup + entry->fixup;

        if (addr != elr_el2) {
            entry++;
            continue;
        }

        write_sysreg(fixup, elr_el2); // 在kvm处理表中寻找入口，并写到elr_el2返回地址中。
        return;
    }

    hyp_panic(host_ctxt); // 若处理表的没有相应的fixup，则直接报panic。
}
```


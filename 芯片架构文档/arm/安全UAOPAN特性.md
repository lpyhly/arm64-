# 安全UAO/PAN特性

## PAN

### 什么是PAN？ 为什么引入？

​		为了防止高特权级访问低特权级的地址。 页表的权限配置中，关于RWX（读写执行）三个动作都有具体的权限配置，而且是会区分el级别的。比如，PXN和UXN就是用来指定该地址在特权级/非特权级是否能够被执行。对于用户态地址，若内核可以随意执行，则有安全风险。 而AP域段只有2bit，用来标识RW的权限时，默认特权级的权限是高于el0的，即没有提供内核不可访问但用户态可访问的选项，这样也就引入了安全风险。

​		页表中没地方改了，所以在`pstate`中引入了PAN域段（Privileged Accesss Never），若置位则不允许内核态访问用户态地址，就是说，当PAN=1且内核态 ldp/stp <用户态地址>时，就会触发异常。（具体例子可以见： https://zhuanlan.zhihu.com/p/365701044）

​		那么，对于`copy_from/to_user`类型的访问，必须在内核态访问用户态地址，该怎么办呢？ --- 解决方式是引入了新的指令ldtr/sttr，用来显示标识这个地址是用户态地址，并且可以去访问。使用了`ldtr/sttr`指令且`UAO=0`时，意味着虽然在`el1/el2`执行该指令，但是表现的像是在`el0`执行，因此即使`PAN`被置位也不会挡该操作。这里要注意的是，这种 Load/store unprivileged指令，只有1/4/8 Byte形式，没有16Byte形式，因此ldp只能转换成2条ldtr指令。

### 内核如何控制PAN的使能与否？

内核中有两个config：`CONFIG_ARM64_PAN`以及`CONFIG_ARM64_SW_TTBR0_PAN`。后面`ARMv8.7`又提出了`enhanced PAN`特性，主要是为了引入内核中仅有执行权限时的fault处理，在内核中也已经被使能起来了（以后再详细分析）。

`CONFIG_ARM64_PAN`表示是否支持硬件版本的`PAN`功能，通过读取`SYS_ID_AA64MMFR1_EL1`的`PAN`位来判断。若支持，则在cpu启动时检测到该特性，就通过`.cpu_enable()`配置`pstate.pan`置1，同时配置`sctlr_el1.span`为0，表示EL切换到`EL1`时，硬件会将`pstate.pan`自动置为1，这样就无需软件单独配置了。

```c
SPAN, bit [23]
 When FEAT_PAN is implemented: Set Privileged Access Never, on taking an exception to EL1.
      0b0   PSTATE.PAN is set to 1 on taking an exception to EL1.
      0b1   The value of PSTATE.PAN is left unchanged on taking an exception to EL1.

     When FEAT_VHE is implemented, and the value of HCR_EL2.{E2H, TGE} is {1, 1}, this bit has no effect on execution at EL0.
```



`CONFIG_ARM64_SW_TTBR0_PAN`用来配置是否在内核中使能PAN功能，不管硬件是否支持。

- 若不使能，则即使硬件支持，也不会将pstate的pan置位；
- 若使能，则即使硬件不支持，也可以使能该功能，因此有软件、硬件两种实现方案。



### 内核中是如何实现PAN的？

即使硬件不支持，也可以使能该功能，因此有软件、硬件两种实现方案：

- 当硬件支持PAN时，就无需过多操作，只需要在进出需要时去使能/禁止pstate的PAN位即可（用户态到内核态的切换不需要修改PAN，因为硬件会判断，el0访问el0内存是合法的，只有特权级访问el0内存才非法，因此无论是用户态还是内核态，一直将PAN置位是没问题的）
- 当硬件不支持PAN时，若要使能PAN功能，则配置`CONFIG_ARM64_SW_TTBR0_PAN`后，内核会在el切换时（kernel_entry/kernel_exit)，增加一段代码，代码逻辑是在进入内核时，修改`ttbr0_el0`为非法地址，从而令访问el0地址时，在地址翻译时发生异常，同时象征性的在`spsr`上将PAN置位。

注意，`ttbr1_el1.asid`可以用来指示当前asid（通过`TCR_el1.A1`来控制`asid`是由`ttbr0`还是`ttbr1`来指定，默认配置BIOS是由`ttbr0`指定，而linux新版本修改为`ttbr1`来指定，在`switch_cpu_to`函数中会修改`asid`），当PAN使能时，在软件方案中会同时修改`ttbr1_el1.asid`域段，在内核态时配置为0(kernel_entry)，回到用户态时再改回去（kernel_exit）。

```c
#ifdef CONFIG_ARM64_SW_TTBR0_PAN
    /*
     * Set the TTBR0 PAN bit in SPSR. When the exception is taken from
     * EL0, there is no need to check the state of TTBR0_EL1 since
     * accesses are always enabled.
     * Note that the meaning of this bit differs from the ARMv8.1 PAN
     * feature as all TTBR0_EL1 accesses are disabled, not just those to
     * user mappings.
     */
SYM_CODE_START_LOCAL(__swpan_entry_el1)
    mrs x21, ttbr0_el1
    tst x21, #TTBR_ASID_MASK        // Check for the reserved ASID
    orr x23, x23, #PSR_PAN_BIT      // Set the emulated PAN in the saved SPSR
    b.eq    1f              // TTBR0 access already disabled
    		// 若ttbr0_el1的asid已经是0了，说明之前已经做过这些动作(比如el0->el1->el1的异常入口），因此不用再重复配置了。
    and x23, x23, #~PSR_PAN_BIT     // Clear the emulated PAN in the saved SPSR
SYM_INNER_LABEL(__swpan_entry_el0, SYM_L_LOCAL) // 若el0->el1，则不用上述检查，直接配置。
    __uaccess_ttbr0_disable x21
1:  ret
SYM_CODE_END(__swpan_entry_el1)
    
static inline void __uaccess_ttbr0_disable(void)
{
        unsigned long flags, ttbr;

        local_irq_save(flags);
        ttbr = read_sysreg(ttbr1_el1);
        ttbr &= ~TTBR_ASID_MASK;
        /* reserved_pg_dir placed before swapper_pg_dir */
        write_sysreg(ttbr - RESERVED_SWAPPER_OFFSET, ttbr0_el1); //配置ttbr0_el1为一个非法地址。
        isb();
        /* Set reserved ASID */
        write_sysreg(ttbr, ttbr1_el1); //配置ttbr1_el1的ASID=0
        isb();
        local_irq_restore(flags);
}
```

​	另外，在内核中还有个封装的函数`uaccess_ttbr0_disable`，会先进行判断，若硬件PAN功能使能，则直接返回不做任何处理。

``` c
static inline bool uaccess_ttbr0_disable(void)
{
        if (!system_uses_ttbr0_pan())  // 没配置sw_ttbr0_pan，或者硬件使能了pan，则直接返回false
                return false;
        __uaccess_ttbr0_disable(); // 需要时才会去修改ttbr的值。
        return true;
}

#ifdef CONFIG_ARM64_SW_TTBR0_PAN
static inline bool system_uses_ttbr0_pan(void)
{   // 硬件不支持或没使能pan，但又使能了sw_ttbr0_pan时为真。
        return IS_ENABLED(CONFIG_ARM64_SW_TTBR0_PAN) &&
                !system_uses_hw_pan();
}
#else
static inline bool system_uses_ttbr0_pan(void)
{  
        return false;
}
#endif
```



## UAO

### 什么是UAO？ 为什么引入？

​		上面的方案又引入了另一个问题，这个方案告诉我们，只要使用ldtr/sttr就可以随意访问任意用户态地址，这个显然是不合理的。因此，引入了`UAO（User Access Override）`特性，同样是做在了`pstate`中，表示用户访问权限的覆盖方案。当`UAO=0`时，还是按照之前方案走，即对于用户态地址，ldp/stp不能访问，ldtr/sttr可以直接访问； 而当`UAO=1`时，则即使是`ldtr/sttr`也不允许被访问了。

​		注意，两个特性都是仅对内核态有作用，对用户态是没有作用的。

**目前内核已经弃用UAO，改用MTE特性中的TCO了。**
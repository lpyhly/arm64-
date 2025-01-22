# BIOS 原理

# 源码解析

## GIC

### GIC初始化流程

plat_hisi_gic_init（TF/products/kp920b/common/lib/gic/plat_gic.c）

 		-> gicv3_distif_init  // gicd的初始化 （open_source/arm-trusted-firmware/TrustedFirmware/drivers/arm/gic/v3/gicv3_main.c)

 		-> gicv3_rdistif_init  // gicr的初始化

 另外，UEFI/hi/kp920b/CoreInitLib/Gic.c中，GicdInit函数也有在运行。



## CORE

在EMU的话，先是由BOOT引导，然后上BIOS。部分配置会由BOOT来进行配置，比如主核部分配置都是在boot，而从核的一定都由BIOS来配置。

``` c
_ModuleEntryPoint() // UEFI/Common/Sec/AArch64/SecEntryPoint.S
    ArmDisableInterrupts // 关中断
    ArmDisableCachesAndMmu // 关MMU和cache
    CoreEarlyInitCommon// early init操作
    _InitMem // 判断当前核是否为主核，若是则： 初始化memory；
    SecondaryCoreEarlyInit //直接ret， 啥也没做
    _SetupSecondaryCoreStack //若是从核，则等主核初始化memory完成后去初始化自己然后去配置从核的栈空间。
```

### 主核

``````c
// 主核：UEFI/Common/Sec/AArch64/SecEntryPoint.S
_InitMem() 
	PrimaryCoreEarlyInit // earlyinit时主核要配置的寄存器，在这里修改
    //获取栈空间
    PrimaryCoreInit // 栈ready后主核要配置的寄存器，在这里修改
	_PrepareArguments -> CEntryPoint
``````



```asm
//KunPeng/TestPkg/ESL/UEFI/Products/Hi1650/Hi1650ESL/Library/PlatformInitLib/Aarch64/PlatformBoot.S
           
ASM_PFX(PrimaryCoreEarlyInit):                                        
/**                                                             
  所有核上电早期都需要进行的芯片级初始化（私有）                
  在系统刚开始启动时调用，此时栈暂时不可用，需要使用汇编代码实现
  临时配置栈到Totem SRAM                                        
**/   
  mov  x17,x30      /*保存LR地址*/ 
  bl ArmEnableInstructionCache                                        
/*  临时措施可以都落在这里，注意这里是主核的配置 */ 
  mrs    x0, S3_1_C15_C2_2                                            
  orr    x0, x0, #(1ULL << 12)                                        
  msr    S3_1_C15_C2_2, x0                                            
#ifndef DEBUG_ON_ESL_PLAT 
  bl OemCoreConfig
  bl OemWaitL3dReady
#endif 
          
  //主核设置临时堆栈，Totem SRAM                                      
  ldr  x0, =FixedPcdGet64(PcdCPUCoresSecStackBaseInSram)              
  ldr  x1, =FixedPcdGet32(PcdCPUCoreSecPrimaryStackSizeInSram)        
  add  x0, x0, x1   
  mov  sp, x0                                                         
                                                                      
  stp x17, x10, [SP, #-16]!     
                                                                      
  bl ASM_PFX(OemUartInitlize)                                         
  bl SerialPortInitialize                                             
                                                                      
  ldp x17,x10,[sp], #16   
  mov   x30,x17 
  ret    
                                                                      
ASM_PFX(SecondaryCoreEarlyInit):                                      
  ret                                                                 
```



### 从核

```c
// UEFI/Products/Hi1650/Hi1650ESL/Library/PlatformInitLib/Aarch64/PlatformBoot.S
SecondaryCoreEarlyInit
```





```c
_SetupSecondaryCoreStack() // UEFI/Common/Sec/AArch64/SecEntryPoint.S
    // 获取从核的栈地址
    SecondaryCoreInit // 初始化堆栈，然后选择一个core去做mBist。
```

```c
// UEFI/Products/Hi1650/Hi1650ESL/Library/PlatformInitLib/PlatformInitLib.c
SecondaryCoreInit() 
```

reset_handler中会找到对应的cpu_ops函数入口，然后br过去。

```c
// open_source/arm-trusted-firmware/TrustedFirmware/include/arch/aarch64/el3_common_macros.S
el3_entrypoint_common  -> bl  reset_handler

declare_cpu_ops linxcore_930, LINXCORE_930_MIDR, \  
    linxcore_930_reset_func, \ 
    linxcore_930_core_pwr_dwn, \ 
    linxcore_930_cluster_pwr_dwn  
// declare_cpu_ops是定义了一个cpu_ops的section，当从核启动的时候，就会跳到这个secion来。
/*                                                                  
 * Declare CPU operations                                           
 *                                                                  
 * _name:                                                           
 *  Name of the CPU for which operations are being specified        
 * _midr:                                                           
 *  Numeric value expected to read from CPU's MIDR                  
 * _resetfunc:                                                      
 *  Reset function for the CPU. If there's no CPU reset function,   
 *  specify CPU_NO_RESET_FUNC                                       
  * _e_handler:                                                      
 *  This is a placeholder for future per CPU exception handlers.    
 * _power_down_ops:                                                 
 *  Comma-separated list of functions to perform power-down         
 *  operatios on the CPU. At least one, and up to                   
 *  CPU_MAX_PWR_DWN_OPS number of functions may be specified.       
 *  Starting at power level 0, these functions shall handle power   
 *  down at subsequent power levels. If there aren't exactly        
 *  CPU_MAX_PWR_DWN_OPS functions, the last specified one will be   
 *  used to handle power down at subsequent levels                  
 */                                                                 
```

从核会走这个接口，`cm_prepare_el3_exit_ns->cm_el2_sysregs_context_restore->el2_sysregs_context_restore`，会改actlr_el2.



## performance模式 & custom模式

performance和custom最大的差别是整了UFS关闭；
uncore 根据流量调频；

![img](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5C36492A87-B3EA-449F-9F81-2413496AF056.png)

IFS： IO die的，目前还没使能；

custom:  lpi on/ufs on
performace 就不影响OS的调频请求了；
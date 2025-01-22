

# vdso原理

- vDSO是为了解决用户态频繁调用的接口在用户内核态切换过程中带来的巨大性能损耗问题而引入的，并且，这个信息是非机密的。

- vDSO在用户态的地址是随机映射的，为了安全考虑。
- vDSO是个完整格式的ELF镜像，可以对其进行符号查找，一般glibc会在第一次调用时去检查当前内核支持的符号及对应版本，在后面的使用中就可以直接用这些接口了。vDSO的符号一般以`__vdso_`或者`__kernel_`开头。
- vDSO的调用不会被`strace`捕获，也不会被`seccomp`屏蔽。

# vdso源码

## 源码阅读小助手

- 内核源码路径： arch/x86/entry/vdso/*

  - ```
    find arch/$ARCH/ -name '*vdso*.so*' -o -name '*gate*.so*'
    ```

- vdso接口函数查看：

  - ```shell
    nm arch/arm64/kernel/vdso/vdso.so.dbg
    ```

- 反汇编： 直接反汇编文件 arch/arm64/kernel/vdso/vdso.so

- vdso基础结构体： arch/arm64/kernel/vdso.c



## 结构体

```c
/**                                                                                
 * struct vdso_data - vdso datapage representation                                 
 * @seq:                timebase sequence counter                                  
 * @clock_mode:         clock mode                                                 
 * @cycle_last:         timebase at clocksource init                               
 * @mask:               clocksource mask                                           
 * @mult:               clocksource multiplier                                     
 * @shift:              clocksource shift                                          
 * @basetime[clock_id]: basetime per clock_id                                      
 * @offset[clock_id]:   time namespace offset per clock_id                         
 * @tz_minuteswest:     minutes west of Greenwich                                  
 * @tz_dsttime:         type of DST correction                                     
 * @hrtimer_res:        hrtimer resolution                                         
 * @__unused:           unused                                                     
 * @arch_data:          architecture specific data (optional, defaults             
 *                      to an empty struct)                                        
 *                                                                                 
 * vdso_data will be accessed by 64 bit and compat code at the same time           
 * so we should be careful before modifying this structure.                        
 *                                                                                 
 * @basetime is used to store the base time for the system wide time getter        
 * VVAR page.                                                                      
 *                                                                                 
 * @offset is used by the special time namespace VVAR pages which are              
 * installed instead of the real VVAR page. These namespace pages must set         
 * @seq to 1 and @clock_mode to VDSO_CLOCKMODE_TIMENS to force the code into       
 * the time namespace slow path. The namespace aware functions retrieve the        
 * real system wide VVAR page, read host time and add the per clock offset.        
 * For clocks which are not affected by time namespace adjustment the              
 * offset must be zero.                                                            
 */                                                                                
struct vdso_data {                                                                 
        u32                     seq;                                               
                                                                                   
        s32                     clock_mode;                                        
        u64                     cycle_last;                                        
        u64                     mask;                                              
        u32                     mult;                                              
        u32                     shift;                                             
                                                                                   
        union {                                                                    
                struct vdso_timestamp   basetime[VDSO_BASES];                      
                struct timens_offset    offset[VDSO_BASES];                        
        };                                                                         
                                                                                   
        s32                     tz_minuteswest;                                    
        s32                     tz_dsttime;                                        
        u32                     hrtimer_res;                                       
        u32                     __unused;                                          
                                                                                   
        struct arch_vdso_data   arch_data;                                         
};                                                                                 
```



```c
static union {                                               
        struct vdso_data        data[CS_BASES];              
        u8                      page[PAGE_SIZE];             
} vdso_data_store __page_aligned_data; 

struct vdso_data *vdso_data = vdso_data_store.data;          
```

## vdso接口

### arm64的vdso接口

```shell
 symbol                   version
──────────────────────────────────────
__kernel_rt_sigreturn    LINUX_2.6.39
__kernel_gettimeofday    LINUX_2.6.39
__kernel_clock_gettime   LINUX_2.6.39
__kernel_clock_getres    LINUX_2.6.39
```



### x86的vdso接口

```shell
       symbol                 version
       ─────────────────────────────────
       __vdso_clock_gettime   LINUX_2.6
       __vdso_getcpu          LINUX_2.6
       __vdso_gettimeofday    LINUX_2.6
       __vdso_time            LINUX_2.6
```



### 用户态获取vdso结构体的方式

``` c
#include <sys/auxv.h>
void *vdso = (uintptr_t) getauxval(AT_SYSINFO_EHDR);
```


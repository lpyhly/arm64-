# linux内核编程指南

[TOC]

## 源码树结构

### interface

（1）arch-independent interface。在include/linux/percpu.h文件中，定义了内核其他模块要使用per cpu机制使用的接口API以及相关数据结构的定义。内核其他模块需要使用per cpu变量接口的时候需要include该头文件

 （2）arch-general interface。在include/asm-generic/percpu.h文件中。如果所有的arch相关的定义都是一样的，那么就把它抽取出来，放到asm-generic目录下。毫无疑问，这个文件定义的接口和数据结构是硬件相关的，只不过软件抽象各个arch-specific的内容，形成一个arch general layer。一般来说，我们不需要直接include该头文件，include/linux/percpu.h会include该头文件。

（3）arch-specific。这是和硬件相关的接口，在arch/arm/include/asm/percpu.h，定义了ARM平台中，具体和per cpu相关的接口代码。



## 常用变量/函数

| 功能                    | 接口                                 |      |
| ----------------------- | ------------------------------------ | ---- |
| 当前核号                | int cpu = smp_processor_id();        |      |
| 从task_struct中获取核号 | task_cpu(p); //struct task_struct *p |      |
|                         |                                      |      |

## 常用宏用法

#### ALTERNATIVE - 指令动态替换

可以根据当前cpu是否支持某些软硬件feature来实现对内核代码的在线优化，即在不关机、不换内核的情况下在线改写某些内核指令，以达到加速内核执行的目的

```asm
/*                                                                     
 * Usage: asm(ALTERNATIVE(oldinstr, newinstr, cpucap));                
 *                                                                     
 * Usage: asm(ALTERNATIVE(oldinstr, newinstr, cpucap, CONFIG_FOO));    
 * N.B. If CONFIG_FOO is specified, but not selected, the whole block  
 *      will be omitted, including oldinstr.                           
 */                                                                    
#作用： if CONFIG_XX == N  ->     NULL;
#      elif cpucap == True  ->   newinstr
#       else          oldinstr

#define ALTERNATIVE(oldinstr, newinstr, ...)   \                       
        _ALTERNATIVE_CFG(oldinstr, newinstr, __VA_ARGS__, 1)           
        


```





## 内核模块

###　Q：模块插入时是怎么进行version检查的？

insmod的动作会调用finit_module()的系统调用接口，具体实现在"kernel/module.c"中，

``` c
finit_module -> load_module 
    -> module_sig_check // #ifdef CONFIG_MODULE_SIG
    // module_sig_check检查失败时的报错信息为：
    // module verification failed: signature and/or required key missing - tainting
    
    -> check_modstruct_version ->   // #ifdef CONFIG_MODVERSIONS
    // 报错信息为： ** disagrees about version of symbol **
    
    -> layout_and_allocate -> check_modinfo    
    // 报错信息为：version magic *** should be *** 或者
    // loading out-of-tree module taints kernel
    
            
static int check_modinfo(struct module *mod, struct load_info *info, int flags)
{
    const char *modmagic = get_modinfo(info, "vermagic");
    int err;

    if (flags & MODULE_INIT_IGNORE_VERMAGIC)
        modmagic = NULL;

    /* This is allowed: modprobe --force will invalidate it. */
    if (!modmagic) {
        err = try_to_force_load(mod, "bad vermagic");
        if (err)
            return err;
    } else if (!same_magic(modmagic, vermagic, info->index.vers)) {
        pr_err("%s: version magic '%s' should be '%s'\n",
               info->name, modmagic, vermagic);
        return -ENOEXEC;
    }


```

### Q：怎么绕过内核模块CRC检查？

详见https://chowdera.com/2021/05/20210527005530858m.html

### Q： 怎么绕过内核的vermagic检查？

1. 修改内核源码的脚本scripts/Makefile.modpost（用来做内核编译时调用的makefile）。

``` c
quiet_cmd_modpost = MODPOST $@
      cmd_modpost = sed 's/ko$$/o/' $< | $(MODPOST) -T -

$(output-symdump): $(MODORDER) $(input-symdump) FORCE
        $(call if_changed,modpost)
        @sleep 30  // 添加一句sleep，可以用来在这时候修改.mod.c文件。

targets += $(output-symdump)
```

2. 修改**.mod.c文件中的vermagic字段的值，修改为正确的vermagic。 正确的vermagic可以通过modinfo *.ko来查看。

``` c
MODULE_INFO(vermagic, "5.10.19-g37aee351391e SMP preempt mod_unload aarch64");
```

3. 等待ko编译完成，就可以直接插入啦！！！

### Q： 怎么定义一个全局变量，可以让所有的内核函数访问到

定义一个全局变量，可以让所有的内核函数访问到。

定义：

```c
static int func_name(void)
{……}
EXPORT_SYMBOL(func_name)
```

其中，EXPORT_SYMBOL做的事情为：

把函数名、函数入口地址放到某个特定的section中，从而方便其他函数进行查找。

 其他函数使用它时：先声明从外部找这个函数，`extern int func_name(void);`然后就可以直接访问啦~

 

使用时需要注意：

必须先insmod mod1，再insmod mod2，否则会找不到符号。

因此，可以在mod2的编译文件中指定：

`KBUILD_EXTRA_SYMBOLS=/path/to/mod1/Module.symvers`

 

### Q: 怎么查看内核符号表？

`cat /proc/kallsyms`

可以获得符号地址。/proc/kallsyms包含了内核中的函数符号(包括没有EXPORT_SYMBOL)、全局变量(用EXPORT_SYMBOL导出的全局变量)

 需要的内核编译选项：

```shell
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y 符号表中包括所有的变量(包括没有用EXPORT_SYMBOL导出的变量)
CONFIG_KALLSYMS_EXTRA_PASS=y
```




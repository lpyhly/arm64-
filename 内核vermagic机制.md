# 内核vermagic机制

在插入ko时有时候会出现vermagic不匹配无法插入的现象，如：

``` shell
version magic '5.10.0-g090653345c3a-dirty SMP mod_unload aarch64' should be '5.10.0-g04885f9fbeb7-dirty SMP mod_unload aarch64'
insmod: can't insert './hly/guest.ko': invalid module format
```

最直接的方式，是在匹配的内核源码下重编内核模块。但是，若相应的内核源码已经找不到了或已被修改过，或者无法更换当前主机内核，该怎么办呢？

此时需要了解什么是vermagic，以及是否可以修改vermagic来实现ko插入。

## vermagic是怎么定义的？

1. 先看include/linux/vermagic.h中，有`VERMAGIC_STRING`的定义

``` c
#include <generated/utsrelease.h>
#include <asm/vermagic.h>

/* Simply sanity version stamp for modules. */
#ifdef CONFIG_SMP
#define MODULE_VERMAGIC_SMP "SMP "
#else
#define MODULE_VERMAGIC_SMP ""
#endif
#ifdef CONFIG_PREEMPT
#define MODULE_VERMAGIC_PREEMPT "preempt "
#elif defined(CONFIG_PREEMPT_RT)
#define MODULE_VERMAGIC_PREEMPT "preempt_rt "
#else
#define MODULE_VERMAGIC_PREEMPT ""
#endif
#ifdef CONFIG_MODULE_UNLOAD
#define MODULE_VERMAGIC_MODULE_UNLOAD "mod_unload "
#else
#define MODULE_VERMAGIC_MODULE_UNLOAD ""
#endif
#ifdef CONFIG_MODVERSIONS
#define MODULE_VERMAGIC_MODVERSIONS "modversions "
#else
#define MODULE_VERMAGIC_MODVERSIONS ""
#endif
#ifdef RANDSTRUCT_PLUGIN
#include <generated/randomize_layout_hash.h>
#define MODULE_RANDSTRUCT_PLUGIN "RANDSTRUCT_PLUGIN_" RANDSTRUCT_HASHED_SEED
#else
#define MODULE_RANDSTRUCT_PLUGIN
#endif

#define VERMAGIC_STRING                         \
    UTS_RELEASE " "                         \
    MODULE_VERMAGIC_SMP MODULE_VERMAGIC_PREEMPT             \
    MODULE_VERMAGIC_MODULE_UNLOAD MODULE_VERMAGIC_MODVERSIONS   \
    MODULE_ARCH_VERMAGIC                        \
    MODULE_RANDSTRUCT_PLUGIN
```

这里，部分内核选项的更改，会引起vermagic的变化，比如`CONFIG_MODVERSIONS`, `CONFIG_PREEMPT`等。

然后，在看一下`UTS_RELEASE`的定义。这个宏是在`generated/utsrelease.h`中定义的，而这个文件是主Makefile自动生成的，生成方式就是将`include/config/kernel.release`中的内容输出为`UTS_RELEASE`宏。

``` makefile
KERNELRELEASE = $(shell cat include/config/kernel.release 2> /dev/null)
uts_len := 64
define filechk_utsrelease.h
    if [ `echo -n "$(KERNELRELEASE)" | wc -c ` -gt $(uts_len) ]; then \
      echo '"$(KERNELRELEASE)" exceeds $(uts_len) characters' >&2;    \
      exit 1;                                                         \
    fi;                                                               \
    echo \#define UTS_RELEASE \"$(KERNELRELEASE)\"
endef

include/generated/utsrelease.h: include/config/kernel.release FORCE
    $(call filechk,utsrelease.h)
```

而`kernel.release`文件也是在`Makefile`中生成的，是通过`KERNELVERSION`变量和`setlocalversion`两部分组装起来的。

``` makefile
filechk_kernel.release = \
    echo "$(KERNELVERSION)$$($(CONFIG_SHELL) $(srctree)/scripts/setlocalversion $(srctree))"

# Store (new) KERNELRELEASE string in include/config/kernel.release
include/config/kernel.release: FORCE
    $(call filechk,kernel.release)
```

先看第一部分`KERNELVERSION`变量的定义，也在`Makefile`中，是将开头的几个版本变量进行组装，顺序是：`VERSION -> PATCH -> SUB -> EXTRA`，注意，这里不包含`NAME`变量的值。

``` makefile
VERSION = 5
PATCHLEVEL = 10
SUBLEVEL = 0
EXTRAVERSION =
NAME = Kleptomaniac Octopus
KERNELVERSION = $(VERSION)$(if $(PATCHLEVEL),.$(PATCHLEVEL)$(if $(SUBLEVEL),.$(SUBLEVEL)))$(EXTRAVERSION)
```

然后再看第二部分，`scripts/setlocalversion`文件，可以单独运行来生成local version，并且可以自己指定srctree的地址。这里的生成方案有几个步骤，分别来看一下。

首先，检查调用脚本的路径（一般来说就是src目录下）和src目录下，是否存在`localversion*`命名的。

``` shell
res="$(collect_files localversion*)"       # 检查当前目录下是否有localversion*文件
if test ! "$srctree" -ef .; then
    res="$res$(collect_files "$srctree"/localversion*)" # 再检查srctree目录。
fi
```

然后，检查是否定义了`CONFIG_LOCALVERSION`（include/config/auto.conf中定义的)，以及LOCALVERASION（调用脚本或编译内核时可以自定义）。多说一句，内核编译过程中，看的是`include/config/auto.conf`，而这个文件在`Makefile`中根据`.config`进行更新及升级。因此，只要调用了`make ***`，就可以获取`.config`的最新内容。

``` shell
res="${res}${CONFIG_LOCALVERSION}${LOCALVERSION}"
```

最后，判断`CONFIG_LOCALVERSION_AUTO`是否使能，若使能，则需要调用`scm_version`来计算完整的版本号，是会包含git tag版本信息的； 若不使能，则仅当本地文件被修改时，才会在后面添加后缀`+`，否则就不添加后缀。

``` shell
if test "$CONFIG_LOCALVERSION_AUTO" = "y"; then
    # full scm version string
    res="$res$(scm_version)"
else
    # append a plus sign if the repository is not in a clean
    # annotated or signed tagged state (as git describe only
    # looks at signed or annotated tags - git tag -a/-s) and
    # LOCALVERSION= is not specified
    if test "${LOCALVERSION+set}" != "set"; then
        scm=$(scm_version --short)
        res="$res${scm:++}"     # scm不为空，则在后面添加一个+，否则不添加后缀
    fi
fi
```

scm_version的逻辑是： 对于git， 如果打了tag，则会使用tag， 否则， 使用字母‘g’再加上最新的commit ID的前7位， 例如 “3.10.49-gde6b4c1”, 若存在未被追踪的修改， 则还会添加” -dirty”字样。



## vermagic可以如何修改？

### 修改内核源码中的vermagic

vim include/linux/vermagic.h

修改VERMAGIC_STRING宏定义即可。（无需重编内核）

同时，内核编译时不使能Automatically append version information to the version string

https://abcdxyzk.github.io/blog/2014/12/22/kernel-vermagic/



### 无需修改内核源码的方式

修改linux内核的scripts/Makefile.modpost文件，通过sleep后修改mod.c文件。

https://developer.aliyun.com/article/1724

 

ps. 若模块加载时报错disagrees about version of symbol module_layout，当确保内核源码是没错的时候，需要修改内核config：

取消勾选 CONFIG_MODVERSIONS

取消勾选 CONFIG_MODULE_SIG



# insmod ko时检查vermagic的逻辑

## 从insmod开始

insmod ko中先调用的是init_module()，这是一个syscall，

```c
init_module(kernel/module/main.c)
    -> load_module
    	-> module_sig_check (检查签名，好像没怎么用过)
        -> early_mod_check
    		-> blacklisted()  -> 如果在blacklist中，则直接报错，不进行加载。
    		-> ...（未完待续）
```

## 禁止insmod ko的方式

根据blacklist的逻辑，可以直接添加cmdline来完成来要求：

	- cmdline中添加`module_blacklist=modname1,modname2,` 用于阻止模块被加载;
	- 若只想在initrd中阻止，而不组织insmod的方式加载，则可以在cmdline中替换成 `modprobe.blacklist=` 
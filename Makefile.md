# Makefile

[TOC]



# Makefile语法

## Makefile入口

PHONY就是入口！



##  Makefile 函数

```Makefile
target ... : prerequisites ...
   command
```

- target

可以是一个object file（目标文件），也可以是一个执行文件，还可以是一个标签（label）。对 于标签这种特性，在后续的“伪目标”章节中会有叙述。

- prerequisites

生成该target所依赖的文件和/或target

- command

prerequisites中如果有一个以上的文件比target文件要新的话，command所定义的命令就会被执行。



##  Makefile规则

| 语法                 | 说明                                                         |      |
| -------------------- | ------------------------------------------------------------ | ---- |
| -include  <filename> | 减号： 如果有文件没有找到的话，make会生成一条警告信息，但不会马上出现致命错误。它会继续载入其它的  文件，一旦完成makefile的读取，make会再重试这些没有找到，或是不能读取的文件，如果还是 不行，make才会出现一条致命信息。 |      |
| %config              | 会匹配所有make *config的规则，比如menuconfig，oldconfig      |      |
| $@                   | 匹配规则名，如：glb.env.mk:     @sed 's/"//g ; s/=/:=/' < glb.env.post > $@ |      |
|                      |                                                              |      |
|                      |                                                              |      |
|                      |                                                              |      |
|                      |                                                              |      |

 # 内核的kbuild系统

## 整体框架

在top的Makefile中，会做这么几件事：

1. 获取全局变量和函数

``` makefile
scripts/Kbuild.include: ;
include scripts/Kbuild.include
# 通过include Kbuild.include这个文件，里面有包含build在内的很多函数、变量定义。
```







## make menuconfig

在Makefile中，会去匹配%config规则。

``` makefile
%config: scripts_basic outputmakefile FORCE
        $(Q)$(MAKE) $(build)=scripts/kconfig $@
```

先看3个依赖项的解释：

``` makefile
FORCE:
# FORCE是一个空目标，没有依赖文件且没有命令部分，由于它没有命令生成FORCE，所以每次都会被更新。
```

 ``` Makefile
scripts_basic:
        $(Q)$(MAKE) $(build)=scripts/basic
        $(Q)rm -f .tmp_quiet_recordmcount
# build变量被定义在scripts/Kbuild.include文件中。
# build := -f $(srctree)/scripts/Makefile.build obj
# 所以，上述等价于： make -f $(srctree)/scripts/Makefile.build obj=scripts/basic 
# 在scripts/basic/Makefile中，只做了一件事，就是生成fixdep这个目标。
    hostprogs-y := fixdep
    always      := $(hostprogs-y)
    $(addprefix $(obj)/,$(filter-out fixdep,$(always))): $(obj)/fixdep
# 所以，上述等价于：scripts/Makefile.build中执行默认指令，传入参数obj=fixdep
# 默认执行入口为__build，参数直接传入的。
 __build: $(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y)) \
      $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target)) \
      $(subdir-ym) $(always)
     @:
#  $(KBUILD_BUILTIN) 和 $(KBUILD_MODULES) 都为空，$(subdir-ym) 也没有在 Makefile 中设置，所以，在 make menuconfig 时，仅编译 $(always)，也就是 fixdep。
 ```

最后看，看主要目标menuconfig。

首先，$(Q)的定义如下，命令前加@表示不需要输出（默认），当make V=1时，则会输出这些命令。

``` makefile
ifeq ($(KBUILD_VERBOSE),1)
  quiet =
  Q =
else
  quiet=quiet_
  Q = @
endif
```

然后继续：

``` makefile
%config: scripts_basic outputMakefile FORCE
	$(Q)$(MAKE) $(build)=scripts/kconfig $@
# 等价于：make -f $(srctree)/scripts/Makefile.build obj=scripts/kconfig menuconfig
```

先分析obj，在scripts/kconfig/Makefile中有定义：

``` shell
menuconfig: $(obj)/mconf
	$< $(silent) $(Kconfig)
# 依赖 scripts/kconfig/mconf（忽略 silent 部分，这是打印相关参数）
# 相对而言，mconf 的生成过程还是比较复杂的：这里先不研究了。
# Kconfig就是指的源码根目录下的Kconfig文件，这只是Kconfig的根文件，整个目录的kconfig形成一棵树。
```





## make install

### 卸载用 make install 编译安装的软件

如果安装的时候指定了prefix，直接删除就好。

如果没有，并且源代码没有提供`make uninstall/distclean/veryclean`的功能，

可以这样做：

1. 找一个临时目录重新安装一遍。比如

   ​	`./configure --prefix=/tmp/to_remove && make install`

2. 然后遍历/tmp/to_remove里的文件，把你原来安装位置的文件都删除。这样的坏处是有些文件夹还可能删除不了（分不清是系统的还是安装上的）

也可以这样做：

`whereis xxx` 找到软件安装目录，`rm -rf` 把这些目录都删除，应该能删除干净，如`whereis python`

# debug技巧

1. `make --debug=v **` 可以打印出非常详细的make过程。
2. make V=1 ** 可以将执行语句逐条打印出来，方便手动复现问题。
3. make V=2 **可以将为什么执行到某句话的原因打印出来，方便梳理逻辑。
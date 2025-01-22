# GCC详解

[TOC]

## GCC编译选项

**GCC编译选项查找大全： https://gcc.gnu.org/onlinedocs/gcc/Option-Index.html**

**summary： https://gcc.gnu.org/onlinedocs/gcc/Option-Summary.html**

以 -g、-f、-m、-O、-W 或 --param 开头的选项会自动传递给 gcc 调用的各种子进程。为了将其他选项传递给这些进程，必须使用 -W 选项。

### -f FLAGS

如果想使用它的反义词，则使用-fno*.

```c
  -fpic         # flag to set position independent code 
  -fno-builtin  # don't recognize build in functions ... 
```



 ### -m  机器相关的模式 MODE

AARCH64相关的选项详见 https://gcc.gnu.org/onlinedocs/gcc/AArch64-Options.html#AArch64-Options

```c
 -mabi=name   #abi mode = name
 -mcpu=cpu
 -march
 -matomic
```

### -w 告警 WARNING

-w 禁止所有告警。

-W* 告警选项：

-Werror 视警告为告警，同时终止编译过程。

 

调试：

-p 生成额外代码来写描述信息，适合prof使用，

-pg 适合gprof使用

-ftest-coverage 生成后缀为.gcno的文件由gcov用于覆盖率统计

 

--sysroot 指定头文件及链接文件的根目录，如果想使用的头文件们

预编译：

-nostdinc 不在标准系统目录下搜索头文件，只搜索-I指定的目录以及当前目录。

-DMACRO=DEFN 定义宏

-UMACRO 取消定义宏

查找目录：

-I <DIR>: 显式包含头文件所在目录：

-L<dir>: 将DIR加到查找-I的目录列表中：

 

-O 优化选项，-O = -O1, -O2用的最多，-O3优化的最多，但默认会使能-finline-function，就是说inline的函数编译时内联完就把原函数删掉了，因此其他函数会找不到它，可能会影响到热补丁。

-fno-inline 忽略所有的inline关键字。

-finline-functions 把所有简单的函数集成到函数中。

-mtune 指定指令集流水架构，泰山的为 -mtune=tsv110

-ansi 关闭所有gcc扩展，如__asm__, __extension__等

-fno-builtin 不识别gcc的内置函数

-pgo优化 -- 两次编译，可以缓解Icache miss

### 编译优化级别O1/O2/O3/Ofast

以gcc 13.2.0为例来看：

https://gcc.gnu.org/onlinedocs/gcc-13.2.0/gcc/Optimize-Options.html

| 优化级别 | 包含的编译选项                                               |                                            |
| -------- | ------------------------------------------------------------ | ------------------------------------------ |
| O1       | -fauto-inc-dec<br/>-fbranch-count-reg<br/>-fcombine-stack-adjustments<br/>-fcompare-elim<br/>-fcprop-registers<br/>-fdce<br/>-fdefer-pop<br/>-fdelayed-branch<br/>-fdse<br/>-fforward-propagate<br/>-fguess-branch-probability<br/>-fif-conversion<br/>-fif-conversion2<br/>-finline-functions-called-once<br/>-fipa-modref<br/>-fipa-profile<br/>-fipa-pure-const<br/>-fipa-reference<br/>-fipa-reference-addressable<br/>-fmerge-constants<br/>-fmove-loop-invariants<br/>-fmove-loop-stores<br/>-fomit-frame-pointer<br/>-freorder-blocks<br/>-fshrink-wrap<br/>-fshrink-wrap-separate<br/>-fsplit-wide-types<br/>-fssa-backprop<br/>-fssa-phiopt<br/>-ftree-bit-ccp<br/>-ftree-ccp<br/>-ftree-ch<br/>-ftree-coalesce-vars<br/>-ftree-copy-prop<br/>-ftree-dce<br/>-ftree-dominator-opts<br/>-ftree-dse<br/>-ftree-forwprop<br/>-ftree-fre<br/>-ftree-phiprop<br/>-ftree-pta<br/>-ftree-scev-cprop<br/>-ftree-sink<br/>-ftree-slsr<br/>-ftree-sra<br/>-ftree-ter<br/>-funit-at-a-time |                                            |
|          | -falign-functions -falign-jumps<br/>-falign-labels -falign-loops<br/>-fcaller-saves<br/>-fcode-hoisting<br/>-fcrossjumping<br/>-fcse-follow-jumps -fcse-skip-blocks<br/>-fdelete-null-pointer-checks<br/>-fdevirtualize -fdevirtualize-speculatively<br/>-fexpensive-optimizations<br/>-ffinite-loops<br/>-fgcse -fgcse-lm<br/>-fhoist-adjacent-loads<br/>-finline-functions<br/>-finline-small-functions<br/>-findirect-inlining<br/>-fipa-bit-cp -fipa-cp -fipa-icf<br/>-fipa-ra -fipa-sra -fipa-vrp<br/>-fisolate-erroneous-paths-dereference<br/>-flra-remat<br/>-foptimize-sibling-calls<br/>-foptimize-strlen<br/>-fpartial-inlining<br/>-fpeephole2<br/>-freorder-blocks-algorithm=stc<br/>-freorder-blocks-and-partition -freorder-functions<br/>-frerun-cse-after-loop<br/>-fschedule-insns -fschedule-insns2<br/>-fsched-interblock -fsched-spec<br/>-fstore-merging<br/>-fstrict-aliasing<br/>-fthread-jumps<br/>-ftree-builtin-call-dce<br/>-ftree-loop-vectorize<br/>-ftree-pre<br/>-ftree-slp-vectorize<br/>-ftree-switch-conversion -ftree-tail-merge<br/>-ftree-vrp<br/>-fvect-cost-model=very-cheap |                                            |
|          | -falign-functions -falign-jumps<br/>-falign-labels -falign-loops<br/>-fcaller-saves<br/>-fcode-hoisting<br/>-fcrossjumping<br/>-fcse-follow-jumps -fcse-skip-blocks<br/>-fdelete-null-pointer-checks<br/>-fdevirtualize -fdevirtualize-speculatively<br/>-fexpensive-optimizations<br/>-ffinite-loops<br/>-fgcse -fgcse-lm<br/>-fhoist-adjacent-loads<br/>-finline-functions<br/>-finline-small-functions<br/>-findirect-inlining<br/>-fipa-bit-cp -fipa-cp -fipa-icf<br/>-fipa-ra -fipa-sra -fipa-vrp<br/>-fisolate-erroneous-paths-dereference<br/>-flra-remat<br/>-foptimize-sibling-calls<br/>-foptimize-strlen<br/>-fpartial-inlining<br/>-fpeephole2<br/>-freorder-blocks-algorithm=stc<br/>-freorder-blocks-and-partition -freorder-functions<br/>-frerun-cse-after-loop<br/>-fschedule-insns -fschedule-insns2<br/>-fsched-interblock -fsched-spec<br/>-fstore-merging<br/>-fstrict-aliasing<br/>-fthread-jumps<br/>-ftree-builtin-call-dce<br/>-ftree-loop-vectorize<br/>-ftree-pre<br/>-ftree-slp-vectorize<br/>-ftree-switch-conversion -ftree-tail-merge<br/>-ftree-vrp<br/>-fvect-cost-model=very-cheap |                                            |
| Ofast    | 使能-ffast-math, -fallow-store-data-races<br/>and the Fortran-specific -fstack-arrays, unless -fmax-stack-var-size is specified, and -fno-protect-parens. It turns off -fsemantic-interposition. | 使能所有O3的优化，以及一些非标类程序的优化 |

### 性能调优选项

梳理所有性能相关的调优选项： （https://gcc.gnu.org/onlinedocs/gcc-13.2.0/gcc/Optimize-Options.htm）

有两类，一是直接添加的参数，如下所示；

二是`--param XX=YY`的方式来添加的，可以通过`gcc --help=param -Q`来查看具体参数名、最小值、最大值、默认值等。

```c
-faggressive-loop-optimizations
-falign-functions=n:m:n2:m2
-falign-jumps=n:m:n2:m2
-falign-labels=n:m:n2:m2
-falign-loops=n:m:n2:m2
-fallow-store-data-races
-fassociative-math
-fauto-inc-dec
-fauto-profile=path
-fbranch-probabilities
-fcaller-saves
-fcode-hoisting
-fcombine-stack-adjustments
-fcompare-elim
-fconserve-stack
-fcprop-registers
-fcrossjumping
-fcse-follow-jumps
-fcse-skip-blocks
-fcx-fortran-rules
-fcx-limited-range
-fdata-sections
-fdce
-fdeclone-ctor-dtor
-fdelayed-branch
-fdelete-null-pointer-checks
-fdevirtualize
-fdevirtualize-at-ltrans
-fdevirtualize-speculatively
-fdse
-fearly-inlining
-fexcess-precision=style
-fexpensive-optimizations
-ffast-math
-ffat-lto-objects
-ffinite-loops
-ffinite-math-only
-ffloat-store
-fforward-propagate
-ffp-contract
-ffunction-sections
-fgcse
-fgcse-after-reload
-fgcse-las
-fgcse-lm
-fgcse-sm
-fgnu-tm.
-fgraphite-identity
-fhoist-adjacent-loads
-fif-conversion
-fif-conversion2
-findirect-inlining
-finline-functions
-finline-functions-called-once
-finline-limit=n
-finline-small-functions
-fipa-bit-cp
-fipa-cp
-fipa-cp-clone
-fipa-icf
-fipa-modref
-fipa-profile
-fipa-pta
-fipa-pure-const
-fipa-ra
-fipa-reference
-fipa-reference-addressable
-fipa-sra
-fipa-stack-alignment
-fipa-strict-aliasing
-fipa-vrp
-fira-algorithm=algorithm
-fira-hoist-pressure
-fira-loop-pressure
-fira-region=region
-fisolate-erroneous-paths-attribute
-fisolate-erroneous-paths-dereference
-fivopts
-fkeep-inline-functions
-fkeep-static-consts
-fkeep-static-functions
-flimit-function-alignment
-flive-patching=level
-flive-range-shrinkage
-floop-block
-floop-interchange
-floop-nest-optimize
-floop-parallelize-all
-floop-strip-mine
-floop-unroll-and-jam
-flra-remat
-flto-compression-level=n
-flto -ffat-lto-objects.
-flto[=n]
-flto-partition=alg
-fmerge-all-constants
-fmerge-constants
-fmodulo-sched
-fmodulo-sched-allow-regmoves
-fmove-loop-invariants
-fmove-loop-stores
-fno-allocation-dce
-fno-branch-count-reg
-fno-defer-pop
-fno-fp-int-builtin-inexact
-fno-function-cse
-fno-guess-branch-probability
-fno-inline
-fno-ira-share-save-slots
-fno-ira-share-spill-slots
-fno-keep-inline-dllexport
-fno-lifetime-dse
-fno-math-errno
-fno-peephole
-fno-peephole2
-fno-printf-return-value
-fno-sched-interblock
-fno-sched-spec
-fno-section-anchors.
-fno-signed-zeros
-fno-toplevel-reorder
-fno-trapping-math
-fno-zero-initialized-in-bss
-fomit-frame-pointer
-foptimize-sibling-calls
-foptimize-strlen
-fpartial-inlining
-fpeel-loops
-fPIC
-fpic
-fpie
-fpredictive-commoning
-fprefetch-loop-arrays
-fprofile-correction
-fprofile-generate option.
-fprofile-partial-training
-fprofile-reorder-functions
-fprofile-reorder-functions
-fprofile-use=path
-fprofile-values
-freciprocal-math
-free
-frename-registers
-freorder-blocks
-freorder-blocks-algorithm=algorithm
-freorder-blocks-and-partition
-freorder-functions
-frerun-cse-after-loop
-freschedule-modulo-scheduled-loops
-frounding-math
-fsanitize=address option.
-fsanitize=kernel-hwaddress.
-fsched2-use-superblocks
-fsched-critical-path-heuristic
-fsched-dep-count-heuristic
-fsched-group-heuristic
-fsched-last-insn-heuristic
-fsched-pressure
-fsched-rank-heuristic
-fsched-spec-insn-heuristic
-fsched-spec-load
-fsched-spec-load-dangerous
-fsched-stalled-insns-dep=n
-fsched-stalled-insns=n
-fschedule-fusion
-fschedule-insns
-fschedule-insns2
-fsection-anchors
-fselective-scheduling
-fselective-scheduling2
-fsel-sched-pipelining
-fsel-sched-pipelining-outer-loops
-fsemantic-interposition
-fshrink-wrap
-fshrink-wrap-separate
-fsignaling-nans
-fsimd-cost-model=model
-fsingle-precision-constant
-fsplit-ivs-in-unroller
-fsplit-loops
-fsplit-paths
-fsplit-wide-types
-fsplit-wide-types-early
-fssa-backprop
-fssa-phiopt
-fstdarg-opt
-fstore-merging
-fstrict-aliasing
-fthread-jumps
-ftracer
-ftree-bit-ccp
-ftree-builtin-call-dce
-ftree-ccp
-ftree-ch
-ftree-coalesce-vars
-ftree-copy-prop
-ftree-dce
-ftree-dominator-opts
-ftree-dse
-ftree-forwprop
-ftree-fre
-ftree-loop-distribute-patterns
-ftree-loop-distribution
-ftree-loop-if-convert
-ftree-loop-im
-ftree-loop-ivcanon
-ftree-loop-linear
-ftree-loop-optimize
-ftree-loop-vectorize
-ftree-parallelize-loops=n
-ftree-partial-pre
-ftree-phiprop
-ftree-pre
-ftree-pta
-ftree-reassoc
-ftree-scev-cprop
-ftree-sink
-ftree-slp-vectorize
-ftree-slsr
-ftree-sra
-ftree-switch-conversion
-ftree-tail-merge
-ftree-ter
-ftree-vectorize
-ftree-vrp
-ftrivial-auto-var-init=choice
-funconstrained-commons
-funit-at-a-time
-funreachable-traps
-funroll-all-loops
-funroll-loops
-funroll-loops -fpeel-loops -ftracer -fvpt
-funsafe-math-optimizations
-funswitch-loops
-fuse-linker-plugin
-fvariable-expansion-in-unroller
-fvect-cost-model=model
-fversion-loops-for-strides
-fvpt
-fweb
-fwhole-program
-fzero-call-used-regs=choice
--gcov=profile.afdo
```



# 链接

## 链接的作用

链接过程：

- 空间和地址分配；

- 符号解析；

- 重定位

链接方式：（实际上还有个隐藏的lds链接脚本，可以通过 `ld --verbose`来查看）

```
ld  lib.o main.o -o main
```

## 链接参数

- -T：-T 参数表示指定链接脚本，用户可以通过 ld -T file 来指定使用自己的链接脚本，而不使用系统默认的，在一些特殊的场景中适用。
- @file: 从文件中读取命令行参数，而不是手动指定，通常在脚本编程时使用这种做法。
- -e entry、--entry=entry：这两个命令是同等效果，显示地指定程序开始的位置，通常情况下，需要指定程序内部的符号，如果给定的参数不是一个符号，链接器会尝试将参数解析成数字，表示从指定的地址开始执行程序。
- -EB、-EL：指定大小端，这会覆盖掉系统默认的大小端设置。
- -L、--library-path=searchdir：指定搜索的目录
- -l ：链接指定的库，库名通常是 libname.a 或者 libname.so，使用该参数时去掉库的前后缀，即 -lname
- -o output、--output=output：指定输出文件名
- -s、--strip-all：丢弃可执行文件中的符号，以减小尺寸。
- -static：不使用动态库，静态地链接
- -nostdlib：默认情况下链接标准库，该参数显示地指明不链接标准库。

## 链接脚本解析

``` c
SECTIONS // 定位符.被定义且初始化为0.
{
    . = 0x10000;   //定位符.被重新赋值为0x10000，表示text段开始的地址
    .text : { *(.text) } // 所有输入文件的.text段都放在输出文件的.text段中，*是通配符
    . = 0x8000000;
    .data : { *(.data) }
    .bss : { *(.bss) } //没有重新赋值.，则默认为0x8000000+sizeof(.data)紧随上一个段的结尾位置
}
```

### 程序入口

按优先级排序：

1. -e指定的入口符号
2. 链接脚本中ENTRY()指定的符号
3. 程序中定义的start符号
4. .text段起始地址
5. 0地址

### C语言中引用链接脚本的变量

注意：链接脚本中操作的是变量的地址，可以覆盖C程序中定义的同名变量。

```c
extern int __bss_start;
int *p = &__bss_start;
printf("bss start addr = 0x%x\n",p);
```



## 自定义链接脚本


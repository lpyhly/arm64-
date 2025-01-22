[TOC]



# glibc源码解读

## 使用指南



### 编译自己的glibc库
 1.下载glibc源代码
		glibc 2.24在make时可能会报错，注意下载最新版本的glibc 2.24或更高版本。实测glibc 2.26是ok的。
			Error: `loc1@GLIBC_2.2.5' can't be versioned to common symbol 'loc1'
	http://www.gnu.org/software/libc/
	
2.解压glibc到当前目录
	1	root@xxx:~# tar zxvf glibc-2.17.tar.gz
	
3.创建glibc的build目录
	不能在glibc-2.17源码目录下./configure，会报错“configure: error: you must configure in a separate build directory”
	

	1	root@xxx:~# mkdir -p build/glibc-build

4.configure glibc
	指定prefix目录，注意，要先cd到build/glibc-build下，生成的make才在这里。
	
	1	root@xxx:~/build/glibc-build# /root/glibc-2.17/configure --prefix=/root/build/glibc-build
 5.编译安装
1	root@xxx:~/build/glibc-build# make -j10
2	root@xxx:~/build/glibc-build# make install
可以看到glibc相关的库都在glibc-build目录下了

	1. 重新编译
	删掉glibc-build目录；
	重新configure，make，make install.
###　使用指定glibc来编译程序

``` shell
$ gcc test_log.c -Wl,--rpath=/home/hly/docker/my_glibc/glibc_build/lib -Wl,--dynamic-linker=/home/hly/docker/my_glibc/glibc_build/lib/ld-linux-x86-64.so.2
# 也可以使用特定标记，如：
$ gcc -Wl,-rpath,'$ORIGIN/../lib'  # 表示程序目录，如somedir/app/下的程序，可以查到somdir/lib下的库。另外，还有$LIB, $PLATFORM等字符串可用。
```


其中/usr/local/lib为glibc动态库的路径，linker为动态装载器。

- 可以指定当前不存在的路径
- 可以使用相对路径
- ld.so在编译glibc时会自动生成的。

### debug们

| 功能                       | 详细说明                                                     |
| -------------------------- | ------------------------------------------------------------ |
| 默认搜索路径               | 默认库路径在cat /etc/ld.so.conf中查看<br />搜索顺序：linux调用so库文件时，先搜索当前路径，然后是系统库目录，提供LD_PRELOAD系统变量可以改变这个顺序，改变后的搜索顺序为 LD_PRELOAD, 当前路径, 系统库目录。 |
| 修改搜索路径               | 1) 将库路径加到LD_LIBRARY_PATH里 <br />2) 执行：ldconfig YOUR_LIB_PATH<br />3) 在/etc/ld.so.conf里加入库所在路径。然后执行：ldconfig |
| 查找函数符号在哪个动态库上 | LD_DEBUG=symbols ./enough                                    |
| 读取glibc的变量            | 可以直接通过`printf("Timesize=%d\n", __TIMESIZE);`来确认该值。 |
|                            |                                                              |
|                            |                                                              |
|                            |                                                              |
|                            |                                                              |
|                            |                                                              |
|                            |                                                              |

## 动态链接/加载器

就是/lib/ld-linux.so.*（旧的是ld.so，只支持a.out格式，新的ld-linux.so支持所有elf格式）

### 动态链接程序的加载流程

1. `_dl_start`函数首先完成ld.so的自重定位，然后进入`_dl_main`函数，开始进行真正的装载.
2. `map_object`将所有`DT_NEEDED`所需要的共享库都加载进来
   1. 先调用`dl_new_object`创建一个map，BFS迭代寻找所有需要的共享库，然后根据名称找到的实际依赖共享库的路径。
   2. 打开共享库得到fd，然后调用`_dl_map_object_from_fd`函数完成最终的映射
   3. 现在，可执行文件本身及所有直接/间接的依赖共享库，已经全部被加载到进程虚拟地址空间。并且它们每个都对应了一个link_map。
3. 加载时重定位
   1. `_dl_relocate_object`进行加载。
   2. 要做的事情：重定位所有GOT表中的项。比如一个模块引用另一个模块的某一个变量，因为在模块被加载前，变量的绝对地址是不知道的，这个也需要在装载时进行修复。此外，延迟绑定（PLT）当中，_dl_runtime_resolve函数的地址也需要保存到GOT表当中去，以便于用户程序调用。
   3. 重定位所有DT_TEXTREL项。如果共享库本身不是PIC的，它内部的一些绝对地址引用（比如函数指针），没法使用RIP相对寻址来实现，只能在装载时，由动态链接器来修正这些地址。这一点在现在已经很不常见了。
      1. TEXTREL类型的重定位，指的是地址在不可写入的内存区域（可以理解为代码段）的重定位。ld没法直接修改TEXTREL里面的值，于是需要先通过mprotect增加写权限，然后修改完了再用mprotect给他去除掉写权限）。



``` c
// GLibc源码中的RTLD函数执行流程
RTLD_START()					(sysdeps/i386/dl-machine.h)
  _dl_start()					(elf/rtld.c)
	elf_machine_load_addr()
	elf_get_dynamic_info()					
	ELF_DYNAMIC_RELOCATE()			(elf/dynamic-link.h)
	  elf_machine_runtime_setup()		(sysdeps/i386/dl-machine.h)
	    _ELF_DYNAMIC_DO_RELOC() 		(sysdeps/i386/dl-machine.h)
		elf_dynamic_do_rel()		(elf/do-rel.h)
	            elf_machine{,_lazy_}rel() 	(sysdeps/i386/dl-machine.h)
  _dl_start_final()				(elf/rtld.c)
	_dl_sysdep_start()			(sysdeps/generic/dl-sysdeps.h)
	  _dl_main()				(elf/rtld.c)
	     process_envvars()			(elf/rtld.c)
	     elf_get_dynamic_info()	
	     _dl_setup_hash()			(elf.dl-lookup.c)
	     _dl_new_object()			(elf/dl-object.c)
	     _dl_map_object()			(elf/dl-load.c)
	     _dl_map_object_from_fd()		(elf/dl-load.c)	
	        add_name_to_object()		(elf/dl-load.c)	
	        _dl_new_object()		(elf/dl-object.c)
	        map_segment()			
	        ELF_{PREFERED,FIXED}_ADDRESS()	
	        mprotect()			
	        munmap()
	       _dl_setup_hash()			(elf/dl-lookup.c)
	     _dl_map_object_deps()		(elf/dl-deps.c)
		preload()		
		   _dl_lookup_symbol()		(elf/dl-lookup.c)
		      do_lookup()
		_dl_relocate_object()		(loop in elf/dl-reloc.c)
  _start()					(main binary)
```







### 自定义动态依赖库

对于一个elf对象，按照如下顺序去查找：

1. 对象编译时指定了对应库的绝对路径（判断依赖项中是否包含斜杠，若包含则认为是绝对路径），就直接使用绝对路径；
2. 若没有，则查看二进制文件的 `DT_RPATH` 动态段（section）属性中指定的目录（如果存在且 DT_RUNPATH 属性不存在）。
3. 查看环境变量 `LD_LIBRARY_PATH`的指定路径（除非可执行文件在安全执行模式下运行，在这种情况下该变量将被忽略）。
4. 查看二进制文件的 `DT_RUNPATH` 动态段属性中指定的目录（如果存在）。搜索此类目录只是为了找到 DT_NEEDED（直接依赖项）条目所需的那些对象，而不适用于这些对象的子对象，这些子对象本身必须具有自己的 DT_RUNPATH 条目。这与 DT_RPATH 不同，DT_RPATH 应用于搜索依赖树中的所有子项。
5. 查看缓存文件 /etc/ld.so.cache，其中包含先前在扩充库路径中找到的候选共享对象的编译列表。但是，如果使用 -z nodeflib 链接器选项链接二进制文件，则会跳过默认路径中的共享对象。安装在硬件功能目录（见下文）中的共享对象优先于其他共享对象。可以通过`/lib/ld-linux-aarch64.so.1 --inhibit-cache`来禁止查找。
6. `/etc/ld.so.conf`文件中定义的默认路径：一般是`/usr/lib64`以及部分子目录。如果使用 -z nodeflib 链接器选项链接二进制文件，则跳过此步骤。


### glibc动态调整运行环境

使用GLIBC_TUNABLES

 比如： GLIBC_TUNABLES=glibc.cpu.hwcaps=-Prefer_No_AVX512,Prefer_No_VZEROUPPER

https://www.gnu.org/software/libc/manual/html_node/Hardware-Capability-Tunables.html

1. 列出可以动态调整的项目： ./ld-linux-x86-64.so.2 --list-tunables

``` shell
$ ./elf/ld-linux-x86-64.so.2 --list-tunables
glibc.rtld.nns: 0x4 (min: 0x1, max: 0x10)
glibc.elision.skip_lock_after_retries: 3 (min: 0, max: 2147483647)
glibc.malloc.trim_threshold: 0x0 (min: 0x0, max: 0xffffffffffffffff)
glibc.malloc.perturb: 0 (min: 0, max: 255)
glibc.cpu.x86_shared_cache_size: 0x130000 (min: 0x0, max: 0xffffffffffffffff)
glibc.pthread.rseq: 1 (min: 0, max: 1)
glibc.mem.tagging: 0 (min: 0, max: 255)
glibc.elision.tries: 3 (min: 0, max: 2147483647)
glibc.elision.enable: 0 (min: 0, max: 1)
glibc.malloc.hugetlb: 0x0 (min: 0x0, max: 0xffffffffffffffff)
glibc.cpu.x86_rep_movsb_threshold: 0x2000 (min: 0x100, max: 0xffffffffffffffff)
glibc.malloc.mxfast: 0x0 (min: 0x0, max: 0xffffffffffffffff)
glibc.rtld.dynamic_sort: 2 (min: 1, max: 2)
glibc.elision.skip_lock_busy: 3 (min: 0, max: 2147483647)
glibc.malloc.top_pad: 0x20000 (min: 0x0, max: 0xffffffffffffffff)
glibc.cpu.x86_rep_stosb_threshold: 0x800 (min: 0x1, max: 0xffffffffffffffff)
glibc.cpu.x86_non_temporal_threshold: 0xe4000 (min: 0x4040, max: 0xfffffffffffffff)
```

### 影响elf的环境变量们

#### LD_BIND_NOW

如果设置为非空字符串，则使动态链接器在程序启动时解析所有符号，而不是将函数调用解析推迟到首次引用它们时的时间点。这在使用调试器时很有用。

#### LD_LIBRARY_PATH

在 LD_LIBRARY_PATH 中指定的路径名中，动态链接器扩展标记 `$ORIGIN、$LIB 和 $PLATFORM`（或名称周围使用花括号的版本），如上面动态字符串标记中所述。因此，例如，以下将导致在包含要执行的程序的目录下的 lib 或 lib64 子目录中搜索库：

``` shell
$ LD_LIBRARY_PATH='$ORIGIN/$LIB' orig
```

 #### LD_PRELOAD

有多种指定要预加载的库的方法，这些方法按以下顺序处理：

(1) LD_PRELOAD 环境变量。
(2) 直接调用动态链接器时的 --preload 命令行选项。
(3) /etc/ld.so.preload 文件（如下所述）。

#### LD_TRACE_LOADED_OBJECTS

如果设置（设置为任何值），将导致程序列出其动态依赖项，就像由 [ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html) 运行一样，而不是正常运行。

然后有许多或多或少模糊的变量，许多已过时或仅供内部使用。

同时，ld-linux-so.1 --list cmd 也可以实现该功能。

####  LD_BIND_NOT
如果将此环境变量设置为非空字符串，则在解析函数符号后不要更新 GOT（全局偏移表）和 PLT（过程链接表）。通过将此变量与 LD_DEBUG（与类别绑定和符号）结合使用，可以观察所有运行时函数绑定。

#### LD_DEBUG

输出有关动态链接器操作的详细调试信息。此变量的内容是以下类别之一，以冒号、逗号或（如果值被引用）空格分隔：

help
在此变量的值中指定 help 不会运行指定的程序，并显示有关可以在此环境变量中指定哪些类别的帮助消息。
all
打印所有调试信息（统计和未使用除外；见下文）。
bindings
显示有关每个符号绑定到哪个定义的信息。
files
显示输入文件的进度。
libs
显示库搜索路径。
reloc
显示重定位处理。
scopes
显示范围信息。
statistics
显示重定位统计信息。
**symbols**
**显示每个符号查找的搜索路径。**
unused
确定未使用的 DSO。
versions
显示版本依赖。

#### LD_DEBUG_OUTPUT

默认情况下，LD_DEBUG 输出写入标准错误。如果定义了 LD_DEBUG_OUTPUT，则输出将写入由其值指定的路径名，后缀为“.”。 （点）后跟附加到路径名的进程 ID。

####　LD_DYNAMIC_WEAK

默认情况下，在搜索共享库以解析符号引用时，动态链接器将解析为它找到的第一个定义。

旧的 glibc 版本（2.2 之前）提供了不同的行为：如果链接器发现一个弱符号，它会记住该符号并继续在剩余的共享库中搜索。如果它随后发现同一符号的强定义，则它会改为使用该定义。 （如果没有找到更多的符号，那么动态链接器将使用它最初找到的弱符号。）

旧的 glibc 行为是非标准的。 （标准做法是弱符号和强符号之间的区别应该只在静态链接时有效。）在 glibc 2.2 中，动态链接器被修改为提供当前行为（这是当时大多数其他实现提供的行为）时间）。

定义 LD_DYNAMIC_WEAK 环境变量（具有任何值）提供了旧的（非标准）glibc 行为，其中一个共享库中的弱符号可能会被随后在另一个共享库中发现的强符号覆盖。 （请注意，即使设置了此变量，共享库中的强符号也不会覆盖主程序中相同符号的弱定义。）

#### LD_HWCAP_MASK

硬件功能掩码。



5.2.14. LD_PROFILE（自 glibc 2.1 起）
要分析的（单个）共享对象的名称，指定为路径名或 soname。分析输出附加到名称为`$LD_PROFILE_OUTPUT/$LD_PROFILE.profile`的文件中。

从 glibc 2.2.5 开始，LD_PROFILE 在安全执行模式下被忽略。

5.2.15. LD_PROFILE_OUTPUT（自 glibc 2.1 起）
应该写入 LD_PROFILE 输出的目录。如果此变量未定义，或定义为空字符串，则默认为 /var/tmp。



## 参考文档

官方说明文档：

​	https://www.gnu.org/software/libc/manual/html_node/index.html#SEC_Contents

pwn相关文档：

​	在安全领域中指的是通过二进制/系统调用等方式获得目标主机的shell

## glibc 概念解读

### PLT
#### 什么是PLT表？

**程序链接表**， procedure linkage table 。用来调用连接器来解析某个外部函数的地址，然后跳转到该函数。

- 编译时是不知道地址的
- 动态链接库都是通过这种方式
- 用于延迟绑定的表，即函数第一次被调用的时候才进行绑定。因此会产生data abort。

#### 为什么需要PLT表？

​		如果只有got表，那么在程序运行开始阶段，就需要把所有用到的动态链接库的所有函数地址都写到got表的对应入口中。但实际上，并不一定全都被使用到，尤其是一些大型程序include的头文件很多，但真正用到的函数却很少。为了提升性能，引入了PLT表，当程序真正跑到某个函数时，才会去查找该函数的入口地址，称为运行时加载。

​		PLT表引入后，程序call的函数名称从`memset@got`改为`memset@plt`，这个函数的入口是一条jmp指令，jmp到的位置`.got.plt`表的`memset@.got.plt`处。编译时，`plt`表的这两条jmp地址是固定的，一个对应的是`.got.plt`表的入口，一个对应的是`.got`表的入口。具体来说，

​		第1个jmp：指向`.got.plt`对应地址，其数值就是当前jmp的下一条指令。因此`jmp *addr`后，其实是跳转到当前指令的下一条指令。只不过是拿`.got.plt`当做跳板进行了一次地址翻译。不过，在运行完第2个jmp后，这个跳板的地址会发生改变，不再指向当前指令的下一条指令，而是修改为memset函数的真正执行入口地址。

​		pushq指令： 参数传递，x86中用堆栈进行参数传递，而arm64中使用通用寄存器。

​		第2个jmp： 所有plt函数对应的跳转地址都是一样的（0x402020），这是符号解析和重定位的公共入口，在跳转到这里之前，会先把当前函数在plt中的编号通过pushq指令丢到栈中去，从而方便公共函数判断是要重定位哪个函数。这里的公共函数，应该就是glibc中的`__dl_runtime_fixup`函数了。这个函数会修改0x40b020地址的值，改成当前环境的libc.so动态库在当前进程中的映射中，memset函数的真正执行地址。

​		注意，plt表是在代码段，而`.got.plt`和`.got`表都是在数据段，可以在运行时修改。

``` c
Disassembly of section .plt:
0000000000402020 <.plt>:
  402020:        pushq  0x8fe2(%rip)         # 40b008 <_GLOBAL_OFFSET_TABLE_+0x8>
  402026:        jmpq   *0x8fe4(%rip)        # 40b010 <_GLOBAL_OFFSET_TABLE_+0x10>
  40202c:        nopl   0x0(%rax)

0000000000402030 <printf@plt>:
  402030:        jmpq   *0x8fe2(%rip)        # 40b018 <printf@GLIBC_2.2.5>
  402036:        pushq  $0x0
  40203b:        jmpq   402020 <.plt>

0000000000402040 <memset@plt>:
  402040:        jmpq   *0x8fda(%rip)        # 40b020 <memset@GLIBC_2.2.5>
      	// rip+0x8fda 地址为0x40b020，是.got.plt表的memset对应入口。
      	// *代码取值，[0x40b020]的地址的值是0x402046，也就是下一条指令入口。
  402046:        pushq  $0x1
  40204b:        jmpq   402020 <.plt> 
      	// 0x402020是plt表的总入口地址。
      
Disassembly of section .got.plt:
000000000040b000 <_GLOBAL_OFFSET_TABLE_>: 
	// 数据段当代码段使用，存储的是地址，因此反汇编的指令是没有意义的。
  40b000:       00 ae 40 00 00 00       add    %ch,0x40(%rsi)
        ...
  40b020:       46 20 40 00             rex.RX and %r8b,0x0(%rax)
  40b024:       00 00                   add    %al,(%rax)
  40b026:       00 00                   add    %al,(%rax)
  40b028:       56                      push   %rsi
  40b029:       20 40 00                and    %al,0x0(%rax)
  40b02c:       00 00                   add    %al,(%rax)
  40b02e:       00 00                   add    %al,(%rax)
  40b030:       66 20 40 00             data16 and %al,0x0(%rax)
  40b034:       00 00                   add    %al,(%rax)
  40b036:       00 00                   add    %al,(%rax)
  40b038:       76 20                   jbe    40b05a <_GLOBAL_OFFSET_TABLE_+0x5a>
```

#### arm64中的实现方式是怎样的？

​		与x86大差不差，不过还是有一定区别。基本原理是：

1. `call <memset@plt>` 
2. memset@plt中，计算出`memset`在`.got.plt`表格中的入口地址，然后取指放到x17中，这里存放的指令地址是<.plt>的入口地址。此时x17=0x400fa0, x16=0x4200a0。
3. 第一次运行时，会跳转到<.plt>，走符号解析和重定位流程，然后修改<got.plt>中的内容，下次再ldr x17的时候，就不是<.plt>的入口地址，而是真正的memset函数实现地址了。

``` assembly
Disassembly of section .plt:
0000000000400fa0 <.plt>:
  400fa0:   a9bf7bf0    stp x16, x30, [sp, #-16]!
  400fa4:   f00000f0    adrp    x16, 41f000 <__FRAME_END__+0x17f00>
  400fa8:   f947fe11    ldr x17, [x16, #4088]
  400fac:   913fe210    add x16, x16, #0xff8
  400fb0:   d61f0220    br  x17
  400fb4:   d503201f    nop
  400fb8:   d503201f    nop
  400fbc:   d503201f    nop
	...
0000000000401100 <memset@plt>:
  401100:   f00000f0    adrp    x16, 420000 <memcpy@GLIBC_2.17>
  401104:   f9405211    ldr x17, [x16, #160]
  401108:   91028210    add x16, x16, #0xa0 // 0x420000+160 = 0x4200a0
  40110c:   d61f0220    br  x17

Disassembly of section .got.plt:
000000000041ffe8 <.got.plt>:
    ...
  4200a0:   00400fa0    .inst   0x00400fa0 ; undefined  

```

#### rela.dyn

got表中存放的是真正的执行地址，但是是lazy binding机制的。因此初始状态很多符号的全0的，在执行时才会去修改got表中的内容。而修改时需要一张表，告诉动态连接器是对哪个符号进行重定位，这个就是rela.dyn重定位表，可以通过`readelf -rW a.out`查看。

``` asm
$ readelf -r a.out

Relocation section '.rela.dyn' at offset 0x450 contains 3 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000041ffd0  000200000401 R_AARCH64_GLOB_DA 0000000000000000 _ITM_deregisterTMClone + 0
00000041ffd8  000500000401 R_AARCH64_GLOB_DA 0000000000000000 __gmon_start__ + 0
00000041ffe0  000700000401 R_AARCH64_GLOB_DA 0000000000000000 _ITM_registerTMCloneTa + 0

Relocation section '.rela.plt' at offset 0x498 contains 6 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000420000  000100000402 R_AARCH64_JUMP_SL 0000000000000000 memcpy@GLIBC_2.17 + 0
000000420008  000300000402 R_AARCH64_JUMP_SL 0000000000000000 malloc@GLIBC_2.17 + 0
000000420010  000400000402 R_AARCH64_JUMP_SL 0000000000000000 __libc_start_main@GLIBC_2.17 + 0
000000420018  000500000402 R_AARCH64_JUMP_SL 0000000000000000 __gmon_start__ + 0
000000420020  000600000402 R_AARCH64_JUMP_SL 0000000000000000 abort@GLIBC_2.17 + 0
000000420028  000800000402 R_AARCH64_JUMP_SL 0000000000000000 printf@GLIBC_2.17 + 0

// 反汇编中，rela.dyn段每4Byte为一组，低32bit存放的是符号地址（addr），高32bit存放的是重定位信息（info）。
// elf加载时需要先将info写到addr中。
0000000000400450 <.rela.dyn>:
  400450:       0041ffd0        .inst   0x0041ffd0 ; undefined
  400454:       00000000        .inst   0x00000000 ; undefined
  400458:       00000401        .inst   0x00000401 ; undefined
  40045c:       00000002        .inst   0x00000002 ; undefined
        ...
  400468:       0041ffd8        .inst   0x0041ffd8 ; undefined
  40046c:       00000000        .inst   0x00000000 ; undefined
  400470:       00000401        .inst   0x00000401 ; undefined
  400474:       00000005        .inst   0x00000005 ; undefined
        ...
000000000041ffc8 <_GLOBAL_OFFSET_TABLE_>:
  41ffc8:       0041fdf8        .inst   0x0041fdf8 ; undefined
        ...

```

​		比如上表，意思是got中本来只有0x41ffc8地址有数值，其他为全0.但是运行时根据rela.dyn段可以将0x41ffd0/d8等地址写入相应的值，就是glibc中对应符号的真正地址。

​		

### GOT

#### 什么是GOT

​		Global Offset Table，全局偏移表，是Linux ELF文件中用于定位**全局变量和函数**的一个表。

#### 为什么需要got表？

​	当使用动态链接库的函数时，需要call <func addr>，需要知道所调用的函数的地址，但是编译过程中，我们是不知道会用到哪个库的这个函数的，也就是无法确认函数地址。

解决方案是：拿一小段数据段过来当做跳板，存放func addr的真实值，这个值在程序运行时就可以确定了，因此运行时解析出地址放过来就好。使用数据段的原因是，地址段不允许运行时修改值，而数据段可以。

那么，这个跳板就是got表了。

比如：

``` c
// 代码段：
	call 0x1004  <memset@got>

// 数据段got表：
0x1000:  < memcpy@got > 的真实地址
    call 0x8888834
0x1004:  < memset@got > 的真实地址 
	call 0x3219473
```

#### 什么是函数重定位？

​		在延迟加载的情况下，每个外部函数的got表都会被初始化成plt表中对应项的地址。当call指令执行时，EIP直接跳转到plt表的一个jmp，这个jmp直接指向对应的got表地址，从这个地址取值。此时这个jmp会跳到保存好的，plt表中对应项的地址，在这里把每个函数重定位过程中唯一的不同点，即一个数字入栈（本例子中write是18h,read是0，对于单个程序来说，这个数字是不变的），然后push got[1]并跳转到got[2]保存的地址。在这个地址中对函数进行了重定位，并且修改got表为真正的函数地址。当第二次调用同一个函数的时候，call仍然使EIP跳转到plt表的同一个jmp，不同的是这回从got表取值取到的是真正的地址，从而避免重复进行重定位。

### glibc 内存管理

​	当编译成共享库的时候，在库中定义的static等全局变量，会放到lib库的段的可读写数据段中，**每个进程独有**，在该进程内生效，进程中的各个线程是共享的。

验证：

​	自定义一个libshare库，打印

```c
(share.h)
#ifndef foo_h__                       
#define foo_h__                       
extern int foo(void);                 
#endif  // foo_h__                    

(share.c)
#include <stdio.h>                    
static int i=0;                       
int foo() {                           
    printf("i add is 0x%x\n", &i);    
    return i++;                       
}                                     
```

测试代码中： 主进程先创建2个线程，然后再创建一个子进程。最后，2个进程 & 2个线程一块儿调用foo()函数。

```c
#define _GNU_SOURCE                                                        
#include <stdio.h>                                                         
#include <stdlib.h>                                                        
#include <sys/types.h>                                                     
#include <unistd.h>                                                        
#include "share.h"                                                         
#include <pthread.h>                                                       
                                                    
void *thread_func(void *arg)                                               
{                                                                          
    for(int i=0; i<100; i++) {                                             
        printf("pid=%d, tid=%d, foo()=%d\n",getpid(), gettid(), foo());
        sleep(1);                                                          
    }                                                                      
}                                                                          
void main()                                                                
{                                                                          
    pthread_t tids[2];                                                     
    pthread_create(&tids[0], NULL, thread_func, NULL);                     
    pthread_create(&tids[1], NULL, thread_func, NULL);                     
    fork();                                                                
    thread_func(NULL);                                                     
    pthread_join(tids[0], NULL);                                           
    pthread_join(tids[0], NULL);                                           
    return;                                                                
}                                                                          
```

测试结果： 

```c
i add is 0xf7f80024      // 因为关掉了randmap，所以每个libshare库都加载到进程的同一个VA地址，
pid=3739203, tid=3739203, foo()=0   // 但是，2个进程的BSS段对应的PA地址是不同的。
i add is 0xf7f80024                          
pid=3739203, tid=3739206, foo()=1    // 子线程们和主进程共享一块内存，一起访问。
i add is 0xf7f80024              
pid=3739207, tid=3739207, foo()=0     // 子进程， 自己独立统计。
i add is 0xf7f80024              
pid=3739203, tid=3739205, foo()=2
i add is 0xf7f80024              
pid=3739203, tid=3739203, foo()=3
i add is 0xf7f80024              
pid=3739203, tid=3739206, foo()=4
i add is 0xf7f80024              
pid=3739207, tid=3739207, foo()=1
i add is 0xf7f80024              
pid=3739203, tid=3739205, foo()=5
i add is 0xf7f80024              
pid=3739203, tid=3739203, foo()=6
i add is 0xf7f80024              
pid=3739203, tid=3739206, foo()=7
i add is 0xf7f80024                 
```

编译方式：

```c
gcc -c share.c -oshare.o      
gcc -shared -o libshare.so share.o // 编译动态库            
gcc -L/home/hly/tmp test.c -lshare -lpthread  // 链接动态库 
LD_LIBRARY_PATH=. taskset -c 1 ./a.out    // 运行测试程序
```

程序链接地址map：

```shell
fffff7f60000-fffff7f70000 r-xp 00000000 fd:05 80092518                   /home/hly/tmp/libshare.so
fffff7f70000-fffff7f80000 r--p 00000000 fd:05 80092518                   /home/hly/tmp/libshare.so
fffff7f80000-fffff7f90000 rw-p 00010000 fd:05 80092518                   /home/hly/tmp/libshare.so
```





## glibc 源码树结构

### sysdeps目录

​        GNU配置名称包含三部分：CPU类型，制造商名称和操作系统。 配置使用这些来选择要查找的系统相关目录的列表。操作系统通常具有*基本操作系统* ; 例如，如果操作系统是Linux，基本操作系统是Unix / sysv。用于选择目录列表的算法很简单： 配置以此顺序列出基本操作系统，制造商，CPU类型和操作系统。然后将所有这些连接起来，并在其之间加上斜杠，以产生目录名；例如，配置“i686-linux-gnu' 结果Unix / sysv / linux / i386 / i686。 配置 然后尝试依次删除列表中的每个元素，因此 unix / sysv / linux 和 Unix / sysv也被尝试过。由于操作系统的确切版本号通常并不重要，因此依次尝试使用不太具体的操作系统名称。

举例来说，以下是配置“i686-linux-gnu' ：

```
sysdeps / i386 / elf
sysdeps / unix / sysv / linux / i386
sysdeps / unix / sysv / linux
sysdeps / gnu
sysdeps / unix / common
sysdeps / unix / mman
sysdeps / unix / inet
sysdeps / unix / sysv / i386 / i686
sysdeps / unix / sysv / i386
sysdeps / unix / sysv
sysdeps / unix / i386
sysdeps / unix
sysdeps / posix
sysdeps / i386 / i686
sysdeps / i386 / i486
sysdeps / libm-i387 / i686
sysdeps / i386 / fpu
sysdeps / libm-i387
sysdeps / i386
sysdeps / wordsize-32
sysdeps / ieee754
sysdeps / libm-ieee754
sysdeps /通用
```



## glibc 反汇编段

从反汇编文件中，可以查看到如下几个段：

##### .dynamic

​	可以通过/proc/<pid>/map查看到，就是函数运行时的真实va地址分布。

##### .dynstr (函数)

​	动态链接字符串表的地址，可以用来查找动态链接库中的函数名的字符串，从而对应到函数的真实地址。

##### .dynsym （变量）

​	动态链接符号表的地址

##### .rela.plt

​	动态链接重定位表的信息



## glibc 宏解读

#### glibc库常用函数的内部函数名对应


| 宏定义                         | libc库中的函数 | 对外呈现的函数 |
| ------------------------------ | -------------- | -------------- |
| libc_hidden_def(__libc_malloc) | __libc_malloc  | malloc         |
| hidden_def(exit)               | exit           | \_\_GI\_\_exit |
|                                |                |                |

### 辅助函数说明

| 宏定义 | libc库中的函数                                    | 说明                                              |
| ------ | ------------------------------------------------- | ------------------------------------------------- |
| weak   | int foo() __attribute(weak)                       | 表示foo函数先从其他地方找定义，找不到再使用这个。 |
| strong |                                                   | 与weak差不多的。                                  |
| alias  | void f() \__attribute__ ((weak, alias ( "__f"))); | 符号f是符号__f的别称。                            |
|        |                                                   |                                                   |

##### _dl_fini

​	_dl_fini->`__do_global_dtors_aux`，其中`__do_global_ctors_aux`和`__do_global_dtors_aux`是gcc中的函数，用于加载动态库的时候构造和析构被加载的动态库中的全局变量（然后才加载库函数）

##### \_dl_runtime_resolve

​	函数作用：第一次加载某个函数的时候，got表中还没有写入真实的地址，这时需要调用本函数将地址写入。注意，这里是延迟加载，就是当真正调用函数时才会去加载。

​	比如，调用read时：跳转过程为：

`read@plt -> read@got -> read@plt -> plt[0] -> _dl_runtime_resolve_xsave -> _dl_fixup`

​	函数行为：先保存寄存器的值到栈中，然后调用`_dl_fixup`，最后再从栈中恢复寄存器。

​	others：感觉动态插桩也可以这样做。

##### _dl_fixup

​	传入的两个参数：一个是rdi寄存器中存储的link_map，一个rsi是GOT表中关于PLT重定位的索引值，后面要根据该索引值写入新的地址。

函数行为：

	1. 从link_map结构中获取符号表symtab、字符串表strtab。

   	2. 从GOT表中计算出最终要修改函数的绝对地址。
   	3. 获取符号的version信息，调用`_dl_fixup->_dl_lookup_symbol_x->do_lookup_x`函数从已装载的共享库中查找最终的符号地址，查找到符号sym后，对其进行重定位，即加上装载地址，保存在value中。
       最后调用elf_machine_fixup_plt函数进行修正。





##### libc_ifunc

需要在运行时判断使用哪个版本的函数时，就可以用这个接口。比如，需要根据不同硬件特性（hwcaps）提供不同的函数实现方案。 具体实现上，以函数foo()为例，有两个版本的实现： `__foo_default` & `__foo_special`。

-  若定义时没有使用libc_hidden_proto(foo)，则可以先在INIT_ARCH()中定义hwcap，然后使用如下方式来进行函数定义。

  ``` c
  libc_ifunc (foo, (hwcap & HWCAP_SPECIAL) ? __foo_special : __foo_default);
  ```

- 若定义时使用了libc_hidden_proto(foo)，则在libc内部和在外部调用需要分开来看。

  - libc内部，可以如下定义，来强制使用`__foo_default`来作为foo函数的具体实现。

    ``` c
    __hidden_ver1 (__foo_default, __GI_foo, __foo_default);
    ```

  - libc外部，可以如下定义，再添加一个`__redirect_foo`作为桥梁，以区别内部的实现。

    ```c
    #define foo __redirect_foo
    #include <foo.h>
    #undef foo
    
    extern __typeof (__redirect_foo) __foo_default attribute_hidden;
    extern __typeof (__redirect_foo) __foo_special attribute_hidden;
    
    libc_ifunc_redirected (__redirect_foo, foo,
               (hwcap & HWCAP_SPECIAL)
               ? __foo_special
               : __foo_default);
    ```

- 若定义时使用了libc_hidden_proto(foo)，且需要libc的内部和外部都使用相同的实现方式，则可如下实现。

  ```c
  libc_ifunc_hidden (foo, foo,
             (hwcap & HWCAP_SPECIAL)
             ? __foo_special
             : __foo_default);
  libc_hidden_def (foo)
  ```

  

用法详见"include/libc-symbols.h"，有详细说明和示例。

`memcpy`就是以这种方式来实现不同版本的混布，关于使用哪一个版本的memcy，在glibc中是会实现编译好所有版本，然后在运行时根据硬件环境来动态判断。注意，即使是静态链接，也是这么做的。判断逻辑如下：

``` c
libc_ifunc (__libc_memcpy,
            (IS_THUNDERX (midr)
         ? __memcpy_thunderx
         : (IS_FALKOR (midr) || IS_PHECDA (midr)
        ? __memcpy_falkor
        : (IS_THUNDERX2 (midr) || IS_THUNDERX2PA (midr)
           ? __memcpy_thunderx2
           : (IS_NEOVERSE_N1 (midr) || IS_NEOVERSE_N2 (midr)
              || IS_NEOVERSE_V1 (midr)
              ? __memcpy_simd
# if HAVE_AARCH64_SVE_ASM
             : (IS_A64FX (midr)
            ? __memcpy_a64fx
            : __memcpy_generic))))));
# else
             : __memcpy_generic)))));
# endif
# undef memcpy
```

其中，判断逻辑是看midr_el1的值：

``` c
// "sysdeps/unix/sysv/linux/aarch64/cpu-features.h"
// 是否为FALKOR:
#define IS_FALKOR(midr) (MIDR_IMPLEMENTOR(midr) == 'Q'                \
                        && MIDR_PARTNUM(midr) == 0xc00)
// 是否为kunpeng
#define IS_KUNPENG920(midr) (MIDR_IMPLEMENTOR(midr) == 'H'             \
                        && MIDR_PARTNUM(midr) == 0xd01)
```

而midr是个全局变量，在"sysdeps/aarch64/multiarch/init-arch.h"头文件中的INIT_ARCH()函数中定义。

##### INIT_ARCH

```asm
#define INIT_ARCH()                               \
  uint64_t __attribute__((unused)) midr =                     \
    GLRO(dl_aarch64_cpu_features).midr_el1;                   \
  unsigned __attribute__((unused)) zva_size =                     \
    GLRO(dl_aarch64_cpu_features).zva_size;                   \
  bool __attribute__((unused)) bti =                          \
    HAVE_AARCH64_BTI && GLRO(dl_aarch64_cpu_features).bti;            \
  bool __attribute__((unused)) mte =                          \
    MTE_ENABLED ();                               \
  bool __attribute__((unused)) sve =                          \
    GLRO(dl_aarch64_cpu_features).sve;
```

##### OPTIMIZE

用来定义同一个函数的不同实现方案。

``` c
#define PASTER1(x,y)    x##_##y
#define EVALUATOR1(x,y) PASTER1 (x,y)
#define PASTER2(x,y)    __##x##_##y
#define EVALUATOR2(x,y) PASTER2 (x,y)

/* Basically set '__redirect_<symbol>' to use as type definition,
   '__<symbol>_<variant>' as the optimized implementation and
   '<symbol>_ifunc_selector' as the IFUNC selector.  */
#define REDIRECT_NAME   EVALUATOR1 (__redirect, SYMBOL_NAME)
#define IFUNC_SELECTOR  EVALUATOR1 (SYMBOL_NAME, ifunc_selector)
#define OPTIMIZE1(name) EVALUATOR1 (SYMBOL_NAME, name)
#define OPTIMIZE2(name) EVALUATOR2 (SYMBOL_NAME, name)
/* Default is to use OP
#define OPTIMIZE(name)  OPTIMIZE2(name)
```

例如，定义foo函数的两个实现`__foo_impl1`和`__foo_impl2`，则可以写作：

``` c
   #define foo __redirect_foo
   #include <foo.h>
   #undef foo
   #define SYMBOL_NAME foo
   #include <ifunc-init.h>

   extern __typeof (REDIRECT_NAME) OPTIMIZE (impl1) attribute_hidden;
   extern __typeof (REDIRECT_NAME) OPTIMIZE (impl2) attribute_hidden;

   static inline void *
   foo_selector (void)
   {
     if (condition)
      return OPTIMIZE (impl2);

     return OPTIMIZE (impl1);
   }

   libc_ifunc_redirected (__redirect_foo, foo, IFUNC_SELECTOR ());

```







## glibc中的系统调用

以times()函数为例，通用实现在"./sysdeps/unix/sysv/linux/times.c"文件中，会调用INTERNAL_SYSCALL_CALL宏，宏展开后其实就是调用svc接口函数。

``` c
 clock_t __times (struct tms *buf)
 {
   clock_t ret = INTERNAL_SYSCALL_CALL (times, buf);
   ......
 }
```

在glibc中有个文件专门放了glibc能使用到的syscall的名字，在sysdeps/unix/sysv/linux/syscall-names.list；

同时，针对不同架构，svc的调用号是不同的，这个存放在不同架构下的./sysdeps/unix/sysv/linux/aarch64/arch-syscall.h文件中。

### exit

对外呈现的是\_\_GI_exit\_\_.

``` c
"stdlib/exit.c"
 exit(status)->__run_exit_handlers(status, &__exit_funcs, true, true)


void attribute_hidden
__run_exit_handlers (int status, struct exit_function_list **listp,
             bool run_list_atexit, bool run_dtors)
{							 // status=-1, run_list_atexit=true, run_dtors=true.
    if (run_dtors)
      __call_tls_dtors (); // 1. 调用析构函数，这里是调用的一个全局的函数。
      // 在文件cxa_thread_atexit_impl.c中，作用是调用tls_dtor_list()，这里貌似没调用到。
==========================================================================
  /* We do it this way to handle recursive calls to exit () made by
     the functions registered with `atexit' and `on_exit'. We call
     everyone on the list and use the status value in the last
     exit (). */
  while (true)
    {    ...
         case ef_cxa:
           cxafct = f->func.cxa.fn; // cxafct = _dl_fini
           cxafct (f->func.cxa.arg, status);  // 2. 调用_dl_fini函数，对于所有loaded objects都需要调用。 _dl_fini -> __do_global_dtors_aux -> _fini-> __cxa_finalize，这里是调用了__cxa_atexit的dso处理函数。
           break;
         }	
	}
  if (run_list_atexit)
    RUN_HOOK (__libc_atexit, ()); // 3. IO cleanup，调用 _IO_cleanup函数清理，比如printf中若没有\n的本来会存放在buffer中，现在全部都直接打印出来了。

  _exit (status);  // 4. 内核态的cleanup，这是一个syscall接口。
}
```




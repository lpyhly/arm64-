# randomize_va_space

## 什么是va空间随机化？

就是`aslr`， elf文件在运行时，会将其stack/heap/mmap/vdso等全部动态分配的地址空间，都作随机化处理。这样，两次运行同一个程序时，就会发现其地址空间是不同的。比如：

```shell
# 地址随机化的一个例子；
echo 2 > /proc/sys/kernel/randomize_va_space
ldd /usr/bin/ls # ldd显示的库的虚拟地址，是当前这个elf运行时的程序的链接库，
ldd /usr/bin/ls # 与上一次的地址不同了。
echo 0 > /proc/sys/kernel/randomize_va_space
ldd /usr/bin/ls
ldd /usr/bin/ls # 与上一次的地址相同。

# 地址随机化的令一个例子：
echo 2 > /proc/sys/kernel/randomize_va_space
cat /proc/sys/kernel/randomize_va_space
ls & pid=$!; cat /proc/$pid/maps > $pid.txt
ls & pid=$!; cat /proc/$pid/maps > $pid.txt
diff *.txt

echo 0 > /proc/sys/kernel/randomize_va_space
ls & pid=$!; cat /proc/$pid/maps > $pid.txt
ls & pid=$!; cat /proc/$pid/maps > $pid.txt
diff *.txt
```



## 怎么配置使能/关闭？

``` shell
# 关闭： 
可以通过内核启动项： norandmaps 配置，也可以通过sysctl配置
$  echo 0 > /proc/sys/kernel/randomize_va_space。

# 使能：
$	echo 2 > /proc/sys/kernel/randomize_va_space
```

总共可以配置3个值：0/1/2。 其中0表示关闭，2表示打开，1表示在开启BRK调试的情况下（CONFIG_COMPAT_BRK）除BRK空间外其余空间随机化。

## 具体实现方式？

在`load_elf_binary`函数中，会对当前task的flags进行相关置位。然后根据这个计算一个load_bias

``` c
load_elf_binary() {
    ...
	if (!(current->personality & ADDR_NO_RANDOMIZE) && randomize_va_space)
        current->flags |= PF_RANDOMIZE;
    ...
    load_bias = ELF_ET_DYN_BASE;             
	if (current->flags & PF_RANDOMIZE)       
        load_bias += arch_mmap_rnd();    // 随机数
    ... 
    e_entry = elf_ex->e_entry + load_bias;    // 后续一堆地址都使用这个load_bias偏移量。
	phdr_addr += load_bias;                       
	elf_bss += load_bias;                         
	elf_brk += load_bias;                         
	start_code += load_bias;                      
	end_code += load_bias;                        
	start_data += load_bias;                      
	end_data += load_bias;                        

}
```



# 拓展阅读：personality

personality表示进程执行域，范围为某个进程；可以通过setarch修改，也可以在C文件中显示指定。

- 可以用来指示如下几种策略，通过修改信号number对应的具体执行动作等方式。

```c
 /* 详细定义在<sys/personality.h> */

       ADDR_COMPAT_LAYOUT (since Linux 2.6.9)
              With this flag set, provide legacy virtual address space layout.

       ADDR_NO_RANDOMIZE (since Linux 2.6.12)
              With this flag set, disable address-space-layout randomization.

       ADDR_LIMIT_32BIT (since Linux 2.2)
              Limit the address space to 32 bits.

       ADDR_LIMIT_3GB (since Linux 2.4.0)
              With this flag set, use 0xc0000000 as the offset at which to search a virtual memory chunk on mmap(2); otherwise use 0xffffe000.

       FDPIC_FUNCPTRS (since Linux 2.6.11)
              User-space function pointers to signal handlers point (on certain architectures) to descriptors.

       MMAP_PAGE_ZERO (since Linux 2.4.0)
              Map page 0 as read-only (to support binaries that depend on this SVr4 behavior).

       READ_IMPLIES_EXEC (since Linux 2.6.8)
              With this flag set, PROT_READ implies PROT_EXEC for mmap(2).

       SHORT_INODE (since Linux 2.4.0)
              No effects(?).

       STICKY_TIMEOUTS (since Linux 1.2.0)
              With this flag set, select(2), pselect(2), and ppoll(2) do not modify the returned timeout argument when interrupted by a signal handler.

       UNAME26 (since Linux 3.1)
              Have  uname(2)  report a 2.6.40+ version number rather than a 3.x version number.  Added as a stopgap measure to support broken applications that could not handle the kernel version-numbering
              switch from 2.6.x to 3.x.

       WHOLE_SECONDS (since Linux 1.2.0)
              No effects(?).

```

# 拓展阅读：setarch

`setarch`命令可以用来修改`personality`，以及修改`elf`的执行环境（修改`uname -m`的显示值），比如在x86上运行amd64的指令等。

作用1：修改执行环境：

``` shell
# x86上：
$ setarch --list
uname26
linux32
linux64
i386
i486
i586
i686
athlon
x86_64

# arm64上：
$ setarch --list
uname26
linux32
linux64
armv7l
armv8l
armh
arm
arm64
aarch64
```

作用2： 修改`personality`

``` shell
/* disables randomization of the virtual address space */
$ setarch -R 
```




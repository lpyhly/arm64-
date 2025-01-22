

[TOC]

# 配套文件解析

#### TF

TF(trust firmware)，包含TEE/REE，在arm中有个opentrustfirmware的开源项目，可以基于此进行tf文件的编译。这个东西运行在el3，负责RAS的处理等。

#### IMU

IMU（integrated management unit）， 是另外一个cpu核，放在IOdie上，与cpu die完全独立。功能包括：RAS故障预处理以 及错误记录上报、安全信任根、能效管理、 芯片内部管理。

#### IMP

IMP（integrated management processor），网卡智能处理器固件，没有OS，是一个单线程系统。集成了NIC，ROCE，UB，ETH，SerDes等特性。功能上主要包括4部分：

1. 初始化NCL config的配置完成一些硬件资源的初始化。
2. 中断处理：CMDQ命令处理等
3. 周期性任务：链路建链等
4. 异常处理： 不可恢复中断

#### IMF



# BOOT/BIOS启动

## BOOT启动

​	boot和bios都是烧在flash中的一段代码，boot可以看做是一级bios，用来引导bios的。在emu中： boot->bios->kernel，在fpga中： bios->kernel。

​	bios主要作用是：

1. 初始化PLL（锁相环）。
2. 初始化串口。 
3. 将flash中的bios代码搬移到ddr中。

为了实现这几个作用，还需要配套的是：

1. 初始化PLL：为了把晶振时钟分频，从而能让时钟调整到高频时钟。初始化后，就可以把时钟升频了。
2. 为了使能串口，还需要初始化GIC。因此在串口有打印前，都是boot在做的事情。
3. 代码搬移到ddr后，运行速率也会加快。

注意，这里还是关MMU的状态，同时也只有c0在运行。而c0的上电运行，是从单板上电后就直接开始了的。

## BIOS启动

第二段代码相当于bios启动了，这里要做的事情是：

1. 逐一唤醒从核；
2. 开mmu；
3. 跳转到el1，去执行kernel代码；



# 完整启动流程

BIOS(flash中读取，可能是lagency BIOS或UEFI BIOS) 

 -> 读取磁盘头，寻找并进入efi分区

 -> efi分区为FAT32格式，UEFI BIOS一定要支持，有了操作系统，就可以去寻找EFI\BOOT\BOOT<MECHINE_TYPE_SHORT_NAME>.EFI文件了，fat 格式的磁盘文件不区分大小写。例如在 openEuler 操作系统上，`BOOTX64.EFI/BOOTAA64.EFI`，实际为 shim 生成的扫描程序，会自动扫描所有 EFI 文件夹下含有 `BOOTX64.CSV/BOOTAA64.CSV`
的文件夹，并根据其内容创建新的启动项，方便下次启动时直接从该目录进行引导启动。例如：`BOOTAA64.CSV`

shimaa64.efi

<img src="D:\at_work\Documents\我的总结文档\images\image-20211229091603978.png" alt="image-20211229091603978" style="zoom:50%;" />

shim: 通常用于对 bootloader(grub2)进行安全检查。
shim: shim是一个简单的EFI应用程序，在运行时会尝试通过标准EFI LoadImage（）和StartImage（）调用来打开并执行另一个应用程序。
如果这些失败（因为在使能 Secure Boot 时会对二进制文件进行检查，有可能二进制应用没有适当的密钥签名），则它将根据内置证书验证二进制文件。
如果成功，并且二进制或签名密钥未列入黑名单，则shim将重定位并执行二进制文件。

#### Q: 如何判断是MBR分区还是GPT分区？

fdisk -l或parted -l, 看Disk label type，若为dos说明为MBR分区，若为gpt说明为GPT分区。

## MBR读取

MBR需要兼容lagacy bios模式以及UEFI模式，占据磁盘最初的512Byte字节山区。

<img src="D:\at_work\Documents\我的总结文档\images\image-20211229104908370.png" alt="image-20211229104908370" style="zoom:50%;" />

``` shell
# 对于包含启动分区的磁盘，我们可以看一下它的MBR信息。
[root@localhost mnt]# hexdump  -C -n 512 /dev/nvme0n1                           
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
*                                                                               
000001b0  00 00 00 00 00 00 00 00  08 90 98 52 00 00 00 00  |...........R....|  
000001c0  01 01 83 3f 20 00 00 08  00 00 00 00 80 07 00 00  |...? ...........|  
000001d0  01 01 83 3f 20 00 00 08  80 07 00 00 80 07 00 00  |...? ...........|  
000001e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|  
00000200                                                                        
```

- 前446Byte中，包含(08 90 98 52)。表示指定的bootloader；

- 接下来的64Bytes可以分成4*16B，每条分区表记录为16Byte，MBR分区表最大支持4个主磁盘分区，或者3主1扩展分区。

  - |                      |                                                              |
    | -------------------- | ------------------------------------------------------------ |
    | 第1字节              | 引导标志。若值为80H表示活动分区，若值为00H表示非活动分区。   |
    | 第2、3、4字节        | 本分区的起始磁头号、扇区号、柱面号。其中：<br/>    磁头号——第2字节；<br/>    扇区号——第3字节的低6位；<br/>    柱面号——为第3字节高2位+第4字节8位。 |
    | 第5字节              | 分区类型符。<br/>    00H——表示该分区未用（即没有指定）；<br/>    06H——FAT16基本分区；<br/>    0BH——FAT32基本分区；<br/>    05H——扩展分区；<br/>    07H——NTFS分区；<br/>    0FH——（LBA模式）扩展分区（83H为Linux分区等）。 |
    | 第6、7、8字节        | 本分区的结束磁头号、扇区号、柱面号。其中：<br/>    磁头号——第6字节；<br/>    扇区号——第7字节的低6位；<br/>    柱面号——第7字节的高2位+第8字节。 |
    | 第9、10、11、12字节  | 逻辑起始扇区号 ，本分区之前已用了的扇区数。                  |
    | 第13、14、15、16字节 | 本分区的总扇区数。                                           |
    |                      |                                                              |
    |                      |                                                              |

  - 第一个分区：（00 00 00 00 01 01 83 3f 20 00 00 08  00 00 00 00），第一个字节0x00代表未激活，不能用来引导；

  - 第二个分区：(80) (07 00 00) (01) (01 83 3f) (20 00 00 08) (80 07 00 00）

  - 分区表数据说明：**(80)(20 21 00)(83)(aa 28 82)(00 08 00 00)(00 00 20 00)**

    - (80):0x80 代表是激活分区，可以用来引导，0x00 代表未激活分区，不能用来引导
    - (20 00 00 08):分区开始扇面，图中 CPU 为小端序，故该值为 0x8000020=134217760，所以该分区开始于 134217760 扇面.
    - (80 07 00 00):分区总共扇面数，因 CPU 是小端序，故该值为 0x780=1920，该扇区的容量为 1920*512KiB/1024/1024/1024

- 最后的2byte为magic number，=0xaa55，表示该磁盘中包含引导分区。

MBR的劣势：

	1. 只支持4个分区；
	2. 磁盘容量最大只支持2T（每个扇区512Byte，扇区idx为32bit标识，所以2^32 * 512B =2TB）。
## GPT读取

对于分区，若为UEFI引导，则分区格式为GUID格式（Globally Unique Identifier），其分区表称为GPT，即GUID partition table。

优势；

	1. 分区数量： 无限制，为每个分区分配一个全局唯一的标识符，是随机生成的字符串。
 	2. 磁盘容量几乎无限制，用扇区号idx由32bit升级到64bit；
 	3. 分区表自带备份；

GPT表格式：

<img src="D:\at_work\Documents\我的总结文档\images\image-20211229143636088.png" alt="image-20211229143636088" style="zoom:50%;" />

LBA（Logical Block Addressing）表示分区，每个分区512Byte大小。

- LBA0： 又称Protective MBR，为了兼容传统MBR，仿照传统MBR模式写的部分。在支持MBR/GPT混合分区表的硬盘中，也存储了MBR格式启动的MBR表头。

  ``` shell
  [root@localhost ~]# hexdump -C -n 512 -s 0 /dev/nvme0n1p1                       
  00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
  *                                                                               
  000001c0  02 00 ee ff ff ff 01 00  00 00 ff ff 3f 06 00 00  |............?...|  
  000001d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
  *                                                                               
  000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|  
  ```

- LBA1：真正的GPT表头，定义了硬盘的可用空间，以及组成分区表的项的大小和数量等。

  ```shell
  [root@localhost ~]# hexdump -C -n 512 -s 512 /dev/nvme0n1p1                     
  00000200  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|  
  00000210  37 42 f0 c2 00 00 00 00  01 00 00 00 00 00 00 00  |7B..............|  
  00000220  ff ff 3f 06 00 00 00 00  22 00 00 00 00 00 00 00  |..?.....".......|  
  00000230  de ff 3f 06 00 00 00 00  a3 32 7b 33 8d 7c 5b 42  |..?......2{3.|[B|  
  00000240  83 e0 0d 92 67 11 ff c4  02 00 00 00 00 00 00 00  |....g...........|  
  00000250  80 00 00 00 80 00 00 00  e9 86 ba cf 00 00 00 00  |................|  
  00000260  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
  ```

  根据下表可知，

  （45 46 49 20 50 41 52 54）： 前8Byte为EFI的标识头，表示当前分区为EFI分区；

  （00 00 01 00）： GPT版本号；

  （5c 00 00 00）：表头大小，0x5c = 92Byte。

  （37 42 f0 c2）： CRC校验

  offset 24：（01 00 00 00 00 00 00 00）：当前LBA的位置为0x1；

  （ff ff 3f 06 00 00 00 00）：备份LBA的位置（在硬盘的最后，若CRC校验错了，则使用备份GPT）

  offset 0x38：（a3 32 7b 33 8d 7c 5b 42）： GUID号

  offset 0x50：（80 00 00 00）： 128个分区entry（分区entry存放在LBA2中）

  offset 0x58：（80 00 00 00）： 每个分区entry为128Byte大小。

  ![image-20211229144351086](D:\at_work\Documents\我的总结文档\images\image-20211229144351086.png)

- LBA2： 描述了各个分区entry，由LBA1可知，每个分区entry为128Byte，最多可以描述128个分区。

  <img src="D:\at_work\Documents\我的总结文档\images\image-20211229145532817.png" alt="image-20211229145532817" style="zoom:50%;" />

  ``` shell
  [root@localhost ~]# hexdump -C -n 512 -s 1024 /dev/nvme0n1p1                    
  00000400  28 73 2a c1 1f f8 d2 11  ba 4b 00 a0 c9 3e c9 3b  |(s*......K...>.;|  
  00000410  7f ff fa cd 3a 2d 4c 43  87 13 64 fc 35 dd 78 38  |....:-LC..d.5.x8|  
  00000420  00 08 00 00 00 00 00 00  ff c7 12 00 00 00 00 00  |................|  
  00000430  00 00 00 00 00 00 00 00  45 00 46 00 49 00 20 00  |........E.F.I. .|  
  00000440  53 00 79 00 73 00 74 00  65 00 6d 00 20 00 50 00  |S.y.s.t.e.m. .P.|  
  00000450  61 00 72 00 74 00 69 00  74 00 69 00 6f 00 6e 00  |a.r.t.i.t.i.o.n.|  
  00000460  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
  *                                                                               
  00000480  af 3d c6 0f 83 84 72 47  8e 79 3d 69 d8 47 7d e4  |.=....rG.y=i.G}.|  
  00000490  94 aa 94 cb ea 4a d6 48  8b e8 59 df a6 55 15 ae  |.....J.H..Y..U..|  
  000004a0  00 c8 12 00 00 00 00 00  ff c7 32 00 00 00 00 00  |..........2.....|  
  000004b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
  *                                                                               
  00000500  79 d3 d6 e6 07 f5 c2 44  a2 3c 23 8f 2a 3d f9 28  |y......D.<#.*=.(|  
  00000510  c5 50 d8 a1 b6 37 b2 4a  a9 eb a3 f0 32 5e 8a de  |.P...7.J....2^..|  
  00000520  00 c8 32 00 00 00 00 00  ff f7 3f 06 00 00 00 00  |..2.......?.....|  
  00000530  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|  
  *                                                                               
  00000600
  ```

  前16byte指定分区类型，例如，EFI系统分区的GUID固定为C12A7328-F81F-11D2-BA4B-00A0C93EC93B，如上图所示。

  offset 0x10 ->0x1f  表示该分区唯一的GUID，全局唯一；

  offset 0x20 ->0x27  ：起始LBA（00 08 00 00 00 00 00 00）， 0x800=2048，因此从2048 * 512Byte地址开始第一个LBA；

  offset 0x28 ->0x2f：末尾LBA

  offset 0x30 ->0x37：属性标签； 00 00 00 00 00 00 00 00，表示系统分区。0x2表示传统BIOS分区。

  offset 0x38 ->0x7f：分区名

  


  - | 分区类型 | GUID |
    | -------------------- | ------------------------------------------------------------ |
  |Unused entry|00000000-0000-0000-0000-000000000000|
  |MBR partition scheme|024DEE41-33E7-11D3-9D69-0008C781F39F|
  |EFI System partition|C12A7328-F81F-11D2-BA4B-00A0C93EC93B|
  |BIOS boot partition[e]|21686148-6449-6E6F-744E-656564454649|
  |||
  |Linux filesystem data[g]|0FC63DAF-8483-4772-8E79-3D69D8477DE4|
  |RAID partition|A19D880F-05FC-4D3B-A006-743F0F84911E|
  |Root partition (x86)[43][44]|44479540-F297-41B2-9AF7-D131D5F0458A|
  |Root partition (x86-64)[43][44]|4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709|
  |Root partition (32-bit ARM)[43][44]|69DAD710-2CE4-4E3C-B16C-21A1D49ABED3|
  |Root partition (64-bit ARM/AArch64)[43][44]|B921B045-1DF0-41C3-AF44-4C6F280D3FAE|
  |/boot partition[43][44]|BC13C2FF-59E6-4262-A352-B275FD6F7172|
  |Swap partition[43][44]|0657FD6D-A4AB-43C4-84E5-0933C84B4F4F|
  |Logical Volume Manager (LVM) partition|E6D6D379-F507-44C2-A23C-238F2A3DF928|
  |/home partition[43][44]|933AC7E1-2EB4-4F13-B844-0E14E2AEF915|
  |/srv (server data) partition[43][44]|3B8F8425-20E0-4F3B-907F-1A25A76F98E8|
  |Plain dm-crypt partition[47][48][49]|7FFEC5C9-2D00-49B7-8941-3EA10A5586B7|
  |LUKS partition[47][48][49][50]|CA7D7CCB-63ED-4C53-861C-1742536059CC|
  |Reserved|8DA63339-0007-60C0-C436-083AC8230908|

- LBA34以后，就根据entry的指示，来表示正式的partition内容了。比如，从entry中可以看到，partition 1格式为efi，从LBA 2048开始的。

## EFI分区

EFI系统分区是一个FAT32格式的物理分区

# kernel 启动

## kernel启动log

``` 
[    0.000000] Booting Linux on physical CPU 0x0300000000 [0x480fd020]
[    0.000000] Linux version 4.19.88-01318-g8ccc55e-dirty (root@localhost.localdomain) (gcc version 9.0.1 20190312 (Red Hat 9.0.1-0.10) (GCC)) #9 SMP Tue Oct 20 15:15:45 EDT 2020
[    0.000000] earlycon: pl11 at MMIO32 0x000000c614080000 (options '')
[    0.000000] bootconsole [pl11] enabled
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: EFI v2.70 by EDK II
[    0.000000] efi:  ACPI 2.0=0x3a420000  MEMATTR=0x3f008018  MEMRESERVE=0x3a71b018 
[    0.000000] ACPI: Early table checksum verification disabled
[    0.000000] ACPI: RSDP 0x000000003A420000 000024 (v02 HISI  )
[    0.000000] ACPI: XSDT 0x000000003A410000 00005C (v01 HISI   HIP09    00000000      01000013)
[    0.000000] ACPI: FACP 0x000000003A360000 000114 (v06 HISI   HIP09    00000000 HISI 20151124)
[    0.000000] ACPI: DSDT 0x000000003A320000 002306 (v02 HISI   HIP09    00000000 INTL 20181213)
[    0.000000] ACPI: GTDT 0x000000003A350000 00007C (v02 HISI   HIP09    00000000 HISI 20151124)
[    0.000000] ACPI: APIC 0x000000003A340000 000518 (v01 HISI   HIP09    00000000 HISI 20151124)
[    0.000000] ACPI: MCFG 0x000000003A330000 00003C (v01 HISI   HIP09    00000000 HISI 20151124)
[    0.000000] ACPI: IORT 0x000000003A310000 000690 (v00 HISI   HIP09    000000FF INTL 20181213)
[    0.000000] ACPI: PCCT 0x000000003A300000 00008A (v01 HISI   HIP09    00000000 HISI 20151124)
[    0.000000] ACPI: SSDT 0x000000003A2F0000 000430 (v02 HISI   HIP09    00000000 INTL 20181213)
[    0.000000] ACPI: NUMA: Failed to initialise from firmware
[    0.000000] NUMA: Faking a node at [mem 0x0000000000000000-0x00000821ffffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x821ffffe780-0x821ffffffff]
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000000000000-0x00000000ffffffff]
[    0.000000]   Normal   [mem 0x0000000100000000-0x00000821ffffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000000000-0x000000003a15ffff]
[    0.000000]   node   0: [mem 0x000000003a160000-0x000000003a1bffff]
[    0.000000]   node   0: [mem 0x000000003a1c0000-0x000000003a29ffff]
[    0.000000]   node   0: [mem 0x000000003a2a0000-0x000000003a2effff]
[    0.000000]   node   0: [mem 0x000000003a2f0000-0x000000003a36ffff]
[    0.000000]   node   0: [mem 0x000000003a370000-0x000000003a40ffff]
[    0.000000]   node   0: [mem 0x000000003a410000-0x000000003a42ffff]
[    0.000000]   node   0: [mem 0x000000003a430000-0x000000003a70ffff]
[    0.000000]   node   0: [mem 0x000000003a710000-0x000000003f6fffff]
[    0.000000]   node   0: [mem 0x000000003f700000-0x000000003f72ffff]
[    0.000000]   node   0: [mem 0x000000003f730000-0x000000003fbfffff]
[    0.000000]   node   0: [mem 0x0000082080000000-0x00000821ffffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x00000821ffffffff]
[    0.000000] psci: probing for conduit method from ACPI.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.1
[    0.000000] ACPI: SRAT not present
[    0.000000] random: get_random_bytes called from start_kernel+0xa0/0x488 with crng_init=0
[    0.000000] percpu: Embedded 3 pages/cpu s118872 r8192 d69544 u196608
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: enabling workaround for Mismatched cache type
[    0.000000] CPU features: detected: GIC system register CPU interface
[    0.000000] CPU features: detected: Virtualization Host Extensions
[    0.000000] CPU features: detected: Hardware dirty bit management
[    0.000000] CPU features: detected: Speculative Store Bypassing Safe (SSBS)
[    0.000000] alternatives: patching kernel code
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 114512
[    0.000000] Policy zone: Normal
[    0.000000] Kernel command line: rdinit=init earlycon=pl011,mmio32,0xc614080000 console=ttyAMA0,115200 acpi=force root=/dev/ram0 initrd=0x07000000,400M pcie_aspm=off
[    0.000000] software IO TLB: mapped [mem 0x3b000000-0x3f000000] (64MB)
[    0.000000] Memory: 6822272K/7335936K available (11132K kernel code, 2190K rwdata, 5184K rodata, 1984K init, 939K bss, 513664K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=16, Nodes=1
[    0.000000] ftrace: allocating 40890 entries in 10 pages
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=16.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=16
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: GIC: Using split EOI/Deactivate mode
[    0.000000] GICv3: Distributor has no Range Selector support
[    0.000000] GICv3: VLPI support, direct LPI support
[    0.000000] GICv3: CPU0: found redistributor 300000000 region 0:0x000000c640100000
[    0.000000] ACPI: SRAT not present
[    0.000000] ITS [mem 0xc660000000-0xc66001ffff]
[    0.000000] ITS@0x000000c660000000: Using ITS number 0
[    0.000000] ITS@0x000000c660000000: allocated 65536 Devices @821c0380000 (flat, esz 8, psz 16K, shr 1)
[    0.000000] ITS@0x000000c660000000: allocated 4096 Interrupt Collections @821c0340000 (flat, esz 16, psz 4K, shr 1)
[    0.000000] ITS@0x000000c660000000: Virtual CPUs too large, reduce ITS pages 512->256
[    0.000000] ITS@0x000000c660000000: allocated 32768 Virtual CPUs @821c0400000 (flat, esz 32, psz 4K, shr 1)
[    0.000000] GICv3: using LPI property table @0x00000821c0350000
[    0.000000] ITS: Using DirectLPI for VPE invalidation
[    0.000000] ITS: Enabling GICv4 support
[    0.000000] GICv3: CPU0: using allocated LPI pending table @0x00000821c0370000
[    0.000000] arch_timer: cp15 timer(s) running at 100.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x171024e7e0, max_idle_ns: 440795205315 ns
[    0.000039] sched_clock: 56 bits at 100MHz, resolution 10ns, wraps every 4398046511100ns
[    0.004193] Console: colour dummy device 80x25
[    0.006911] ACPI: Core revision 20180810
[    0.010766] Calibrating delay loop (skipped), value calculated using timer frequency.. 200.00 BogoMIPS (lpj=400000)
[    0.014058] pid_max: default: 32768 minimum: 301
[    0.016400] Security Framework initialized
[    0.055894] Dentry cache hash table entries: 1048576 (order: 7, 8388608 bytes)
[    0.078477] Inode-cache hash table entries: 524288 (order: 6, 4194304 bytes)
[    0.082332] Mount-cache hash table entries: 16384 (order: 1, 131072 bytes)
[    0.085374] Mountpoint-cache hash table entries: 16384 (order: 1, 131072 bytes)
[    0.103124] ACPI PPTT: No PPTT table found, cpu topology may be inaccurate
[    0.112937] ASID allocator initialised with 32768 entries
[    0.116131] rcu: Hierarchical SRCU implementation.
[    0.121494] Platform MSI: ITS@0xc660000000 domain created
[    0.123576] PCI/MSI: ITS@0xc660000000 domain created
[    0.125981] Remapping and enabling EFI services.
[    0.141890] smp: Bringing up secondary CPUs ...
[    0.157643] Detected VIPT I-cache on CPU1
[    0.158073] GICv3: CPU1: found redistributor 300000001 region 1:0x000000c640140000
[    0.158386] GICv3: CPU1: using allocated LPI pending table @0x00000821c0500000
[    0.158873] CPU1: Booted secondary processor 0x0300000001 [0x480fd020]
[    0.176726] Detected VIPT I-cache on CPU2
[    0.177222] GICv3: CPU2: found redistributor 300000200 region 2:0x000000c640200000
[    0.177539] GICv3: CPU2: using allocated LPI pending table @0x00000821c0510000
[    0.178025] CPU2: Booted secondary processor 0x0300000200 [0x480fd020]
[    0.195626] Detected VIPT I-cache on CPU3
[    0.196124] GICv3: CPU3: found redistributor 300000201 region 3:0x000000c640240000
[    0.196477] GICv3: CPU3: using allocated LPI pending table @0x00000821c0520000
[    0.196988] CPU3: Booted secondary processor 0x0300000201 [0x480fd020]
[    0.213663] Detected VIPT I-cache on CPU4
[    0.214139] GICv3: CPU4: found redistributor 300010000 region 4:0x000000c640300000
[    0.214405] GICv3: CPU4: using allocated LPI pending table @0x00000821c0530000
[    0.214868] CPU4: Booted secondary processor 0x0300010000 [0x480fd020]
[    0.232597] Detected VIPT I-cache on CPU5
[    0.232938] GICv3: CPU5: found redistributor 300010001 region 5:0x000000c640340000
[    0.233099] GICv3: CPU5: using allocated LPI pending table @0x00000821c0540000
[    0.233512] CPU5: Booted secondary processor 0x0300010001 [0x480fd020]
[    0.251365] Detected VIPT I-cache on CPU6
[    0.251750] GICv3: CPU6: found redistributor 300010200 region 6:0x000000c640400000
[    0.251967] GICv3: CPU6: using allocated LPI pending table @0x00000821c0550000
[    0.252387] CPU6: Booted secondary processor 0x0300010200 [0x480fd020]
[    0.271048] Detected VIPT I-cache on CPU7
[    0.271380] GICv3: CPU7: found redistributor 300010201 region 7:0x000000c640440000
[    0.271556] GICv3: CPU7: using allocated LPI pending table @0x00000821c0560000
[    0.271939] CPU7: Booted secondary processor 0x0300010201 [0x480fd020]
[    0.290534] Detected VIPT I-cache on CPU8
[    0.291143] GICv3: CPU8: found redistributor 300020000 region 8:0x000000c640500000
[    0.291474] GICv3: CPU8: using allocated LPI pending table @0x00000821c0570000
[    0.292049] CPU8: Booted secondary processor 0x0300020000 [0x480fd020]
[    0.311181] Detected VIPT I-cache on CPU9
[    0.311503] GICv3: CPU9: found redistributor 300020001 region 9:0x000000c640540000
[    0.311648] GICv3: CPU9: using allocated LPI pending table @0x00000821c0580000
[    0.312076] CPU9: Booted secondary processor 0x0300020001 [0x480fd020]
[    0.332028] Detected VIPT I-cache on CPU10
[    0.332498] GICv3: CPU10: found redistributor 300020200 region 10:0x000000c640600000
[    0.332699] GICv3: CPU10: using allocated LPI pending table @0x00000821c0590000
[    0.333123] CPU10: Booted secondary processor 0x0300020200 [0x480fd020]
[    0.352849] Detected VIPT I-cache on CPU11
[    0.353211] GICv3: CPU11: found redistributor 300020201 region 11:0x000000c640640000
[    0.353418] GICv3: CPU11: using allocated LPI pending table @0x00000821c05a0000
[    0.353841] CPU11: Booted secondary processor 0x0300020201 [0x480fd020]
[    0.373414] Detected VIPT I-cache on CPU12
[    0.374085] GICv3: CPU12: found redistributor 300030000 region 12:0x000000c640700000
[    0.374406] GICv3: CPU12: using allocated LPI pending table @0x00000821c05b0000
[    0.374953] CPU12: Booted secondary processor 0x0300030000 [0x480fd020]
[    0.394790] Detected VIPT I-cache on CPU13
[    0.395192] GICv3: CPU13: found redistributor 300030001 region 13:0x000000c640740000
[    0.395359] GICv3: CPU13: using allocated LPI pending table @0x00000821c05c0000
[    0.395726] CPU13: Booted secondary processor 0x0300030001 [0x480fd020]
[    0.416182] Detected VIPT I-cache on CPU14
[    0.416676] GICv3: CPU14: found redistributor 300030200 region 14:0x000000c640800000
[    0.416902] GICv3: CPU14: using allocated LPI pending table @0x00000821c05d0000
[    0.417300] CPU14: Booted secondary processor 0x0300030200 [0x480fd020]
[    0.437241] Detected VIPT I-cache on CPU15
[    0.437710] GICv3: CPU15: found redistributor 300030201 region 15:0x000000c640840000
[    0.437936] GICv3: CPU15: using allocated LPI pending table @0x00000821c05e0000
[    0.438359] CPU15: Booted secondary processor 0x0300030201 [0x480fd020]
[    0.442050] smp: Brought up 1 node, 16 CPUs
[    0.561294] SMP: Total of 16 processors activated.
[    0.562772] CPU features: detected: Privileged Access Never
[    0.564727] CPU features: detected: User Access Override
[    0.566300] CPU features: detected: Scalable Vector Extension
[    0.567970] CPU features: detected: Data cache clean to the PoU not required for I/D coherence
[    0.570757] CPU features: detected: Instruction cache invalidation not required for I/D coherence
[    0.573335] CPU features: detected: Stage-2 Force Write-Back
[    0.575066] CPU features: detected: CRC32 instructions
[    0.628630] SVE: maximum available vector length 32 bytes per vector
[    0.630507] SVE: default vector length 32 bytes per vector
[    0.632559] CPU: All CPU(s) started at EL2
[    0.683964] devtmpfs: initialized
[    0.722330] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.727579] futex hash table entries: 4096 (order: 2, 262144 bytes)
[    0.756514] pinctrl core: initialized pinctrl subsystem
[    0.805677] DMI not present or invalid.
[    0.859846] NET: Registered protocol family 16
[    0.923954] audit: initializing netlink subsys (disabled)
[    0.936086] audit: type=2000 audit(0.792:1): state=initialized audit_enabled=0 res=1
[    1.035615] cpuidle: using governor menu
[    1.038589] Detected 1 PCC Subspaces
[    1.046784] Registering PCC driver as Mailbox controller
[    1.062025] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    1.105627] DMA: preallocated 256 KiB pool for atomic allocations
[    1.122883] ACPI: bus type PCI registered
[    1.125211] acpiphp: ACPI Hot Plug PCI Controller Driver version: 0.5
[    1.155944] Serial: AMBA PL011 UART driver
[    2.123233] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    2.125384] HugeTLB registered 512 MiB page size, pre-allocated 0 pages
[    2.294366] ACPI: Added _OSI(Module Device)
[    2.296855] ACPI: Added _OSI(Processor Device)
[    2.298146] ACPI: Added _OSI(3.0 _SCP Extensions)
[    2.299599] ACPI: Added _OSI(Processor Aggregator Device)
[    2.301077] ACPI: Added _OSI(Linux-Dell-Video)
[    2.302354] ACPI: Added _OSI(Linux-Lenovo-NV-HDMI-Audio)
[    2.343950] ACPI: 2 ACPI AML tables successfully acquired and loaded
[    2.380181] ACPI: Interpreter enabled
[    2.381620] ACPI: Using GIC for interrupt routing
[    2.383987] ACPI: MCFG table detected, 1 entries
[    2.385945] ACPI: IORT: SMMU-v3[148000000] Mapped to Proximity domain 0
[    2.391062] ACPI: IORT: SMMU-v3[100000000] Mapped to Proximity domain 0
[    2.395663] ACPI: IORT: SMMU-v3[140000000] Mapped to Proximity domain 0
[    2.399638] ACPI: IORT: SMMU-v3[400148000000] Mapped to Proximity domain 2
[    2.405807] ACPI: IORT: SMMU-v3[400100000000] Mapped to Proximity domain 2
[    2.411289] ACPI: IORT: SMMU-v3[400140000000] Mapped to Proximity domain 2
[    3.323689] ARMH0011:00: ttyAMA0 at MMIO 0xc614080000 (irq = 5, base_baud = 0) is a SBSA
[    3.327714] console [ttyAMA0] enabled
[    3.327714] console [ttyAMA0] enabled
[    3.329322] bootconsole [pl11] disabled
[    3.329322] bootconsole [pl11] disabled
[    4.042884] vgaarb: loaded
[    4.056405] SCSI subsystem initialized
[    4.093167] ACPI: bus type USB registered
[    4.099034] usbcore: registered new interface driver usbfs
[    4.106638] usbcore: registered new interface driver hub
[    4.125263] usbcore: registered new device driver usb
[    4.213709] pps_core: LinuxPPS API ver. 1 registered
[    4.215702] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    4.217981] PTP clock support registered
[    4.227755] EDAC MC: Ver: 3.0.0
[    4.285460] Registered efivars operations
[    4.416055] clocksource: Switched to clocksource arch_sys_counter
[    5.114459] VFS: Disk quotas dquot_6.6.0
[    5.118334] VFS: Dquot-cache hash table entries: 8192 (order 0, 65536 bytes)
[    5.137129] pnp: PnP ACPI init
[    5.205976] pnp: PnP ACPI: found 0 devices
[    7.097111] NET: Registered protocol family 2
[    7.148039] tcp_listen_portaddr_hash hash table entries: 4096 (order: 0, 65536 bytes)
[    7.152047] TCP established hash table entries: 65536 (order: 3, 524288 bytes)
[    7.160644] TCP bind hash table entries: 65536 (order: 4, 1048576 bytes)
[    7.177277] TCP: Hash tables configured (established 65536 bind 65536)
[    7.181946] UDP hash table entries: 4096 (order: 1, 131072 bytes)
[    7.186093] UDP-Lite hash table entries: 4096 (order: 1, 131072 bytes)
[    7.194348] NET: Registered protocol family 1
[    7.217071] RPC: Registered named UNIX socket transport module.
[    7.218024] RPC: Registered udp transport module.
[    7.218676] RPC: Registered tcp transport module.
[    7.220157] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    7.225281] Unpacking initramfs...

--IPOP Time:2020/10/20_17:13:09--

--IPOP Time:2020/10/20_17:23:09--
[   91.331228] rcu: INFO: rcu_sched self-detected stall on CPU
[   91.332290] rcu: 	4-....: (5249 ticks this GP) idle=70e/1/0x4000000000000002 softirq=2300/2300 fqs=2625 
[   91.333515] rcu: 	 (t=5250 jiffies g=3977 q=5)
[   91.334239] NMI backtrace for cpu 4
[   91.334788] CPU: 4 PID: 1 Comm: swapper/0 Not tainted 4.19.88-01318-g8ccc55e-dirty #9
[   91.335832] Call trace:
[   91.336378]  dump_backtrace+0x0/0x178
[   91.336920]  show_stack+0x24/0x30
[   91.337463]  dump_stack+0xb4/0xdc
[   91.337975]  nmi_cpu_backtrace+0xa0/0xe8
[   91.338543]  nmi_trigger_cpumask_backtrace+0x18c/0x1b8
[   91.339297]  arch_trigger_cpumask_backtrace+0x30/0x3c
[   91.340036]  rcu_dump_cpu_stacks+0xf4/0x138
[   91.340669]  rcu_check_callbacks+0x6f0/0x8b8
[   91.341286]  update_process_times+0x34/0x78
[   91.341927]  tick_sched_handle.isra.0+0x44/0x68
[   91.342555]  tick_sched_timer+0x50/0xa0
[   91.343084]  __hrtimer_run_queues+0x120/0x350
[   91.343676]  hrtimer_interrupt+0x11c/0x2d0
[   91.344309]  arch_timer_handler_phys+0x3c/0x50
[   91.344965]  handle_percpu_devid_irq+0x90/0x230
[   91.345601]  generic_handle_irq+0x34/0x50
[   91.346156]  __handle_domain_irq+0x6c/0xc0
[   91.346724]  gic_handle_irq+0x6c/0x180
[   91.347239]  el1_irq+0xb8/0x140
[   91.347691]  __memset+0x170/0x188
[   91.348244]  free_initrd_mem+0x40/0x78
[   91.348782]  populate_rootfs+0x110/0x134
[   91.349329]  do_one_initcall+0x54/0x1e8
[   91.349852]  kernel_init_freeable+0x290/0x334
[   91.350472]  kernel_init+0x18/0x110
[   91.350972]  ret_from_fork+0x10/0x1c
[   92.066770] Freeing initrd memory: 409600K
[   92.118909] hw perfevents: enabled with armv8_pmuv3_0 PMU driver, 1 counters available
[   92.122691] kvm [1]: 16-bit VMID
[   92.125166] kvm [1]: GICv4 support disabled
[   92.125790] kvm [1]: vgic-v2@fe020000
[   92.126550] kvm [1]: GIC system register CPU interface enabled
[   92.145301] kvm [1]: vgic interrupt IRQ1
[   92.162403] kvm [1]: VHE mode initialized successfully
[   92.421796] Initialise system trusted keyrings
[   92.509621] workingset: timestamp_bits=44 max_order=17 bucket_order=0
[   93.610514] NFS: Registering the id_resolver key type
[   93.614599] Key type id_resolver registered
[   93.618321] Key type id_legacy registered
[   93.622534] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[   93.693257] 9p: Installing v9fs 9p2000 file system support
[   94.081037] Key type asymmetric registered
[   94.081951] Asymmetric key parser 'x509' registered
[   94.085191] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 244)
[   94.086279] io scheduler noop registered
[   94.086875] io scheduler deadline registered
[   94.094376] io scheduler cfq registered (default)
[   94.096323] io scheduler mq-deadline registered
[   94.096994] io scheduler kyber registered
[   96.469288] EINJ: EINJ table not found.
[   96.498065] ACPI GTDT: found 1 SBSA generic Watchdog(s).
[   98.590691] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[   99.114576] SuperH (H)SCI(F) driver initialized
[   99.194765] msm_serial: driver initialized
[   99.290132] ACPI PPTT: No PPTT table found, cache topology may be inaccurate
[   99.302475] ACPI PPTT: No PPTT table found, cache topology may be inaccurate
[   99.305881] cacheinfo: Unable to detect cache hierarchy for CPU 0
[   99.852992] loop: module loaded
[  101.056778] libphy: Fixed MDIO Bus: probed
[  101.090785] tun: Universal TUN/TAP device driver, 1.6
[  101.355071] libphy: Hisilicon MII Bus: probed
[  101.438313] VFIO - User Level meta-driver version: 0.3
[  101.738287] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[  101.741299] ehci-pci: EHCI PCI platform driver
[  101.750935] ehci-orion: EHCI orion driver
[  102.650426] i2c /dev entries driver
[  103.902707] Synopsys Designware Multimedia Card Interface Driver
[  104.376700] ledtrig-cpu: registered to indicate activity on CPUs
[  104.628743] usbcore: registered new interface driver usbhid
[  104.629883] usbhid: USB HID core driver
[  105.613698] NET: Registered protocol family 10
[  105.782194] Segment Routing with IPv6
[  105.790772] NET: Registered protocol family 17
[  105.798263] bridge: filtering via arp/ip/ip6tables is no longer available by default. Update your scripts to load br_netfilter if you need this.
[  105.813358] 9pnet: Installing 9P2000 support
[  105.821424] Key type dns_resolver registered
[  105.986877] registered taskstats version 1
[  105.990612] Loading compiled-in X.509 certificates
[  106.056403] hctosys: unable to open rtc device (rtc0)
[  106.153850] Freeing unused kernel memory: 1984K
[  106.196598] Run init as init process
Starting OpenBSD Secure Shell server: sshd
  generating ssh RSA key...
[  107.557731] random: ssh-keygen: uninitialized urandom read (32 bytes read)
  generating ssh ECDSA key...
[  112.608343] random: ssh-keygen: uninitialized urandom read (32 bytes read)
  generating ssh DSA key...
[  112.808616] random: ssh-keygen: uninitialized urandom read (32 bytes read)
  generating ssh ED25519 key...
[  115.485452] random: ssh-keygen: uninitialized urandom read (32 bytes read)
[  115.896291] random: sshd: uninitialized urandom read (32 bytes read)
```

### secondary核的启动

``` c
// secondary_start_kernel中打印，
[    0.141890] smp: Bringing up secondary CPUs ...
[    0.157643] Detected VIPT I-cache on CPU1
[    0.158073] GICv3: CPU1: found redistributor 300000001 region 1:0x000000c640140000
[    0.158386] GICv3: CPU1: using allocated LPI pending table @0x00000821c0500000
[    0.158873] CPU1: Booted secondary processor 0x0300000001 [0x480fd020]
......
[    0.437241] Detected VIPT I-cache on CPU15
[    0.437710] GICv3: CPU15: found redistributor 300030201 region 15:0x000000c640840000
[    0.437936] GICv3: CPU15: using allocated LPI pending table @0x00000821c05e0000
[    0.438359] CPU15: Booted secondary processor 0x0300030201 [0x480fd020]
[    0.442050] smp: Brought up 1 node, 16 CPUs
// smp_cpus_done()中打印下面一行: 设置
[    0.561294] SMP: Total of 16 processors activated.
[    0.562772] CPU features: detected: Privileged Access Never
```

单独看一下这个函数。

``` c
void __init smp_cpus_done(unsigned int max_cpus)
{
    pr_info("SMP: Total of %d processors activated.\n", num_online_cpus());
    setup_cpu_features(); // 打印system的所有capabilities。详见《cpufeature的检测》
    hyp_mode_check(); // check是否使能EL2的VHE功能。
    apply_alternatives_all(); //
    mark_linear_text_alias_ro();
}
```



# 源码研读

## cpufeature的检测

有3个地方会检测，分别对应3种不同的检测范围。分别是：

1.  **SCOPE_BOOT_CPU**

   只检测primary boot CPU的特性。在启动其他核之前就要先做。

   ``` c
    CPU features: enabling workaround for Mismatched cache type
    CPU features: detected: GIC system register CPU interface
    CPU features: detected: Virtualization Host Extensions
    CPU features: detected: Hardware dirty bit management
    CPU features: detected: Speculative Store Bypassing Safe (SSBS)
   ```

2.  **SCOPE_SYSTEM**

   启动secondary CPUs后进行检测。检测所有CPU，只有全部匹配时认为该特性使能。一般用于检测ID寄存器中的特性。

   ​		在smp_cpus_done -> setup_cpu_features -> setup_system_capabilities -> update_cpu_capabilities -> cpus_set_cap中。通过检测`cpu_hwcaps_ptrs`矩阵中对应的特性成员是否存在，若存在则将`cpu_hwcaps`中对应index的bit置位。

   ​		bit map全局数据结构`cpu_hwcap`中，每个bit代码一个特性，设置这个bitmap的函数只有`cpus_set_cap`。

   ``` c
   CPU features: detected: Privileged Access Never
   CPU features: detected: User Access Override
   CPU features: detected: Scalable Vector Extension
   CPU features: detected: Data cache clean to the PoU not required for I/D coherence
   CPU features: detected: Instruction cache invalidation not required for I/D coherence
   CPU features: detected: Stage-2 Force Write-Back
   CPU features: detected: CRC32 instructions
   ```

3. **LOCAL_CPU**

   检测所有CPU，只要有1个匹配就认为该特性使能。



## 内存空间的初始化

start_kernel()->mm_init()->mem_init()

从vmlinux.lds.S文件中获取各个段地址，比如`(_text, _etext), (__init_begin, __init_end), (_sdata, _edata)`等。

4.16内核之后就不会把这些信息打印出来了。

## 内核页表的初始化

start_kernel -> setup_arch（架构相关模块的初始化) -> paging_init -> map_mem->create_mapping

1. 会映射从.stext开始到.init.begin结束的段地址，包含内核的text，data，rodata等段。
2. 会将mm_struct->pgd字段赋值，并copy到ttbr_e1去。
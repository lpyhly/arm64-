# 安全SB指令

## SB是什么？

SB： speculation barrier， 是一个控制投机的barrier指令。 ARMv8.0新特性，v8.5强制实现。

在SB完成之前，任何程序序排在SB之后的指令：

1. 不能作为控制流/数据投机执行的结果被旁路观察到；
2. 当预测一条指令会产生异常但没产生异常时，可以投机执行。

对SB指令的投机执行：

1. 不能是控制流投机的结果     
2. 不能是数据投机的结果
3. 可以是预测可能生成异常的指令但没生成异常的结果

当满足如下条件时，SB指令完成：

1. It is known that it is not speculative
2. 所有程序序在SB指令之前的根据指令预测产生的数据值已经确认后。

### SB对应的ID寄存器

ID_AA64ISAR1_EL1.SB， bit[39:36]

![image-20220908091608783](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20220908091608783.png)

0b0000,  SB instruction is not implemented

0b0001,  SB instruction is implemented

### 为什么要有SB

为了消除spectre和meltdown影响引入的，通过这个指令，显式告诉硬件，不允许后面的指令投机执行。作用有些类似isb + dsb，即不允许指令 & 数据流的投机执行，但对性能的影响上应该是弱于后者的，因为只要硬件确认了后续指令不是投机执行的，就可以继续走了，而不需要等待前面的指令全部commit。

SB后面的指令都不能投机执行，这里包括后面指令对cache也不能造成影响。除了一个例外，就是猜测后面的指令可能会产生异常，但这个不太懂。

### 在linux中的使用

https://patchwork.kernel.org/project/linux-arm-kernel/patch/1543322517-470-2-git-send-email-will.deacon@arm.com/

定义了一个sb()的接口封装，与mb,rmb,wmb等指令在同一个文件中声明。（include/asm/barrier.h)

这个函数在支持SB特性时，表现为  `sb + nop` ;不支持SB特性时，表现为 `dsb nsh + isb`。

1. 在set_fs()函数中，替代了曾经的dsb+isb序列。（最新版本的内核中已经删除了set_fs函数）
2. 在eret后添加sb指令，防止某些cpu投机执行eret之后的指令，因为eret后返回到了低权限state，因此投机执行可能会导致侧信道攻击。添加方式举例如下：

``` c
diff --git a/arch/arm64/kernel/entry.S b/arch/arm64/kernel/entry.S
index 039144ecbcb2..a7fc77ab4a0a 100644
--- a/arch/arm64/kernel/entry.S
+++ b/arch/arm64/kernel/entry.S
@@ -363,6 +363,7 @@  alternative_insn eret, nop, ARM64_UNMAP_KERNEL_AT_EL0
 	.else
 	eret
 	.endif
+	sb
 	.endm
```
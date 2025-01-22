#各种 trap exit处理流程

[TOC]

# WFx

![image-20201210110647841](D:\at_work\Documents\我的总结文档\images\image-20201210110647841.png)

## WFE

​		round-robin不断轮询同一个vm的其他vcpu，看哪个vcpu上有唤醒事件
​		把CPU让给同一虚拟机的其他CPU(yield_to函数)，必须是之前被调度走的(vcpu->preempted=true)且runable的。
​		return 1，回到guest

## WFI

​		kvm_vcpu_block
​			如果只有这一个线程，且未超时，则一直循环查中断是否到达
​			否则，只尝试一小段时间，看有没有wakeup事件；然后schedule(TASK_INTERRUPTIBLE)循环等待
​				每次block的时间会动态伸缩，开始时block时间比较短
​			gic v4使能doorbell ？
​			中断到来后，置位KVM_REQ_UNHALT
​			gic v4关闭doorbell ？
​			然后schedule调度，标志本进程可以被抢占

# msr/mrs

![image-20201210110728832](D:\at_work\Documents\我的总结文档\images\image-20201210110728832.png)

# stage 2缺页异常处理

​	只是告诉vm我分配给你了好大好大的内存，让vm的va->ipa流程畅通无阻，但实际上只是空头支票，在stage 2翻译的时候会缺页，这个时候才真正拿该qemu进程的hva去到host申请物理内存，建立stage 2页表。后面vm再访问这段地址，就不会stage 2缺页了，而是正常做了。

![image-20201210110758687](D:\at_work\Documents\我的总结文档\images\image-20201210110758687.png)

## stage 2 页表的创建

当stage 1缺页异常后（无论是IABT还是DABT），会trap出来到`kvm_handle_guest_abort`，首先将ipa转换成hva，然后在`user_mem_abort`中去分配内存来建立stage 2页表。

### stage 2页表的默认属性

``` c
#define PAGE_S2         __pgprot(_PROT_DEFAULT | PAGE_S2_MEMATTR(NORMAL) | PTE_S2_RDONLY | PAGE_S2_XN) 
#define PAGE_S2_DEVICE      __pgprot(_PROT_DEFAULT | PAGE_S2_MEMATTR(DEVICE_nGnRE) | PTE_S2_RDONLY | PAGE_S2_XN)

static int user_mem_abort(struct kvm_vcpu *vcpu, phys_addr_t fault_ipa,
              struct kvm_memory_slot *memslot, unsigned long hva,
              unsigned long fault_status)
{
    gfn_t gfn = fault_ipa >> PAGE_SHIFT;
    ...... 
    pgprot_t mem_type = PAGE_S2; // normal空间的默认属性。
    ......
    if (kvm_is_device_pfn(pfn)) {
        mem_type = PAGE_S2_DEVICE;  // device 空间的默认属性。
        flags |= KVM_S2PTE_FLAG_IS_IOMAP;
    }
    ......
    pte_t new_pte = kvm_pfn_pte(pfn, mem_type); // 将mem_type写成pte格式。
}
```

从上述代码看，stage 2页表默认属性为：

-  s2uxn = 10， 在stage 2不允许在EL1或EL0执行

  - PTE_S2_XN=10，表示：

    当EL2使用**AArch64**时:   XN[1:0]

    00  在stage 2允许在EL1或EL0执行

    01  在stage 2不允许在EL1执行，允许在EL0执行

    10  在stage 2不允许在EL1或EL0执行

    11  在stage 2允许在EL1执行，不允许在EL0执行

- rdonly，HAP[2:1] = 0，只给read权限。
  - 若当前为写操作，在`user_mem_abort`中会进行判断，单独调用`kvm_s2pte_mkwrite`来修改pte的RDONLY为RDWR。
  - 若当前操作需要执行权限，在`user_mem_abort`中会进行判断，单独调用`kvm_s2pte_mkexec`来修改pte的xn为00，在stage 2允许在EL1或EL0执行。
- 普通memattr为NORMAL，device属性为nGnRE
- 其他默认属性先不care。

### s2uxn特性


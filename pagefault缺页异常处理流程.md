# pagefault缺页异常处理流程

[TOC]

思维导图： mmap一个文件（从file->va的建立） –> 访问该文件发生pagefault（从va获取pa并建立页表）的流程分析。

# 1      怎么获取va

## 1.1    内核态

内核va/pa是固定的，va就是0 – 1G地址空间，因此TTBR1_EL1的值对于所有进程都是一样的（不考虑arm某个会在el0修改ttbr1_el1的值的新特性）。内核页表也是在启动的时候就建好了的（start_kernel中），而内核启动的时候默认从某个固定地址启动（可以配置），默认这个地址就存放着内核的第一个执行函数地址。

因此，只要知道了内核的va，在任何时候都可以直接对应到pa。

## 1.2    用户态

32bit中，用户态的每个进程都独享一段4G的地址空间，其中高1G统一给内核使用，剩下3G供进程自己的用户态用，包含stack段，data段，text段，bss段，heap段等。整个进程的地址空间对应一个struct mm_struct（task->mm），每个段的一块连续地址对应一个VMA（由struct vm_area_struct标识），比如memory mapping段包含mmap的文件以及动态链接库文件，那么每一个文件就对应一个vma。

## 1.3    vma

``` c
 const struct file_operations xfs_file_operations = {
     .llseek     = xfs_file_llseek,
     .read_iter  = xfs_file_read_iter,
     .write_iter = xfs_file_write_iter,
     .splice_read    = generic_file_splice_read,
     .splice_write   = iter_file_splice_write,
     .iopoll     = iomap_dio_iopoll,
     .unlocked_ioctl = xfs_file_ioctl,
 #ifdef CONFIG_COMPAT
     .compat_ioctl   = xfs_file_compat_ioctl,
 #endif
     .mmap       = xfs_file_mmap, // mmap对应。
     .mmap_supported_flags = MAP_SYNC,
     .open       = xfs_file_open,
     .release    = xfs_file_release,
     .fsync      = xfs_file_fsync,
     .get_unmapped_area = thp_get_unmapped_area,
     .fallocate  = xfs_file_fallocate,
     .fadvise    = xfs_file_fadvise,
     .remap_file_range = xfs_file_remap_range,
 };
```

当用户态mmap一个文件时，vm_area_struct的成员vm_file可以指向struct file结构体，然后通过struct address_space指向 struct inode，也就能找到真正的文件了。注意，这里struct file是和进程相关的，因此每个进程需要使用磁盘文件时，都需要先open一下，就是将代表文件物理存储信息的inode与进程相关联，媒介就是struct file结构体。

因此，通过VMA就可以建立起va到真实访问对象的映射，比如malloc/mmap等接口。此时，还没有真正将文件内容放在ddr上，也没有建立页表，因为还没有分配PA。

``` c
vma->vm_ops = &xfs_file_vm_ops;
```



## 1.4    各个vma段的区分

### 1.4.1       why区分？

​    可以供mmu去分配不同的执行权限给不同的地址段，以及dcache、icache的区分，还有annoymous、file-based的区分。

### 1.4.2       how 区分？

首先区分代码段和数据段，其中代码段只有`text`段，应该会进入到ICache的，怎么进去的还不清楚。

然后区分`file-based`和`annoymous`。前者是会体现在编译完成后的二进制文件大小中的，而后者不会。代码段一定是file-based。数据段中，对于编译后执行前就能确定的东西则为file-based的，可以直接扔到data段中，比如静态变量，常量，初始化的全局变量（data段是rw的，所以可以在运行后进行修改）；对于执行后才知道值的，就是annoymous，分别放在stack段，heap段，bss段中。

annoymous中，stack和heap是共用一段地址空间的，stack从高往低写，heap从低往高写。bss段主要存放的是mmap的文件、没有赋初值的全局变量（可以先不分配内存，执行后再分配）。

``` c
// 两种全局变量定义方式，前者的elf会大很多。
int ar[300000] = {1, 2, 3, 4, 5, 6 };
int ar[300000];
```

# 2      怎么获取pa

### 2.1     pagefault的类别

4类：（from 宋宝华）

1.   heap区域的cow，比如malloc后的写。是minor pagefault，只有vma信息，还没有页表，需要重新申请一页内存，并修改权限为R+W，不涉及IO操作。
2.   地址区域非法：访问落在非法区，没有对应的vma。
3.   protection fault：写没有写权限的区域，执行没有执行权限的区域等。
4.   major pagefault：页表被交换到swap或磁盘中，或者要访问的数据本来就在磁盘中。是major pagefault，涉及IO操作。



## 2.2    pagefault后会发生什么？

通过va访问时，会发生`pagefault`，因为找不到页表。

根据地址空间，pagefault可以分为3类：

1）内核态发生pagefault，va是内核态地址。

2）内核态发生pagefault，va是用户态地址。比如copy_from_user后，内核态使用用户态未建立过页表的内容。

3）用户态发生pagefault，va是用户态地址。进入handle_mm_fault处理。

对于case 1/2，硬件走el1_sync处理；对于case 3，走el0_sync处理。其中，

case 1失败会发生kernel oops；

case 2若一通走下来还是失败，也可以走fixup流程返回，就不会发生kernel oops了；fixup对于每个va都有对应的fixup代码入口，可以在失败时发生跳转过去。若没有fixup，则依然会kernel oops。

case 3 若失败，只会在用户态报错，不会影响内核态。

## 2.3    major & minor pagefault

然后，会进行权限检查，

1. 若读了没有读权限的地址，则触发错误，发出SIGSEGV信号杀死进程；

2. 若写了没有写权限的地址，判断是否为COW的地址；若是则触发do_wp_page，若不是则返回异常。

3. 若权限检查通过，则为正常的缺页异常，继续判断va对应的是mmap空间？file？swap？从而进入不同的函数处理。

此时，有两种大类，

1. minor pagefault：物理页表已经被load到ddr了，这种损耗就很小，因为不需要访问磁盘；比如：

   -  多核进程同时发生pagefault，其他进程已经做了load页表的事情了；
   -  匿名页

2. major pagefault：需要重新load页表的，包含：

   - 之前没有建立过页表的：
   - 之前建立了页表，但是目前页表在swap空间；

## 2.4    pagefault处理流程

### 2.4.1       pagefault流程人话介绍

**S1：**根据va在task->mm->mmap中获取到对应的vma（struct *vm_area_struct），进行权限检查，若执行了data/stack/heap，或者写了text段等，则为权限检查失败报错。若检查通过，则将va，vma，mm_flags传给handle_mm_fault处理。



**S2：**`handle_mm_fault`中，先建立pte以上的页表们([3])。然后调用`handle_pte_fault`区分不同情况创建或修改pte页表：

- 若pte未分配且file-based，则调用`do_fault()`然后对应到具体fs的fault()函数处理；

- 若pte未分配且anonymous的，则调用`do_anonymous_page`直接分配；

- 若pte已分配但在swap中，调用`do_swap_page`；

- 若pte已分配但需要numa-balanced，则调用`do_numa_page`处理；

- 若pte已分配但没有写权限，而flags中表示本次是写操作，则调用`do_wp_page`进行写时复制操作；

  

tips：

[1] 是否为file-based是通过判断vma->vm_ops是否为NULL来判断的，若为file-based则vm_ops应该映射到具体文件系统指针；

[2] mm_flags表示当前操作类型，是在do_page_fault函数中通过esr传递的fault类型判断的，包含是否为user态，是否需要执行，是否需要写等。会构造vm_flags和mm_flags，前者仅用于__do_page_fault中和vma->vm_flags进行比较，后者才会真正传递到handle_mm_fault中。

``` c
（arch/arm64/mm/fault.c）
static int __kprobes do_page_fault(unsigned long addr, unsigned int esr,
                   struct pt_regs *regs)
{
    const struct fault_info *inf;
    struct mm_struct *mm = current->mm;
    vm_fault_t fault, major = 0;
    unsigned long vm_flags = VM_ACCESS_FLAGS;
    unsigned int mm_flags = FAULT_FLAG_DEFAULT;
	...

    if (user_mode(regs))
        mm_flags |= FAULT_FLAG_USER;

    if (is_el0_instruction_abort(esr)) {
        vm_flags = VM_EXEC;
        mm_flags |= FAULT_FLAG_INSTRUCTION;
    } else if (is_write_abort(esr)) {
        vm_flags = VM_WRITE;
        mm_flags |= FAULT_FLAG_WRITE;
    }
```

[3] handle_mm_fault-> 先建立高阶页表，通过alloc_page()获取一个page页，包含pgd，p4d（=pgd），pud，pmd。其中，pgd是从vma的上层mm_struct mm->pgd中获取的，也就是说task->mm会保存一级页表pgd。

 

### 2.4.2   do_read_fault

(1)  调用do_read_fault ->vm_ops->map_pages这个钩子函数可以关联到具体的文件系统，一般用filemap_map_pages实现。

(2)  调用do_read_fault -> __do_fault()进而调用vm_ops.fault()函数来完成页面的申请。

(3) 获取当前页表项pte。

(4) 将新生成的PTE entry设置到硬件页表项中。

``` c
static vm_fault_t do_read_fault(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    vm_fault_t ret = 0;

    /*
     * Let's call ->map_pages() first and use ->fault() as fallback
     * if page by the offset is not ready to be mapped (cold cache or
     * something).
     */
    if (vma->vm_ops->map_pages && fault_around_bytes >> PAGE_SHIFT > 1) {
        ret = do_fault_around(vmf); // 围绕在缺页异常地址周围提前映射尽可能多的页面，
        		//提前建立进程地址空间和page cache的映射关系有利于减少发生缺页终端的次数。
        		//这里只是和现存的page cache提前建立映射关系，而不会去创建page cache。
        		// 会调用vma->vm_ops->map_pages的。
        if (ret) // ret ！=0表示pagefault已经解决了，可以返回啦~~
            return ret;
    }

    ret = __do_fault(vmf);
    if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
        return ret;

    ret |= finish_fault(vmf);
    unlock_page(vmf->page);
    if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
        put_page(vmf->page);
    return ret;
}
```



### 2.4.3      do_shared_fault

(1)读取文件到fault_page中

(2)使页面变为可写页面(与do_read_page()函数不同之处)

(3)获取fault_page对应的pte

(4)将新生成的PTE entry设置到硬件页表中

(5)将page标记为dirty(与do_read_page()函数不同之处)

(6)通过balance_dirty_pages_ratelimited()来平衡并回写一部分脏页。



###　2.4.4　do_wp_page

``` c
 static vm_fault_t wp_page_copy(struct vm_fault *vmf)
 {
     struct vm_area_struct *vma = vmf->vma;
     struct mm_struct *mm = vma->vm_mm;
     struct page *old_page = vmf->page;
     struct page *new_page = NULL;
     pte_t entry;
     int page_copied = 0;
     struct mmu_notifier_range range;

     if (unlikely(anon_vma_prepare(vma))) // 关联一个anon_vma实例到vma
         goto oom;

     if (is_zero_pfn(pte_pfn(vmf->orig_pte))) { // 判断原来的页表项映射的页是0页，就分配高端可移动的页并用0初始化，应该是对cow的页进行先读后写的操作时。
         new_page = alloc_zeroed_user_highpage_movable(vma,
                                   vmf->address);
         if (!new_page)
             goto oom;
     } else {
         new_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vma, 
                 vmf->address); // 如果不是0页，分配高端可移动的页，
         if (!new_page)
             goto oom;

         if (!cow_user_page(new_page, old_page, vmf)) { // 然后将原来的页拷贝到新页
             /*
              * COW failed, if the fault was solved by other,
              * it's fine. If not, userspace would re-fault on
              * the same address and we will handle the fault
              * from the second attempt.
              */
             put_page(new_page);
             if (old_page)
                 put_page(old_page);
             return 0;
         }
     }

     if (mem_cgroup_charge(new_page, mm, GFP_KERNEL))
         goto oom_free_new;
     cgroup_throttle_swaprate(new_page, GFP_KERNEL);

     __SetPageUptodate(new_page); // 设置新的页标识位为PageUptodate，表示页中包含有效数据。
      mmu_notifier_range_init(&range, MMU_NOTIFY_CLEAR, 0, vma, mm,
             vmf->address & PAGE_MASK,
             (vmf->address & PAGE_MASK) + PAGE_SIZE);
 mmu_notifier_invalidate_range_start(&range);

 /*
  * Re-check the pte - we dropped the lock
  */
 vmf->pte = pte_offset_map_lock(mm, vmf->pmd, vmf->address, &vmf->ptl);
 if (likely(pte_same(*vmf->pte, vmf->orig_pte))) { // 发生缺页异常时获得页表项和现在锁住之后获得页表项内容相同的情况，主要情况。
     if (old_page) {
         if (!PageAnon(old_page)) {
             dec_mm_counter_fast(mm,
                     mm_counter_file(old_page));
             inc_mm_counter_fast(mm, MM_ANONPAGES);
         }
     } else {
         inc_mm_counter_fast(mm, MM_ANONPAGES);
     }
     flush_cache_page(vma, vmf->address, pte_pfn(vmf->orig_pte));
     entry = mk_pte(new_page, vma->vm_page_prot);
     entry = pte_sw_mkyoung(entry);
     entry = maybe_mkwrite(pte_mkdirty(entry), vma); // 设置页表项值属性为脏和可写，此时还没有正式写入内存中（如果vma有可写属性，这个时候将页表项修改为了可写，fork的时候修改为只读这个地方修改了回来）
     /*
      * Clear the pte entry and flush it first, before updating the
      * pte with the new entry. This will avoid a race condition
      * seen in the presence of one thread doing SMC and another
      * thread doing COW.
      */
     ptep_clear_flush_notify(vma, vmf->address, vmf->pte); // 将页表项原有的值清除，然后刷新地址发生缺页地址对应的tlb（这一行操作很重要）
     page_add_new_anon_rmap(new_page, vma, vmf->address, false);
     lru_cache_add_inactive_or_unevictable(new_page, vma);
     /*
      * We call the notify macro here because, when using secondary
      * mmu page tables (such as kvm shadow page tables), we want the
      * new page to be mapped directly into the secondary page table.
      */
     set_pte_at_notify(mm, vmf->address, vmf->pte, entry); // 将构建好的页表项值写入到页表项条目中，这个时候页表项修改才会生效。
     update_mmu_cache(vma, vmf->address, vmf->pte);
     ......
 }
 }

```





### 2.4.5  关于不同pagesize路径不同的研究

1.  #### 先check mmap传入的系统调用参数。

   ​	strace -f -o 1.txt taskset -c 1 ./lat_pagefault -N1 /tmp/haha

   ​	发现两边传入的参数是一致的，都是PROT_READ/MAP_SHARED

   

2.  #### check 内核态do_mmap处理的函数参数，通过tracepoint。

   ​	也是一致的。

   

3.  #### check做mmap_region的参数。

``` c
perf probe -a 'mmap_region vm_flags'
perf record -e probe:mmap_region -R -C 1 sleep 1
```

​	4k： 全是0xd1 = VM_READ | VM_MAYREAD | VM_MAYEXEC | VM_MAYSHARE

​	64k： 也全是0xd1

4. #### check做pagefault时可能走不同分支的地方。

|      | perf  probe -a 'do_fault:35 vmf->flags'                      |
| ---- | ------------------------------------------------------------ |
|      | perf record -e  probe:do_fault -R ./lat_pagefault  /tmp/haha |

``` c
 if (!vma->vm_ops->fault) {
     ...
} else if (!(vmf->flags & FAULT_FLAG_WRITE))
     ret = do_read_fault(vmf);
 else if (!(vma->vm_flags & VM_SHARED))
     ret = do_cow_fault(vmf);
 else
     ret = do_shared_fault(vmf);
```

4k:   vmf->flags 0x54

64k：vmf->flags 0x54 = FAULT_FLAG_ALLOW_RETRY |  FAULT_FLAG_KILLABLE | FAULT_FLAG_USER

​	所以都走第一个分支 ->do_read_fault

回到do_read_fault的代码，可以看到先进行map_pages，若不行，才调用fault。

``` c
static vm_fault_t do_read_fault(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    vm_fault_t ret = 0;

    /*
     * Let's call ->map_pages() first and use ->fault() as fallback
     * if page by the offset is not ready to be mapped (cold cache or
     * something).
     */
    if (vma->vm_ops->map_pages && fault_around_bytes >> PAGE_SHIFT > 1) {
        ret = do_fault_around(vmf);
        if (ret)
            return ret;
    }

    ret = __do_fault(vmf);
```

__do_fault-> {vma->vm_ops->fault()}对于tmpfs调用的是shmem_fault，实际上64k上看到的确进入了这个函数。

``` c
 static const struct vm_operations_struct shmem_vm_ops = {
     .fault      = shmem_fault,
     .map_pages  = filemap_map_pages,
 #ifdef CONFIG_NUMA
     .set_policy     = shmem_set_policy,
     .get_policy     = shmem_get_policy,
 #endif
 };
```

逐行trace，发现64k/4k都进了do_read_fault, 其中64k全部去了__do_fault（一个filemap_map_pages都没有，也是比较奇怪）， 4k全部去了filemap_map_pages。

``` c
static vm_fault_t do_read_fault(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    vm_fault_t ret = 0;

    /*
     * Let's call ->map_pages() first and use ->fault() as fallback
     * if page by the offset is not ready to be mapped (cold cache or
     * something).
     */
    if (vma->vm_ops->map_pages && fault_around_bytes >> PAGE_SHIFT > 1) {
        ret = do_fault_around(vmf);
        if (ret)
            return ret;
    }

    ret = __do_fault(vmf);
    if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
        return ret;

    ret |= finish_fault(vmf);
    unlock_page(vmf->page);
    if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
        put_page(vmf->page);
    return ret;
}
```



怀疑点：

1. **64k下，vm_ops->map_pages函数根本没有映射；（已排除)**

    ---- perf probe -a 'do_fault:35 vma->vm_ops->map_pages'

    ---- perf record -e probe:do_fault -g ./lat_pagefault /tmp/haha

   ---- 对应system.map查看，确实是存在的，而且对应的是filemap_map_pages函数。

2. **64k下，vm_ops->map_pages返回false；**

   --- 这里没有实际调用函数，而只是判断函数是否存在。

   ----- 函数指针是存在的，不为空，可以继续判断。

3. **fault_around_bytes     >> PAGE_SHIFT <1**

   --- 实际check，发现PAGE_SHIFT使用的是CONFIG_ARM64_PAGE_SHIFT这个宏，由config定义的。在4k下是12， 在64k下是16, fault_around_bytes=0x10000，***所以这个值在4k下>1，64k下=1***

4. **do_fault_around返回false。**

   -----函数的作用是：将附近的64k全都filemap_map_pages一下。

   ---- 实际check，64k页表下根本没有调用过这个函数。

 

- 继续研究.map_files和.fault函数的差别，包括作用和性能的差别。

  - map_files: 处理minor fault，即page已经在ddr中了，只需要建立页表映射即可。

  - fault：page可能不在ddr中，需要先alloc page，然后再进行映射。

- 关于 fault_around_bytes >> PAGE_SHIFT > 1 这个语句的来源。

- - **fault_around_bytes当前默认是64k，是为了一次映射多个已经在ddr中的page的，当PAGESIZE为4k，则可以映射16个page，当PAGESIZE为64k，只会映射一个页，也就不需要进入do_fault_around了。**

  - 这个值可以在**debugfs中修改:  

    ​		`/sys/kernel/debug/fault_around_bytes`

- 实测：1620 64k页表中：为64k时，8m file的fault时间为0.58us，改成256k，8m file的fault时间降低为0.29us。

# 3. 怎么构造pagefault？

## lat_pagefault源码解析

1. 测试一个 256M 映射在tmpfs的文件。
2. 测试一个 8M 映射在tmpfs的文件。

``` shell
case 1（benchmark_mmap）:  mmap + munmap
case 2（benchmark）:       mmap + （每次访问1byte，每隔1 pagesize访问一次） + munmap

pagefault的用时 = (case 2的用时 - case 1的用时)/访问的次数
```

源码：

``` c
void
benchmark(iter_t iterations, void* cookie)
{
        int     i;
        int     sum = 0;
        state_t *state = (state_t *) cookie;

        while (iterations-- > 0) {
                for (i = 0; i < state->npages; ++i) {
                        sum += *(state->where + state->pages[i]);
                } // 	每次访问一个页，页的大小与kernel的PAGESIZE一致。
				  //    结果是访问每个页的时延。
                munmap(state->where, state->size);
                state->where = mmap(0, state->size,
                                    PROT_READ, MAP_SHARED, state->fd, 0);
#ifdef  MS_INVALIDATE
                if (msync(state->where, state->size, MS_INVALIDATE) != 0) {
                        perror("msync");
                        exit(1);
                }
#endif
        }
        use_int(sum);
}

void
benchmark_mmap(iter_t iterations, void* cookie)
{
        int     sum = 0;
        state_t *state = (state_t *) cookie;

        while (iterations-- > 0) {
                munmap(state->where, state->size);
                state->where = mmap(0, state->size,
                                    PROT_READ, MAP_SHARED, state->fd, 0);
#ifdef  MS_INVALIDATE
                if (msync(state->where, state->size, MS_INVALIDATE) != 0) {
                        perror("msync");
                        exit(1);
                }
#endif
        }
        use_int(sum);
}

```

## lat_mmap与lat_pagefault的区别

`lat_pagefault`是读操作，所以触发的是read fault;

`lat_mmap`是写操作，所以触发的是 do_shared_fault.



# pagefault的debug方式

## 地址trace

通过debugfs来调试，只有x86实现了，arm64还未实现。

可以区分user态和kernel态，事件：

​	/sys/kernel/debug/tracing/events/exceptions/page_fault_user

​	/sys/kernel/debug/tracing/events/exceptions/page_fault_kernel

可以得到page fault发生的pid、核号、时间戳，fault地址、pc以及error_code等。

``` c
spawn-4163  [002] dN.. 437085.938627: page_fault_kernel: address=0x7f9b35fbf7d0 ip=__put_user_4 error_code=0x3
```

## fault个数

可以通过perf工具来采集。

`perf stat -e faults -I 1000`



#  虚拟机内的pagefault

## user_mem_abort



# 1     不同的fs

主要区别在挂载的文件file->f_op->mmap()对应的函数不同，在函数中，主要做的就是挂载不同的vma->vm_ops钩子从而使得do_fault/map时候的处理函数不同。

## 1.1    ramfs

fs/ramfs/file-mmu.c中定义了struct file_operations。这里关注mmap。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

generic_file_mmap中，将vma->vm_ops = &generic_file_vm_ops

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)

## 1.2    tmpfs

mm/shmem.c中，struct file_operations

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

vma->vm_ops = &shmem_vm_ops，有fault中的：

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)

 

 

## 1.3    xfs

fs/xfs/xfs_file.c中，struct file_operations结构体对应的实例的mmap钩子为：

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

其中，xfs_file_mmap做的事情只是把vma->vm_ops填充，供后续pagefault时调用。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image006.jpg)














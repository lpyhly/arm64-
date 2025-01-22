# 伙伴系统



# 内存分配接口研究

## 0. 几种分配函数的比较

![伙伴系统中各个分配函数之间的关系](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cclip_image001-1678502730330.png)

| 基本原理   | 接口                                                 | 分配原理                                                     | 最大内存 | 其他                                                         |
| ---------- | ---------------------------------------------------- | ------------------------------------------------------------ | -------- | ------------------------------------------------------------ |
| alloc_page | __get_free_pages(mask,  order) __get_free_page(mask) | 直接对页框进行操作,  返回分配内存块的虚拟地址，而不是page实例 | 4MB      | 适用于分配较大量的连续物理内存                               |
| slab       | kmem_cache_alloc                                     | 基于slab机制实现                                             | 128KB    | 适合需要频繁申请释放相同大小内存块时使用                     |
| slab       | kmalloc                                              | 基于kmem_cache_alloc实现                                     | 128KB    | 最常见的分配方式，需要小于页框大小的内存时可以使用           |
|            | vmalloc                                              | 建立非连续物理内存到虚拟地址的映射  返回4k对齐地址           |          | 物理不连续，适合需要大内存，但是对地址连续性没有要求的场合   |
| alloc_page | dma_alloc_coherent                                   | 基于__alloc_pages实现                                        | 4MB      | 适用于DMA操作                                                |
|            | ioremap                                              | 实现已知物理地址到虚拟地址的映射                             |          | 适用于物理地址已知的场合，如设备驱动                         |
|            | alloc_bootmem                                        | 在启动kernel时，预留一段内存，内核看不见                     |          | 小于物理内存大小，内存管理要求较高                           |
| alloc_page | alloc_pages(mask,  order)                            | 分配2order2order页并返回一个struct  page的实例，表示分配的内存块的起始页 |          | NUMA-include/linux/gfp.h,  line 466  UMA-include/linux/gfp.h?v=4.7,  line 476 |
| alloc_page | alloc_page(mask)                                     | 是前者在order =  0情况下的简化形式，只分配一页               |          | include/linux/gfp.h?v=4.7,  line 483                         |
| alloc_page | get_zeroed_page(mask)                                | 分配一页并返回一个page实例，页对应的内存填充0（所有其他函数，分配之后页的内容是未定义的） |          | mm/page_alloc.c?v=4.7,  line 3900                            |
| alloc_page | get_dma_pages(gfp_mask,  order)                      | 用来获得适用于DMA的页.                                       |          | include/linux/gfp.h?v=4.7,  line 503                         |
| put_page   |                                                      | 释放页                                                       |          |                                                              |

### alloc_page原理

``` shell
alloc_pages ( gfp_mask: 非direct_reclaim，支持compound地址混合，nowarn，noretry）
  -> alloc_pages_current
  获取task的内存分配策略：交织（内存分配覆盖所有节点）/bind（特定节点集）/preferred（默认，指定节点优先）
    -> __alloc_pages_nodemask -- 伙伴分配机制的核心函数
#       -> get_page_from_freelist 从freelist中获取page
#       -> __alloc_pages_slowpath 若freelist中获取不到，从全局内存池中分配page。
 
```

然后，对这两种情况分别分析：

``` shell
# get_page_from_freelist
一些检查，若当前zone不满足，则node_reclaim尝试回收本区域内存；后面再重新检查zone
-> rmqueue  区分order==0以及order!=0两种情况。

#  case 1：order == 0
  使用冷热页机制分配页表（仅对order=0有效），从per-cpu列表中寻找page，防止申请页表前多cpu抢同一个锁，页表释放后也会先进入到本cpu的list中,保证某个页一直在同一个cpu中申请释放，提高cache命中率。
	-> rmqueue_pcplist
	   关中断
	   -> __rmqueue_pcplist
		  取出当前cpu的pcp（per-cpu-page结构体），根据前一类型取出对应链表pcp->lists[migratetype]
		  若list为空，则从伙伴系统中分配pcp->batch个页到list中
		  若设置了COLD，则从list尾部获取一页，否则从链表头获取一页。
	  开中断
#  case 2: order != 0
    spinlock
    -> __rmqueue_smallest 找恰好满足当前MIGRATE_TYPE的page
        从当前order的zonelist中获取free page，若获取不到，则逐级到上一层获取。
        -> expand
            若从上一层获取到，则需要将多余的page还回到对应的list中。
    -> 如果找不到，调用__rmqueue
        ->再试一次__rmqueue_smallest
        ->失败，则调用__rmqueue_fallback从其他迁移类型中申请page，从最高order申请，从而减少碎片化。
    spin_unlock 
 
```

``` shell
# __alloc_pages_slowpath 
  若freelist中获取不到，从全局内存池中分配page。
  -> 若设置了__GFP_KSWAPD_RECLAIM，则唤醒kswap进程来控制页表的分配；
  -> 再调用get_page_from_freelist试一下
      若还是失败，则使用更繁杂的方式申请：
      1. direct_compact 将空闲页面链表中的小页面块合并成大页面块（小order合并成大order），再分配页表。
      2. direct_reclaim 回收一些最近很少用的pagecache中的页面，以腾出更过空间。
      3. 考虑OOM（out of memory)的情况，调用out_of_memory杀死申请内存最多的进程，然后重新分配内存。
```

### put_page原理

``` shell
# __put_page 
查看是否为复合页（order>0 && __GFP_COMP)，若是，则调用__put_compound_page, 否则调用__put_single_page.
#   __put_compound_page
   对于hugetlbfs，一律调用下面函数
   ->  get_compound_page_dtor  获取指针函数
	 -> free_compound_page
	  -> __free_pages_ok
	        获取migratetype
		    关中断
			统计当前cpu释放的page数目
			-> free_one_page
				spin_lock  zone->lock   zone以numa node为单位
				-> __free_one_page
					找到伙伴page，若伙伴page空闲且可以合并，则将改伙伴page从list上delete下来，order++，继续寻找下一级order是否有空闲的伙伴，直到最高的order。
					将合并后的page放到对应order的list中（单纯的链表添加操作）。
				spin_unlock					
 			开中断
```



# 1      mmap的流程

## 1.1    mmap仅用于用户态

mmap可以映射文件 & 匿名空间，后者可以用于父子进程通信（fork时会将memory map空间一同映射过去），也可以认为是单纯的malloc一段空间。

映射文件时，可以给不同的参数，比如可读PROT_READ，可写PROT_WRITE等，若不标注可写则说明是只读空间，后续pagefault时对应的函数不同。

## 1.2    mmap以pagesize为单位

4k/16k/64k的粒度来映射，而且是pagesize对齐的。这个可以通过传入do_mmap的参数pgoff看出，这个值的单位是page。

## 1.3    mmap的常用flags

MAP_SHARED：多个进程mmap同一片地址或同一个file时，对当前file的写入时会被所有人感知到的；

MAP_PRIVATE：与上面相反，copy-on-write形式，所作的修改不会被其他人感知。

MAP_ANNOYMOUS：匿名映射。

## 1.4    mmap系统调用执行流程

1. 检查参数，并根据传入的映射类型设置vma的flags.

2. 进程查找其虚拟地址空间，找到一块空闲的满足要求的虚拟地址空间.

3. 根据找到的虚拟地址空间初始化vma.

4. 设置vma->vm_file.

5. 根据文件系统类型，将vma->vm_ops设为对应的file_operations.

6. 将vma插入mm的链表中.

前2步在do_mmap中处理；后4步在mmap_region中处理。

## 1.5    mmap源码分析

arch/arm64/kernel/sys.c 定义了sys_mmap的调用 

-> ksys_mmap_pgoff. (mm/mmap.c) 判断是否为匿名页/大页，进行一些处理;

-> vm_mmap_pgoff 安全审计，

-> do_mmap_pgoff->do_mmap 终于开始有实际内容啦~~

### 1.5.1       do_mmap

就是拿到了va，并将需要映射的file+pgoff准备好。具体做了3件事：

\1.   根据映射长度len获取到了未被分配过的一段len长度的va地址；

\2.   将file_based/annoymous需要映射的pgoff值计算出来，前者需要映射的是file+pgoff，后者为pgoff（file=0）。

\3.   根据传入的flags设置对应的vm_flags。

对应代码来看：

Ø 对flag属性进行区分处理，比如若为PROT_READ，则多数情况下再添加一个PROT_EXEC；

Ø 某些异常情况检测，比如非对齐/overflow等。

Ø 从地址空间获取一个va地址。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

Ø 添加vm_flags；

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)

Ø 对于file-backend的情况，先从struct file中获取inode，调用file_mmap_ok判断当前file及其offset是否可以映射。

Ø 对于匿名映射，判断传入的flag来设置不同的vm_flags。对于MAP_SHARED，则文件偏移量pgoff=0（这里没有文件，所以按照内存0地址来理解），同时设置vm_flags=VM_SHARED|VM_MAYSHRE， 对于MAP_PRIVATE则只设置pgoff=addr>>PAGE_SHIFT（pgoff的单位是page）。

Ø 调用mmap_region进行va<->file+pgoff的映射。

### 1.5.2       mmap_region

mmap_region：申请一个vma，然后进行填充

​    ![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image006.jpg)

对于file_based，再填充vm_file和vm_ops（vm_ops通过call_mmap填充）： 

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image008.jpg)

 

其中call_mmap调用file->f_op->mmap()钩子函数链接到具体fs，具体函数可以参见内核中的fs/xfs（或其他文件系统名）/xfs_file.c的struct file_operations结构体对应的实例的mmap钩子。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image010.jpg)

其中，xfs_file_mmap做的事情只是把vma->vm_ops填充，供后续pagefault时调用。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image012.jpg)

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image014.jpg)

然后，将申请的新vma加入mm的vma链表，更新mm。



# 影响lat_mmap的因素

### 文件系统类型 

映射文件的文件系统不同，性能会有差异。在1630上实测不同fs下mmap 2m性能：

 2.87（tmpfs） 3.48（nonefs）   4.11（xfs）  5.53（ext4）  3.99（ext2）

默认使用的映射文件是./haha，因此和当前文件系统类型有关。

- 在emu上，如果是fileimg形式，则映射的是ext2的ramblock；如果是minifs形式，则为nonefs；

- 在样片上，默认是nonefs格式，即/不挂载文件系统，直接从内存中读取。



# malloc

### 测试项解析

- malloc + free 4kB大小的内存空间。其中。 会在init时初始化`1024`个buf指针。然后在iterations中先依次调用1024次malloc接口，再依次调用1024次free接口。对每个4k范围的地址的第一个byte进行写操作，此时会触发pagefault真正建立页表。

- 最后统计的结果，是单次malloc+写操作+free的结果。即总时间/loop的次数/1024。

  ```c
  # define MAX_MALLOC_NUM 1024
  void
  loop_malloc(iter_t iterations, void *cookie)
  {
      state_t *state = (state_t *) cookie;
      unsigned int i = 0;
      TYPE *buf[MAX_MALLOC_NUM];
      TYPE res = 0;
      while (iterations-- > 0) {
          for (i = 0; i < MAX_MALLOC_NUM; i++) {
              buf[i] = (TYPE *)malloc(state->nbytes); // 分配
              *(buf[i]) = i;    // 写
          }
          for (i = 0; i < MAX_MALLOC_NUM; i++) { 
              free (buf[i]); // 释放
          }
      }
  }
  ```

  

### 热点分析

热点主要在写操作引起的`pagefault -> do_anonymous_page`上。64k页表热点更多的在`clear_page`，因为每次操作的是64k，比4k要大16倍。

```shell
# 4k页表上：
-   99.99%     0.00%  userdef_malloc  userdef_malloc    [.] _start                     
     _start                                                                            
     __libc_start_main@GLIBC_2.17                                                      
     __libc_start_call_main                                                            
     main                                                                              
     benchmp                                                                           
   - benchmp_child                                                                     
      - 99.59% loop_malloc                                                             
         - 75.59% _int_malloc                                                          
            - 63.89% el0_sync                                                          
               - el0_sync_handler                                                      
                  - 63.44% el0_da                                                      
                     - 58.84% do_mem_abort                                             
                        - 58.67% do_translation_fault                                  
                           - 56.37% do_page_fault                                      
                              - 54.77% handle_mm_fault                                 
                                 - 50.38% __handle_mm_fault                            
                                    - 48.56% handle_pte_fault                          
                                       - 47.24% do_anonymous_page                      
                                          - 26.41% alloc_pages_vma                     
                                             - 24.89% __alloc_pages                    
                                                - 24.04% get_page_from_freelist        
                                                   - 16.77% prep_new_page              
                                                        16.44% clear_page              
                                                     5.53% rmqueue_pcplist.constprop.0 
                                               0.55% __next_zones_zonelist             
                                          - 9.16% mem_cgroup_charge                    
                                               1.97% consume_stock                     
                                               1.35% get_mem_cgroup_from_mm.part.0     
                                               1.33% try_charge                        
                                          - 6.86% lru_cache_add_inactive_or_unevictable
                                             - 6.79% lru_cache_add                     
                                                - __pagevec_lru_add                    
                                                     1.17% release_pages               
                                          - 1.12% page_add_new_anon_rmap               
                                             - 0.80% __mod_lruvec_state                
                                                - 0.70% __mod_memcg_lruvec_state       
                                                     cgroup_rstat_updated              
                                            0.67% cgroup_throttle_swaprate             
                                         0.65% native_queued_spin_unlock               
                             1.22% down_read_trylock                                   
                             0.82% up_read                                             
            - 3.70% sysmalloc                                                          
               - 1.85% el0_sync                                                        
                    el0_sync_handler                                                   
                  - el0_da                                                             
                     - 1.58% do_mem_abort                                              
                        - do_translation_fault                                         
                           - 1.55% do_page_fault                                       
                              - handle_mm_fault                                        
                                 - __handle_mm_fault                                   
                                    - handle_pte_fault                                 
                                       - 1.23% do_anonymous_page                       
                                          - 0.80% alloc_pages_vma                      
                                             - __alloc_pages                           
                                                  get_page_from_freelist               
               - 1.62% __default_morecore@GLIBC_2.17                                   
                  - 1.60% brk                                                          
                     - el0_sync                                                        
                       el0_sync_handler                                                
                       el0_svc                                                         
                     - do_el0_svc                                                      
                        - 1.40% el0_svc_common.constprop.0                             
                           - 1.15% __arm64_sys_brk                                     
                              - 1.05% __se_sys_brk                                     
                                   0.97% do_brk_flags                                  
         - 23.06% __libc_free                                                          
            - 22.81% _int_free                                                         
               - 20.86% systrim.constprop.0                                            
                    __default_morecore@GLIBC_2.17                                      
                    brk                                                                
                    el0_sync                                                           
                    el0_sync_handler                                                   
                    el0_svc                                                            
                    do_el0_svc                                                         
                  - el0_svc_common.constprop.0                                         
                     - 20.79% __arm64_sys_brk                                          
                        - __se_sys_brk                                                 
                           - 20.76% __do_munmap                                        
                              - unmap_region                                           
                                 - 13.92% tlb_finish_mmu                               
                                    - 13.87% tlb_flush_mmu                             
                                       - free_pages_and_swap_cache                     
                                          - release_pages                              
                                               6.16% free_unref_page_list              
                                             - 1.78% mem_cgroup_uncharge_list          
                                                  1.66% uncharge_page                  
                                               0.81% free_unref_page_prepare.part.0    
                                 - 6.69% unmap_vmas                                    
                                      unmap_single_vma                                 
                                    - unmap_page_range                                 
                                       - 6.47% zap_pte_range                           
                                          - 4.40% page_remove_rmap                     
                                               0.58% __mod_lruvec_state                
                                               0.57% __mod_node_page_state             
```

```shell
# 64k页表上
 99.76%     0.00%  userdef_malloc   userdef_malloc      [.] _start                    
  _start                                                                              
  __libc_start_main                                                                   
  main                                                                                
  benchmp                                                                             
- benchmp_child                                                                       
   - 98.49% loop_malloc                                                               
      - 84.16% _int_malloc                                                            
         - 53.46% el0_sync                                                            
            - el0_sync_handler                                                        
               - 52.82% el0_da                                                        
                  - do_mem_abort                                                      
                     - 52.74% do_translation_fault                                    
                        - 52.39% do_page_fault                                        
                           - 51.96% handle_mm_fault                                   
                              - 51.55% __handle_mm_fault                              
                                 - handle_pte_fault                                   
                                    - 51.00% do_anonymous_page                        
                                       - 47.84% alloc_pages_vma                       
                                          - 47.72% __alloc_pages_nodemask             
                                             - 47.44% get_page_from_freelist          
                                                - 46.50% prep_new_page                
                                                     46.01% clear_page                
                                                - 0.84% rmqueue                       
                                                     0.79% arch_local_irq_restore     
                                       - 1.10% lru_cache_add_inactive_or_unevictable  
                                          - 0.98% lru_cache_add                       
                                               0.78% arch_local_irq_restore           
                                         1.06% mem_cgroup_charge                      
                 0.59% local_daif_restore                                             
         - 26.28% sysmalloc                                                           
            - 22.72% el0_sync                                                         
               - el0_sync_handler                                                     
                  - 22.34% el0_da                                                     
                       do_mem_abort                                                   
                     - do_translation_fault                                           
                        - 22.11% do_page_fault                                        
                           - 21.84% handle_mm_fault                                   
                              - 21.67% __handle_mm_fault                              
                                 - 21.52% handle_pte_fault                            
                                    - 21.29% do_anonymous_page                        
                                       - 20.04% alloc_pages_vma                       
                                          - __alloc_pages_nodemask                    
                                             - 19.80% get_page_from_freelist          
                                                - 19.46% prep_new_page                
                                                     19.19% clear_page                
                                         0.51% mem_cgroup_charge                      
            - 3.26% __default_morecore                                                
               - 3.18% __brk                                                          
                  - 2.98% el0_sync                                                    
                       el0_sync_handler                                               
                       el0_svc                                                        
                     - do_el0_svc                                                     
                        - 2.70% el0_svc_common.constprop.0                            
                           - 2.19% invoke_syscall                                     
                              - __arm64_sys_brk                                       
                                 - 2.09% __do_sys_brk                                 
                                    - 1.68% do_brk_flags                              
                                         0.76% vma_merge                              
                                         0.54% perf_event_mmap                        
      - 13.18% _int_free                                                              
         - 8.17% systrim.isra.0.constprop.0                                           
              __default_morecore                                                      
              __brk                                                                   
              el0_sync                                                                
              el0_sync_handler                                                        
              el0_svc                                                                 
              do_el0_svc                                                              
            - el0_svc_common.constprop.0                                              
               - 8.10% invoke_syscall                                                 
                    __arm64_sys_brk                                                   
                  - __do_sys_brk                                                      
                     - __do_munmap                                                    
                        - 8.04% unmap_region                                          
                           - 5.82% unmap_vmas                                         
                              - unmap_page_range                                      
                                   3.13% __flush_tlb_range.isra.0                     
                                   1.60% tlb_flush                                    
                                 - 1.09% zap_pmd_range                                
                                      1.04% zap_pte_range                             
                           - 2.04% tlb_finish_mmu                                     
                              - 2.02% free_pages_and_swap_cache                       
                                 - 1.36% release_pages                                
                                      1.01% arch_local_irq_restore                    
                                   0.63% arch_local_irq_restore                       
           0.51% unlink_chunk.isra.0                                                  
        0.70% malloc                                                                  
     0.58% malloc                                                                     
```



### 性能影响因素

malloc性能与pagesize相关，在1620虚拟机上实测：同样的内核源码文件系统，只修改pagesize，性能：

|                     | 4k页表  | 64k页表 | 提升      |
| ------------------- | ------- | ------- | --------- |
| userdef_malloc 64   | 0.023   | 0.023   | 0%        |
| userdef_malloc 1k   | 0.290   | 0.092   | **+215%** |
| userdef_malloc 4k   | 1.17 us | 0.27 us | **+333%** |
| userdef_malloc 64k  | 1.58 us | 3.68 us | -57%      |
| userdef_malloc 512k | 3.75 us | 4.47 us | -16%      |

原因分析：

	1. TLB miss，导致硬件走pagewalk流程重新建立TLB entry；
 	2. pagewalk时发生PTE页表缺页，软件触发pagefault建立页表。

详细分析：

-  64Byte映射：第一次映射后，后面的256次都不会缺页&TLB miss，即使有缺页，也只有3次/1023次访问，循环次数足够多的情况下，可以cover住这部分性能损失；对于64k页表，整个1024次都不会缺页&TLB miss。 
- 4kByte映射：在4k页表下，若没有预取机制，每次都会缺页； 而64k页表下，16次才会TLB miss一次。
- 64kB映射： 都是每次访问都缺页，但64k需要一次clear_page 64kB，而4k只需要clear_page 4kB;
- 512kB映射：都是每次访问都缺页（？，可能会有around机制，导致提前预取？），但64k需要一次clear_page 64kB，而4k只需要clear_page 4kB;



​		bpf统计得知，触发的pagefault次数和性能差异基本一致，即在4k页表上缺页次数是64k页表的3倍。

``` c
bpftrace -e 'software:page-faults:1 { @[comm] = count(); }' -c 'sleep 1'
```




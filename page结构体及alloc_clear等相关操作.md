# page结构体

结构体偏移：

```c
printk("lru=%d\n", offsetof(struct page, lru));
printk("mapping=%d\n", offsetof(struct page, mapping));
printk("private=%d\n", offsetof(struct page, private));
printk("rcu_head=%d\n", offsetof(struct page, rcu_head));
printk("index=%d\n", offsetof(struct page, index));
printk("_mapcount=%d\n", offsetof(struct page, _mapcount));
printk("refcount=%d\n", offsetof(struct page, _refcount));
```
``` c
struct page {
    flags         -- 0x00
    lru=8
	mapping=24
    index=32
    private=40
    _mapcount=48
    refcount=52
} // 共0x40 Byte

struct page {
    unsigned long flags;        /* Atomic flags, some possibly
                     * updated asynchronously */
    /*
     * Five words (20/40 bytes) are available in this union.
     * WARNING: bit 0 of the first word is used for PageTail(). That
     * means the other users of this union MUST NOT use the bit to
     * avoid collision and false-positive PageTail().
     */
    union {
        struct {    /* Page cache and anonymous pages */
            /**
             * @lru: Pageout list, eg. active_list protected by
             * zone_lru_lock.  Sometimes used as a generic list
             * by the page owner.
             */
            struct list_head lru;
            /* See page-flags.h for PAGE_MAPPING_FLAGS */
            struct address_space *mapping;
            pgoff_t index;      /* Our offset within mapping. */
            /**
             * @private: Mapping-private opaque data.
             * Usually used for buffer_heads if PagePrivate.
             * Used for swp_entry_t if PageSwapCache.
             * Indicates order in the buddy system if PageBuddy.
             */
            unsigned long private;
        };
        struct {    /* slab, slob and slub */
            union {
                struct list_head slab_list; /* uses lru */
                struct {    /* Partial pages */
                    struct page *next;
#ifdef CONFIG_64BIT
                    int pages;  /* Nr of pages left */
                    int pobjects;   /* Approximate count */
#else
                    short int pages;
                    short int pobjects;
#endif
                };
            };
            struct kmem_cache *slab_cache; /* not slob */
            /* Double-word boundary */
            void *freelist;     /* first free object */
            union {
                void *s_mem;    /* slab: first object */
                unsigned long counters;     /* SLUB */
                struct {            /* SLUB */
                    unsigned inuse:16;
                    unsigned objects:15;
                    unsigned frozen:1;
                };
            };
        };
        struct {    /* Tail pages of compound page */
            unsigned long compound_head;    /* Bit zero is set */

            /* First tail page only */
            unsigned char compound_dtor;

```





# alloc_pages

alloc_pages->
				alloc_pages_current
					__alloc_pages_nodemask
						get_page_from_freelist

## 	get_page_from_freelist

`get_page_from_freelist`函数做的是：

1. 先检查一些flag是否符合要求
2. 然后检查watermark水线是否允许分配。

首先通过WARTERMARK_LOW来循环检测，当没有能满足在WARTERMARK_LOW之上的zone时系统会以WARTERMARK_MIN为标准以同样的方式对zone进行检测，如果连满足WARTERMARK_MIN的zone也没有了，那就会启用lowmem_reserve的内存。当然一般在没有满足WARTERMARK_LOW的zone时就会唤醒kswapd内核线程来对页框进行回收。

1. rmqueue函数从系统中分配物理页框。
2. 分配成功后，调用prep_new_page做一些检测。其中，若gfp_flags中含有__GFP_ZERO，则调用clear_highpage将page页面全部清零。



# clear_page

``` c
__handle_mm_fault -> do_wp_fault -> wp_page_copy(某一种clear_page的来源)
    	// 造成写时缺页的是0页（可能是先读后写），
    	// 需要重新分配一个页表，并将这个PAGE写全0来初始化
    	-> alloc_zeroed_user_highpage_movable 
    		-> clear_user_highpage
    			-> clear_page 
```


[TOC]

# 从进程到文件

1. 从进程描述符中找到该进程打开的文件的struct file结构体。

2. 从file结构体中获取dentry，即目录项（路径）；也可能已经缓存了inode，则可以跳过步骤3。

3. 从目录项中获取inode结构体。

4. inode即真正对应着文件，通过block块来标识，从inode就可以去到设备驱动来找文件了。

   

**内核中用inode结构表示具体的文件，而用file结构表示打开的文件描述符。**

## 进程->file结构体

file结构体是与进程息息相关的变量。从进程任务来一层层看：

- task_struct -> files，为struct file_struct *结构类型。主要包含：

  - fdt为struct file *类型的指针，指向自己的fd_array成员的第一个变量，可以通过这个来访问到具体的file，之所以还专门搞一个出来，是为了防止打开的文件超过预设个数，即fd_array成员放不开了，就需要再弄一个链表出来链接，这时候就用到fdt成员了。

- files->fd_array， 为struct file *[]指针数组，存放的是当前进程打开的所有文件，是struct file *的数组。

  ![img](D:\at_work\Documents\我的总结文档\images\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppbmtpbmcwMQ==,size_16,color_FFFFFF,t_70)

## file到dentry

struct file结构体中有个成员叫fpath，fpath中有个成员叫dentry。

## file到inode

 struct inode *inode = file->f_mapping->host;

可以获取到inode指针。

## inode到page

通过`filemap_get_pages()`从inode中获取一梭罗的pages。 

- 先调用`filemap_get_read_batch`，看能否直接从pagecache中获取，若不行，则需要从磁盘中同步预读`page_cache_sync_readahead`），然后重新`filemap_get_read_batch`； 此时应该已经能从pagecache中读取到了，要是再读取不到，则调用`filemap_create_folio-> filemap_alloc_folio+ filemap_add_folio+ filemap_read_folio` 进行读取。
- 从fbatch中获取到了folio们，从中判断是否需要预取（比如，对于最后一个folio，`filemap_get_read_batch`就有可能会将ReadAhead置位，从而引发提前预取动作，这里预取算法考虑的事项很多，不在此详述），若需要，则调用`filemap_readahead`进行异步预读操作，因为这里预取的页不是我们现在就要的，所以选择异步预取，只发指定，不需要等待完成。
- 最后，需要检查一下是否是最新的，若不是则需要`filemap_update_page`进行更新。

# 重要结构体们

## struct file

file结构体是受rcu机制保护的，对于该结构体的操作，需要使用rcu接口进行。但是，对于file内的成员变量的操作，就不受rcu保护了。

![images-struct-file](D:\at_work\Documents\我的总结文档\images\images-struct-file.png)

``` c
// 4.19/5.10内核：
struct file {
    union {
        struct llist_node   fu_llist;                       -- 0x0, size 8B
        struct rcu_head     fu_rcuhead;                     -- 0x0, size 16B
    } f_u;													-- 0x0
    struct path     f_path;									-- 0x10
        // f_path: 包含dentry（dentry结构）和mnt(根目录）两个成员，用于确定文件路径
    struct inode        *f_inode;   /* cached value */      -- 0x20
    const struct file_operations    *f_op;                  -- 0x28
    spinlock_t      f_lock;                                 -- 0x30
    enum rw_hint        f_write_hint;                       -- 0x34
    atomic_long_t       f_count;                            -- 0x38
    unsigned int        f_flags;                            -- 0x40
        // 对应于open时指定的flag
    fmode_t         f_mode;			                        -- 0x44
        // 读写模式：open的mod_t mode参数
    struct mutex        f_pos_lock;                         -- 0x48
    loff_t          f_pos;		                 			-- 0x88
        // 该文件在当前进程中的文件偏移量
    struct fown_struct  f_owner;               				-- 0x70
        // 该结构的作用是通过信号进行I/O时间通知的数据。
    const struct cred   *f_cred;             			    -- 0x90
    struct file_ra_state    f_ra;                 			-- 0x98
    u64         f_version;               					-- 0xb8
#ifdef CONFIG_SECURITY
    void            *f_security;		                 	-- 0xc0
#endif
    /* needed for tty driver, and maybe others */
    void            *private_data;			                -- 0xc8

#ifdef CONFIG_EPOLL
    /* Used by fs/eventpoll.c to link all the hooks to this file */
    struct list_head    f_ep_links; 			            -- 0xd0
    struct list_head    f_tfile_llink;			            -- 0xe0
#endif /* #ifdef CONFIG_EPOLL */
    struct address_space    *f_mapping;			            -- 0xf0
    errseq_t        f_wb_err;			                    -- 0xf8
    errseq_t        f_sb_err; /* for syncfs */              -- 5.10内核新增
} __randomize_layout
  __attribute__((aligned(4)));  /* lest something weird decides that 2 is OK */
// 共0x100 = 256 B
```

## struct inode

```c
struct inode {
        umode_t                 i_mode; // inode的权限
        unsigned short          i_opflags;
        kuid_t                  i_uid;
        kgid_t                  i_gid;
        unsigned int            i_flags;
        const struct inode_operations   *i_op;
        struct super_block      *i_sb;
        struct address_space    *i_mapping;

#ifdef CONFIG_SECURITY
        void                    *i_security;
#endif
        /* Stat data, not accessed from path walking */
        unsigned long           i_ino; // inode序号，全局唯一
        union {
                const unsigned int i_nlink; // hard link的个数
                unsigned int __i_nlink;
        };
        dev_t                   i_rdev; // 对应的devcie代码
        loff_t                  i_size; // size
        struct timespec64       i_atime; // 三个time
        struct timespec64       i_mtime;
        struct timespec64       i_ctime;
        spinlock_t              i_lock; // osqlock !!!!!
        unsigned short          i_bytes;
        u8                      i_blkbits;
        u8                      i_write_hint;
        blkcnt_t                i_blocks;
#ifdef __NEED_I_SIZE_ORDERED
        seqcount_t              i_size_seqcount;
#endif

        /* Misc */
        unsigned long           i_state;
        struct rw_semaphore     i_rwsem;

        unsigned long           dirtied_when;   /* jiffies of first dirtying */
        unsigned long           dirtied_time_when;

        struct hlist_node       i_hash;
        struct list_head        i_io_list;      /* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
        struct bdi_writeback    *i_wb;          /* the associated cgroup wb */

        /* foreign inode detection, see wbc_detach_inode() */
        int                     i_wb_frn_winner;
        u16                     i_wb_frn_avg_time;
        u16                     i_wb_frn_history;
#endif
        struct list_head        i_lru;          /* inode LRU list */
        struct list_head        i_sb_list;
        struct list_head        i_wb_list;      /* backing dev writeback list */
        union {
                struct hlist_head       i_dentry;
                struct rcu_head         i_rcu;
        };
        atomic64_t              i_version;
        atomic64_t              i_sequence; /* see futex */
        atomic_t                i_count;
        atomic_t                i_dio_count;
        atomic_t                i_writecount;
#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
        atomic_t                i_readcount; /* struct files open RO */
#endif
        union {
                const struct file_operations    *i_fop; /* former ->i_op->default_file_ops */
                void (*free_inode)(struct inode *);
        };
        struct file_lock_context        *i_flctx;
        struct address_space    i_data;
        struct list_head        i_devices;
        union {
                struct pipe_inode_info  *i_pipe;
                struct cdev             *i_cdev;
                char                    *i_link;
                unsigned                i_dir_seq;
        };

        __u32                   i_generation;

#ifdef CONFIG_FSNOTIFY
        __u32                   i_fsnotify_mask; /* all events this inode cares about */
        struct fsnotify_mark_connector __rcu    *i_fsnotify_marks;
#endif

#ifdef CONFIG_FS_ENCRYPTION
        struct fscrypt_info     *i_crypt_info;
#endif

#ifdef CONFIG_FS_VERITY
        struct fsverity_info    *i_verity_info;
#endif

        void                    *i_private; /* fs or device private pointer */
} __randomize_layout;
```



# fork中关于file的操作：copy_files

​		所有进程均操作同一个文件，进程都是同一个父进程fork出来的。在fork时，会`copy_files -> dup_fd`去通过操作新进程的`struct file_struct -> fd`指针，来更新`struct file *fd_array[]`变量。

``` c
// dup_fd
old_fds = old_fdt->fd; 
new_fds = new_fdt->fd; // struct file **类型指针，指向自身file_struct->fd_array[]。

for (i = open_files; i != 0; i--) {
        struct file *f = *old_fds++;
        if (f) {
                get_file(f);
        } else {
                __clear_open_fd(open_files - i, new_fdt);
        }
        rcu_assign_pointer(*new_fds++, f); // 更新fd_array[]，这里只是把struct file*复制过来了，也就是说struct file与父进程是同一份。
}
spin_unlock(&oldf->file_lock);
```



# dup

1. 先检查文件权限
2. 检查文件引用数是否为0
3. 引用数+1

``` c
dup(fs/file.c)
    __arm64_sys_dup
 	   -> ksys_dup
    		-> fget_raw -> __fget(fd, 0)
int ksys_dup(unsigned int fildes)
{
    int ret = -EBADF;
    struct file *file = fget_raw(fildes); // 	给定一个文件描述符int fd，然后从current进程中
                              // 的fdt列表上把fd对应的struct file拿下来。

    if (file) {
        ret = get_unused_fd_flags(0); // 获取一个没有用到的fd描述符。
        if (ret >= 0)
            fd_install(ret, file);  // 将current进程中的fdt->fd[fd]填充
                               // 为之前的struct file指针，从而实现两个fd指向同一个file的操作。
        else
            fput(file);
    }
    return ret;
}

```

其中，`__fget_files`在不同内核中的实现是不同的。

``` c
// 5.10内核
static struct file *__fget_files(struct files_struct *files, unsigned int fd,
                 fmode_t mask, unsigned int refs)
{
    struct file *file;

    rcu_read_lock();
loop:
    file = files_lookup_fd_rcu(files, fd); // 0.从rcu上将fdtable薅下来，然后
    			// 将fd对应的文件指针再薅下来。
    			// 这里不操作struct file内部结构体，只是获取了file指针
    if (file) {
        /* File object ref couldn't be taken.
         * dup2() atomicity guarantee is the reason
         * we loop to catch the new file (or NULL pointer)
         */
        if (file->f_mode & mask)  	// 1. 先检查文件权限
            file = NULL;
        else if (!get_file_rcu_many(file, refs)) // 2.若file当前引用数不为0，则令引用数+refs 
            	// 若file引用数=0，说明文件描述符已经不存在，需要重新获取。
            	// 调用arch_atomic64_fetch_add_unless,实现为try_cmpxchg
            	// 操作的是file->f_count
            goto loop;
    }
    rcu_read_unlock();

    return file;
}
#define get_file_rcu_many(x, cnt)       \
        atomic_long_add_unless(&(x)->f_count, (cnt), 0)
// 真正实现是read+CAS。
// -> arch_atomic64_read + arch_atomic64_try_cmpxchg
```

``` c
// 5.18内核， 实现不同，但是成员变量的操作是一致的。
__fget_files -> __fget_files_rcu
static inline struct file *__fget_files_rcu(struct files_struct *files,
    unsigned int fd, fmode_t mask, unsigned int refs)
{
    // 简化版
    struct fdtable *fdt = rcu_dereference_raw(files->fdt);
    fdentry = fdt->fd + array_index_nospec(fd, fdt->max_fds);
    file = rcu_dereference_raw(*fdentry);
    if (unlikely(!file))
        return NULL;
    if (unlikely(file->f_mode & mask)) // 读取file->f_mode
        return NULL;
	if (unlikely(!get_file_rcu_many(file, refs))) // 对f_count做read+CAS
         continue;
    return file;
}
```



## dup对应的锁行为

涉及到struct file中的2个变量：

​    f_count  -  0x38  ： 文件引用数，atomic_long_t类型

​	f_mode  -  0x44   :  uint类型， 文件权限管理，比如该文件是否可读、可写等，或者是指示是否为O_PATH。 

dup中：

1. f_mode(0x44)的纯读操作(__fget())，若MASK=NULLL（这个case一定是NULL），则
2. f_count(0x38)的atomic_long_read读(file_count())     -> rs操作，非锁操作
3. f_count(0x38)的cas操作-while循环。
4. 若MASK ！= NULL，则没有上述操作。

# close

1. 先检查文件引用数是否为0
2. 再检查文件mode是否为FMODE_PATH（即是否为O_PATH（location only）类型），若是，则需要特殊操作
   - O_PATH： 若open时指定flags=O_PATH，表示该文件不会真正被操作，只是想借助其路径而已。
   - 因此，O_PATH文件在open的时候引用数不会+1，也不支持read  write  fchmod等文件操作
   - 可以用来防止因为路径指向的对象被更改导致的时间竞赛漏洞，因为事先通过路径获取了该文件夹的struct file了。
   - 因为引用数没有+1， 所以O_PATH对应的close,dup,openat,fcntl等需要特殊操作。
   - 详见 https://qastack.cn/unix/392546/what-am-i-supposed-to-use-o-path-for-and-how
3.  dnotify_flush在fsnotify机制中，因为文件已经关闭了，因此destroy对该文件的监听。
   - dnotify -> fsnotify，是linux中的文件监听机制，通过f_inode结构体来索引调用。
4. 将文件引用数-1， 调用fput()。

``` c
SYS_close // fs/open.c，系统调用close的内核接口
    -> close_fd // fs/file.c，从current中获取到对应fd的文件struct file *
    	-> filp_close // 执行close操作的主要函数, fs/open.c
/*
 * "id" is the POSIX thread ID. We use the
 * files pointer for this..
 */
int filp_close(struct file *filp, fl_owner_t id)
{
    int retval = 0;
    if (!file_count(filp)) {  // 1. 先判断file引用数是否为0，为f_count的RS读
        printk(KERN_ERR "VFS: Close: file count is 0\n");
        return 0;
    }

    if (filp->f_op->flush) 
        retval = filp->f_op->flush(filp, id);
    			// stdin没有flush钩子函数，perf跟踪确认为NULL。

    if (likely(!(filp->f_mode & FMODE_PATH))) { // 3. 判断是否为O_PATH
        dnotify_flush(filp, id);  // 将附加到当前inode上的所有dnotify_struct结构体解绑，
        	// 这个类似monitor，可以在文件发生access、motify等操作的时候触发特定的动作。
        locks_remove_posix(filp, id);
    }
    fput(filp);    // 4. 修改f_count引用数
    return retval;
}


fput
    -> fput_many(refs=1)
void fput_many(struct file *file, unsigned int refs)
{
    if (atomic_long_sub_and_test(refs, &file->f_count)) { 
        // 对f_count做ldadd减一.      
        // 令f_count引用数减一, 若减-1后结果为0，则返回True。
        // 若该文件真的没有人再用了，才会进入到这里面。
        // 最终调用的是__lse_atomic64_fetch_add，只有ldadd操作，没有RS。
        struct task_struct *task = current;  
		
        if (likely(!in_interrupt() && !(task->flags & PF_KTHREAD))) {
            init_task_work(&file->f_u.fu_rcuhead, ____fput);
            // ____fput-> __fput，释放文件的最后一个引用。
            // 其中，f_u为struct file的第一个变量，且fu_llist和fu_rcuhead在同一个位置。
            if (!task_work_add(task, &file->f_u.fu_rcuhead, TWA_RESUME))
                return;
        }

        if (llist_add(&file->f_u.fu_llist, &delayed_fput_list))
            schedule_delayed_work(&delayed_fput_work, 1);
    }
}

```

## close对应的锁行为

涉及到struct file中的3个变量：

​	*f_inode  -  0x20  ： 文件inode信息， struct inode *指针

​    *f_op       -  0x28   :    文件ops操作的钩子函数的指针，struct file_operations *。

​    f_count  -  0x38  ： 文件引用数，atomic_long_t类型

​	f_mode  -  0x44   :  uint类型， 文件权限管理，比如该文件是否可读、可写等，或者是指示是否为O_PATH。 



1. f_count（0x38)的atomic read操作（L3看不见，filp_close）
2. f_mode（0x44)的纯读操作（filp_close）
3. f_op      （0x28)的纯读操作（filp_close）
4. f_inode（0x20)的纯读操作（dnotify_flush)
5. f_count（0x38)的ldadd原子减操作。

![image-20210823140318872](D:\at_work\Documents\我的总结文档\images\image-20210823140318872.png)



# open

``` c
SYS_open
    -> do_sys_open
    	-> do_sys_openat2 
    		-> build_open_flags //对flag进行权限及合规检查，并构造open_flags结构体
    		-> get_unused_fd_flags //获取fd
    		-> do_filp_open //通过fd获取struct file*指针，只是孤零零的指针，不挂载任何进程。
    			-> path_openat(flags|LOOKUP_RCU) //先通过RCU方式解析文件路径，若有其他人修改了正在查找的目录则可能报错，那么就使用ref-walk REF方式，再不行就REVAL方式，最终获取到。
    		-> fsnotify_open
    			-> fsnotify_file
    				-> fsnotify_parent //通知父节点，是为了更新mtime么？
    		-> fd_install // 将file*挂载到fdtable fdt中，写时需要使用rcu保护。
    
 static struct file *path_openat(struct nameidata *nd,
             const struct open_flags *op, unsigned flags)
 {
     struct file *file;
     int error;

     file = alloc_empty_file(op->open_flag, current_cred()); 
     	// 1. 通过__alloc_file分配file结构体，新的。
     	// 同时f_count置1(atomic_long_set直接写，无原子操作)
     	// 然后初始化部分内部成员 f_owner，f_lock, f_pos_lock,f_flags,f_mode等
	 ...

     if (unlikely(...)) // 创建临时文件
     else {
         const char *s = path_init(nd, flags); //确定查找的其实目录，初始化nd->path
         while (!(error = link_path_walk(s, nd)) &&  //解析文件路径的每个分量，最后一个分量除外
                (s = open_last_lookups(nd, file, op)) != NULL) //解析最后一个分量，并打开文件
             ;
         if (!error)
             error = do_open(nd, file, op); // 3. 对不同f_mode进行分别处理，
         		// 然后调用vfs_open -> do_dentry_open对file结构体的f_inode, f_mapping等
         		// 进行赋值操作。
         terminate_walk(nd); // 结束查找，释放解析文件路径的过程中保存的目录项和挂载描述符。
     }
     if (likely(!error)) {
         if (likely(file->f_mode & FMODE_OPENED)) // 4. 若打开成功，则返回file结构体
             return file;
         WARN_ON(1);
         error = -EINVAL;
     }
    ...
 }
```

### link_path_walk

这是一个名字解析函数，可以将pathname转换成最终的dentry。

``` c
// (fs/namei.c)
// 返回0说明成功，通过nd来保存dentry。
static int link_path_walk(const char *name, struct nameidata *nd)
```






# read

基本调用关系。

``` c
read (glibc)
-> el0_svc->__arm64_sys_read->ksys_read->
    -> vfs_read->__vfs_read
   // 这里会调用具体fs的钩子函数。
ssize_t __vfs_read(struct file *file, char __user *buf, size_t count,
           loff_t *pos)
{
    if (file->f_op->read) 	// 若read钩子存在，则直接调用.
        return file->f_op->read(file, buf, count, pos);
    else if (file->f_op->read_iter) // 否则，调用new_sync_read，其实就是调用了read_iter钩子。
        return new_sync_read(file, buf, count, pos);
    else
        return -EINVAL;
}
```

以ramfs为例，

``` c
// 以ramfs为例来看一下其f_op结构体的定义（fs/ramfs/file_mmu.c)
const struct file_operations ramfs_file_operations = {
    .read_iter  = generic_file_read_iter,
     // read操作会调用generic_file_read_iter，其实也是所有能使用page cache的fs的read_iter路径。当然，如果是直接调用IO而不需要pagecache缓存的，就不是这个接口了。具体的可以参见不同fs的file_operations结构体说明。
    .write_iter = generic_file_write_iter,
    .mmap       = generic_file_mmap,
    .fsync      = noop_fsync,
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
    .llseek     = generic_file_llseek,
    .get_unmapped_area  = ramfs_mmu_get_unmapped_area,
};
```

通用文件读路径函数`generic_file_read_iter`，在5.10进行了重构，因此分别看一下。



### generic_file_read_iter-5.18

5.18内核中，看起来更清爽了。generic_file_read_iter函数中仅判断是否为directIO，然后分为两个路线。若非directIO，则表示需要经过pagecached这一层，将IO的页映射到struct page中，则调用filemap_read函数。

``` c
generic_file_read_iter
    if IOCB_DIRECT : 
		-- if IOCB_NOWAIT: filemap_range_needs_writeback
        		else: filemap_write_and_wait_range
        -- file_accessed();
        -- direct_IO()
    else: 
 		filemap_read()
```

首先介绍几个结构体：

``` c

struct folio_batch {
    unsigned char nr;
    bool percpu_pvec_drained;
    struct folio *folios[PAGEVEC_SIZE]; // PAGEVEC_SIZE固定15，加其他则整结构体为16*8的size
};
struct folio {} 
// struct page的抽象，当前与前者同构。因为page仅可表示PAGE_SIZE大小的页，而后者可以表示完整的compound的页。
// 对齐方式上有强制性， 具体来说是offset与size是相同的。
/* It is a power-of-two in size, and it is aligned to that same power-of-two.  */
```

然后看filemap_read函数，主要作用是：将data从pagecache中拷贝到iter指针指向的用户态地址。若pagecache中尚未准备好data，则需要readahead和readpage首先取到pagecache中。详细流程为：

1. 获取一个folio_batch（包含最多16个folio页）
2. 对folio_batch中的每个页，copy_to_iter（从用户态到内核态）并标记为accessed；
3. 若没读完，返回S1继续。
4. 若读完了，则修改文件的mtime。// file_accessed()

``` c
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter,
        ssize_t already_read)
{
    struct file *filp = iocb->ki_filp;
    struct file_ra_state *ra = &filp->f_ra; // Track a file's readahead state
    struct address_space *mapping = filp->f_mapping;
    struct inode *inode = mapping->host;
    struct folio_batch fbatch;
    bool writably_mapped;
    loff_t isize, end_offset;

    ......
    do {
        cond_resched();
        error = filemap_get_pages(iocb, iter, &fbatch); // 先获取一向量的抽象页
        if (error < 0)
            break;
        ......
        end_offset = min_t(loff_t, isize, iocb->ki_pos + iter->count);

        writably_mapped = mapping_writably_mapped(mapping); // 用户态文件可能已经被修改了。
        
		folio_mark_accessed(fbatch.folios[0]);
        for (i = 0; i < folio_batch_count(&fbatch); i++) { // 对每一个folio操作
            struct folio *folio = fbatch.folios[i];
            size_t fsize = folio_size(folio);
            size_t offset = iocb->ki_pos & (fsize - 1);
            size_t bytes = min_t(loff_t, end_offset - iocb->ki_pos,
                         fsize - offset);
            size_t copied;

            if (i > 0)
                folio_mark_accessed(folio);

            if (writably_mapped)
                flush_dcache_folio(folio); // 标记当前page的标记为dirty，表示这个page已经被内核修改过了，当返回用户态时需要flush一下（详见 __sync_icache_dcache）

            copied = copy_folio_to_iter(folio, offset, bytes, iter);

            already_read += copied;
            iocb->ki_pos += copied;
            ra->prev_pos = iocb->ki_pos;
        }
    ......
    } while (iov_iter_count(iter) && iocb->ki_pos < isize && !error);

    file_accessed(filp);
    return already_read ? already_read : error;
}
```

#### filemap_get_pages函数分析

详见 [inode到page](#inode到page)， 其中获取page的函数为 `filemap_get_read_batch`， 使用rcu_read锁来进行保护。

``` c
// filemap_get_pages -> filemap_get_read_batch
static void filemap_get_read_batch(struct address_space *mapping,
                pgoff_t index, pgoff_t max, struct folio_batch *fbatch)
{
        XA_STATE(xas, &mapping->i_pages, index);
        struct folio *folio;

        rcu_read_lock();
        for (folio = xas_load(&xas); folio; folio = xas_next(&xas)) {
                if (xas_retry(&xas, folio))
                        continue;
                if (xas.xa_index > max || xa_is_value(folio))
                        break;
                if (xa_is_sibling(folio))
                        break;
                if (!folio_try_get_rcu(folio))
                        goto retry;

                if (unlikely(folio != xas_reload(&xas)))
                        goto put_folio;

                if (!folio_batch_add(fbatch, folio)) // 将获取到的folio添加进fbatch中
                        break;
                if (!folio_test_uptodate(folio)) // 判断是否加uptodate标志位
                        break;
                if (folio_test_readahead(folio)) // 判断是否加预取标志位
                        break;
                xas_advance(&xas, folio->index + folio_nr_pages(folio) - 1);
                continue;
put_folio:
                folio_put(folio);
retry:
                xas_reset(&xas);
        }
        rcu_read_unlock();
}
```





### generic_file_read_iter-4.19

通用文件读路径函数`generic_file_read_iter`的l流程为：

1. 循环在内存中寻找所读取内容是否在内存中缓存
   1. 如果PageCache命中失败，使用`page_cache_async_readahead/page_cache_sync_readahead`会从磁盘中读取页，并进行预读。
2. 判断页是否是最新，以免读到脏数据；
   1. 如果非最新则需要调用`address_space_operations`中`readpage`函数进行读操作获取最新页，读页的函数最后都会调用`submit_bio`。   
   2. 此外，如果内存已经没有`page cache`，则需要调用函数`page_cache_alloc`来进行分类page并加入到`page_cache_lru`。
3. 然后通过`copy_page_to_iter`将内存中数据复制到用户空间。

4. 最后通过函数`file_accessed`来更新文件访问时间。

![](https://yqfile.alicdn.com/37c6679f050fc64c38348e29d88a3df1be469576.png)



  ->  `generic_file_buffered_read` ，接下来，重点分析函数`generic_file_buffered_read`

通过`perf probe -a '__vfs_read file->f_flags'`并将file->f_flags对应到iocb->ki_flags（详见函数`iocb_flags`），发现read过程中，ki_flags的

这个函数很长，只截取主干进行分析

``` c
// iocb: 待读取的文件的iocb
// iter：目的地，一定是用户态的地址空间
// written： 已经copy过来的字节数
ssize_t generic_file_buffered_read(struct kiocb *iocb,
        struct iov_iter *iter, ssize_t written) 
{
    struct file *filp = iocb->ki_filp;
    struct address_space *mapping = filp->f_mapping; // 获取管理缓冲区的对象
    struct inode *inode = mapping->host;  
    	// 获取inode，open()时已经将struct file和真正的磁盘inode对应上了。
    	// __dentry_open() {  f->f_mapping = inode->i_mapping; }
    struct file_ra_state *ra = &filp->f_ra;
    loff_t *ppos = &iocb->ki_pos;
    pgoff_t index;
    pgoff_t last_index;
    pgoff_t prev_index;
    unsigned long offset;      /* offset into pagecache page */
    unsigned int prev_offset;
    int error = 0;
    
    index = *ppos >> PAGE_SHIFT; // 确定本次读取的是文件中的第几个页
	prev_index = ra->prev_pos >> PAGE_SHIFT; 
	prev_offset = ra->prev_pos & (PAGE_SIZE-1);
	last_index = (*ppos + iter->count + PAGE_SIZE-1) >> PAGE_SHIFT;
	offset = *ppos & ~PAGE_MASK;
    
    for (;;) { // 行为主体
         struct page *page;
         pgoff_t end_index;
         loff_t isize;
         unsigned long nr, ret;

         cond_resched();
 find_page:
         page = find_get_page(mapping, index); // 在pageCache中获取特定index的page，后面详述
          if (!page) {     // 若pageCache中找不到
             if (iocb->ki_flags & IOCB_NOIO)
                 goto would_block;
             page_cache_sync_readahead(mapping, // 需要去磁盘中进行同步预读，读到ddr中。
                     ra, filp, index, last_index - index);
             page = find_get_page(mapping, index); // 预读后再重新获取一次
             if (unlikely(page == NULL))
                 goto no_cached_page;
         }
         if (PageReadahead(page)) { // 如果page包含ReadAhead标志，则去预读
             if (iocb->ki_flags & IOCB_NOIO) {
                 put_page(page);
                 goto out;
             }
             page_cache_async_readahead(mapping, // 这里再进行一次异步预读操作。
                     ra, filp, page,
                     index, last_index - index);
         }       
         if (!PageUptodate(page)) { // 如果page不是最新的，就去获取最新的page
             goto page_not_up_to_date;
             ......
             goto page_not_up_to_date_locked;
			 .....
         }
page_ok:   
         ret = copy_page_to_iter(page, offset, nr, iter); // copy到用户态
         offset += ret;
         index += offset >> PAGE_SHIFT;
         offset &= ~PAGE_MASK;
         prev_offset = offset;

         put_page(page); // 释放当前page，即计数-1
         written += ret;
         if (!iov_iter_count(iter)) // 如果全部copy完了
             goto out;
         if (ret < nr) {
             error = -EFAULT;
             goto out;
         }
         continue;
page_not_up_to_date: //当page需要更新时才会获取lock
        /* Get exclusive access to the page ... */
        if (iocb->ki_flags & IOCB_WAITQ)
            error = lock_page_async(page, iocb->ki_waitq); 
        				//获取page的锁，在readpage钩子函数中释放。
        else
            error = lock_page_killable(page); 
        	// 只是设置page->flags的PG_locked = 1，
        	// 设置的时候直接设置，没有锁，
            // 返回时会调用到atomic_long_fetch_or_acquire()来读取。

page_not_up_to_date_locked:
		// some 判断条件
readpage:
        if (iocb->ki_flags & (IOCB_NOIO | IOCB_NOWAIT)) {
            unlock_page(page); // page->flags的PG_locked清零。没有锁操作。
            put_page(page); // refcount--，使用atomic_dec操作。
            goto would_block;
        }
        /*
         * A previous I/O error may have been due to temporary
         * failures, eg. multipath errors.
         * PG_error will be set again if readpage fails.
         */
        ClearPageError(page);
        /* Start the actual read. The read will unlock the page. */
        error = mapping->a_ops->readpage(filp, page); // 把最新的page从磁盘中读出来，后面详述      
out:
    ra->prev_pos = prev_index;
    ra->prev_pos <<= PAGE_SHIFT;
    ra->prev_pos |= prev_offset;

    *ppos = ((loff_t)index << PAGE_SHIFT) + offset;
    file_accessed(filp);
    return written ? written : error;
}
```



#### find_get_page函数分析

作用是从page cache中获取struct page。

传入参数mapping和offset都是从struct file中获取的，这个struct file就是open()/create()接口通过fs将io设备映射过来的。其中，mapping对应的是该文件的inode，因此是全局唯一的，offset表示当前正在获取的page是struct file的第几个page。

直接调用`pagecache_get_page`。此时的fgp_flags=0，也就是什么FGP flag都不会被设置。

``` c
static inline struct page *find_get_page(struct address_space *mapping,
                    pgoff_t offset)
{
    return pagecache_get_page(mapping, offset, 0, 0);
}
```

``` c
// if fgp_flags = 0, pagecache_get_page函数作用就只是调用find_get_entry.
struct page *pagecache_get_page(struct address_space *mapping, pgoff_t offset,
    int fgp_flags, gfp_t gfp_mask)
{
    struct page *page;
    page = find_get_entry(mapping, offset); 
    	// find_get_entry尝试从当前的page中获取page cache slot
	......
    return page;
} 
```



##### find_get_entry函数分析

find_get_entry函数中，需要注意的有：

１.　函数在rcu锁内部进行操作，因此自己读完后，可能被其他人修改掉了。这里的rcu锁保护的是整个pagecache，不是特定的file或者page，注意，rcu对读是没有影响的，只是声明了一下而已。

２.　radix tree是用来管理pagecache的，使用了基数树这种数据结构，特点是空间换时间，方便查找。

find_get_entry函数流程是：

- 首先，在pagecache这个大池子中查找，我需要的inode+offset对应的page是否已经在pagecache中。若不在，直接返回NULL。
- 若在，则需要获取这个page了。获取时，是需要修改page的refcount，真正完成对当前page的引用的。

``` c
// * @mapping: the address_space to search
// * @offset: the page cache index
struct page *find_get_entry(struct address_space *mapping, pgoff_t offset)
{
    void **pagep;
    struct page *head, *page;

    rcu_read_lock();
repeat:
    page = NULL;
    pagep = radix_tree_lookup_slot(&mapping->i_pages, offset); // 仅查找pagecache
    if (pagep) {
        page = radix_tree_deref_slot(pagep);
        if (unlikely(!page))
            goto out;
        if (radix_tree_exception(page)) {
            if (radix_tree_deref_retry(page))
                goto repeat;
            /*
             * A shadow entry of a recently evicted page,
             * or a swap entry from shmem/tmpfs.  Return
             * it without attempting to raise page count.
             */
            goto out;
        }

        head = compound_head(page);
        if (!page_cache_get_speculative(head)) // 真正获取该page，即修改page的refcount++。
            goto repeat;

        /* The page was split under us? */
        if (compound_head(page) != head) {
            put_page(head);
            goto repeat;
        }

        /*
         * Has the page moved?
         * This is part of the lockless pagecache protocol. See
         * include/linux/pagemap.h for details.
         */
        if (unlikely(page != *pagep)) {
            put_page(head);
            goto repeat;
        }
    }
out:
    rcu_read_unlock();

    return page;
}

```

#### readpage函数分析

```c
// 上述generic_file_buffered_read中，有调用mapping->a_ops->readpage函数。
	// 在ramfs中(fs/ramfs/inode.c)
static const struct address_space_operations ramfs_aops = {
    .readpage   = simple_readpage,
    .write_begin    = simple_write_begin,
    .write_end  = simple_write_end,
    .set_page_dirty = __set_page_dirty_no_writeback,
};
```

关于ramfs的读操作函数simple_readpage，如果能调用到这里，说明要读的page之前在内存中是不存在的，因此也不会有数据。这里会将page对应的一块pagesize大小的地址清零，然后返回作为是最新的数据。正常来说不会调用到这里。

``` c
int simple_readpage(struct file *file, struct page *page)
{
    clear_highpage(page);     // memset
    flush_dcache_page(page); // 清除page->flags中的PG_dcache_clean位
    SetPageUptodate(page);   // 设置page->flags中的PG_uptodate位
    unlock_page(page);       // 解锁page
    return 0;
}

// clear_highpage中，不仅是针对HIGHMEM，arm64中没有highmem的概念。这里是单纯的clear page。
static inline void clear_highpage(struct page *page)
{
    void *kaddr = kmap_atomic(page); 
    	// kmap_atomic中，先关抢占，关pagefault，然后调用page_address(page)
    clear_page(kaddr); // memset PAGESIZE
    kunmap_atomic(kaddr); // 开pagefault使能，开抢占。其他啥也没干。
}
// 关于kmap_atomic， 逐层分解：
#define page_address(page) lowmem_page_address(page)  ==》 return page_to_virt(page);
#define virt_to_page(x)     pfn_to_page(virt_to_pfn(x))
#define virt_to_pfn(x)      __phys_to_pfn(__virt_to_phys((unsigned long)(x)))

void unlock_page(struct page *page)
{
    BUILD_BUG_ON(PG_waiters != 7);
    page = compound_head(page);
    VM_BUG_ON_PAGE(!PageLocked(page), page);
    if (clear_bit_unlock_is_negative_byte(PG_locked, &page->flags))
        // 清除PG_locked位，同时测试PG_waiters是否置位，如果有说明有人在等这个page，就wake up一下。
        wake_up_page_bit(page, PG_locked);
}
```

#### 主体copy函数：copy_folio_to_iter

```c
copy_folio_to_iter -> copy_page_to_iter ->循环调用 _copy_to_iter

size_t copy_page_to_iter(struct page *page, size_t offset, size_t bytes,
                         struct iov_iter *i)
{
        size_t res = 0;
        page += offset / PAGE_SIZE; // first subpage
        offset %= PAGE_SIZE;
        while (1) {     // 拷贝的size是看传入的size与page_size的大小，单次最大不超过page_size;
                size_t n = __copy_page_to_iter(page, offset, 
                                min(bytes, (size_t)PAGE_SIZE - offset), i);
                res += n;
                bytes -= n;
                if (!bytes || !n)
                        break;
                offset += n;
                if (offset == PAGE_SIZE) {
                        page++;
                        offset = 0;
                }
        }
        return res;
}    
        
size_t _copy_to_iter(const void *addr, size_t bytes, struct iov_iter *i)
{
        if (unlikely(iov_iter_is_pipe(i))) // // 是否为pipe管道的copy,若是则调用memcpy
                return copy_pipe_to_iter(addr, bytes, i);
        if (iter_is_iovec(i))
                might_fault();
        iterate_and_advance(i, bytes, base, len, off, // 判断iter是用户态还是内核态
                copyout(base, addr + off, len), // 用户态，则copyout->raw_copy_to_user来操作
                memcpy(base, addr + off, len) // 内核态，直接memcpy
        )
        return bytes;
}
```





# write



``` c
write ->  __GI__libc_write(glibc)
    ->el0_sve -> __arm64_sys_write->ksys_write->vfs_write->__vfs_write
    	-> generic_file_write_iter
// 通用函数，write data to a file.
// iocb: IO state structure
// from: 含有数据的iov_iter结构体。
ssize_t generic_file_write_iter(struct kiocb *iocb, struct iov_iter *from)         
{
    struct file *file = iocb->ki_filp;                                             
    struct inode *inode = file->f_mapping->host;                               
    ssize_t ret;                                                                     
    inode_lock(inode);    // 首先获取inode锁
    ret = generic_write_checks(iocb, from);  
    if (ret > 0)                                                               
        ret = __generic_file_write_iter(iocb, from);  // 主体函数
    inode_unlock(inode);       // 释放inode锁                    

    if (ret > 0)
        ret = generic_write_sync(iocb, ret);
    return ret;
}

// inode_lock
static inline void inode_lock(struct inode *inode)               
{                                                                
    down_write(&inode->i_rwsem);                                 
} 
// inode_unlock
static inline void inode_unlock(struct inode *inode)
{
    up_write(&inode->i_rwsem);
}
```

### inode锁的获取释放

`inode_lock`函数获取写锁，调用`down_write`函数。`inode_unlock`函数释放写锁，调用`up_write`获取写锁。

####  rw_semaphore结构体

4.19内核版本：

![image-20220428144746499](D:\at_work\Documents\我的总结文档\images\image-20220428144746499.png)

5.10内核版本：

``` c
/*
 * For an uncontended rwsem, count and owner are the only fields a task
 * needs to touch when acquiring the rwsem. So they are put next to each
 * other to increase the chance that they will share the same cacheline.
 *
 * In a contended rwsem, the owner is likely the most frequently accessed
 * field in the structure as the optimistic waiter that holds the osq lock
 * will spin on owner. For an embedded rwsem, other hot fields in the
 * containing structure should be moved further away from the rwsem to
 * reduce the chance that they will share the same cacheline causing
 * cacheline bouncing problem.
 */
struct rw_semaphore {
    atomic_long_t count;                                           // + 00
    atomic_long_t owner;                                           // + 04
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* spinner MCS lock */       // + 08
#endif
    raw_spinlock_t wait_lock;                                      // + 0c
    struct list_head wait_list;                                    // + 10
#ifdef CONFIG_DEBUG_RWSEMS
    void *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

#### down_write 获取写锁

4.19内核版本

``` c
//    -> down_write 获取写锁
void __sched down_write(struct rw_semaphore *sem)           
{                                                           
    might_sleep();                                   
    rwsem_acquire(&sem->dep_map, 0, 0, _RET_IP_); // 只有DEBUG_LOCK_ALLOC使能才用
                                                            
    LOCK_CONTENDED(sem, __down_write_trylock, __down_write); // 先__down_write_trylock（对wait_lock结构体加spinlock锁同时关中断），若获取成功则直接加锁；否则需要调用正式的__down_write()来获取锁。
    rwsem_set_owner(sem);  // WRITE_ONCE(sem->owner, current);
}
```

5.10内核版本

```c
void __sched down_write(struct rw_semaphore *sem)
{
    might_sleep();
    rwsem_acquire(&sem->dep_map, 0, 0, _RET_IP_); // 只有DEBUG_LOCK_ALLOC使能才用
    LOCK_CONTENDED(sem, __down_write_trylock, __down_write);
     // LOCK_CONTENDED宏，只有在CONFIG_LOCK_STAT=y时才会去调用__down_write_trylock函数，否则
    // #define LOCK_CONTENDED(_lock, try, lock)  lock(_lock) //直接调用__down_write函数
}
//  -> __down_write_trylock
static inline int __down_write_trylock(struct rw_semaphore *sem)
{
    long tmp;
	......
    tmp  = RWSEM_UNLOCKED_VALUE;
    if (atomic_long_try_cmpxchg_acquire(&sem->count, &tmp,
                        RWSEM_WRITER_LOCKED)) { 
        rwsem_set_owner(sem);	
        return true;
    }
    return false;
}
// -> __down_write
static inline void __down_write(struct rw_semaphore *sem)
{
    long tmp = RWSEM_UNLOCKED_VALUE;

    if (unlikely(!atomic_long_try_cmpxchg_acquire(&sem->count, &tmp,
                              RWSEM_WRITER_LOCKED))) // count变量的cas操作
        rwsem_down_write_slowpath(sem, TASK_UNINTERRUPTIBLE); //真正用到osq_lock的地方
    else
        rwsem_set_owner(sem);//  owner变量的STSET
        				//	atomic_long_set(&sem->owner, (long)current);
}
```

##### osq_lock

调用栈：` down_write -> rwsem_down_write_slowpath -> rwsem_optimistic_spin -> osq_lock`





#### up_write 释放写锁

4.19 & 5.10内核版本实现一致。

``` c
//    -> up_write 释放写锁
void up_write(struct rw_semaphore *sem)
{
    rwsem_release(&sem->dep_map, _RET_IP_); // 只有DEBUG_LOCK_ALLOC使能才用
    __up_write(sem);
}
//         -> __up_write函数实现
static inline void __up_write(struct rw_semaphore *sem)
{
    long tmp;
    ......
    rwsem_clear_owner(sem); // owner变量的STSET操作，atomic_long_set(&sem->owner, 0);
    tmp = atomic_long_fetch_add_release(-RWSEM_WRITER_LOCKED, &sem->count); //count 变量的ldaddl，减一操作，具体实现待考证。
    if (unlikely(tmp & RWSEM_FLAG_WAITERS))
        rwsem_wake(sem, tmp);
}
```





### 内核write主体函数：__generic_file_write_iter

关于函数主体`__generic_file_write_iter`继续分析。

``` c
 __generic_file_write_iter
 // 先更新struct inode中的mtime/ctime：   
     file_update_time -> inode_update_time -> inode->i_op->update_time钩子函数真正更新
 // 判断是否为direct IO操作，若不是则直接调用generic_perform_write函数。
     


ssize_t generic_perform_write(struct file *file,
                struct iov_iter *i, loff_t pos)
{
     do {
        struct page *page;
         offset = (pos & (PAGE_SIZE - 1));
         bytes = min_t(unsigned long, PAGE_SIZE - offset, iov_iter_count(i));
	     ...
        status = a_ops->write_begin(file, mapping, pos, bytes, flags,
                        &page, &fsdata); // 分配并获取内核态的page

        copied = copy_page_from_iter_atomic(page, offset, bytes, i); // 执行copy操作，从用户态copy到内核态，每次操作bytes字节，这里bytes要么是PAGE_SIZE，要么就是不足PAGE_SIZE的剩余长度。
        flush_dcache_page(page); // 当前page标记为clean状态，写完后会标记为dirty状态。

        status = a_ops->write_end(file, mapping, pos, bytes, copied,
                        page, fsdata); // 将内核态的数据写入最终的磁盘位置，但实际上此时数据不一定真正写到磁盘上，因为大多数数据写为异步I/O
     } while(iov_iter_count(i)); // i->count表示剩余需要copy的byte数。
}
```

上面讲过，a_ops在ramfs中的write_begin/write_end钩子函数分别是simple_write_begin/simple_write_end。

- 关于write_begin:

  - Linux文件系统VFS层将应用程序的写入数据分割成页面大小。对于 每个页面，VFS会检查其是否已经为其创建了buffer_head结构，若没有创建，则为其创建 buffer_head；若已经创建则检查每个buffer_head的状态，包括buffer_head是否已经与物理磁盘块建立映射等.
  - 在ramfs中，调用的simple_write_begin只会看pagecache中是否有合适的page，若没有，则直接返回error。

- 关于write_end:

  - 对于块设备，需要调用generic_write_end通过相应的driver写入块设备的磁盘；
  - 对于non-block-device，如ramfs，可以调用simple_write_end（与simple_readpage搭配使用），只做了写page后最简单的操作。


下面分析这两个函数。

#### write_begin

``` c
simple_write_begin //
    -> grab_cache_page_write_begin // 创建或找到一个page，返回locked page。
    	-> pagecache_get_page
```

在函数`grab_cache_page_write_begin`中，设置了`fgp_flags`为`FGP_LOCK|FGP_WRITE|FGP_CREAT`，因此，传递给pagecache_get_page的参数就与read流程不同了。

``` c
struct page *grab_cache_page_write_begin(struct address_space *mapping,
                    pgoff_t index, unsigned flags)
{
    struct page *page;
    int fgp_flags = FGP_LOCK|FGP_WRITE|FGP_CREAT;

    if (flags & AOP_FLAG_NOFS)
        fgp_flags |= FGP_NOFS;

    page = pagecache_get_page(mapping, index, fgp_flags,
            mapping_gfp_mask(mapping));
    if (page)
        wait_for_stable_page(page);

    return page;
}
```



``` c
#define FGP_LOCK        0x00000002
#define FGP_CREAT       0x00000004
#define FGP_WRITE       0x00000008
#define FGP_NOFS        0x00000010 // ramfs中，包含且只包含上述4项。
struct page *pagecache_get_page(struct address_space *mapping, pgoff_t offset,
        int fgp_flags, gfp_t gfp_mask)
{
    struct page *page;
    repeat:
        page = find_get_entry(mapping, offset); // rs + cas
		......
        if (fgp_flags & FGP_LOCK) { // 会进入
            if (fgp_flags & FGP_NOWAIT) { // 不进入
					...
            } else {
                lock_page(page); // go here.
                		// 将page->flags的PG_locked位置位。
	......
    return page;
}
```

#### write_end

``` c
int simple_write_end(struct file *file, struct address_space *mapping,
            loff_t pos, unsigned len, unsigned copied,
            struct page *page, void *fsdata)
{
    struct inode *inode = page->mapping->host;
    loff_t last_pos = pos + copied;

    /* zero the stale part of the page if we did a short copy */
    if (!PageUptodate(page)) {
        if (copied < len) {
            unsigned from = pos & (PAGE_SIZE - 1);

            zero_user(page, from + copied, len - copied);
        }
        SetPageUptodate(page);
    }
    /*
     * No need to use i_size_read() here, the i_size
     * cannot change under us because we hold the i_mutex.
     */
    if (last_pos > inode->i_size)
        i_size_write(inode, last_pos);

    set_page_dirty(page); // 标记为dirty
    unlock_page(page);    // 释放锁
    put_page(page);       // _refcount--，这里page应该不会释放，因为page是一个file所有的。

    return copied;
}
```

#### 主体copy函数：copy_page_from_iter_atomic

```c
copy_page_from_iter_atomic ->
    __iterate_and_advance // 判断iov是用户态还是内核态地址，分别调用copyin和memcpy函数
if 用户态： 
    copyin -> raw_copy_from_user(to, from, n);
if 内核态：
    memcpy(p+off, base, len);
    
```







# lseek

``` c
lseek系统调用 -> ksys_lseek ->
SYSCALL_DEFINE3(lseek, unsigned int, fd, off_t, offset, unsigned int, whence)
{
    return ksys_lseek(fd, offset, whence);
}

```


# stat

查看文件状态，也是syscall的一种。stat在glibc 2.29中调用了79号syscall，在内核中（include/uapi/asm-generic/unistd.h）对应的是sys_newfstatat，可以查看fs/file.c中的定义。 调用关系：

``` c
__do_sys_newstat
    -> vfs_statx
    	-> 1. user_path_at->user_path_at_empty->filename_lookup 
    		// 将路径名转化成struct path结构体。
        -> 2. vfs_getattr // 核心函数，获取文件状态。
    	-> 3. path_put // path结构体引用释放
    		-> dput // 释放path->dentry
    		-> mntput
```

dentry的引用加一的时候：

``` c
// 将路径名转化成struct path结构体。
filename_lookup 
-> path_lookupat()
    -> complete_walk
	    -> unlazy_walk
		    -> legitimize_path
 			   -> lockref_get_not_dead  // 会将dentry的引用+1
    			// 使用的是cas操作，如果锁没人用，则直接atomic操作；如果锁有人用，
    			// 则需要spinlock先抢锁再操作。
```



dentry的引用减一的时候：

``` c
dput
    -> fast_dput // 快速路径
    	-> lockref_put_return // 通过cas将dentry->d_lockref->count减一
    // 若快速不成功，则进入慢速，这时已经有了dentry->d_lock锁了。
    -> retain_dentry // // 通过cas将dentry->d_lockref->count减一
    	// 如果当前dentry有d_delete的钩子则调用，则count不能减到0，最多减到1.如/dev/null
    	-> case 1: lockref_put_or_lock 
    		-> 可以直接调用的情况： 直接cas操作
    		-> 不可以直接调用的情况： 先spinlock住dentry->d_lockref->lock，然后cas，最后释放lock。
            // 判断标准： old.lock.rlock.raw_lock == NULL，也就是没有人获得锁。
    	// 否则，调用下面的，可以减到0.
    	-> case 2: lockref_put_return
    
```

关于lockref结构体：

``` c
struct lockref { // 共16 Byte
    union {
#if USE_CMPXCHG_LOCKREF
        aligned_u64 lock_count; // 8Byte对齐，      8 Byte
#endif
        struct {
            spinlock_t lock;    // 4 Byte
            int count;          // 4 Byte
        };
    };
};

```

关于dentry结构体：

``` c
struct dentry {
    /* RCU lookup touched fields */
    unsigned int d_flags;       /* protected by d_lock */
    seqcount_t d_seq;       /* per dentry seqlock */
    struct hlist_bl_node d_hash;    /* lookup hash list */
    struct dentry *d_parent;    /* parent directory */
    struct qstr d_name;
    struct inode *d_inode;      /* Where the name belongs to - NULL is
                     * negative */
    unsigned char d_iname[DNAME_INLINE_LEN];    /* small names */

    /* Ref lookup also touches following */
    struct lockref d_lockref;   /* per-dentry lock and refcount */
    const struct dentry_operations *d_op;
    struct super_block *d_sb;   /* The root of the dentry tree */
    unsigned long d_time;       /* used by d_revalidate */
    void *d_fsdata;         /* fs-specific data */

    union {
        struct list_head d_lru;     /* LRU list */
        wait_queue_head_t *d_wait;  /* in-lookup ones only */
    };
    struct list_head d_child;   /* child of parent list */
    struct list_head d_subdirs; /* our children */
    /*
     * d_alias and d_rcu can share memory
     */
    union {
        struct hlist_node d_alias;  /* inode alias list */
        struct hlist_bl_node d_in_lookup_hash;  /* only for in-lookup ones */
        struct rcu_head d_rcu;
    } d_u;

    /* negative dentry under this dentry, if it's dir */
    atomic_t d_neg_dnum;
    KABI_RESERVE(1)
    KABI_RESERVE(2)
} __randomize_layout;

```



# 性能相关

文件操作中，有两个地方与性能相关：

1. auditd
   1. 若auditd开启，则相关的文件操作会被audit跟踪记录。
2. security
   1. 若CONFIG_SECURITY开启，会在inode操作（如getattr）时进行安全hook的检查，对每个安全相关的hook一一排查，看看是否有不符合的。
3. trace
   1. 应该也会起作用，会trace各种操作，比如security的操作也会进行记录。

# pagecache的作用

https://zhuanlan.zhihu.com/p/38681930

pagecache是为了将实际IO的内容缓存在内存，从而提升访问效率而提出的概念。

**pageCache的特点：**

1. pageCache在vfs层面，可以屏蔽具体的fs实现，以及屏蔽具体的dev类型（比如blk dev，char dev等）。

2. pageCache以page为单位进行缓存，可以直接缓存文件内容，相比bufferCache以block为单位的操作，可以不用查找到具体fs的具体block，而可以直接通过inode号找到文件的具体内容。

   ​	tips:  访问磁盘时，是通过磁盘索引也就是block的具体位置来进行访问，bufferCache做的就是这个操作，block位置与page的映射，是具体fs自己来实现的，通过指针来关联起来。因此，对于某个具体的file的访问，就需要从vfs到具体fs，再转换成bio的位置，再交给磁盘驱动去读写。pageCache可以只到vfs，而不用到下面的其他操作。

**pagecache的结构：**

1. pageCache是通过radix tree来管理的。
2. 



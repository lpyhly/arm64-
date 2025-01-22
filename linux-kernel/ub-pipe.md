# ub-pipe

## pipe源码解读

pipe()函数先调用sys_pipe2创建管道描述符，同时关联上ops操作，当对该fd进行read(), write()时，会调用pipe_read(), pipe_write()进行pipe的读写操作。

* pipe文件系统初始化
  * 注册pipefs
    * register_filesystems

* pipe系统调用
  * pipe调用do_pipe
  * do_pipe()
    * f1&f2 get_empty_filep分配filep数据结构
    * inode = get_pipe_inode()从pipe文件系统获得inode
      * new_inode()
      * pipe_new()新建pipe
        * __get_free_pages(GFP_USER)为该pipe分配一页内存（4KB）
        * inode->i_pipe = kmalloc(sizeof(struct pipe_inde_info), GFP_KERNEL)分配pipe信息结构
    * i&j = get_unused_fd()获取两个fd
    * dentry = d_alloc()从pipefs分配dentry
    * d_add(dentry, inode)将inode插入到dentry中
    * 将f1设置成O_RDONLY，将f2设置成O_WRONLY
    * 进程的files列表中，files[i] = f1, files[j] = f2

* 实现函数
  * pipe
    * pipe_read
    * pipe_write



## 创建管道

sys_pipe-> sys_pipe2->do_pipe

1. 创建2个struct file，pipefifo_fops的钩子函数赋过去。（create_pipe_files）
2. 将管道相关的2个fd复制回用户态。

``` c
// 针对pipe的文件操作实例
const struct file_operations pipefifo_fops = {
     .open          = fifo_open, 
     .llseek          = no_llseek,
     .read          = new_sync_read,
     .read_iter     = pipe_read,
     .write          = new_sync_write,
     .write_iter     = pipe_write,
     .poll          = pipe_poll,
     .unlocked_ioctl     = pipe_ioctl,
     .release     = pipe_release,
     .fasync          = pipe_fasync,
};
```

其中，当调用read接口时，会ksys_read->vfs_read，然后调用file->f_op->read()接口，这里是new_sync_read。这个函数会将传入的file/buf/len/offset封装为struct kiocb和iov_iter，然后调用.read_iter钩子函数，进入到pipe_read()来处理。

``` c
static ssize_t new_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
{
    struct iovec iov = { .iov_base = buf, .iov_len = len };
    struct kiocb kiocb;
    struct iov_iter iter;
    ssize_t ret;

    init_sync_kiocb(&kiocb, filp);
    kiocb.ki_pos = (ppos ? *ppos : 0);
    iov_iter_init(&iter, READ, &iov, 1, len);

    ret = call_read_iter(filp, &kiocb, &iter);
    BUG_ON(ret == -EIOCBQUEUED);
    if (ppos)
        *ppos = kiocb.ki_pos;
    return ret;
}
```



## 相关结构体

pipe是一个ring buffer，由struct pipe_inode_info结构体标识。有head，tail指针，以及最大容量limit。可以通过tail - head来判断pipe内当前数据量。

```c
// 旧内核：4.19
struct pipe_inode_info { // 共136 bytes，且不会有对齐，就是说只保证8bytes对齐而已。
    struct mutex mutex;  // 32 bytes（实测）
    wait_queue_head_t wait; // 24 byte
    unsigned int nrbufs, curbuf, buffers; // 12 byte
    unsigned int readers; // 以下6个共24 bytes
    unsigned int writers;
    unsigned int files;
    unsigned int waiting_writers;
    unsigned int r_counter;
    unsigned int w_counter;
    // 以上共92byte，但后面一个是8byte，所以会有4byte的空余，所以共92+4=96bytes.
    struct page *tmp_page; // 以下5个共40bytes
    struct fasync_struct *fasync_readers;
    struct fasync_struct *fasync_writers;
    struct pipe_buffer *bufs;
    struct user_struct *user;
};
```



``` c
/**
 *  struct pipe_inode_info - a linux kernel pipe
 *  @mutex: mutex protecting the whole thing
 *  @rd_wait: reader wait point in case of empty pipe
 *  @wr_wait: writer wait point in case of full pipe
 *  @head: The point of buffer production
 *  @tail: The point of buffer consumption
 *  @max_usage: The maximum number of slots that may be used in the ring
 *  @ring_size: total number of buffers (should be a power of 2)
 *  @tmp_page: cached released page
 *  @readers: number of current readers of this pipe
 *  @writers: number of current writers of this pipe
 *  @files: number of struct file referring this pipe (protected by ->i_lock)
 *  @r_counter: reader counter
 *  @w_counter: writer counter
 *  @fasync_readers: reader side fasync
 *  @fasync_writers: writer side fasync
 *  @bufs: the circular array of pipe buffers
 *  @user: the user who created this pipe
 **/
struct pipe_inode_info {
    struct mutex mutex;
    wait_queue_head_t rd_wait, wr_wait;
    unsigned int head;
    unsigned int tail;
    unsigned int max_usage;
    unsigned int ring_size;
    unsigned int readers;
    unsigned int writers;
    unsigned int files; 					// 所关联的文件
    unsigned int r_counter;
    unsigned int w_counter;
    struct page *tmp_page;
    struct fasync_struct *fasync_readers;
    struct fasync_struct *fasync_writers;
    struct pipe_buffer *bufs;				// pipe buf, 真正存数据的地方。
    struct user_struct *user;
};
```

其中，bufs是真正存数据的地方，由struct pipe_buffer所承载。包含data所在的page/offset/len。

``` c
/**
 *  struct pipe_buffer - a linux kernel pipe buffer
 *  @page: the page containing the data for the pipe buffer
 *  @offset: offset of data inside the @page
 *  @len: length of data inside the @page
 *  @ops: operations associated with this buffer. See @pipe_buf_operations.
 *  @flags: pipe buffer flags. See above.
 *  @private: private data owned by the ops.
 **/
struct pipe_buffer {
    struct page *page;
    unsigned int offset, len;
    const struct pipe_buf_operations *ops;
    unsigned int flags;
    unsigned long private;
};
```

![img](http://img.blog.csdn.net/20150824013717117?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## write管道



管道写函数通过将字节复制到 VFS 索引节点指向的物理内存而写入数据，而管道读函数则通过复制物理内存中的字节而读出数据。当然，内核必须利用一定的机制同步对管道的访问，为此，内核使用了锁、等待队列和信号。
当写进程向管道中写入时，它利用标准的库函数write()，系统根据库函数传递的文件描述符，可找到该文件的 file 结构。file 结构中指定了用来进行写操作的函数（即写入函数）地址，于是，内核调用该函数完成写操作。写入函数在向内存中写入数据之前，必须首先检查 VFS 索引节点中的信息，同时满足如下条件时，才能进行实际的内存复制工作：
1.内存中有足够的空间可容纳所有要写入的数据；
2.内存没有被读程序锁定。
如果同时满足上述条件，写入函数首先锁定内存，然后从写进程的地址空间中复制数据到内存。否则，写入进程就休眠在 VFS 索 引节点的等待队列中，接下来，内核将调用调度程序，而调度程序会选择其他进程运行。写入进程实际处于可中断的等待状态，当内存中有足够的空间可以容纳写入 数据，或内存被解锁时，读取进程会唤醒写入进程，这时，写入进程将接收到信号。当数据写入内存之后，内存被解锁，而所有休眠在索引节点的读取进程会被唤醒。


Linux 管道对阻塞之前一次写操作的大小有限制。 专门为每个管道所使用的内核级缓冲区确切为 4096 字节。 除非reader清空管道，否则一次超过 4K 的写操作将被阻塞。 实际上这算不上什么限制，因为读和写操作是在不同的线程中实现的。（若64k pagesize，则最大为64k）

http://luodw.cc/2016/08/01/pipeof/

``` c
static ssize_t pipe_write(struct kiocb *iocb, struct iov_iter *from)
{
    struct file *filp = iocb->ki_filp;
    struct pipe_inode_info *pipe = filp->private_data;
    unsigned int head;
    ssize_t ret = 0;
    size_t total_len = iov_iter_count(from);
    ssize_t chars;
    bool was_empty = false;
    bool wake_next_writer = false;
    ...
    // 先判断pipe管道是否为空，若不空说明此时没有reader在等待（否则数据早就被取走了），这个标记会在后面使用。
    was_empty = pipe_empty(head, pipe->tail);
    chars = total_len & (PAGE_SIZE-1);  // 一次最多写一个page的size。
    if (chars && !was_empty) { // 若非空，则尝试把新数据merge到上一个buf中
        unsigned int mask = pipe->ring_size - 1;
        struct pipe_buffer *buf = &pipe->bufs[(head - 1) & mask];
        int offset = buf->offset + buf->len;

        if (pipe_buf_can_merge(buf) && offset + chars <= PAGE_SIZE) {
            ret = pipe_buf_confirm(pipe, buf);
            if (ret)
                goto out;

            ret = copy_page_from_iter(buf->page, offset, chars, from);
            if (unlikely(ret < chars)) {
                ret = -EFAULT;
                goto out;
            }

            buf->len += ret;
            if (!iov_iter_count(from))
                goto out;
        }
    }
	
    // 此时pipe是空的，可以继续执行pipe_write操作。  
    for (;;) {
        head = pipe->head;
        if (!pipe_full(head, pipe->tail, pipe->max_usage)) {
            unsigned int mask = pipe->ring_size - 1;
            struct pipe_buffer *buf = &pipe->bufs[head & mask];
            struct page *page = pipe->tmp_page;
            int copied;

            if (!page) {
                page = alloc_page(GFP_HIGHUSER | __GFP_ACCOUNT); // 分配一个page
                pipe->tmp_page = page;
            }
            spin_lock_irq(&pipe->rd_wait.lock);
            head = pipe->head;
            pipe->head = head + 1; 			// buf编号+1， 即使用新的buf
            spin_unlock_irq(&pipe->rd_wait.lock);

            /* Insert it into the buffer array */
            buf = &pipe->bufs[head & mask];	// 将page填充到新的buf中，此时page还是空的。
            buf->page = page;
            buf->ops = &anon_pipe_buf_ops;
            buf->offset = 0;
            buf->len = 0;
            buf->flags = 0;
            pipe->tmp_page = NULL;

            copied = copy_page_from_iter(page, 0, PAGE_SIZE, from); // 真正执行copy动作的地方，每次固定copy PAGE_SIZE大小。
            ret += copied;
            buf->offset = 0;
            buf->len = copied;

            if (!iov_iter_count(from))	// pipe中的内容都copy完了就可以退出循环了。
                break;
        } // if
        if (!pipe_full(head, pipe->tail, pipe->max_usage))
            continue;
    } // for
    __pipe_unlock(pipe);
    if (was_empty) { // 若写之前pipe是空的，那有可能有reader在等待，所以写完后在这里唤醒reader过来读。
    	wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPOLLRDNORM);
    	kill_fasync(&pipe->fasync_readers, SIGIO, POLL_IN);
	}
    if (ret > 0 && sb_start_write_trylock(file_inode(filp)->i_sb)) {
        int err = file_update_time(filp);  // 更新 mtime和ctime
        if (err)
            ret = err;
        sb_end_write(file_inode(filp)->i_sb);
    }
    return ret;
}


```

其中，真正执行从user到kernel的copy动作的是copy_page_from_iter，依次调用 -> copy_page_from_iter_iovec -> copyin

``` c
static size_t copy_page_from_iter_iovec(struct page *page, size_t offset, size_t bytes,
             struct iov_iter *i)
{
        kaddr = kmap_atomic(page);
        to = kaddr + offset;
        /* first chunk, usually the only one */
        left = copyin(to, buf, copy);
}
static int copyin(void *to, const void __user *from, size_t n)
{
    if (access_ok(from, n)) {
        kasan_check_write(to, n);
        n = raw_copy_from_user(to, from, n);
    }
    return n;
}

/* arch/arm64中的实现 */
#define raw_copy_from_user(to, from, n)                 \
({                                  \
    unsigned long __acfu_ret;                   \
    uaccess_enable_not_uao();                   \
    __acfu_ret = __arch_copy_from_user((to),            \
                      __uaccess_mask_ptr(from), (n));   \
    uaccess_disable_not_uao();                  \
    __acfu_ret;                         \
})
/* arch/arm64/lib/copy_from_user.S  */
SYM_FUNC_START(__arch_copy_from_user)
    add end, x0, x2
#include "copy_template.S"
    mov x0, #0              // Nothing to copy
    ret
SYM_FUNC_END(__arch_copy_from_user)
    
/* arch/x86的实现 */
static __always_inline unsigned long
raw_copy_from_user(void *to, const void __user *from, unsigned long n)
{
    return __copy_user_ll(to, (__force const void *)from, n);
}
unsigned long __copy_user_ll(void *to, const void *from, unsigned long n)
{
    __uaccess_begin_nospec();
    if (movsl_is_ok(to, from, n))
        __copy_user(to, from, n);	// 这里调用rep movsl指令来执行copy任务
    else
        n = __copy_user_intel(to, from, n); // 这里调用大片的movl指令
    __uaccess_end();
    return n;
}

```

最后的copy_template.S中调用ldp/stp进行copy操作，64B一次循环。值得注意的是，这里软件来保证源地址是16B对齐的，而目的地址的对齐处理由硬件来保证。（ Copy a buffer from src to dest (alignment handled by the hardware)） ？？？





## read管道

struct file -> read()钩子关联到pipe_read(), 参数为struct kiocb *iocb和iov_iter *to， 前者包含了内核态的pipe信息，后者包含了用户态的接收数据缓冲区地址和大小。

主要做了以下事情：

1. 先用自己结构体中的mutex_lock锁住pipe，

2. 判断当前pipe是否有内容，若没有内容，则释放cpu，执行interruptible的调度。

3. 若有内容，则循环执行以下内容直到pipe中head = tail，即pipe为空：将iocb->pipe->bufs[head]结构体内容填充到to->pipe->bufs[head]中，这里只是单纯的拷贝了struct page指针/offset/len等信息，并更新head++。回去继续判断源pipe是否还有未读的内容。

4. 释放锁

5. 到了这里说明pipe中已经没有要读的数据了，因此设置interruptible的schedule并让出cpu，若被中断，则返回，否则继续判断。

6. file_access()设置file的access time。这里的file是源文件描述符（iocb->ki_filp)，也就是用户态调用read()接口时传入的fd[0]。

    

其中，步骤3由函数copy_page_to_iter调用copy_page_to_iter_pipe()承载；步骤1/4由\__pipe_lock/__pipe_unlock实现，步骤5由函数wait_event_interruptible_exclusive调用wake_up_interruptible实现，主要方式为for(;;) {if(confition} break; schedule()}.

管道的读取过程和写入过程类似。但是，进程可以在没有数据或内存被锁定时立即返回错误信息，而不是阻塞该进程，这依赖于文件或管道的打开模式。反之，进程可以休眠在索引节点的等待队列中等待写入进程写入数据。当所有的进程完成了管道操作之后，管道的索引节点被丢弃，而共享数据页也被释放。

pipe_read函数在2019年底进行了修改，先看修改前的版本，4.19内核：

``` c
// 4.19的旧版本 - 这里精简一下
static ssize_t
pipe_read(struct kiocb *iocb, struct iov_iter *to)
{
        size_t total_len = iov_iter_count(to);
        struct file *filp = iocb->ki_filp;
        struct pipe_inode_info *pipe = filp->private_data;
        int do_wakeup;
        ssize_t ret;

        do_wakeup = 0;
        ret = 0;
        __pipe_lock(pipe);
        for (;;) {
                int bufs = pipe->nrbufs;
                if (bufs) { // 如果有需要读的内容
                        int curbuf = pipe->curbuf;
                        struct pipe_buffer *buf = pipe->bufs + curbuf;
                        size_t chars = buf->len;
                        size_t written;
                        // 读进来                    	
                        written = copy_page_to_iter(buf->page, buf->offset, chars, to);
                        
                        ret += chars;
                        buf->offset += chars;
                        buf->len -= chars;
                        if (!buf->len) { // buf->len = 0，表示当前buf已经读完了。
                                pipe_buf_release(pipe, buf);
                                curbuf = (curbuf + 1) & (pipe->buffers - 1);
                                pipe->curbuf = curbuf;
                                pipe->nrbufs = --bufs;
                                do_wakeup = 1; // 设置唤醒标志，可以去唤醒写进程了。
                        }
                        total_len -= chars;
                        if (!total_len) // 已经达到用户设置的最大可读字节数，就可以返回了。
                                break;  /* common path: read succeeded */
                }
                if (bufs)       /* More to do? */
                        continue;
                if (!pipe->writers) // 实测pipe->writers = 1, pipe->waiting_writers = 0.
                        break;
                if (!pipe->waiting_writers) {
                        /* syscall merging: Usually we must not sleep
                         * if O_NONBLOCK is set, or if we got some data.
                         * But if a writer sleeps in kernel space, then
                         * we can wait for that data without violating POSIX.
                         */
                        if (ret)
                                break;
                        if (filp->f_flags & O_NONBLOCK) {
                                ret = -EAGAIN;
                                break;
                        }
                }
                if (signal_pending(current)) { // 检查是否有信号需要处理
                        if (!ret)
                                ret = -ERESTARTSYS;
                        break;
                }
                if (do_wakeup) { // 由于单次pipe_write只能写一个buf，因此当前buf读完了并且还没达到reader要求读的字节数，就唤醒writer继续写。
                        wake_up_interruptible_sync_poll(&pipe->wait, EPOLLOUT | EPOLLWRNORM);
                        kill_fasync(&pipe->fasync_writers, SIGIO, POLL_OUT);
                }
                pipe_wait(pipe); // 如果当前pipe中没有读到东西且writers还处于waiting状态（sleep）或没有writer，则会执行到这里。
        }
        __pipe_unlock(pipe);

        /* Signal writers asynchronously that there is more room. */
        if (do_wakeup) {
                wake_up_interruptible_sync_poll(&pipe->wait, EPOLLOUT | EPOLLWRNORM);
                kill_fasync(&pipe->fasync_writers, SIGIO, POLL_OUT);
        }
        if (ret > 0)
                file_accessed(filp);
        return ret;
}
```



``` c
// 5.7的内核对pipe_read进行了修改。
static ssize_t
pipe_read(struct kiocb *iocb, struct iov_iter *to)
{
    size_t total_len = iov_iter_count(to);
    struct file *filp = iocb->ki_filp;
    struct pipe_inode_info *pipe = filp->private_data;
    bool was_full, wake_next_reader = false;
    ssize_t ret;

    __pipe_lock(pipe);  // s1

    was_full = pipe_full(pipe->head, pipe->tail, pipe->max_usage);
    for (;;) {
        unsigned int head = pipe->head;
        unsigned int tail = pipe->tail;
        unsigned int mask = pipe->ring_size - 1;

        if (!pipe_empty(head, tail)) {
            struct pipe_buffer *buf = &pipe->bufs[tail & mask];
            size_t chars = buf->len;
            size_t written;
            written = copy_page_to_iter(buf->page, buf->offset, chars, to); // s2
            ret += chars;
            buf->offset += chars;
            buf->len -= chars;
            total_len -= chars;
            if (!pipe_empty(head, tail))    /* More to do? */  跳到for(;;)的开头继续copy
                continue;
        }
        if (ret)
            break;	// 只有真正读到信息才会跳出循环，否则会一直阻塞在for循环中。
        wake_next_reader = true;
    }	// for循环结束
    if (pipe_empty(pipe->head, pipe->tail))
        wake_next_reader = false;
    __pipe_unlock(pipe);	// 解锁

    if (wake_next_reader)  // 跳出循环后，又有新的信息到了。
        wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPOLLRDNORM);
    if (ret > 0)
        file_accessed(filp);	// 设置文件的atime
    return ret;
}
```

## 睡眠与唤醒

 pipe_write热点中看到有wake_up_interruptible_sync_poll占比高，这个是说第一次pipe读完后，发现又有新消息到了，所以需要再唤醒下一个reader来读。

这里使用的是同步的wakeup函数，这个函数会设置wake_flags标志位，从而在调度选择cpu时，保证自己不会被调度到别的cpu去。这个函数通常用在当前task知道自己会很快被调度回来的情况。同时，由于当前进程和将被wakeup的进程在同一个cpu上，也可以防止新的进程被唤醒后抢占当前进程，因为当前进程做完后自己调度出去，才能让后面的进程进来。

这里，pipe_read发现有新的信息要写，就可以同步调用pipe_write进来写，反正写完后又会将pipe_read调度回来去读。

### 唤醒

```c
wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPILLRDNORM)
    -> __wake_up_sync_key
        -> __wake_up_common_lock(&pipe->rd_wait, TASK_INTERRUPTIBLE, 1, WF_SYNC, EPOLLIN | EPILLRDNORM)

static void __wake_up_common_lock(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key)
{
    unsigned long flags;
    wait_queue_entry_t bookmark;

    bookmark.flags = 0;
    bookmark.private = NULL;
    bookmark.func = NULL;
    INIT_LIST_HEAD(&bookmark.entry);

    do {
        spin_lock_irqsave(&wq_head->lock, flags); // 获得锁，同时关中断防止被打断。
        nr_exclusive = __wake_up_common(wq_head, mode, nr_exclusive,
                        wake_flags, key, &bookmark);
        spin_unlock_irqrestore(&wq_head->lock, flags);
    } while (bookmark.flags & WQ_FLAG_BOOKMARK);
}
```

调用了__wake_up_common，这是wakeup函数的核心代码，其中的参数nr_exclusive标识是唤醒一切任务，还是只唤醒non-exclusive的任务和一个特定的exclusive任务。这里nr_exclusive=1，是后者。

``` c
static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key,
            wait_queue_entry_t *bookmark)
{
    wait_queue_entry_t *curr, *next;
    int cnt = 0;

    lockdep_assert_held(&wq_head->lock);
    // 从rd_wait队列中取出第一个
    curr = list_first_entry(&wq_head->head, wait_queue_entry_t, entry);

    list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
        unsigned flags = curr->flags;
        int ret;
        if (flags & WQ_FLAG_BOOKMARK)
            continue;

        ret = curr->func(curr, mode, wake_flags, key); // 这里是调用的rd_wait所在结构体的func钩子;其中rd_wait是struct wait_queue_Entry的head。
        if (ret < 0)
            break;
        if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;

        if (bookmark && (++cnt > WAITQUEUE_WALK_BREAK_CNT) &&
                (&next->entry != &wq_head->head)) {
            bookmark->flags = WQ_FLAG_BOOKMARK;
            list_add_tail(&bookmark->entry, &next->entry);
            break;
        }
    }

    return nr_exclusive;
}
                                                                                
```

### 睡眠

再看睡眠，pipe_read中的热点有pipe_wait。这个函数会设置INTERRUPTIBLE然后将自己调度出去。

``` c
void pipe_wait(struct pipe_inode_info *pipe)
{
    DEFINE_WAIT(rdwait); 
    DEFINE_WAIT(wrwait);

    /*
     * Pipes are system-local resources, so sleeping on them
     * is considered a noninteractive wait:
     */
    prepare_to_wait(&pipe->rd_wait, &rdwait, TASK_INTERRUPTIBLE); // 将当前任务加入到wq_entry队列中，并将任务状态设置为INTERRUPTIBLE。
    prepare_to_wait(&pipe->wr_wait, &wrwait, TASK_INTERRUPTIBLE);
    pipe_unlock(pipe); // 释放锁并调度出去。
    schedule();
    finish_wait(&pipe->rd_wait, &rdwait); // 再次调度到当前节点后，设置当前任务状态为RUNNING
    finish_wait(&pipe->wr_wait, &wrwait);
    pipe_lock(pipe); // 获取锁
}

// DEFINE_WAIT
```

其中，DEFINE_WAIT()用来设置wait_queue，其中的func函数就是唤醒时调用的那个函数。这里是autoremove_wake_function，这个函数的作用就是从队列wq_entry中拿下来第一个任务并唤醒。后续操作见上述代码注释。

``` c
#define DEFINE_WAIT_FUNC(name, function)                    \
    struct wait_queue_entry name = {                    \
        .private    = current,                  \
        .func       = function,                 \
        .entry      = LIST_HEAD_INIT((name).entry),         \
    }

#define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)
```

pipe_wait函数会在


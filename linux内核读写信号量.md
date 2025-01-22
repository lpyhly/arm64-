# linux 内核读写信号量

[TOC]



## 概述

- 若writer来了：
  - 发现rwsem为空，则修改count为ACTIVE_WRITE, 修改owner为自己，做事情去了；
  - 发现当前有writer在做，
    - 没有人等，则获取osq_lock后，在owner自旋等当前的writer释放owner，就可以做了。
    - 有其他writer在等owner（不在wait list中），则writer2一定获取了osq lock，其他writer都需要自旋等osq lock。
    - 有其他reader在等，此时reader只能在wait list中。那么writer依然可以获取osq lock后，在owner自旋，伺机插队。
    - 有其他reader和writer在等，此时reader在wait list中，writer spin在owner上，则其他writer需要自旋等osq lock。
  - 发现当前有reader在做， 则不能自旋在osq lock了。
    - 没有人等，则把自己加入到wait list中，然后睡过去等唤醒。
    - 有其他writer在等，同上。
- 若reader来了：
  - 发现rwsem为空，则修改count为ACTIVE_READ, 修改owner为anonymous，做事情去了；
  - 若发现有1个或几个reader在读，直接增加count值，做事情去了。
  - 若发现有writer在写，则直接进入到wait list中等待。
- 若writer走了：修改owner为空，count减去ACTIVE_WRITE。
  - 看看wait队列，若为空就不管了。
  - wait队列有人，则rwsem_wake唤醒第一个waiter。
    - 若为writer，则唤醒它就好了。
    - 若为reader，则唤醒所有队列头的reader，并增加count值为reader数目。
- 若reader走了：count减一
  - 看看wait队列，若为空就不管了。
  - wait队列有人，则rwsem_wake第一个waiter（一定是writer）。
    - 若为writer，则唤醒它就好了。
    - （不会发生）若为reader，则唤醒所有队列头的reader，并增加count值为reader数目。

## 读写信号量的特点

1. 同一时刻最多有一个写者（writer）获得锁；

2. 同一时刻可以有多个读者（reader）获得锁；

3. 同一时刻写者和读者不能同时获得锁；

由于读者可以同时获得锁，因此提高了系统的并发程度，进而提高了系统的性能。

![img](D:\at_work\Documents\我的总结文档\images\up_write.png)



下面是当锁处于不同状态时锁的内部状态。后面将描述Linux的实现是如何保证锁能够正确工作。

1. 锁空闲。reader=0, writer=0, count=0, wait_list=empty。
2. 读者获得锁且等待链表空。reader=N, writer=0, count=0x0000000N, wait_list=empty。
3. 读者获得锁且等待链表非空。reader=N, writer=0, count=0xffff000N, wait_list=not-empty。
4. 写者获得锁且等待链表空。reader=0, writer=1, count=0xffff0001, wait_list=empty。
5. 写者获得锁且等待链表非空。reader=0, writer=1, count=0xfffe0001, wait_list=not-empty。

## rw_semphore结构体

读写锁的实现，是使用rw_semphore信号量结构体。

``` c
struct rw_semaphore {                            // offside: 
    atomic_long_t count;　　　　　　　　　　　　　　　// 0xa0
    struct list_head wait_list;                  // 0xa8
    raw_spinlock_t wait_lock;                    // 0xb8
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq;            // 0xbc
    /*
     * Write owner. Used as a speculative check to see
     * if the owner is running on the cpu.
     */
    struct task_struct *owner;                   // 0xc0
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

各个成员变量的作用：

1. **count**： 用来标识当前读用户数、写用户数、等待队列用户数；64 bit，高32位用于waiting队列（用count

   .waiting标识，只有0和0xffffffff两种情况），低32位用于active队列（用count.active标识）。其中，

   - 空闲状态： count.active = 0 && count.waiting = 0
- 只有读者： count.active > 0 && count.waiting = 0
   - 只有写者： count = 0xffffffff00000001(即count.active=0x1, count.waiting=0xffffffff)
   - 写者到来： count.waiting  = 0xffffffff. 可能是获取写锁成功，也可能是waiting的写者。
     - 若count.active = 1，可能有一个成功拿到锁的写者，也可能有一个读者和一个waiting的写者；
     - 若count.active = x，可能有x-1个读者和一个waiting的写者，或者x个读者和一个waiting的写者。

2. **wait_lock**:  锁，当wait队列中的用户想去操作count时，先要获取到这个锁，是qspinlock的锁。

3. **wait_list**：等待列表，用于管理在该信号量上睡眠的任务。

4. **osq_lock**：锁，对于首次修改count失败后的用户，需要先获取到osq lock，才能再次发起对count的修改。若已经自旋在osq lock了，就不需要进入waiting队列了。

5. **owner**: 当前获取锁的进程的task_struct。

   - 若有多个读者，则owner为最后一个获取锁的读者。
   - reader释放锁的时候，不会去修改owner；
   - writer释放锁的时候，会将owner置为NULL。

   owner可能有4个值：

   - 0
   - READER_OWNED // 有或曾经有reader
   - ANONYMOUSLY_OWNED // 目前有writer，其他writer不能spin在owner上。
   - 其他值  // 目前有writer，其他writer可以spin在owner上。

## read lock的宏观流程

![img](D:\at_work\Documents\我的总结文档\images\SouthEast)

## write lock的宏观流程

![img](D:\at_work\Documents\我的总结文档\images\SouthEast2)

## 读写锁的应用

### task_struct ->mmap中的读写锁

task_struct -> mmap为struct mm_struct 类型的结构体，其中包含mmap_sem变量，为struct rw_semaphore类型的读写锁。

当进程创建时，会调用dup_mm -> dup_mmap函数，先获取写锁，然后对所有vma进行copy操作，最后再释放写锁。

- 一个进程只有一把读写锁。

``` c
static __latent_entropy int dup_mmap(struct mm_struct *mm,
                    struct mm_struct *oldmm)
{
    if (down_write_killable(&oldmm->mmap_sem)) { // 获取父进程的mmap的信号量
        retval = -EINTR;
        goto fail_uprobe_end;
    }
    flush_cache_dup_mm(oldmm);
    uprobe_dup_mmap(oldmm, mm);
    /*
     * Not linked in yet - no deadlock potential:
     */
    down_write_nested(&mm->mmap_sem, SINGLE_DEPTH_NESTING);  // 获取子进程的mmap的信号量
......
    for (mpnt = oldmm->mmap; mpnt; mpnt = mpnt->vm_next) {
        tmp = vm_area_dup(mpnt);
        anon_vma_fork(tmp, mpnt); // 处理反向映射？ 
        tmp->vm_flags &= ~(VM_LOCKED | VM_LOCKONFAULT);
        tmp->vm_next = tmp->vm_prev = NULL;
        file = tmp->vm_file;
        if (file) { // 如果是文件，则
            struct inode *inode = file_inode(file);
            struct address_space *mapping = file->f_mapping;

            get_file(file);
            if (tmp->vm_flags & VM_DENYWRITE)
                atomic_dec(&inode->i_writecount);
            i_mmap_lock_write(mapping); 
             // 直接调用down_write，传参为
             // oldmm->mmap->vm_file->f_mapping.
             // 这里可能会出现
            if (tmp->vm_flags & VM_SHARED)
                atomic_inc(&mapping->i_mmap_writable);
            flush_dcache_mmap_lock(mapping);
            /* insert tmp into the share list, just after mpnt */
            vma_interval_tree_insert_after(tmp, mpnt,
                    &mapping->i_mmap);
            flush_dcache_mmap_unlock(mapping);
            i_mmap_unlock_write(mapping);
        }

out:
    up_write(&mm->mmap_sem);
    flush_tlb_mm(oldmm);
    up_write(&oldmm->mmap_sem);
    dup_userfaultfd_complete(&uf);
}

```

会调用down_write函数的有以下几类，但只有前两个会进入到osq lock，也就是file based情况下。估计其他情况的rwsem可以顺利获取，而file based情况下传参为一个**共享的sem变量**，因为在获取到vm_file时已经指向了同一个文件。

**unlink_file_vma** 

**dup_mmap**

anon_vma_clone

unlink_anon_vmas

```  c
do_exit -> exit_mm -> mmput -> exit_mmap
-> free_pgtables 
    -> unlink_anon_vmas
    -> unlink_file_vma
```

``` c
_do_fork -> dup_mm -> dup_mmap
    -> anon_vma_fork
    	-> anon_vma_clone
    -> if(file) {i_mmap_lock_write} -> down_write
```



# 详细代码分析

以32位的count值为例，高16bit代表的是`waiting part`，低16bit代表的是`active part`；

- `RWSEM_UNLOCKED_VALUE`：值为0，表示锁未被持有，没有读者也没有写者；
- `RWSEM_ACTIVE_BIAS`：值为1，，该值用于定义`RWSEM_ACTIVE_READ_BIAS`和`RWSEM_ACTIVE_WRITE_BIAS`；
- `RWSEM_WAITING_BIAS`：值为-65536，当有任务需要加入到等待列表中时，count值需要加`RWSEM_WAITING_BIAS`，有任务需要从等待列表中移除时，count值需要减去`RWSEM_WAITING_BIAS`；
- `RWSEM_ACTIVE_READ_BIAS`：值为1，当有读者去获取锁的时候，count值将加`RWSEM_ACTIVE_READ_BIAS`，释放锁的时候，count值将减去`RWSEM_ACTIVE_READ_BIAS`；
- `RWSEM_ACTIVE_WRITE_BIAS`，值为-65535，当有写者去获取锁的时候，count值将加`RWSEM_ACTIVE_WRITE_BIAS`，释放锁的时候，count值需要减去`RWSEM_ACTIVE_WRITE_BIAS`；

## down_write

![img](D:\at_work\Documents\我的总结文档\images\1771657-20200517220129970-1749261016.png)

### down_write函数源码解读

目的：

1. 要获取写锁，退出时一定是获取到了写锁，否则会在函数内等待。
2. 等待时会uninterruptible的睡过去，等wfe唤醒。

流程简介：

1. 二话不说，上来直接原子的修改count变量（ldadd）
2. 若此前没有writer也没有reader，则获取锁成功，将自己的task_struct写入owner就可以退出了。
3. 若前面有writer或者reader，则需要先回退自己刚写过的count变量（ldadd），
4. 若当前获取锁的是writer，则可以使用osq lock，自旋在osq lock中等待获取锁。
5. 当获取到osq lock后，还需要再去check当前的owner是否被前一个人释放（即为NULL），如果前一个人还没完成，就持续读owner，直到owner值发生变化，再重新去check此时owner是否为NULL。
6. 终于等到owner=NULL后，就可以继续修改count变量去获得写锁，然后修改owner为自己。
7. 若不可以使用osq lock（当前获取锁的是reader），或者osq lock失败了，则去获取wait_lock锁后(raw_spin_lock)，进入到waiting队列中，UNINTERRUPTIBLE的睡过去等待被wfe唤醒。
8. 每次醒来都会去cas瞅瞅自己能不能获取到count的ACTIVE锁，若不能，释放wait_lock继续睡过去。
9. 若能获取count的ACTIVE锁（即cas成功交换，说明前面所有的reader和writer都完成了），就可以欢快的醒来，获取wait_lock锁后退出waiting队列，前面的cas操作已经成功修改了count变量了，也就是ACTIVE锁已经获取到了。

变种：

- down_write_killable:   将uninterruptible睡过去修改为killable的睡过去，其余都一样；

- down_write_nested:   只有rwsem_acquire参数不同，而这个函数又只用于debug，所以认为与down_write是一致的。

``` c
/*
 * lock for writing
 */
void __sched down_write(struct rw_semaphore *sem)
{
    might_sleep();
    rwsem_acquire(&sem->dep_map, 0, 0, _RET_IP_);
    // 只有DEBUG_LOCK_ALLOC使能时才用到，这里没用。

    LOCK_CONTENDED(sem, __down_write_trylock, __down_write);
	// 默认直接调用__down_write(sem)
    // 但如果定义了CONFIG_LOCK_STAT： 
    // 则先__down_write_trylock（对wait_lock结构体加spinlock锁同时关中断），若获取成功则直接lock_acquired(锁的dep_map变量）， 否则需要调用正式的__down_write(sem)来获取锁。
    
    rwsem_set_owner(sem);
    // 就一句话：　WRITE_ONCE(sem->owner, current);
}
```

关于`__down_write`函数，有不同的实现，这里需要check一下arm64走的是哪一条分支。

1.  down_write函数是在文件"kernel/locking/rwsem.c"中，文件头首先include了<linux/rwsem.h>文件。
2.  在include/linux/rwsem.h中，会判断是否定义了宏CONFIG_RWSEM_GENERIC_SPINLOCK（历史遗留，arm64在4.x已经不会定义这个宏了）。
3. 若没有定义，则去include<asm/rwsem.h>，这里会去判断不同的架构下的rwsem实现。
4. arch/arm64/include/generated/asm/rwsem.h 文件中只有一句话，#include <asm-generic/rwsem.h>，也就是要去include/asm-generic/rwsem.h文件中查找。
5. 因此，下面研究include/asm-generic/rwsem.h的`__down_write`函数实现。

``` c
// 研究__down_write的实现：
static inline void __down_write(struct rw_semaphore *sem)
{
    long tmp;
    tmp = atomic_long_add_return_acquire(RWSEM_ACTIVE_WRITE_BIAS,
                         &sem->count); 
    // sem->count的atomic add操作，尝试获取写锁。
    if (unlikely(tmp != RWSEM_ACTIVE_WRITE_BIAS))
    // 仅当没有reader也没有writer时才能成功。
    // 具体实现：只有count的写锁部分之前是0，才可能成功；
    // 否则，一定有读者或者其他写者，因此进入到failed函数慢慢获取。
        rwsem_down_write_failed(sem);
    // 这里，就需要进入waiting队列等待了;
    // 会在这里面一直自旋（可能会sleep）直到等到前面所有人都做完，然后自己获取count的写锁。
}
```

``` c
 static inline struct rw_semaphore *
__rwsem_down_write_failed_common(struct rw_semaphore *sem, int state)
{
    long count;
    bool waiting = true; /* any queued threads before us */
    struct rwsem_waiter waiter;
    struct rw_semaphore *ret = sem;
    DEFINE_WAKE_Q(wake_q);

    /* undo write bias from down_write operation, stop active locking */
    count = atomic_long_sub_return(RWSEM_ACTIVE_WRITE_BIAS, &sem->count);
     // 撤销前面写count的动作。

    /* do optimistic spinning and steal lock if possible */
    if (rwsem_optimistic_spin(sem)) 
        return sem;
	// 再尝试获取锁，这里使用osq_lock锁，只要能spin就可以一直spin在osq_lock上等锁，
    // 直到前面的人都做完了，成功获取锁为止。直接返回。
    // 如果能获取到，则不需要再进waiting队列了。
    // 目测正常情况下都不需要再进waiting队列了。
     
    /*
     * Optimistic spinning failed, proceed to the slowpath
     * and block until we can acquire the sem.
     */
    waiter.task = current;
    waiter.type = RWSEM_WAITING_FOR_WRITE;
    // waiting队列需要标识两个域段： 1. 你是哪个任务。 2. 你是writer还是reader。

    raw_spin_lock_irq(&sem->wait_lock); // 获取wait_lock锁，为了操作wait_list.

    /* account for this before adding a new element to the list */
    if (list_empty(&sem->wait_list))  
        waiting = false;
	// 若wait_list中没有其他人，说明是第一个waiter
     
    list_add_tail(&waiter.list, &sem->wait_list); // 将自己添加到wait_list中。
    /* we're now waiting on the lock, but no longer actively locking */
    if (waiting) {
        // 若有其他waiter，则需要判断一个场景： 当前没有active的writer，但有wait的reader，那么就去叫醒其中一个reader过去工作啦。
        count = atomic_long_read(&sem->count);

        /*
         * If there were already threads queued before us and there are
         * no active writers, the lock must be read owned; so we try to
         * wake any read locks that were queued ahead of us.
         */
        if (count > RWSEM_WAITING_BIAS) {
            __rwsem_mark_wake(sem, RWSEM_WAKE_READERS, &wake_q);
            /*
             * The wakeup is normally called _after_ the wait_lock
             * is released, but given that we are proactively waking
             * readers we can deal with the wake_q overhead as it is
             * similar to releasing and taking the wait_lock again
             * for attempting rwsem_try_write_lock().
             */
            wake_up_q(&wake_q);

            /*
             * Reinitialize wake_q after use.
             */
            wake_q_init(&wake_q);
        }

    } else
        count = atomic_long_add_return(RWSEM_WAITING_BIAS, &sem->count);
	// 第一个waiter直接修改count值，告诉其他人有一个waiter了。

    /* wait until we successfully acquire the lock */
    set_current_state(state); // 设置为uninterrupible状态，方便睡过去。
     
    // 进入漫长的等ACTIVE流程。
    while (true) {
        if (rwsem_try_write_lock(count, sem))
            // cas唤醒，这里只是拿到了wait_lock，没有去拿osq lock。
            break;
        raw_spin_unlock_irq(&sem->wait_lock);
		// 等前面所有的active writer或reader写完，可以睡过去等唤醒。
        /* Block until there are no active lockers. */
        do {
            if (signal_pending_state(state, current))
                goto out_nolock;

            schedule();
            set_current_state(state);
        } while ((count = atomic_long_read(&sem->count)) & RWSEM_ACTIVE_MASK);

        raw_spin_lock_irq(&sem->wait_lock);
    }
    // 终于拿到count的ACTIVE状态啦，快离开waiting_list吧。
    __set_current_state(TASK_RUNNING);
    list_del(&waiter.list);
    raw_spin_unlock_irq(&sem->wait_lock);

    return ret;

// 若发现跑出去处理signal了，就退出waiting list，不再等锁了。
out_nolock:
    __set_current_state(TASK_RUNNING);
    raw_spin_lock_irq(&sem->wait_lock);
    list_del(&waiter.list);
    if (list_empty(&sem->wait_list))
        atomic_long_add(-RWSEM_WAITING_BIAS, &sem->count);
    else
        __rwsem_mark_wake(sem, RWSEM_WAKE_ANY, &wake_q);
    raw_spin_unlock_irq(&sem->wait_lock);
    wake_up_q(&wake_q);

    return ERR_PTR(-EINTR);
}
```



### up_write函数源码解读

释放写锁：

1. owner置为NULL，count--。
2. 若有waiter，则唤醒wait list中的第一个人。

``` c
// up_write: owner置为NULL，然后调用__up_write
// __up_write: count值减掉RWSEM_ACTIVE_WRITE_BIAS后，判断是否还有waiter，若有则rwsem_wake唤醒。

// qspinlock获取wait_lock，把count清零并唤醒第一个waiter，释放锁。
// __rwsem_wake: 从wait list中取出第一个waiter。
	// 1. 若第一个waiter是writer，则去唤醒这个进程(wake_up_process)，不会修改count的值。
	// 2. 若第一个waiter是reader，则需要先判断： 
	//     1） 若是被writer唤醒，则先atomic add改count，成功才放心的将owner设置为annoymous，继续。
    //     2） 若是被writer唤醒，但发现count已经被spin的writer偷偷改了，则放弃本次reader唤醒，ret。
	//     3） 若是被reader唤醒（即不是第一个被唤醒的reader），则直接继续下面的动作。
	// 3. 唤醒队列头的所有连续read-waiter，count增加上所唤醒的reader数。
    // 4. 若wait list中没有writer了，则count.waiting=0.
    // 5. 唤醒上面的所有reader。
```



### rwsem_optimistic_spin源码解读

这里的osq lock只是用来保护count变量的。为了避免频繁去原子操作count变量。

流程：

 	1. 先尝试直接cas修改count变量；
 	2. 若成功，则直接返回，也没有osq lock什么事了。
 	3. 若失败，则说明有竞争，
      	1. ldadd回退count变量值。
      	2. 尝试获取osq lock，减少count变量的频繁修改。
      	3. 当拿到osq lock后，再去cas操作count型变量，完成后释放osq lock。

``` c
static bool rwsem_optimistic_spin(struct rw_semaphore *sem)
{
    bool taken = false;

    preempt_disable();

    /* sem->wait_lock should not be held when doing optimistic spinning */
    if (!rwsem_can_spin_on_owner(sem)) 
        // 是否可以spin在owner上，若不能则直接返回false
        // 如果owner是reader，则不能spin
        goto done;

    if (!osq_lock(&sem->osq)) // osq_lock，若不成功则返回false
        goto done;

    /*
     * Optimistically spin on the owner field and attempt to acquire the
     * lock whenever the owner changes. Spinning will be stopped when:
     *  1) the owning writer isn't running; or
     *  2) readers own the lock as we can't determine if they are
     *     actively running or not.
     */
    while (rwsem_spin_on_owner(sem)) { 
        // rwsem_spin_on_owner函数：
        // 需要check当前持锁的是否完成，即owner是否为NULL，
        // 若不是，则一直读owner变量，直到owner发生变化才返回。
        /*
         * Try to acquire the lock
         */
        if (rwsem_try_write_lock_unqueued(sem)) {
            // 判断count值是否为0或者只有waiting list的人，
            // 若是，则cas将count置为RWSEM_ACTIVE_WRITE_BIAS，并写owner为自己。
            // 若不是，则返回false。
            taken = true;
            break;
        }

        /*
         * When there's no owner, we might have preempted between the
         * owner acquiring the lock and setting the owner field. If
         * we're an RT task that will live-lock because we won't let
         * the owner complete.
         */
        if (!sem->owner && (need_resched() || rt_task(current)))
            break;

        /*
         * The cpu_relax() call is a compiler barrier which forces
         * everything in this loop to be re-loaded. We don't need
         * memory barriers as we'll eventually observe the right
         * values at the cost of a few extra spins.
         */
        cpu_relax();
    }
    osq_unlock(&sem->osq);
done:
    preempt_enable();
    return taken;
}
```



### 锁操作梳理 

``` c
// rwsem_optimistic_spin的锁操作
1. owner的读操作(rwsem_can_spin_on_owner)
2. osq的xchg操作（osq_lock)，用于修改osq queue的tail指针。
3. owner的读操作（rwsem_spin_on_owner)
4. owner的读操作(rwsem_spin_on_owner)
5. 

```



``` c
    // 获取写锁
	1. count的add(__down_write)
     判断add后的值，若获取成功，则直接跳到第n步；
     否则，
    2. count的add(__rwsem_down_write_failed_common) // 回退count值
    3. rwsem_optimistic_spin // 尝试获取osq，详见该函数的锁操作梳理。
    n. owner的RU写(down_write)
    // 释放写锁
    1. owner的RU写(up_write)
    2. countd的ldadd减一操作（__up_write)
    3. 如果还有其他waiter，则进入rwsem_wake
        	- 
        
```



## down_read

``` c
// down_read: 调用__down_read（）函数，然后设置owner为READER_OWNED
// __down_read:
static inline void __down_read(struct rw_semaphore *sem)
{
    if (unlikely(atomic_long_inc_return_acquire(&sem->count) <= 0)) 
        // 直接count.active +1，表示reader+1.
        rwsem_down_read_failed(sem); 
    	// <=0意味着当前有人正在写，或者有writer在wait list中，
    	// 那么就需要等待了。
}

static inline struct rw_semaphore __sched *
__rwsem_down_read_failed_common(struct rw_semaphore *sem, int state)
{
    long count, adjustment = -RWSEM_ACTIVE_READ_BIAS; // 之前加了1，这里需要先-1.
    struct rwsem_waiter waiter;
    DEFINE_WAKE_Q(wake_q);

    waiter.task = current;
    waiter.type = RWSEM_WAITING_FOR_READ;

    raw_spin_lock_irq(&sem->wait_lock); // 先获取wait_lock，才能进入wait list。
    if (list_empty(&sem->wait_list)) 
        // 如果自己是wait list中的第一个，则需要更正count的值
        // 如果不是第一个，则别人已经更正过了。
        adjustment += RWSEM_WAITING_BIAS;
    list_add_tail(&waiter.list, &sem->wait_list); // 入队。

    /* we're now waiting on the lock, but no longer actively locking */
    count = atomic_long_add_return(adjustment, &sem->count); 
    	// count修改成WAITING_BIAS的状态。

    /*
     * If there are no active locks, wake the front queued process(es).
     *
     * If there are no writers and we are first in the queue,
     * wake our own waiter to join the existing active readers !
     */
    if (count == RWSEM_WAITING_BIAS ||
        (count > RWSEM_WAITING_BIAS &&
         adjustment != -RWSEM_ACTIVE_READ_BIAS))
        __rwsem_mark_wake(sem, RWSEM_WAKE_ANY, &wake_q);

    raw_spin_unlock_irq(&sem->wait_lock);
    wake_up_q(&wake_q);

    /* wait to be given the lock */
    while (true) {
        set_current_state(state);
        if (!smp_load_acquire(&waiter.task)) {
            // 在唤醒队列任务前，会先store_release将task置为NULL，来释放reader在这里的自旋。
            /* Matches rwsem_mark_wake()'s smp_store_release(). */
            break;
        }
        if (signal_pending_state(state, current)) { // state=UNINTERRUPTIBLE，所以不会进去。
            raw_spin_lock_irq(&sem->wait_lock);
            if (waiter.task)
                goto out_nolock;
            raw_spin_unlock_irq(&sem->wait_lock);
            break;
        }
        schedule();
    }

    __set_current_state(TASK_RUNNING);
    return sem;
out_nolock:
    list_del(&waiter.list);
    if (list_empty(&sem->wait_list))
        atomic_long_add(-RWSEM_WAITING_BIAS, &sem->count);
    raw_spin_unlock_irq(&sem->wait_lock);
    __set_current_state(TASK_RUNNING);
    return ERR_PTR(-EINTR);
}

```




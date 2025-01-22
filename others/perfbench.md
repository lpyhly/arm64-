# 测试脚本

``` shell
# sched
core=1
nr_th=16

cores=0-$((nr_th-1))

taskset -c $core perf bench sched messaging -g 1 -l 100
taskset -c $core perf bench sched messaging -g 1 -l 100 -p
taskset -c $core perf bench sched messaging -g 1 -l 100 -t
taskset -c $core perf bench sched messaging -g 1 -l 100 -t -p
taskset -c $cores perf bench sched messaging -g ${nr_th} -l 100
taskset -c $cores perf bench sched messaging -g ${nr_th} -l 100 -p
taskset -c $cores perf bench sched messaging -g ${nr_th} -l 100 -t
taskset -c $cores perf bench sched messaging -g ${nr_th} -l 100 -t -p

# syscall
taskset -c $core perf bench syscall basic -l 100

# mem
taskset -c $core perf bench mem find_bit

# futex
taskset -c $core perf bench futex hash -f 1 -t 1    # 单线程，1个futex
taskset -c $core perf bench futex hash -f 1024 -t 1    # 单线程多个futex
taskset -c $cores perf bench futex hash -f 1 -t ${nr_th}    # 多线程
taskset -c $cores perf bench futex hash -f 1024 -t ${nr_th}    # 多线程
taskset -c $core perf bench futex wake -t 1 -w 1        # 单线程
taskset -c $cores perf bench futex wake -t ${nr_th} -w 1       # 多线程
taskset -c $cores perf bench futex wake -t ${nr_th} -w 4
taskset -c $cores perf bench futex wake -t ${nr_th} -w ${nr_th}
taskset -c $core perf bench futex wake-parallel -t 1 -w 1        # 单线程
taskset -c $cores perf bench futex wake-parallel -t ${nr_th} -w 1       # 多线程
taskset -c $cores perf bench futex wake-parallel -t ${nr_th} -w 4
taskset -c $cores perf bench futex wake-parallel -t ${nr_th} -w ${nr_th}
taskset -c $core perf bench futex requeue -t 1 -q 1
taskset -c $cores perf bench futex requeue -t ${nr_th} -q 4
taskset -c $cores perf bench futex requeue -t ${nr_th} -q ${nr_th}
taskset -c $core perf bench futex lock-pi -t 1           # 单线程 单锁
taskset -c $core perf bench futex lock-pi -t 1 -M        # 单线程 多锁
taskset -c $cores perf bench futex lock-pi -t ${nr_th}
taskset -c $cores perf bench futex lock-pi -t ${nr_th} -M

#epoll
taskset -c $core perf bench epoll ctl -t 1 -f 64
taskset -c $core perf bench epoll ctl -t 1 -f 64 -R
taskset -c $core perf bench epoll ctl -t ${nr_th} -f 64
taskset -c $core perf bench epoll ctl -t ${nr_th} -f 64 -R
taskset -c $core perf bench epoll wait -t 1 -f 64
taskset -c $core perf bench epoll wait -t 1 -R -f 64
taskset -c $core perf bench epoll wait -t 1 -B -f 64
taskset -c $core perf bench epoll wait -t 1 -E -f 64
taskset -c $core perf bench epoll wait -t 1 -S -f 64
taskset -c $core perf bench epoll wait -t 1 -R -f 64
taskset -c $core perf bench epoll wait -t 1 -m -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -R -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -B -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -E -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -S -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -R -f 64
taskset -c $cores perf bench epoll wait -t ${nr_th} -m -f 64

# internals
taskset -c $core perf bench internals synthesize -s # 单线程
taskset -c $core perf bench internals synthesize -t -m 1 -M ${nr_th}  # 多线程
taskset -c $core perf bench internals kallsyms-parse
taskset -c $core perf bench internals inject-build-id -m 1 -n 1
taskset -c $core perf bench internals inject-build-id -m 10 -n 10
taskset -c $core perf bench internals inject-build-id -m 100 -n 100

```

# 源码助手

总的列表参见： linux/tools/perf/builtin-bench.c

具体实现参见： linux/tools/perf/bench/*

# sched

messaging和pipe两个子项，几乎是把hackbench的实现搬过来的。

## messaging

每个group有20个发送端 到 20个接收端，每次write 100 Byte（这些都是代码中固定的），循环次数/group数目可以用户指定，没有配置绑核方式的地方。

参数： -p： 支持pipe创建和socketpair创建连接两种方式；默认socketpair方式

​         -t： 多线程 or 多进程，默认进程。

​        -g： group数目，group间是没有关系的，只是并发而已。默认group=10

​        -l： 循环次数，默认100次，每次通信中发送信息的次数，默认100次。

原理详见《[hackbench](onenote:benchmark项目介绍.one#hackbench&section-id={19809E89-E15F-4516-A367-DACD164B9C7F}&page-id={A53798CF-8966-4FF5-95AD-A9FFFF77DD0D}&base-path=//C:/Users/h00424781/Documents/OneNote 笔记本/688)》

这里，与lmbench和unixbench的实现都不一样。

lmbench： 多个进程间链式串行模式通信；

unixbench-ctx：两个进程两个pipe管道，每个进程都监听自己的read，收到后就向自己的write发信息，循环执行，单次发送4B（sizeof(long))，多进程之间互相独立。

perfbench：每个sender都向所有receiver发送，共发送loops * num_fds(接收端个数) 个信息，发完就退出。每个receiver都接收所有的发送信息。单次发送100 Byte。判断任务结束的方式是，所有sender都发完信息了，所有receiver都接收完该收的信息了（提前计算好），使用wait()接口/pthread_join（）阻塞接口。

## pipe

和unixbench实现一致，单次信息4Byte。只不过可以选择使用线程模式。

参数： -l: loop

​    -T: 线程还是进程

原理：

使用waitpid()接口/pthread_join（）阻塞接口。

# syscall

1. 1. 纯测试getppid接口的调用时间。

 

# mem

包含memcpy/memset/find_bit函数，提供get_cycles()和gettimeofday两种计数模式。

这里的cycles使用的是sys_perf_event_open(

参数： -s： 大小，可识别KB/MB/GB单位（大写）

​       -f： funciton： memcpy/memset

   -l: 循环次数

​       -c：使用cycles还是Bps作单位

 

##memcpy：

使用memset清0方式来初始化。

memcpy实现方式默认选用glibc的memcpy()，即直接调用当前系统中的memcpy()函数，动态链接。

在x86上又提供了其他3种方案，对应于arch/x86/lib/memcpy_64.S中的memcpy实现（内核中的实现）。包含unrolled版本，movsq版本，movsb版本。

 

使用方式：

perf bench mem memcpy -f help

perf bench mem memcpy -f all

##memset:

同上。

 

##find_bit:

首先遍历 bitmap的长度，从 1 bits -> 2048 bits，2次幂增加。对于每种长度，再遍历set为1的bit位数，从1 -> bitmap长度，2次幂递增。set的时候，bit位数平均选择，比如2048bits下set 256个bit，就会选择bit 0, 8, 16, 32……

 

在主循环中，提供了两种实现方式，

1. 会调用for_each_set_bit函数，去对每个被set为1的位进行操作。
2. 遍历每个bit，判断是否为0/1，使用test_bit函数。

操作很简单，就是两个全局变量的加法操作，一个用来计数（accumulator），一个用来保序&防编译优化。

static unsigned int accumulator;

static unsigned int use_of_val;

static noinline void workload(int val)

{

​    use_of_val += val;

​    accumulator++;

}

参数： -I inner_iterations： 内层迭代次数。单次迭代操作就是一个for_each_set_bit，即遍历bitmap中所有set的bit们。

​      -o outer_iterations： 外层迭代次数，和上面作用差不多。

 

# futex

有hash，wake，wake-parallel, requeue, lock-pi几种子项。

- worker： 多个子线程，每个子线程的工作就是：操作threads_starting变量减一，直到全部线程都减一后，pthread_cond_signal来告知parent进程去解锁futex。 子线程wait在thread_worker变量上等待parent去唤醒。

- - 醒来后，就可以进入主要循环了： 不断的通过某种futex函数去操作futex，最后计数就是该futex函数的完成速率。

 

- parent：创建子线程们，然后futex在自己的thread_parent的futex上等待子线程全部进入主要循环任务（即thread_starting减到0后被唤醒），然后向子线程们发信号唤醒它们（pthread_cond_broadcast直接广播唤醒所有子线程）。

- - 接下来就sleep一段时间，这段时间留给子线程们去futex_wait()操作，最后令全局变量done=TRUE，让子线程们同时退出循环，最后去统计时间和ops。

- 测试过程中，线程是一一绑核的。

- 参数：

- - t：线程数
  - r：运行次数（默认为10次）
  - S：共享futex，是给内核的attr参数，表示该futex是当前进程独享的，只用于进程内的线程间同步，从而实现性能加速。
  - m：调用mlockall()，锁住所有现在及未来的内存。将进程全部va空间加锁，防止swap。

不同子项的worker进程中的主循环是不同的。

## hash：

- 调用futex_wait()，每个worker依次调用nfutexes个futex锁（可传参，-f指定），每个worker的锁都是独立的，所以互相之间完全没有竞争。一个ops对应的是一个futex锁。 
- 测试的就是无竞争状态下futex_wait()函数对某个地址作hash去寻找其对应结构体的能力。

 

## lock-pi: 带优先级继承的futex

- 一个ops对应的是： 调用futex_lock_pi和futex_unlock_pi函数，中间会usleep(1)。锁可以是全局一把，也可以是每个线程独享，通过参数(-M)传递。

 

## requeue：

子线程们通过futex_wait或futex_wait_requeue_pi(根据-p参数传递是否使用pi版本futex）来睡眠，父线程唤醒子线程后先sleep 0.1s等子线程们都进入到wait，然后才启动机时：

- 调用futex_cmp_requeue/futex_cmp_requeue_pi唤醒某个子线程，并将最多q个子线程转移到futex2的队列上，q通过参数指定（-q），默认为1.返回值为被唤醒的线程数，或者是被requeue到futex2的线程数。
- 不断重复上述过程，直到所有nthreads个线程要么被唤醒，要么被转移到futex2，就结束计时。

结束计时后，再把futex2上的睡眠线程全部唤醒，pthread_join做完自己的事情后消亡，下一次循环时再重新pthread_create创建新线程，当然这些操作已经不会记录了。

上述过程为1次循环，用户可以指定循环次数，通过-r参数。

 

## wake：

通过-r指定循环次数，单次循环内容：

子线程futex_wait()等待，父线程循环调用futex_wake()，直到把所有子线程都唤醒。子线程们共享一把futex锁。

 

## wake_parallel:

有多个线程并行来做唤醒动作。

总线程数： nthreads， 一一绑核的那种

负责唤醒动作的线程数： nwakes， -w指定，互相之间并行。 

单次循环： 

每个线程需要通过futex_wait()唤醒nthreads/nwakes个线程，负责唤醒动作的线程自己统计这个时间，最后求和。

 

# epoll

epoll机制会注册一个新的文件系统eventpollfs，是红黑树实现的，对于epoll_create就会返回一个文件描述符fd，属于这个文件系统。与select的区别在于，后者是轮询，文件数最多1024个；而epoll采用回调函数机制，若有事件发生，会自行调用callback函数，并且内核态与用户态通过mmap共享，减少赋值开销，文件数上限是最大可以打开的文件数目（/proc/sys/fs/file-max， ulimit -n）。

文件都比较活跃时，select效率更高，因为不需要大量回调函数； 反之，则epoll效率更高，因为不需要轮询。

主要就3个函数： epoll_create， epoll_ctl， epoll_wait

``` c
// 注册要监听的事件类型，包括ADD，MOD，DEL的操作，先ep_find进行红黑树的查找。
int epoll_create(int size); // 参数无意义
// 创建一个eventpoll对象，分配一个文件描述符作为返回值，后面就用这个epoll描述符来操作了。
```



``` c
// 注册要监听的事件类型，包括ADD，MOD，DEL的操作，先ep_find进行红黑树的查找。
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
/* 
- epfd：  就是epoll_create创建的对象，后面的epoll_wait参数也是这个对象；
- op： ADD/MOD/DEL 注册新的fd到epfd中/修改fd/删除fd
- fd： 需要监听的fd
epoll_event需要配置的成员由events和data。
events可以是以下几个宏的集合：
 EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
 EPOLLOUT：表示对应的文件描述符可以写；
 EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
 EPOLLERR：表示对应的文件描述符发生错误；
 EPOLLHUP：表示对应的文件描述符被挂断；
 EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
 EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
data为文件描述符，一般配置为event.data.fd = fd，这个fd就是socket或者是的。
返回值： 0为成功，否则失败。
*/
```

``` c
// 等待事件的产生，一定是epoll_ctl中注册的事件。
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
/* 
events: 用户态需要先分配好内存，内核会把事件复制到events数组中，供用户态直接查看。
maxevents： 本次可以返回的最大事件数据
timeout：超时时间，单位ms， 若为0表示epoll_wait不会等待立即返回。
*/
```



## epoll-ctl

单纯测试epoll-ctl的接口性能，不会真的收到事件，也就不会去调用回调函数。默认线程一一绑核，线程间是互相独立的，描述符也是线程独有的，没有共享。

参数： 

 	-t： 线程数，默认与当前环境核数一致。

​	-f：每个线程检测的文件描述符数目，默认64。

​	-r：运行时间（s）

​	-n： 不绑核，默认是绑核的

​	-R： 随机操作，随机fds

​	-v： verbose



父线程与子线程交互流程与futex一致，通过pthead_cond_t变量thread_parent和thread_worker来实现同步。当子线程一直在循环运行，直到父线程sleep一定时间后发送done就退出循环。

主循环中，若非random模式，则遍历每个文件描述符，并依次调用add/mod/del的epoll操作；若为random模式，则调用do_random_epoll_op，随机选择一个fd，随机选择一种操作。每次epoll操作中间会nanosleep 250 ns。



## epoll-wait

参数：

​	-t/-r/-f/-n/-R/-v同上；

​	-m： 多个epoll实例，即每个线程都有自己的epoll对象，若不选中则共用同一个epoll对象（默认选择，同epoll-ctl)；

​				多epoll实例的场景： httpd服务监听众多tcp短链接们。

​	-B： epoll_wait是否要阻塞式等待，影响的是epoll_ctl的传参；

​	-S/-E：都是epoll_ctl的传参调整。

epoll_wait同时监控多个文件描述符。有两种模式：

计时的部分为：

- do_threads 为每一个线程epoll_create创建epfd（若不指定-m则公用同一个epfd），然后对每个线程，都epoll_ctl将文件描述符们逐一添加到每一个epfd中。最后，pthread_create真正创建子线程们。
  - 子线程的workerfn会与父线程交互，未等待done信号前，每个子线程会epoll_wait等待，每次仅处理一个fd，当等到事件后调用read读取，ops++。

- 创建一个写线程，对所有线程的所有描述符依次 write()写入，若为random模式，则线程和fds都会先随机化。单次遍历完成后，nanosleep睡眠5ns后继续，直到计时结束后的1s后父线程置位wdone变量，才停止该写线程。

这里的计时时间比较长，在创建子线程前就开始计时了，不知道为啥。



# numa

参数：

-p：进程数

-t：每个进程的线程数

-G/-P/-L/-T： 总内存（MB）/进程内存(MB)/进程锁定内存/线程内存

-R/-W/-B/-Z/-r: 数据处理方式：读/写/后向/写零/随机漫步。

-z/-I/-0/-x:   初始化方案： bzero/随机初始化/仅cpu0初始化/每xs将线程0迁移到另一个核（核0 & 最后一个核间震荡）

-l/-s/-u: 最大loop数，最长时间（二者取最小值）， 每次loop间sleep的us数

-H: 是否使能thp，可以使能/关闭/保持默认，软件上会madvise()去配置thp。

-C： 绑核，前N个任务绑核到特定cpu（其余的不绑）

-M： 前N个任务绑到特定numa node（其余的不绑）



worker_thread:

​	对于global_data, process_data, thread_data, 加锁的process_data， 依次调用do_work去遍历访问全部bytes_**字节（用户可配），每次操作8Byte，访问方式读/写/读+写（用户指定），访问顺序用户指定。



# internals

perf内的功能们。

## synthesized

 对`/proc/<pid>/task/<pid>/`下的各级目录做遍历，然后只是对全局变量event_count做原子加，最后统计加的次数除以时间。 中间的perf操作比较复杂。大致是：

`perf_event__prepare_comm -> perf_event__synthesize_fork -> perf_tool__process_synth_event -> perf_event__synthesize_mmap_events`从而实现perf操作的执行。



## kallsyms

从`/proc/kallsyms`中依次读取每个字符，并进行处理，解析出来每一个symbol_type和 symbol_name，然后调用`bench_process_symbol`直到文件结尾。该函数啥也不做，直接return。

性能统计仅看时间。

``` c
$ tail /proc/kallsyms
ffff800008c148f0 t disable_discard      [dm_mod]
```

## inject-buildid

测试`perf inject -b --buildid-all`的性能，具体原理没看懂，大致意思是把perf.data的信息重新加工一下，增加一下想要的内容，从而供后面的perf report读取。这里增加的就是buildid相关的信息。

会调用`nftw`函数去遍历/usr/lib下所有层的每个文件，直到找到`nr_mmaps`个含有build_id的文件为止。然后对每个文件，构建一个struct perf_event，先填充好header，然后将stdout通过pipe管道写进这个perf_event中。这里，stdout就是`perf inject -b --buildid-all`的输出。
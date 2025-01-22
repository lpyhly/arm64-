[TOC]

# tips们

1. 不同时间接口的分类：

https://www.kernel.org/doc/html/latest/core-api/timekeeping.html#c.ktime_get_coarse_real_ts64

2. clocksource解析：

http://www.wowotech.net/timer_subsystem/clocksource.html



# arm架构的时间寄存器

## 1.1   以arch timer为基础的

硬件系统寄存器，每个核一个，与整系统同步，1620为100MHz

- **第一类： 当前时间寄存器**

| 寄存器名称    | 功能                                                         | 说明                                   |
| ------------- | ------------------------------------------------------------ | -------------------------------------- |
| `cntpct_el0`  | physical timer                                              |                                        |
| `cntvct_el0`  | virtual timer 软件读取当前时间用这个，在host中（包含user/kernel）=cntpct，在guest中=cntpct - cntvoff。 | msr/msr读写，不保序，需要isb自行保序。 |
| `cntvoff_el2` | offset                                                       |                                        |
|            |                            |                                                              |

​     ===》》》 单位是arch timer cycle数，对于100MHz的timer，则每个cycle = 10ns。

​    ===》》》 gettimeofday使用的就是cntvct的值。

  

- **第二类： 倒计时定时寄存器**

基本计时单位还是使用上述的`cntpct/cntvct`，只是会设置一个超时触发GIC中断的机制。因此，单位依然是arch timer cycle数。

| 寄存器名称    | 功能                  | 说明                                                         |
| ------------- | --------------------- | ------------------------------------------------------------ |
| CNTP_CVAL_EL0 | 设置一个最大值max_val | 当cntpct >=  max_val时，可以产生一个中断（是否中断由cntp_ctl控制) |
| CNTP_TVAL_EL0 | down-count的值        | 32-bit TimerValue register。该寄存器配合system  counter可以实现一个32 bit signed downcounter（有的时候，使用downcounter会让软件逻辑更容易，看ARM  generic timer的设计人员考虑的多么周到）。开始的时候，我们可以设定TimerValue寄存器的值为1000（假设我们想down count 1000，然后触发中断），向该寄存器写入1000实际上也就是设定了CompareValue register的值是system  counter值加上1000。随着system  counter的值不断累加，TimerValue register的值在递减，当值<=0的时候，便会向GIC触发中断 |
| CNTP_CTL_EL0  | 控制寄存器            |                                                              |

另外还有一套对应的虚拟定时器：   CNTV_CVAL_EL0, CNTV_TVAL_EL0, and CNTV_CTL_EL0



- **第三类：arch timer频率寄存器**

`cntfrq_el0`： 系统arch timer的频率。1620/688为100MHz，是soc的特性，cpu freq也是根据这个来分频得到3GHz的。

linux通过`arch_timer_get_cntfrq()`读取这个寄存器去获取`arch timer`的频率。注意，这里的`arch timer`与`/proc/interrupts`中的arch_timer不是一个概念。后者是gic定时上报中断的，定时时间是由CONFIG_HZ决定的，这个是软件可配置的，用于进程调度的。

## 1.2 以cpu timer为基础的

`pmccnt_el0`： 直接统计cpu的cycle数。

内核态可以直接查看，用户态需要先在内核态打开pmc寄存器后，才可以查看。

## 1.3 以jiffies为基础的

 每次timer中断时更新，更新时间间隔由CONFIG_HZ决定，一般为100，250，1000。 是纯软件行为，与linux调度一致。



# 时钟类型盘点

## 时钟类型clock

在Linux内核中，我们可以发现主要有这么几种不同类型的时钟（clock)：

1. CLOCK_REALTIME

2. CLOCK_MONOTONIC

3. CLOCK_ RAW

4. CLOCK_BOOTTIME

具体来说：

- CLOCK_REALTIME，可以理解为wall time，即是实际的时间。用户可以使用命令（date）或是系统调用去修改。如果使用了NTP， 也会被NTP修改。当系统休眠（suspend）时，仍然会运行的（系统恢复时，kernel去作补偿）。

 

- CLOCK_MONTONIC，是单调时间，即从系统开始到现在持续的时间。用户不能修改这个时间，但是当系统进入休眠（suspend）时，CLOCK_MONOTONIC是不会增加的。

 

- CLOCK_MONOTONIC_RAW，和CLOCK_MONOTONIC类似，但不同之处是MONOTONIC_RAW不会受到NTP的影响。CLOCK_MONOTONIC会受到NTP的影响，而是说当NTP server 和本地的时钟硬件之间有问题，NTP会影响到CLOCK_MONOTONIC的频率，但是MONOTONIC_RAW则不会受其影响。 其区别可以参考 Difference between MONOTONIC and MONOTONIC_RAW

 

- CLOCK_BOOTTIME，与CLOCK_MONOTONIC类似，但是当suspend时，会依然增加。可以参考LWN的这篇文章 introduce CLOCK_BOOTTIME

## 时钟源clocksource

表示时钟源的抽象结构体，通过`cat /sys/devices/system/clocksource/clocksource0/available_clocksource`可以查看系统可用时钟源。

clocksource结构体定义在：include/linux/clocksource.h，其中有个(*read)钩子函数，需要不同时钟源驱动自行实现，当读取clocksource结构体时就调用这个函数了。

### arm_arch_timer的驱动实现

``` c
// drivers/clocksource/arm_arch_timer.c
static struct clocksource clocksource_counter = {        
        .name   = "arch_sys_counter",                    
        .id     = CSID_ARM_ARCH_COUNTER,                 
        .rating = 400,                                   
        .read   = arch_counter_read,                     
        .flags  = CLOCK_SOURCE_IS_CONTINUOUS,            
};                                                       
```

 其中的read()操作最终调用的__arch_counter_get_cntvct()，就是`mrs cntvct`

`rating`代表精度，数值越高代表精度越高，用来帮助内核选择适当精度的timer，400已经是非常高了。

## timekeeper结构体

```c
// include/linux/timekeeper_internal.h
struct timekeeper {                                                           
        struct tk_read_base     tkr_mono;                                     
        struct tk_read_base     tkr_raw;                                      
        u64                     xtime_sec;                                    
        unsigned long           ktime_sec;                                    
        struct timespec64       wall_to_monotonic;                            
        ktime_t                 offs_real;                                    
        ktime_t                 offs_boot;                                    
        ktime_t                 offs_tai;                                     
        s32                     tai_offset;                                   
        unsigned int            clock_was_set_seq;                            
        u8                      cs_was_changed_seq;                           
        ktime_t                 next_leap_ktime;                              
        u64                     raw_sec;                                      
        struct timespec64       monotonic_to_boot;                            
                                                                              
        /* The following members are for timekeeping internal use */          
        u64                     cycle_interval;                               
        u64                     xtime_interval;                               
        s64                     xtime_remainder;                              
        u64                     raw_interval;                                            		 u64                     ntp_tick;                                                 
        s64                     ntp_error;                                    
        u32                     ntp_error_shift;                              
        u32                     ntp_err_mult;                                       
        u32                     skip_second_overflow;                         
#ifdef CONFIG_DEBUG_TIMEKEEPING                                               
        long                    last_warning;                                   
        int                     underflow_seen;                               
        int                     overflow_seen;                                
#endif                                                                        
};                                                                            
```

 其中，`tkr_mono`表示`CLOCK_MONOTONIC`类型的base structure

`xtime_sec`表示`CLOCK_REALTIME`的当前时间，单位为s（REAL需要结合RTC时间）。

`ktime_sec`表示`CLOCK_MONOTONIC`的当前时间，单位为s（MONO是单调递增的，和系统的时间域没有关系）

另外，realtime， boottime，tai的时间都从mono转换过去，raw的时间单独测试。

 

 tk_read_base结构体是定时器输出(timekeeping readout)的基础结构体，其中包含了clocksource结构体。

 ```c
struct tk_read_base {                                
        struct clocksource      *clock;              
        u64                     mask;                
        u64                     cycle_last;          
        u32                     mult;                
        u32                     shift;               
        u64                     xtime_nsec;          
        ktime_t                 base;                
        u64                     base_real;           
};                                                   
 ```

###  timekeeper结构体的更新

timekeeper结构体是在jiffies到时的时候更新，因此coarse的时间精度就与jiffies的时间相同。若CONFIG_HZ = 100，则10ms更新一次。



# 内核态获取时间

```c
ktime_t ktime_get(void)   // 用于可靠的时间戳和精确测量短时间间隔。在系统启动时启动，但在挂起期间停止

ktime_t ktime_get_boottime(void）//类似于ktime_get，但是但在挂起时不会停止例如，这可用于需要通过挂起操作与其他计算机同步的密钥过期时间。

ktime_t ktime_get_real(void)//返回相对于1970年的时间，类似于用户态的gettimeofday.这适用于需要在重新启动期间保持的所有时间戳，如inode时间，但在内部使用时应避免使用，因为它可能由于从用户空间执行的闰秒更新、NTP adjustment settimeofday（）操作而向后跳转

ktime_t ktime_get_raw(void)//类似于ktime_get，但运行速度与硬件时钟源相同，而不需要（NTP）调整时钟漂移。这在内核中也很少需要。

```

根据用户的要求，有一些变量以不同的格式返回时间：

``` c
u64 ktime_get_ns(void)
u64 ktime_get_boottime_ns(void)
u64 ktime_get_real_ns(void)
u64 ktime_get_clocktai_ns(void)
u64 ktime_get_raw_ns(void)

 
time64_t ktime_get_seconds(void)
time64_t ktime_get_boottime_seconds(void)
time64_t ktime_get_real_seconds(void)
time64_t ktime_get_clocktai_seconds(void)
time64_t ktime_get_raw_seconds(void)

void ktime_get_ts64(struct timespec64 *ts)；       //CLOCK_MONOTONIC
void ktime_get_real_ts64(struct timespec64 *)；    //CLOCK_REALTIME
void ktime_get_boottime_ts64(struct timespec64 *)；//CLOCK_BOOTTIME
void ktime_get_raw_ts64(struct timespec64 *)；     //CLOCK_MONOTONIC_RAW 
```



## 时间函数的分类

原型：ktime_t ktime_get(void) -- 对应的clock为:CLOCK_MONOTONIC

都是通过tk_core.timekeeper这个timekeeper结构体来进行操作。这个结构体是**全局变量**。

### 时钟不同

1. ktime_get -- CLOCK_MONOTONIC 最可靠精确的，系统启动后开始计时。

2. ktime_get_real -- CLOCK_REALTIME， 使用协调世界时（UTC）返回相对于1970年开始的时间，与gettimeofday（）相同。

3. ktime_get_raw  -- CLOCK_MONOTONIC_RAW，与硬件时钟源相同速率

​       

​       

### 返回类型不同 

1. 返回ktime_t; ktime_** 最原始的。

   2.返回ns： ktime_get_**_ns

   3.返回timespec64： ktime_get_**_ts64  s和ns分开，就可以少一次除法操作，适用于timespec/timeval等结构体。

​       

### 粒度不同

1. 粗粒度的： ktime_get_**_seconds 返回秒级时间，不需要去访问时钟硬件。

2. 较粗粒度的： ktime_get_coarse_**  比non-coarse更快，使用CLOCK_MONOTONIC_COARSE 和CLOCK_REALTIME_COARSE，用CONFIG_HZ来进行计数，通过读取jiffies变量实现，因此可以避免访问硬件，从而比访问硬件快100 cpu cycle。 -- 可用于inode时间戳。

3. 细粒度的：ktime_get**

​    

### 过时的函数

比如getnstimeofday, do_gettimeofday等等。

​       

## 2.5      内核时间函数使用方式

### 2.5.1    细粒度      

所有的ktime_get家族的函数，是精确获取的方式。分两部分：

1. 其中粗粒度的时间是直接读取timekeeper结构体中的变量获取

对于ktime_t返回类型：base = tk->tkr_mono.base

对于timespec64返回类型：sec = tk->xtime_sec

2. 而细粒度的部分是调用了timekeeping_get_ns(&tk->tkr_mono)进行细粒度校准，需要读取硬件timer。

对于mono时钟和real时钟，使用的clocksource是tkr_mono；

对于raw时钟，使用的是tkr_raw;（内核不常用）

 

其中，ktime_get -> timekeeping_get_ns -> timekeeping_get_delta这条线中，

timekeeping_get_delta是直接调用的clocksource结构体的read()函数，这个函数在arch timer的driver中对应的就是mrs cntvct

 

 **内核中的tkr_mono挂载的是arch timer，由arm_arch_timer的driver来实现。**

```c
#include <linux/timex.h> 
struct timespec64 ts_start, ts_end;
struct timespec64 ts_delta;
 
ktime_get_boottime_ts64(&ts_start);
// ...函数逻辑
ktime_get_boottime_ts64(&ts_end);
 
// 计算时间差
ts_delta = timespec64_sub(ts_end, ts_start);
 
pr_notice("[DB] time consumed: %lld (ns)\n",timespec64_to_ns(&ts_delta));
```

 

### 2.5.3    粗粒度

所有的ktime_get_coarse*家族的函数，是直接读timekeeper结构体变量值的，忽略细粒度的变化，即不需要读取硬件timer。通过tk_xtime(tk)获取tk->xtime_sec和tk->tkr_mono.xtime_nsec.

 

# 用户态获取时间

## 分类

### 按时钟源分类

wall clock： 绝对时间，包含因为NTP、timeZone调整的时间。

\-     time, gettimeofday, clock_getime

进程相关的clock：进程自己统计的运行时间。

\-     clock，getrusage，times

### 按返回结果精度分类

\-     ns级别： clock_gettime

\-     us级别：gettimeofday，clock，

\-     s级别： time

## timeout/sleep/usleep

`timeout`为`busybox`或者`coreutils`中的命令，是linux系统基本的小程序。以`busybox`为例来看，

`timeout(seconds)`调用了`sleep(seconds)`接口来实现的，`sleep`调用的是`nanosleep`来实现的。而`nanosleep`接口在glibc中有两个地方实现，第一个是./sysdeps/unix/sysv/linux/nanosleep.c，第二个是./posix/nanosleep.c。 当`__TIMESIZE == 64`的时候，前者仅定义了nanosleep64，而真正的`nanosleep`的实现在后者。这里注意，可以直接通过`printf("Timesize=%d\n", __TIMESIZE);`来确认该值（通过bpf抓取也可以确认使用的是clock_nanosleep）

```c
timeout        // busybox
  -> sleep            // coreutils, coreutils/sleep.c
     -> nanosleep     // glibc, posix/nanosleep.c
         -> __clock_nanosleep(CLOCK_REALTIME, 0, requested_time, remaining)  // linux, kernel/time/posix-timers.c
    		-> 
```

内核中的具体实现是：

```c
// 省流版：
clock_nanosleep
	-> common_nsleep
    	-> hrtimer_nanosleep
			->do_nanosleep(mode=HRTIMER_MODE_REL)
    // 这个函数中,通过睡眠来实现的。
    // 将自己设置成TASK_FREEZABLE|TASK_INTERRUPTIBLE，然后把delta time写到timer中，添加至timerqueue中。然后scheduler()调度出去，等到回来的时候继续判断，直到task为空，就说明真的到期了，设置回TASK_RUNNING，返回就好了。这里，怎么用到的硬件timer来触发中断的还没完全搞清楚。
    			-> enqueue_hrtimer  // hrtimer入队操作。

// 详细版
SYSCALL_DEFINE4(clock_nanosleep, const clockid_t, which_clock, int, flags,     
                const struct __kernel_timespec __user *, rqtp,                 
                struct __kernel_timespec __user *, rmtp)                       
{                                                                              
        const struct k_clock *kc = clockid_to_kclock(which_clock);  //获取时钟源，这里是clock_relatime,             
        struct timespec64 t;                                                   
                                                                               
        if (!kc)                                                               
                return -EINVAL;                                                
        if (!kc->nsleep)                                                       
                return -EOPNOTSUPP;                                           
        if (get_timespec64(&t, rqtp))                                          
                return -EFAULT;                                       
        if (!timespec64_valid(&t))                                             
                return -EINVAL;                                                
        if (flags & TIMER_ABSTIME)                                             
                rmtp = NULL;                                                   
        current->restart_block.fn = do_no_restart_syscall;                     
        current->restart_block.nanosleep.type = rmtp ? TT_NATIVE : TT_NONE;    
        current->restart_block.nanosleep.rmtp = rmtp;                          
                                                                               
        return kc->nsleep(which_clock, flags, &t);   // nsleep实际调用。    
}      
static const struct k_clock clock_realtime = {                       
        .nsleep                 = common_nsleep,                   
};      

static int common_nsleep(const clockid_t which_clock, int flags,       
                         const struct timespec64 *rqtp)                
{                                                                      
        ktime_t texp = timespec64_to_ktime(*rqtp);                     
        return hrtimer_nanosleep(texp, flags & TIMER_ABSTIME ?       //flgas=0，走REL  
                                 HRTIMER_MODE_ABS : HRTIMER_MODE_REL,  
                                 which_clock);                         
} 
 static int __sched do_nanosleep(struct hrtimer_sleeper *t, enum hrtimer_mode mode)
 {                                                                                 
         struct restart_block *restart;                                            
                                                                                   
         do {                                                                      
                 set_current_state(TASK_INTERRUPTIBLE|TASK_FREEZABLE);             
                 hrtimer_sleeper_start_expires(t, mode);                           
                                                                                   
                 if (likely(t->task))                                              
                         schedule();                                               
                                                                                   
                 hrtimer_cancel(&t->timer);                                        
                 mode = HRTIMER_MODE_ABS;                                          
                                                                                   
         } while (t->task && !signal_pending(current));                            
                                                                                   
         __set_current_state(TASK_RUNNING);                                        
                                                                                   
         if (!t->task)                                                             
                 return 0;                                                         
                                                                                   
         restart = &current->restart_block;                                        
         if (restart->nanosleep.type != TT_NONE) {                                 
                 ktime_t rem = hrtimer_expires_remaining(&t->timer);               
                 struct timespec64 rmt;                                            
                                                                                   
                 if (rem <= 0)                                                     
                         return 0;                                                 
                 rmt = ktime_to_timespec64(rem);                                   
                                                                                   
                 return nanosleep_copyout(restart, &rmt);                          
         }                                                                         
         return -ERESTART_RESTARTBLOCK;                                            
 }                                                                                 
```





## time() 

实现方式：arm64通过`clock_gettime(COARSE)`接口读取（glibc2.31以上版本，2.29还只是通过__kernel_gettimeofday读取），x86通过`vdso_time()`接口读取， 

单位：秒，直接读取vdso结构体中的成员变量，不涉及硬件接口。

耗时：arm上约88ns，x86上约32ns

### glibc中的定义

在文件 `sysdeps/unix/sysv/linux/time.c`中，

``` c
# define HAVE_TIME_VSYSCALL             "__vdso_time" 
			// 符号定义。
# define INIT_ARCH() void *vdso_time = dl_vdso_vsym (HAVE_TIME_VSYSCALL);
			// 在VDSO link map中寻找对应的符号。这个map在内核的vdso.lds.S中定义。
libc_ifunc (time,
        vdso_time ? VDSO_IFUNC_RET (vdso_time) 
              : (void *) time_syscall); // 定义函数time，若存在__vdso_time的
            					//内核vsyscalls调用，则直接调用，否则就退化到
            					//time_syscall的实现中。
// 其中，time_syscall的实现为：
static time_t
time_syscall (time_t *t)
{
  return INLINE_SYSCALL_CALL (time, t); // 其实就是调用glibc自己的time(t)函数。
}

```

### x86的实现

#### glibc

x86的实现为：

``` c
// time()接口在x86会去到文件sysdeps/unix/sysv/linux/x86/time.c中
#define USE_IFUNC_TIME  // 当定义这个宏时，则直接调用vdso函数，而不需要syscall到内核了。
#include <sysdeps/unix/sysv/linux/time.c>
```

``` c
// 再看sysdeps/unix/sysv/linux/time.c
// 如果定义了这个宏
#ifdef USE_IFUNC_TIME
static time_t
time_syscall (time_t *t)
{
  return INLINE_SYSCALL_CALL (time, t); //调用内核的sys_time() -> ktime_get_real_seconds（）
    		// 读取timekeeper的xtime_sec域段来返回，
    		// 详见 kernel/time/time.c
}

# undef INIT_ARCH
# define INIT_ARCH() \
  void *vdso_time = dl_vdso_vsym (HAVE_TIME_VSYSCALL);
		// 从vdso_vsym中获取HAVE_TIME_VSYSCALL对应的函数，若获取成功，则可以在下面直接返回了。
libc_ifunc (time,
        vdso_time ? VDSO_IFUNC_RET (vdso_time) 
              : (void *) time_syscall); // 若获取不到，则fallback真正陷入内核去处理。

#else /* USE_IFUNC_TIME  */
// 如果没有定义上面的宏
__time64_t
__time64 (__time64_t *timer)
{
  struct __timespec64 ts;
  __clock_gettime64 (TIME_CLOCK_GETTIME_CLOCKID, &ts);
    // sysdeps/unix/sysv/linux/clock_gettime.c
    // 先在vdso中依次查找vdso_gettime64/vdso_gettime，若有则可以直接调用了；
    // 若没有，则需要执行真正的syscall函数，调用sys_gettime64/sys_gettime来获取。

  if (timer != NULL)
    *timer = ts.tv_sec;
  return ts.tv_sec;
}
weak_alias (__time, time)
#endif
```



#### 内核

​	x86的vdso接口有（vdso.lds.S）

``` c
 （arch/x86/entry/vdso/vclock_gettime.c）
__kernel_old_time_t __vdso_time(__kernel_old_time_t *t)
 {
     return __cvdso_time(t);
 }

static __maybe_unused __kernel_old_time_t __cvdso_time(__kernel_old_time_t *time)
{
    return __cvdso_time_data(__arch_get_vdso_data(), time);
}

static __maybe_unused __kernel_old_time_t
__cvdso_time_data(const struct vdso_data *vd, __kernel_old_time_t *time)
{
    __kernel_old_time_t t;

    if (IS_ENABLED(CONFIG_TIME_NS) &&          // 若使能了TIME的namespace
        vd->clock_mode == VDSO_CLOCKMODE_TIMENS)
        vd = __arch_get_timens_vdso_data();

    t = READ_ONCE(vd[CS_HRES_COARSE].basetime[CLOCK_REALTIME].sec); // 获取s，直接读取结构体。

    if (time)
        *time = t;

    return t;
}
```

### arm64的实现

#### glibc

​	glibc 2.31之前是调用gettimeofday（）的vdso接口，需要读取硬件时间。现在是调用gettime()的vdso接口+COARSE的参数，可以不用读取硬件时间。

​	https://sourceware.org/pipermail/glibc-cvs/2019q4/067885.html

##### before 2.31

调用__kernel_gettimeofday

##### after 2.31

修改后：

``` c
（time/time.c）

time_t
time (time_t *timer)
{
  struct timespec ts;
  __clock_gettime (TIME_CLOCK_GETTIME_CLOCKID, &ts); 
    		// #define TIME_CLOCK_GETTIME_CLOCKID CLOCK_REALTIME_COARSE
    		// 具体链接方式详见上述x86的glibc代码

  if (timer)
    *timer = ts.tv_sec;
  return ts.tv_sec;
}
```

####　内核

 gettime() -> __cvdso_clock_gettime_common()

```c
 static __always_inline int
 __cvdso_clock_gettime_common(const struct vdso_data *vd, clockid_t clock,
                  struct __kernel_timespec *ts)
 {
     u32 msk;
     msk = 1U << clock;
     if (likely(msk & VDSO_HRES))
         vd = &vd[CS_HRES_COARSE]; // 需要继续do_hres
     else if (msk & VDSO_COARSE) 
         	// gettime()的glibc接口参数是COARSE，可以匹配这里，直接返回不需要到do_hres接口。
         return do_coarse(&vd[CS_HRES_COARSE], clock, ts);
     else if (msk & VDSO_RAW)
         vd = &vd[CS_RAW];  // 需要继续do_hres
     else
         return -1;

     return do_hres(vd, clock, ts); // 这里会去读取硬件timer。
 }
```

``` c
static __always_inline int do_coarse(const struct vdso_data *vd, clockid_t clk,
                     struct __kernel_timespec *ts)
{
    const struct vdso_timestamp *vdso_ts = &vd->basetime[clk];
    u32 seq;

    do {
        /*
         * Open coded to handle VDSO_CLOCK_TIMENS. See comment in
         * do_hres().
         */
        while ((seq = READ_ONCE(vd->seq)) & 1) { 
            if (IS_ENABLED(CONFIG_TIME_NS) &&
                vd->clock_mode == VDSO_CLOCKMODE_TIMENS) 
                	// 仅在time的namespace使能时才会进入。
                return do_coarse_timens(vd, clk, ts);
            cpu_relax();
        }
        smp_rmb(); // 实现上是一个dmb ishld

        ts->tv_sec = vdso_ts->sec;
        ts->tv_nsec = vdso_ts->nsec;
    } while (unlikely(vdso_read_retry(vd, seq))); // 实现中有一个smp_rmb()
    	// 再重新读一下vd->seq（序列号）看看这期间是否有人修改了vdso结构体，若被改了则重新读取。

    return 0;
}
```

这一个函数中就存在２个dmb，实际操作中挺影响性能的。但这里必须要保证这个序关系：

```shell
1.  读 vd->seq 确认当前vdso的序列号
2.  读 vdso_ts->sec/nsec，各自是8 Byte
3.  再重新读vd->seq确认当前vdso的序列号，如果和1相同，说明中间没有vdso的变化，因此2读到的一定是正确的值；而如果3和1不一样，则无法保证2读到的是是哪一个时刻的值，可能sec是旧值，nsec是新值，那就乱了。所以必须重新读。

因为保证1->2->3的3个读操作必须是串行执行，因此1/2之间得有rmb，2/3之间得有rmb。
# 无法直接使用atomic指令读取这两个8Byte的变量，因为atomic本身仅支持最大4Byte（casp可以到8B），而LD64B指令是上到了64Byte，没有其他中间状态。因此，只能接受两个dmb了。
```



如果time的namespace使能，则会进入`do_coarse_timens`函数：

``` c
static __always_inline int do_coarse_timens(const struct vdso_data *vdns, clockid_t clk,
                        struct __kernel_timespec *ts)
{
    const struct vdso_data *vd = __arch_get_timens_vdso_data(vdns);
    const struct vdso_timestamp *vdso_ts = &vd->basetime[clk];
    const struct timens_offset *offs = &vdns->offset[clk];
    u64 nsec;
    s64 sec;
    s32 seq;

    do {
        seq = vdso_read_begin(vd);
        sec = vdso_ts->sec;
        nsec = vdso_ts->nsec;
    } while (unlikely(vdso_read_retry(vd, seq)));

    /* Add the namespace offset */
    sec += offs->sec;
    nsec += offs->nsec;

    /*
     * Do this outside the loop: a race inside the loop could result
     * in __iter_div_u64_rem() being extremely slow.
     */
    ts->tv_sec = sec + __iter_div_u64_rem(nsec, NSEC_PER_SEC, &nsec);
    ts->tv_nsec = nsec;
    return 0;
}
// 其中，vdso_read_retry中有一个barrier，实现上是一个dmb ishld。
static __always_inline u32 vdso_read_retry(const struct vdso_data *vd,
                       u32 start)
{
    u32 seq;

    smp_rmb();
    seq = READ_ONCE(vd->seq);
    return seq != start;
}
```



## times

接口形式为：`times (struct tms *buffer)`，是要传参的，结构体struct tms是用户态、内核态均有定义的。

单位为jiffies，这里指的是USER_HZ为单位，目前内核的实现是固定为100.

``` c
 struct tms
   {
     clock_t tms_utime;      /* User CPU time.  */
     clock_t tms_stime;      /* System CPU time.  */

     clock_t tms_cutime;     /* User CPU time of dead children.  */
     clock_t tms_cstime;     /* System CPU time of dead children.  */
   };
```

### glibc中的定义

通用实现在"./sysdeps/unix/sysv/linux/times.c"文件中，会调用INTERNAL_SYSCALL_CALL宏，宏展开后其实就是调用svc接口函数。

``` c
 clock_t __times (struct tms *buf)
 {
   clock_t ret = INTERNAL_SYSCALL_CALL (times, buf);
   ......
 }
```

### 内核中的实现

在kernel/sys.c中，

``` c
static void do_sys_times(struct tms *tms)
{
    u64 tgutime, tgstime, cutime, cstime;

    thread_group_cputime_adjusted(current, &tgutime, &tgstime);
    // 从task_struct->signal中获取当前进程含子进程的运行时间，不需要读硬件。
    cutime = current->signal->cutime;
    cstime = current->signal->cstime;
    tms->tms_utime = nsec_to_clock_t(tgutime);
    tms->tms_stime = nsec_to_clock_t(tgstime);
    tms->tms_cutime = nsec_to_clock_t(cutime);
    tms->tms_cstime = nsec_to_clock_t(cstime);
}

SYSCALL_DEFINE1(times, struct tms __user *, tbuf)
{
    if (tbuf) {
        struct tms tmp;

        do_sys_times(&tmp);
        if (copy_to_user(tbuf, &tmp, sizeof(struct tms)))
            return -EFAULT;
    }
    force_successful_syscall_return();
    return (long) jiffies_64_to_clock_t(get_jiffies_64());
}
```



##  clock()

未研究，应该是us级别的精度，使用clock_gettime()获取。

## clock_gettime()

调用vdso的clock_gettime()接口。

精度：根据传入参数来获取不同的精度。支持获取ns（硬件实现），s（time函数的实现方式，传入参数为COARSE，不会去访问硬件）。

 

###  源码

``` c
// kernel/time/posix-timers.c
SYSCALL_DEFINE2(clock_gettime, const clockid_t, which_clock,
    struct kernel_timespec user *, tp)
{
  const struct k_clock *kc = clockid_to_kclock(which_clock);
  struct timespec64 kernel_tp;
  int error;

  if (!kc)
    return -EINVAL;

  error = kc->clock_get_timespec(which_clock, &kernel_tp);
    
  if (!error && put_timespec64(&kernel_tp, tp))
    error = -EFAULT;
  return error;
}

```

可以自己指定clocksource，不同的参数对应的是不同的钩子函数。

``` c
static const struct k_clock * const posix_clocks[] = {
   [CLOCK_REALTIME]    = &clock_realtime,
   [CLOCK_MONOTONIC]    = &clock_monotonic,
   [CLOCK_PROCESS_CPUTIME_ID] = &clock_process,
   [CLOCK_THREAD_CPUTIME_ID]  = &clock_thread,
   [CLOCK_MONOTONIC_RAW]    = &clock_monotonic_raw,
   [CLOCK_REALTIME_COARSE]   = &clock_realtime_coarse,
   [CLOCK_MONOTONIC_COARSE]  = &clock_monotonic_coarse,
   [CLOCK_BOOTTIME]    = &clock_boottime,
   [CLOCK_REALTIME_ALARM]   = &alarm_clock,
   [CLOCK_BOOTTIME_ALARM]   = &alarm_clock,
   [CLOCK_TAI]     = &clock_tai,
 };
```

其中，

\-     CLOCK_PROCESS_CPUTIME_ID对应的是当前进程的执行时间，通过读取task_struct的u_time和st_time成员变量来获取，这两个值会在HZ timer以及调度时进行更新，因此精度为HZ。

 

###  用法

```c
struct timespec time_start={0, 0},time_end={0, 0};
clock_gettime(CLOCK_REALTIME, &time_start);
printf("start time %llus,%llu ns\n", time_start.tv_sec, time_start.tv_nsec);
```



## getrusage

## gettimeofday

通过vdso gettimeofday获取us，（读的硬件cntvct，但API限制只能到us级别）

### glibc源码

```c
// time/gettimeodfay.c
// 调用的是CLOCK_REALTIME
/* Get the current time of day, putting it into *TV.                  
   If *TZ is not NULL, clear it.                                      
   Returns 0 on success, -1 on errors.  */                            
int                                                                   
___gettimeofday (struct timeval *restrict tv, void *restrict tz)      
{                                                                     
  if (__glibc_unlikely (tz != 0))                                     
    memset (tz, 0, sizeof (struct timezone));                         
                                                                      
  struct timespec ts;                                                 
  if (__clock_gettime (CLOCK_REALTIME, &ts))                          
    return -1;                                                        
                                                                      
  TIMESPEC_TO_TIMEVAL (tv, &ts);                                      
  return 0;                                                           
}                                                                               
```



###   内核源码

调用vdso库的__kernel_gettimeofday接口，这是个封装，在arm64中实际调用的是__cvdso_gettimeofday(lib/vdso/gettimeofday.c)。流程是:

a)     先获取vdso结构体；

b)    然后从结构体中调用获取时间的函数，传入的参数为**CLOCK_REALTIME**，调用`do_hres`函数

c)     并返回timespec结构体的ts。

 

其中，do_hres函数的实现是：

先获取vdso中存放的时间以及上次存放的cycle数last， 然后通过硬件接口（msr cntvct）获取当前的cycle数cycles，计算cycles-last过去了多少ns，以此更新vdso结构体。

 

当clock_mode = 0时，说明vDSO使能，可以正常读取cntvct。否则，就需要返回错误，不能走vDSO流程，只能去syscall gettimeofday了。

   



# 时间转换函数

gmtime & localtime & mktime，不直接获取时间，而是涉及到时间格式的转换，他们的区别在于时区的选择不同。

## 时区设置

1. 先看环境变量`TZ`，若被设置了：
   1. 若为相对路径，则再读取环境变量`TZDIR`进行拼接，然后读取该路径的文件作为时区；比如，`TZ="Asia/Shanghai"`
   2. 若为绝对路径，则直接读取；
   3. 若为Null，表示不指定特定时区，这是就使用最简单的默认时区，即默认设置为`Universal`时区，也就是读取`/usr/share/zoneinfo/Universal`文件。
2. 若环境变量`TZ`不存在（注意，TZ存在但是空值，不属于这种情况）
   1. 则读取默认文件`/etc/localtime`，这个可能是一个软链接，链接到`/usr/share/zoneinfo/**`的某个具体时区。这里一般在安装OS的时候就进行了配置，也可以后续`tzselect`进行修改。
   2. tips：若想知道当前时区配置，则`timedatectl`即可。

## gmtime

### 用户态使用方式

有两种，`gmtime` & `gmtime_r`，后者将dta存储在用户提供的结构体中。

返回的是一个指针，实际的内存是内部通过static申请的静态内存，因此可能会被后续的日期时间调用函数所覆盖。

``` c
struct tm *gmtime(const time_t *timer) //使用 timer 的值来填充 tm 结构，并用协调世界时（UTC）也被称为格林尼治标准时间（GMT）表示。
    
int main ()
{
   time_t rawtime;
   struct tm *info;
 
   time(&rawtime);
   /* 获取 GMT 时间 */
   info = gmtime(&rawtime);
   
   printf("当前的世界时钟：\n");
   printf("伦敦：%2d:%02d\n", (info->tm_hour+BST)%24, info->tm_min);
   printf("中国：%2d:%02d\n", (info->tm_hour+CCT)%24, info->tm_min);
 
   return(0);
}

```



### glibc中的定义

``` c
(time/gmtime.c)
/* Return the `struct tm' representation of *T in UTC.  */
struct tm *
__gmtime64 (const __time64_t *t)
{
  return __tz_convert (*t, 0, &_tmbuf);
}
```

```c
struct tm *
__tz_convert (__time64_t timer, int use_localtime, struct tm *tp)
{
  long int leap_correction;
  int leap_extra_secs;

  __libc_lock_lock (tzset_lock);

  /* Update internal database according to current TZ setting.
     POSIX.1 8.3.7.2 says that localtime_r is not required to set tzname.
     This is a good idea since this allows at least a bit more parallelism.  */
   // 仅进程的第一次调用会真正运行，其他时候直接判断一行就出来了。
  tzset_internal (tp == &_tmbuf && use_localtime);
    // gmtime/gmtime_r的use_localtime = 0, localtime_r的tp!=_tmbuf，
    // 只有localtime可能为1，但是这个函数用的很少了。

    // __use_tzfile全局变量，在函数__tzfile_read中被配置。
  if (__use_tzfile)  
    __tzfile_compute (timer, use_localtime, &leap_correction,
              &leap_extra_secs, tp);
  else  // 小文件系统走这里。
    {
      if (! __offtime (timer, 0, tp))  
	    	tp = NULL;
      else
		    __tz_compute (timer, tp, use_localtime);
		    leap_correction = 0L;
 	    	eap_extra_secs = 0;
    }

  __libc_lock_unlock (tzset_lock);

  if (tp)
  {
      if (! use_localtime)
 	 {
  	   tp->tm_isdst = 0;
   	   tp->tm_zone = "GMT";
   	   tp->tm_gmtoff = 0L;
  	  }

  	  if (__offtime (timer, tp->tm_gmtoff - leap_correction, tp))
        tp->tm_sec += leap_extra_secs;
 	  else
        tp = NULL;
  }

  return tp;
}
```

#### __tzfile_internal

（time/tzset.c）， 该函数会解析`TZ`变量，但一个进程只会调用一次，因为使用static变量`is_initialized`来进行判断，第一次设置完了后第二次发现已经为1，就直接`return`了。

配置时有几种情况：

1. 用户主动设置TZ变量为空，即TZ存在且TZ=空字符串 。
  
   1. 那么，这里会明确使用Universal时区，即`tz = "Universal"`
   2. 在`__tzfile_read`中，会读取`TZDIR`进行拼接，然后读取`/local/share/zoneinfo/Universal`文件，进行后续的解析。
   
2. 用户主动设置TZ变量为冒号，即TZ=：，表示自定义的时区语法
  
   1. 这里会忽略冒号，剩余信息按照普通语法继续解析。
   
3. 若和之前的时区一致（全局变量`old_tz`会保存上一次的值），则直接return；

4. 若tz变量没有被设置，则设置为TZDEFAULT的默认值，即/etc/localtime。

   ```makefile
   （glibc/Makeconfig）
   # Where to install the "localtime" timezone file; this is the file whose
   # contents $(localtime) specifies.  If this is a relative pathname, it is
   # relative to $(zonedir).  It is a good idea to put this somewhere
   # other than there, so the zoneinfo directory contains only universal data,
   # localizing the configuration data elsewhere.
   ifeq ($(origin prefix),undefined) 
   prefix = /usr/local  # 实际上看，这里应该不会进入，因此prefix=NULL
   endif
   sysconfdir = $(prefix)/etc
   localtime-file = $(sysconfdir)/localtime
   datadir = $(prefix)/share
   zonedir = $(datadir)/zoneinfo
   posixrules-file = posixrules
   
   （time/Makefile)
   tz-cflags = -DTZDIR='"$(zonedir)"' \
           -DTZDEFAULT='"$(localtime-file)"' \
           -DTZDEFRULES='"$(posixrules-file)"'
   ```

配置完tz文件后，就调用`__tzfile_read`读取这个文件，在`__tzfile_read`函数中，若读取成功则配置`__use_tzfile=1`。

1. 若读取成功，则该函数直接返回；
2. 若读取失败，则使用UTC作为时区规范，并设置全局变量`__daylight`,`__timezone`,`__tzname`以及`tz_rules`。 这里，就是用的最简单的方案，所以比`__tzfile_read`中的逻辑要简单很多。



#### __tzfile_read

1. 判断传入的file：
   1. 若file指针为空，则使用默认值TZDEFAULT；
   2. **若file指向'\0'，则直接返回，此时 `__use_tzfile=0`**
   3. 若file是一个相对路径，则从环境变量`TZDIR`中获取对应的路径，进行拼接。
2. 若file并没有被修改过，则`__use_tzfile=1`后直接返回。
3. **`fopen`打开文件，若文件打开失败，则直接返回，此时 `__use_tzfile=0`**
4. 从文件中获取`struct tzhenad`类型的各种信息，并进行一大堆的判断和解析，比如时区结构体的大小，是否夏日时区，时区编号等，最终将解析到的信息写到全局变量`__daylight`,`__timezone`中，最后设置`__use_tzfile=1`，表示可以从文件中读取到，因此下一次还是用这个文件。



#### __offtime

```assembly
 {
        __time64_t days, rem, y;
        const unsigned short int *ip;
        days = t / SECS_PER_DAY;
 0x0000fffff7dc33e0 <+0>:     mov     x4, #0x2957                     // #10583
 0x0000fffff7dc33e4 <+4>:     stp     x29, x30, [sp, #-48]!
 0x0000fffff7dc33e8 <+8>:     movk    x4, #0xce51, lsl #16
 0x0000fffff7dc33ec <+12>:    movk    x4, #0xc8a0, lsl #32
 0x0000fffff7dc33f0 <+16>:    mov     x3, #0x5180                     // #20864
 0x0000fffff7dc33f4 <+20>:    movk    x4, #0x1845, lsl #48
 0x0000fffff7dc33f8 <+24>:    movk    x3, #0x1, lsl #16
 0x0000fffff7dc33fc <+28>:    mov     x29, sp
 0x0000fffff7dc3400 <+32>:    smulh   x8, x0, x4
 0x0000fffff7dc3404 <+36>:    stp     x19, x20, [sp, #16]
 0x0000fffff7dc3408 <+40>:    stp     x21, x22, [sp, #32]
 0x0000fffff7dc340c <+44>:    asr     x8, x8, #13
 0x0000fffff7dc3410 <+48>:    sub     x8, x8, x0, asr #63
        rem = t % SECS_PER_DAY;
 0x0000fffff7dc3414 <+52>:    msub    x0, x8, x3, x0
        rem += offset;
        while (rem < 0)
 0x0000fffff7dc3418 <+56>:    adds    x1, x0, x1
 0x0000fffff7dc341c <+60>:    b.pl    0xfffff7dc36b0 <__offtime+720>  // b.nfrst
 0x0000fffff7dc3424 <+68>:    adds    x1, x1, x3
 0x0000fffff7dc3428 <+72>:    b.mi    0xfffff7dc3420 <__offtime+64>  // b.first
          {
            rem += SECS_PER_DAY;
            --days;
 0x0000fffff7dc3420 <+64>:    sub     x8, x8, #0x1
          }
        while (rem >= SECS_PER_DAY)
 0x0000fffff7dc36b0 <+720>:   mov     x0, #0x517f                     // #20863
 0x0000fffff7dc36b4 <+724>:   movk    x0, #0x1, lsl #16
 0x0000fffff7dc36b8 <+728>:   cmp     x1, x0
 0x0000fffff7dc36bc <+732>:   b.le    0xfffff7dc342c <__offtime+76>
 0x0000fffff7dc36c0 <+736>:   mov     x3, #0xffffffffffffae80         // #-20864
 0x0000fffff7dc36c4 <+740>:   movk    x3, #0xfffe, lsl #16
 0x0000fffff7dc36d0 <+752>:   cmp     x1, x0
 0x0000fffff7dc36d4 <+756>:   b.gt    0xfffff7dc36c8 <__offtime+744>
 0x0000fffff7dc36d8 <+760>:   b       0xfffff7dc342c <__offtime+76>
          {
            rem -= SECS_PER_DAY;
 0x0000fffff7dc36c8 <+744>:   add     x1, x1, x3
            ++days;
 0x0000fffff7dc36cc <+748>:   add     x8, x8, #0x1
         }
       tp->tm_hour = rem / SECS_PER_HOUR;
0x0000fffff7dc342c <+76>:    mov     x3, #0x6f81                     // #28545
0x0000fffff7dc3430 <+80>:    lsr     x4, x1, #4
0x0000fffff7dc3434 <+84>:    movk    x3, #0x4d5e, lsl #16
0x0000fffff7dc3438 <+88>:    mov     x0, #0x4925                     // #18725
0x0000fffff7dc343c <+92>:    movk    x3, #0x2b3c, lsl #32
0x0000fffff7dc3440 <+96>:    movk    x0, #0x2492, lsl #16
0x0000fffff7dc3444 <+100>:   movk    x3, #0x91a, lsl #48
0x0000fffff7dc3448 <+104>:   add     x6, x8, #0x4
0x0000fffff7dc344c <+108>:   movk    x0, #0x9249, lsl #32
0x0000fffff7dc3450 <+112>:   mov     x5, #0x8888888888888888         // #-8608480567731124088
0x0000fffff7dc3454 <+116>:   umulh   x4, x4, x3
0x0000fffff7dc3458 <+120>:   movk    x0, #0x4924, lsl #48
0x0000fffff7dc345c <+124>:   movk    x5, #0x8889
0x0000fffff7dc3460 <+128>:   mov     x14, #0x5c29                    // #23593
0x0000fffff7dc3464 <+132>:   smulh   x0, x6, x0
0x0000fffff7dc3468 <+136>:   mov     x17, #0x1eb8                    // #7864
0x0000fffff7dc346c <+140>:   lsr     x4, x4, #3
0x0000fffff7dc3470 <+144>:   mov     x16, #0x8f5c                    // #36700
0x0000fffff7dc3474 <+148>:   mov     x15, #0x1eb0                    // #7856
0x0000fffff7dc3478 <+152>:   mov     x18, #0xa3d6                    // #41942
0x0000fffff7dc347c <+156>:   asr     x0, x0, #1
0x0000fffff7dc3480 <+160>:   lsl     x3, x4, #3
0x0000fffff7dc3484 <+164>:   sub     x0, x0, x6, asr #63
0x0000fffff7dc3488 <+168>:   sub     x3, x3, x4
0x0000fffff7dc348c <+172>:   mov     x13, #0x33e7                    // #13287
0x0000fffff7dc3490 <+176>:   mov     x11, #0xd70b                    // #55051
0x0000fffff7dc3494 <+180>:   add     x3, x4, x3, lsl #5
0x0000fffff7dc3498 <+184>:   lsl     x7, x0, #3
0x0000fffff7dc349c <+188>:   sub     x0, x7, x0
0x0000fffff7dc34a0 <+192>:   movk    x14, #0xc28f, lsl #16
0x0000fffff7dc34a4 <+196>:   sub     x1, x1, x3, lsl #4
0x0000fffff7dc34a8 <+200>:   subs    x0, x6, x0
0x0000fffff7dc34ac <+204>:   add     w3, w0, #0x7
0x0000fffff7dc34b0 <+208>:   movk    x17, #0xeb85, lsl #16
0x0000fffff7dc34b4 <+212>:   csel    w0, w0, w3, pl  // pl = nfrst
0x0000fffff7dc34b8 <+216>:   str     w0, [x2, #24]
0x0000fffff7dc34bc <+220>:   umulh   x0, x1, x5
0x0000fffff7dc34c0 <+224>:   movk    x16, #0xf5c2, lsl #16
0x0000fffff7dc34c4 <+228>:   movk    x15, #0xeb85, lsl #16
0x0000fffff7dc34c8 <+232>:   movk    x18, #0x3d70, lsl #16
0x0000fffff7dc34cc <+236>:   movk    x13, #0x2ce, lsl #16
0x0000fffff7dc34d0 <+240>:   movk    x11, #0x70a3, lsl #16
0x0000fffff7dc34d4 <+244>:   lsr     x0, x0, #5
0x0000fffff7dc34d8 <+248>:   str     w0, [x2, #4]
0x0000fffff7dc34dc <+252>:   movk    x14, #0x28f5, lsl #32
0x0000fffff7dc34e0 <+256>:   movk    x17, #0xb851, lsl #32
0x0000fffff7dc34e4 <+260>:   lsl     x3, x0, #4
0x0000fffff7dc34e8 <+264>:   movk    x16, #0x5c28, lsl #32
0x0000fffff7dc34ec <+268>:   sub     x0, x3, x0
0x0000fffff7dc34f0 <+272>:   movk    x15, #0xb851, lsl #32
0x0000fffff7dc34f4 <+276>:   movk    x18, #0xd70a, lsl #32
0x0000fffff7dc34f8 <+280>:   movk    x13, #0x3e6c, lsl #32
0x0000fffff7dc34fc <+284>:   movk    x11, #0xa3d, lsl #32
0x0000fffff7dc3500 <+288>:   sub     x1, x1, x0, lsl #2
0x0000fffff7dc3504 <+292>:   mov     x9, #0x7b2                      // #1970
0x0000fffff7dc3508 <+296>:   movk    x14, #0x8f5c, lsl #48
0x0000fffff7dc350c <+300>:   movk    x17, #0x51e, lsl #48
0x0000fffff7dc3510 <+304>:   movk    x16, #0x28f, lsl #48
0x0000fffff7dc3514 <+308>:   movk    x15, #0x51e, lsl #48
0x0000fffff7dc3518 <+312>:   movk    x18, #0xa3, lsl #48
0x0000fffff7dc351c <+316>:   movk    x13, #0x2ce3, lsl #48
0x0000fffff7dc3520 <+320>:   mov     x12, #0x16d                     // #365
0x0000fffff7dc3524 <+324>:   movk    x11, #0xa3d7, lsl #48
0x0000fffff7dc3528 <+328>:   str     w1, [x2]
0x0000fffff7dc352c <+332>:   str     w4, [x2, #8]
       rem %= SECS_PER_HOUR;
       tp->tm_min = rem / 60;
       tp->tm_sec = rem % 60;
       /* January 1, 1970 was a Thursday.  */
       tp->tm_wday = (4 + days) % 7;
       if (tp->tm_wday < 0)
         tp->tm_wday += 7;
       y = 1970;
     #define DIV(a, b) ((a) / (b) - ((a) % (b) < 0))
     #define LEAPS_THRU_END_OF(y) (DIV (y, 4) - DIV (y, 100) + DIV (y, 400))
       while (days < 0 || days >= (__isleap (y) ? 366 : 365))
0x0000fffff7dc3530 <+336>:   b       0xfffff7dc3628 <__offtime+584>
0x0000fffff7dc3534 <+340>:   asr     x0, x0, #6
0x0000fffff7dc3538 <+344>:   sub     x0, x0, x8, asr #63
0x0000fffff7dc353c <+348>:   add     x1, x0, x9
0x0000fffff7dc3540 <+352>:   msub    x0, x0, x12, x8
0x0000fffff7dc3544 <+356>:   sub     x0, x1, x0, lsr #63
         {
           /* Guess a corrected year, assuming 365 days per year.  */
           __time64_t yg = y + days / 365 - (days % 365 < 0);
           /* Adjust DAYS and Y to match the guessed year.  */
           days -= ((yg - y) * 365
0x0000fffff7dc3548 <+360>:   subs    x7, x0, #0x1
0x0000fffff7dc354c <+364>:   add     x1, x0, #0x2
0x0000fffff7dc3550 <+368>:   csel    x1, x1, x7, mi  // mi = first
0x0000fffff7dc3554 <+372>:   negs    x4, x7
0x0000fffff7dc3558 <+376>:   and     x3, x7, #0x3
0x0000fffff7dc355c <+380>:   and     x4, x4, #0x3
0x0000fffff7dc3560 <+384>:   smulh   x10, x7, x11
0x0000fffff7dc3564 <+388>:   csneg   x4, x3, x4, mi  // mi = first
0x0000fffff7dc3568 <+392>:   subs    x6, x9, #0x1
0x0000fffff7dc356c <+396>:   asr     x5, x7, #63
0x0000fffff7dc3570 <+400>:   add     x10, x10, x7
0x0000fffff7dc3574 <+404>:   asr     x1, x1, #2
0x0000fffff7dc3578 <+408>:   asr     x22, x6, #63
0x0000fffff7dc357c <+412>:   csel    x20, x20, x6, mi  // mi = first
0x0000fffff7dc3580 <+416>:   asr     x30, x10, #6
0x0000fffff7dc3584 <+420>:   smulh   x3, x6, x11
0x0000fffff7dc3588 <+424>:   sub     x21, x30, x5
```







### 指令级分析

在波形上check：

``` c  
// 用户态调用gmtime -> 进入glibc的gmtime入口。
   0000000000093bc0 <gmtime@@GLIBC_2.17>:
   93bc0:   f9400000    ldr x0, [x0]
   93bc4:   d0000702    adrp    x2, 175000 <_res@GLIBC_2.17+0x308>
   93bc8:   52800001    mov w1, #0x0                    // #0
   93bcc:   91236042    add x2, x2, #0x8d8
   93bd0:   140007ac    b   95a80 <tzset@@GLIBC_2.17+0xb0>
// 进入__tz_convert()函数
   95a80:   a9ba7bfd    stp x29, x30, [sp, #-96]!
   95a84:   910003fd    mov x29, sp
   95a88:   a9025bf5    stp x21, x22, [sp, #32]
   95a8c:   d00006d6    adrp    x22, 16f000 <sys_sigabbrev@@GLIBC_2.17+0x1f8>
   95a90:   2a0103f5    mov w21, w1
   95a94:   f9477ac4    ldr x4, [x22, #3824]
   95a98:   a90153f3    stp x19, x20, [sp, #16]
   95a9c:   d00006f4    adrp    x20, 173000 <__abort_msg@@GLIBC_PRIVATE+0xa98>
   95aa0:   9124e283    add x3, x20, #0x938
   95aa4:   aa0203f3    mov x19, x2
   95aa8:   9101d063    add x3, x3, #0x74
   95aac:   f9001bf7    str x23, [sp, #48]
   95ab0:   aa0003f7    mov x23, x0
   95ab4:   f9400080    ldr x0, [x4]
   95ab8:   f9002fe0    str x0, [sp, #88]
   95abc:   d2800000    mov x0, #0x0                    // #0
   95ac0:   52800020    mov w0, #0x1                    // #1
       // 加锁 __libc_lock_lock (tzset_lock);
   95ac4:   885ffc62    ldaxr   w2, [x3]
   95ac8:   35000062    cbnz    w2, 95ad4 <tzset@@GLIBC_2.17+0x104> // 不跳转
   95acc:   88017c60    stxr    w1, w0, [x3] // 8.6ns长时延
   95ad0:   35ffffa1    cbnz    w1, 95ac4 <tzset@@GLIBC_2.17+0xf4>
   95ad4:   7100005f    cmp w2, #0x0
   95ad8:   54000a41    b.ne    95c20 <tzset@@GLIBC_2.17+0x250>  // b.any
   95adc:   710002bf    cmp w21, #0x0
   95ae0:   90000700    adrp    x0, 175000 <_res@GLIBC_2.17+0x308>
   95ae4:   91236000    add x0, x0, #0x8d8
   95ae8:   fa401260    ccmp    x19, x0, #0x0, ne  // ne = any
   95aec:   1a9f17e0    cset    w0, eq  // eq = none
   95af0:   97fffd69    bl  95094 <adjtime@@GLIBC_2.17+0xa94> // 跳转
// 进入 tzset_internal()函数
       0000000000095094 <adjtime@@GLIBC_2.17+0xa94>:
95094:   stp   x29, x30, [sp, #-48]!
95098:   eor   w0, w0, #0x1
9509c:   mov   x29, sp
950a0:   stp   x19, x20, [sp, #16]
950a4:   adrp  x20, 173000 <__abort_msg@@GLIBC_PRIVATE+0xa98>
950a8:   stp   x21, x22, [sp, #32]
950ac:   add   x21, x20, #0x938
950b0:   ldr   w1, [x21, #112]
950b4:   cmp   w1, #0x0
950b8:   cset  w1, ne  // ne = any
950bc:   tst   w0, w1
950c0: ↓ b.ne  951ac <adjtime@@GLIBC_2.17+0xbac>  // 实际会跳转
    ......
951ac:   ldp   x19, x20, [sp, #16]                                               
951b0:   ldp   x21, x22, [sp, #32]                                               
951b4:   ldp   x29, x30, [sp], #48                                               
951b8: ← ret                                                                     
// 回到__tz_convert()中    
   95af4:   90000700    adrp    x0, 175000 <_res@GLIBC_2.17+0x308>
   95af8:   b9491000    ldr w0, [x0, #2320]
   95afc:   340005c0    cbz w0, 95bb4 <tzset@@GLIBC_2.17+0x1e4> // 跳转
		95bb4:   mov   x2, x19                                                     
		95bb8:   mov   x0, x23                                                     
		95bbc:   mov   x1, #0x0                        // #0                       
		95bc0:   bl    933e0 <c32rtomb@@GLIBC_2.17+0x20>        // 跳转
// 到__offtime()中， 满ipc（6）计算。
// 返回            
        95bc4:   cbnz  w0, 95c0c <tzset@@GLIBC_2.17+0x23c>   // 跳转                      
     //  if (! __offtime (timer, 0, tp))
     //    tp = NULL;
     //  else      // 走else分支。
     //    __tz_compute (timer, tp, use_localtime);
     //    leap_correction = 0L;
     //    leap_extra_secs = 0;
            
        	95c0c:   mov   w2, w21                                
			95c10:   mov   x1, x19                                
			95c14:   mov   x0, x23                                
			95c18:   bl    95240 <adjtime@@GLIBC_2.17+0xc40>   
// 跳转到__tz_compute()分支
   95240:       a9bd7bfd        stp     x29, x30, [sp, #-48]!
   95244:       d00006e4        adrp    x4, 173000 <__abort_msg@@GLIBC_PRIVATE+0xa98>
   95248:       910003fd        mov     x29, sp
   9524c:       b9401429        ldr     w9, [x1, #20]
   95250:       311db53f        cmn     w9, #0x76d
   95254:       111db125        add     w5, w9, #0x76c
   95258:       54001180        b.eq    95488 <adjtime@@GLIBC_2.17+0xe88>  // 不跳
   9525c:       9124e083        add     x3, x4, #0x938
   95260:       b9403863        ldr     w3, [x3, #56]
   95264:       6b0300bf        cmp     w5, w3
   95268:       54000440        b.eq    952f0 <adjtime@@GLIBC_2.17+0xcf0>  // 跳转
       。。。。。。
   952f0:       9124e083        add     x3, x4, #0x938
   952f4:       b9406863        ldr     w3, [x3, #104]
   952f8:       6b0300bf        cmp     w5, w3
   952fc:       540004a0        b.eq    95390 <adjtime@@GLIBC_2.17+0xd90>  // 跳转
		。。。。。。
   95390:       34000302        cbz     w2, 953f0 <adjtime@@GLIBC_2.17+0xdf0>
       。。。。。。
   953f0:       a8c37bfd        ldp     x29, x30, [sp], #48
   953f4:       d65f03c0        ret
// 从__tz_compute返回到__tz_convert()中
            95c1c: ↑ b     95bcc <tzset@@GLIBC_2.17+0x1fc>        
   95bcc:       9124e282        add     x2, x20, #0x938
   95bd0:       b9004fff        str     wzr, [sp, #76]
   95bd4:       9101d042        add     x2, x2, #0x74
   95bd8:       f9002bff        str     xzr, [sp, #80]
   95bdc:       885f7c40        ldxr    w0, [x2] // 耗时1.6ns
   95be0:       8801fc5f        stlxr   w1, wzr, [x2] // 耗时6.6ns
   95be4:       35ffffc1        cbnz    w1, 95bdc <tzset@@GLIBC_2.17+0x20c>
   95be8:       7100041f        cmp     w0, #0x1
   95bec:       54fffa4d        b.le    95b34 <tzset@@GLIBC_2.17+0x164> // 跳转
       。。。。。。
   96b34:       b40001f3        cbz     x19, 95b70 <tzset@@GLIBC_2.17+0x1a0>
   95b38:       34000335        cbz     w21, 95b9c <tzset@@GLIBC_2.17+0x1cc> //跳转
   95b9c:       b00004a0        adrp    x0, 12a000 
   95ba0:       d2800003        mov     x3, #0x0                        // #0
   95ba4:       91148000        add     x0, x0, #0x520
   95ba8:       b900227f        str     wzr, [x19, #32]
   95bac:       a902827f        stp     xzr, x0, [x19, #40]
   95bb0:       17ffffe4        b       95b40 <tzset@@GLIBC_2.17+0x170>
       。。。。。。
   95b40:       aa1703e0        mov     x0, x23
   95b44:       f9402be1        ldr     x1, [sp, #80]
   95b48:       aa1303e2        mov     x2, x19
   95b4c:       cb010061        sub     x1, x3, x1
   95b50:       97fff624        bl      933e0 <c32rtomb@@GLIBC_2.17+0x20>
       。。。。。。
   933e0:       d2852ae4        mov     x4, #0x2957                     // #10583
   933e4:       a9bd7bfd        stp     x29, x30, [sp, #-48]!
   933e8:       f2b9ca24        movk    x4, #0xce51, lsl #16
   933ec:       f2d91404        movk    x4, #0xc8a0, lsl #32
   933f0:       d28a3003        mov     x3, #0x5180                     // #20864
   933f4:       f2e308a4        movk    x4, #0x1845, lsl #48
   933f8:       f2a00023        movk    x3, #0x1, lsl #16
   933fc:       910003fd        mov     x29, sp
   93400:       9b447c08        smulh   x8, x0, x4
   93404:       a90153f3        stp     x19, x20, [sp, #16]
   93408:       a9025bf5        stp     x21, x22, [sp, #32]
   9340c:       934dfd08        asr     x8, x8, #13
   93410:       cb80fd08        sub     x8, x8, x0, asr #63
   93414:       9b038100        msub    x0, x8, x3, x0
   93418:       ab010001        adds    x1, x0, x1
   9341c:       540014a5        b.pl    936b0 <c32rtomb@@GLIBC_2.17+0x2f0>  // b.nfrst
   93420:       d1000508        sub     x8, x8, #0x1
   93424:       ab030021        adds    x1, x1, x3
   93428:       54ffffc4        b.mi    93420 <c32rtomb@@GLIBC_2.17+0x60>  // b.first
       .....

```

### 锁操作解析

这里存在着两把锁，一个是lock，一个是unlock，当前的实现是nolse的，也就是使用exclusive方式去实现。

```assembly
#   __libc_lock_lock (tzset_lock) 对应的反汇编
   0x0000fffff7dc5a94 <+20>:    ldr     x4, [x22, #3824]
   0x0000fffff7dc5a98 <+24>:    stp     x19, x20, [sp, #16]
   0x0000fffff7dc5a9c <+28>:    adrp    x20, 0xfffff7ea3000 <proc_file_chain_lock>
   0x0000fffff7dc5aa0 <+32>:    add     x3, x20, #0x938
   0x0000fffff7dc5aa4 <+36>:    mov     x19, x2
   0x0000fffff7dc5aa8 <+40>:    add     x3, x3, #0x74
   0x0000fffff7dc5aac <+44>:    str     x23, [sp, #48]
   0x0000fffff7dc5ab0 <+48>:    mov     x23, x0
   0x0000fffff7dc5ab4 <+52>:    ldr     x0, [x4]
   0x0000fffff7dc5ab8 <+56>:    str     x0, [sp, #88]
   0x0000fffff7dc5abc <+60>:    mov     x0, #0x0                        // #0
   0x0000fffff7dc5ac0 <+64>:    mov     w0, #0x1                        // #1
   0x0000fffff7dc5ac4 <+68>:    ldaxr   w2, [x3]
   0x0000fffff7dc5ac8 <+72>:    cbnz    w2, 0xfffff7dc5ad4 <__tz_convert+84>
   0x0000fffff7dc5acc <+76>:    stxr    w1, w0, [x3]
   0x0000fffff7dc5ad0 <+80>:    cbnz    w1, 0xfffff7dc5ac4 <__tz_convert+68>
   0x0000fffff7dc5ad4 <+84>:    cmp     w2, #0x0
   0x0000fffff7dc5ad8 <+88>:    b.ne    0xfffff7dc5c20 <__tz_convert+416>  // b.any

#  __libc_lock_unlock (tzset_lock)
0x0000fffff7dc5bdc:       40 7c 5f 88     ldxr    w0, [x2]
0x0000fffff7dc5be0:       5f fc 01 88     stlxr   w1, wzr, [x2]
0x0000fffff7dc5be4:       c1 ff ff 35     cbnz    w1, 0xfffff7dc5bdc <__tz_convert+348>
```



```c
("time/tzset.c")
/* This locks all the state variables in tzfile.c and this file.  */
__libc_lock_define_initialized (static, tzset_lock)

struct tm * __tz_convert (__time64_t timer, int use_localtime, struct tm *tp)
{
  __libc_lock_lock (tzset_lock);
.........
  __libc_lock_unlock (tzset_lock);
.........
}

("sysdeps/nptl/libc-lockP.h")
# ifndef __libc_lock_lock
#  define __libc_lock_lock(NAME) \
  ({ lll_lock (NAME, LLL_PRIVATE); 0; })
# endif

("sysdeps/nptl/lowlevellock.h")
#define __lll_lock(futex, private)                                      \
  ((void)                                                               \
   ({                                                                   \
     int *__futex = (futex);                                            \
     if (__glibc_unlikely                                               \
         (atomic_compare_and_exchange_bool_acq (__futex, 1, 0)))        \
       {                                                                \
         if (__builtin_constant_p (private) && (private) == LLL_PRIVATE) \
           __lll_lock_wait_private (__futex);                           \
         else                                                           \
           __lll_lock_wait (__futex, private);                          \
       }                                                                \
   }))
#define lll_lock(futex, private)        \
  __lll_lock (&(futex), private)

("sysdeps/aarch64/atomic-machine.h")
# define atomic_compare_and_exchange_bool_acq(mem, new, old)    \
  __atomic_bool_bysize (__arch_compare_and_exchange_bool, int,  \
                        mem, new, old, __ATOMIC_ACQUIRE)

#  define __arch_compare_and_exchange_bool_64_int(mem, newval, oldval, model) \
  ({                                                                    \
    typeof (*mem) __oldval = (oldval);                                  \
    !__atomic_compare_exchange_n (mem, (void *) &__oldval, newval, 0,   \
                                  model, __ATOMIC_RELAXED);             \
  })

// 最后，__atomic_compare_exchange_n这个函数是在gcc中定义的。
```







## localtime_r

与`gmtime`类似，区别在于`gmtime`使用UTC世界时区，而`localtime`使用的是本地时区。

有两种，`localtime` & `localtime_r`，后者将data存储在用户提供的结构体中。

###用户态使用方式

``` c
struct tm* localtime_r( const time_t* timer, struct tm* result );
```

线程安全的。

返回的是一个指针，实际的内存是内部通过static申请的静态内存，所以通过localtime调用后的返回值不及时使用的话，很有可能被其他线程localtime/gmtime调用所覆盖掉。

### glibc中的定义

``` c
(time/localtime.c)
/* The C Standard says that localtime and gmtime return the same pointer.  */
struct tm _tmbuf;
struct tm * __localtime64_r (const __time64_t *t, struct tm *tp)
{
  return __tz_convert (*t, 1, tp);
}

// 如果用户不提供场地，那就借用glibc的全局静态结构_tmbuf来保存，只返回一个指针给用户。
struct tm * __localtime64 (const __time64_t *t)
{
  return __tz_convert (*t, 1, &_tmbuf);
}
```

可以看到，与`gmtime`调用的接口是一样的，只是第二个参数由`0`变成了`1`

### 热点分析

``` c
// 1620 evb上的单核localtime_r()函数热点。
97.84%     8.52%  libc-2.31.so         [.] __tz_convert       
- 18.03% __tz_convert                                          
   - 27.29% __tzfile_compute                                   
      + 14.60% __tzset_parse_tz                                
        8.90% __offtime                                        
        6.84% __tz_compute                                     
     7.07% __offtime                                           
- 8.52% _start                                                 
     __libc_start_main                                         
     main                                                      
     benchmp_wave                                              
   - benchmp_child                                             
      - 7.94% benchmark                                        
           __tz_convert                                        
```





## mktime

### 用户态使用方式

### glibc中的定义

mktime代码包含两部分：

__tzset() 来进行时区的设置；

__mktime_Internal来进行计算。

从1620的热点中看（kernel 5.10，glibc 2.31），tzset占比75%， mktime_internal占比23%

``` shell
  - 98.65% benchmark                               
     - 75.20% mktime                               
        + __tzset                                  
     - 23.03% __mktime_internal                    
        + 21.32% ranged_convert                    
    0.68% mktime                                   
```

其中，mktime_internal中热点比较分散。

``` shell
23.03% __mktime_internal                                  
 - 21.32% ranged_convert                                  
    - 20.47% __tz_convert                                 
       - 17.80% __tzfile_compute                          
          - 14.66% __tzset_parse_tz                       
             - 13.21% parse_offset                        
                - 12.45% __isoc99_sscanf                  
                   - 9.05% __vfscanf_internal             
                        1.25% __GI_____strtoull_l_internal
                        0.52% __uflow                     
                     1.06% _IO_str_init_static_internal   
               0.90% parse_tzname                         
            1.48% __offtime                               
            1.41% __tz_compute                            
         1.31% __offtime                                  
```

tzset主要耗在xstat64的系统调用上(68%热点)。

`__xstat64(68%) ->el0_sve->__arm64_sys_newfstatat-> __do_sys_newfstatat->vfs_statx(54%)+cp_new_stat(4%) `

再展开看：

``` c
54.67% vfs_statx                                    
 - 50.90% user_path_at_empty                        
    - 44.94% filename_lookup                        
       - 42.48% path_lookupat.isra.0                
          - 21.96% link_path_walk.part.0            
             - 14.33% walk_component                
                - 8.26% lookup_fast                 
                     __d_lookup_rcu                 
                - 2.99% step_into                   
                     1.00% __follow_mount_rcu.isra.0
                     0.93% __lookup_mnt             
                - 2.13% handle_dots.part.0          
                     0.98% step_into                
                     0.94% follow_dotdot_rcu        
             - 2.25% inode_permission               
                  0.81% generic_permission          
                  0.73% security_inode_permission   
          + 6.88% walk_component                    
          + 6.05% complete_walk                     
            3.58% hash_name                         
          + 3.33% path_init                         
       - 2.14% putname                              
            1.37% kmem_cache_free                   
            0.65% __cmpxchg_double                  
    + 5.76% getname_flags.part.0                    
 + 2.04% path_put                                   
 + 1.22% vfs_getattr_nosec                          
```



### 内核实现


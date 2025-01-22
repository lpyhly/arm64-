[TOC]



# 默认编译选项

gcc -O -DRUSAGE -DHAVE_uint=1 -DHAVE_int64_t=1 -DHAVE_pmap_clnt_h -DHAVE_socklen_t -DHAVE_DRAND48 -DHAVE_SCHED_SETAFFINITY=1   -c lib_tcp.c -o ../bin//lib_tcp.o



gcc -O -DRUSAGE -DHAVE_uint=1 -DHAVE_int64_t=1 -DHAVE_pmap_clnt_h -DHAVE_socklen_t -DHAVE_DRAND48 -DHAVE_SCHED_SETAFFINITY=1   -o ../bin//userdef_sigaction userdef_sigaction.c ../bin//lmbench.a -lm

# lmbench中的调度策略

## 函数解析

### 设置调度：handle_schduler函数解析

benchmp中调度的接口函数为`handle_scheduler`，其中3个参数：

1. childno：只是个标记，与pid无关，在lmbench中用来区分多进程的不同进程，因此范围设定为`[0, parallel-1]`;
2. benchproc：只是个标记。当**测试程序**自己会fork子进程时才会用到（benchmp自己的fork无关）。测试程序的主进程标记为`0`，测试程序创建的子进程们标记为`[1, nbenchprocs]`。
3. nbenchprocs：测试程序的子进程个数，当**测试程序**自己会fork子进程时才会用到，否则置为`0`.
### 获取总核数

获取可以运行程序的核数，linux中通过glibc中的变量`_SC_NPROCESSORS_ONLN`来获取online的core个数。

``` c
int sched_ncpus()                                     
{                                                 
......
#elif defined(_SC_NPROCESSORS_ONLN)               
        /* AIX, Solaris, and Linux interface */   
        return sysconf(_SC_NPROCESSORS_ONLN);     
#else                                             
        return 1;                                 
#endif                                            
}                                         
```

### 设置调度
linux中通过`sched_setaffinity`来将任务绑定到不同的核上。


## 具体调度策略

通过变量`LMBENCH_SCHED`来指定调度策略。有如下几种：

| 调度策略               | 说明                                                         | 代码体现                                                     |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| DEFAULT（默认）        | 什么都不做                                                   | 直接return 0                                                 |
| SINGLE                 | 全部分配到cpu 0                                              | cpu = 0                                                      |
| BALANCED               | 每个进程绑定在单独的固定核，使用的核为{0，1，……，进程数-1}   | cpu = childno                                                |
| BALANCED_SPREAD        | 每个进程绑定在单独的固定核，但使用的核号与上面不同：childno越接近，实际使用的核号越遥远。 | cpu = reverse_bits(childno);//返回值是入参的高低位对调的结果，比如共16个核，则0001返回1000,0011返回1100. |
| (不常用）UNIQUE        | 以benchprocs粒度分配core，每个benchprocs绑定在单独的固定核   | cpu = childno * (nbenchprocs + 1) + benchproc;               |
| (不常用）UNIQUE_SPREAD | 以benchprocs粒度分配core，每个benchprocs绑定在单独的固定核，但使用的核号与上面不同：childno/benchproc越接近，实际使用的核号越遥远。 | cpu = reverse_bits(childno * (nbenchprocs + 1) + benchproc); |
| (不常用）CUSTOM        | 用户自定义核号。比如指定到：LMBENCH_SCHED="COMSTOM 1 2 3"，则进程会根据childno依次绑定到c1,c2,c3,c1,c2,c3…… | cpu = custom(sched + strlen("CUSTOM"), childno);             |
| (不常用) CUSTOM_SPREAD | 用户自定义核号。比如指定到：LMBENCH_SCHED="COMSTOM_SPREAD 1 2 3"，则进程会根据childno+benchproc依次绑定到c1,c2,c3,c1,c2,c3…… | cpu = custom(sched + strlen("CUSTOM_SPREAD"),             childno * (nbenchprocs + 1) + benchproc); |




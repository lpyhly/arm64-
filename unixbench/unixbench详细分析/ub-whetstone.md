# whetstone

# 源码解读

## ub测试内容分析

包含 C/C++ Whetstone Benchmark Single or Double Precision。 

其中，有6类测试，分别是：

1. Original concept        Brian Wichmann NPL      1960's
2. Original author         Harold Curnow  CCTA     1972
3. Self timing versions    Roy Longbottom CCTA     1978/87
4. Optimisation control    Bangor University       1987/90
5. C/C++ Version           Roy Longbottom          1996
6. Compatibility & timers  Al Aburto               1996

## ub源码分析

### 编译选项

``` c
gcc -o ./pgms/whetstone-double -DTIME -Wall -pedantic -Wno-unused-variable -O2 -fomit-frame-pointer -fforce-addr -ffast-math -Wall -DDP -DUNIX -DMYUNIXBENCH ./src/whets.c -lm
```



### 初始化代码分析

``` c
main：
    循环训练，找到最合适的循环次数； TimeUsed
    运行测试：    -> whetstones(xtra,x100,calibrate); 
		/* xtra: 单个session内的循环次数； x100: 数据量; calibrate: 0则开启内部打印。 */
	计算结果校验（和正确值作对比）
	性能结果收集&打印
```

在`whetstones`函数中，根据测试内容分成6段，依次运行，段内循环执行，且每个段分别计时（性能结果仅和计算时间相关，注意），保存计算结果并根据情况判断是否打印内部的信息，最后做时间相加。所以，不能直接按照IPC来计算`whetstone`的性能。



### loop主循环代码分析

## 指令流分析

### ESL指令流分析
[TOC]



#规格参数

## 1630V200

**整型：**

​	**3个ALU，每个ALU有2条pipeline，但是比688更灵活，两个pipeline可以并发执行，pipeline0可以走1cycle的指令或者br，pipeline1可以走1c/2c/mac/std/stdp/div等指令。其中ALU 0/1/2的pipeline 0/1均可以走add， ALU 0/1/2的pipeline 1可以走mul，ALU 0/1/2的pipeline 1随意一个都可以走div，但一次只能执行1条，ALU 0/1/2的pipeline 0都可以走branch。**

​	**add一拍，3*2的并发； mul 2-3拍（64bit为3拍）；div 8拍，1个MDU单元。**

**浮点：**

​	**add 2拍，4条FSU pipe； mul 3拍，4条FSU pipe；div 7-10拍，4条pipe可并发且迭代为3-6 cycle。**



## 688

**整型：**

​	**4个ALU，每个ALU有2条pipeline。其中ALU 0/1/2/3均可以走add， ALU2/3可以走mul/branch， ALU 3 可以走div。**

​	**add一拍，4个ALU单元； mul 2-3拍（64bit为3拍），2个MDU单元；div 8拍，1个MDU单元。**

**浮点：**

​	**4条pipeline**

**add 2拍，4条FSU pipe； mul 3拍，4条FSU pipe；div 7-10拍，4条pipe可并发且迭代为3-6 cycle。**

具体：

- 4个ALU单元

- 2个MDU单元

- 2个BRU单元：分支跳转
- 2个LSU单元： 2 * 256 bit的LS， 2 * 128 bit的ST
- FSU： 4条pipe

## 1620

**总的来说：**

​	**整型： add+lsl为2拍，3个ALU单元； mul 3-4拍（64bit为4拍），1个MDU单元；div 8拍，1个MDU单元。**

​	**浮点： add 2拍，2条FSU pipe； mul 5拍，2条FSU pipe；div 7-10拍，pipe不可并发。**

``` c
688优化点 - 时延
    1. 整型mul减少1拍；
    2. 整型add + lsl减少1拍；
    3. 浮点add、mul减少了2拍；
688优化点 - 并发度
    1. FSU pipe从2增加到4
    2. MDU单元从1个增加到2个。
    3. 浮点div可通过迭代并发了。
    
其他：
    1. 整型div只提升了频率比；
    2. 整型div的back-to-back还比1620差了1拍。
```



3个ALU： 包含1个纯ALU,和2个ALU/BRU， 作用是整型运算以及分支操作；

1个MDU：multi-cycle，作用包含整型shilt-ALU/乘/除/CRC。

2个LSU：2 * 128的LDA0/LDA1 + 2 * 64的STA0/STA1

2个FSU：2 * 128bit，asimd运算，fp运算，crypto运算。

# lmbench-计算指令详解

## 综述

编译器优化的项目如下，check代码是需要注意。

- int bit， int64 bit操作的串并行操作都进行了优化；

- int add，int64 add的并行操作进行了优化。

## 整型
### integer bit

#### 串行

1. 源码

``` c
while (iterations-- > 0) {                   
    HUNDRED(r ^= iterations; s ^= r; r |= s;)
}                                            
```

认为上述为3条指令，结果为单条指令的时延。

1. 汇编pattern

``` c
eor w0, w0, w1
eor w2, w0, w3
orr w0, w0, w3
```

3. 分析

   实测688为2拍3个，latency = 0.22 ns，符合设计规格。其中第一条用1拍，第二、三条没有依赖，因此可以并发去做，用1拍。

#### 并行

1. 源码

并发情况比较特殊。与lat_ops的实现不同，lat_ops使用iteration作为中间值。 而这里的并发指令的数据流pattern为：

``` c
r##N ^= s##N; s##N ^= r##N; r##N |= s##N;
```

1. 汇编

编译器将这种pattern进行了优化，原理是s = (r ^ s) ^ s = r 。因此并发情况下的指令流的反汇编为（没有用到eor，变成了mov + orr）

``` c
// 单并发
mov w2, w0
orr w0, w0, w2
// 三并发
mov w2, w19
mov w3, w20
mov w4, w0
orr w0, w0, w4
orr w20, w20, w3
orr w19, w19, w2
```

3. 分析

   实测这种模式下，并发度为2.28。 其中，单并发的两条指令时延为0.33ns（1拍做完mov + orr，因为mov可以在rename阶段完成，因此认为mov操作0拍完成，那么理论IPC = 2)，多并发时延为0.15ns，实际IPC已经达到5.x了。
   
   在1630V200，并发度为3.47。原因是dispatch最大发射数从6增大到8，因此最高IPC提升了。目测3.47的并发度的IPC可以接近7了。

#### x86

1. 汇编pattern

``` c
// 串行：
xor    %eax,%edi
mov    %edi,%edx
xor    %ecx,%edx
or     %ecx,%edi
```

 2. 分析

    ​	在6248测试， integer mod时延为0.27ns=0.67 cycle，即2拍完成3个操作，根据源码分析，3个操作刚好对应上述4条汇编指令。

    

#### x86并行

1. 汇编pattern

   ``` c
   // 5并发
   jmp    40159d <integer_bit_4+0x72>
   mov    %ebx,%edx
   mov    %ebp,%ecx
   mov    %r12d,%r8d
   mov    %r13d,%r9d
   mov    %edi,%r11d
   or     %r11d,%edi
   or     %r9d,%r13d
   or     %r8d,%r12d
   or     %ecx,%ebp
   or     %edx,%ebx
   sub    $0x1,%rax
   cmp    $0xffffffffffffffff,%rax
   jne    401579 <integer_bit_4+0x4e>
   ```

2. 分析

   ​	在6248测试，integer mod最大并发度为2.34，对应时延为0.28 cycle；在8380测试，最大并发度为4.64，对应时延为0.31ns * 2.3GHz/4.64 =0.15 cycle。 上述指令均无依赖，理论最大值与最大IPC强相关，目测Intel 8380的IPC有所提升。

   

### integer add

#### 串行

1. 源码

   ``` c
   a = a + a + i
   ```

   上述指令认为是2个加法操作。

2. 汇编

   反汇编出来是1条指令

   ``` c
   add w0, w1, w0, lsl #1
   ```

3. 分析

   关于 add + lsl， Intel上可以1拍commit掉上述这2个操作（实测），688这里有dts单，目前已经修复为1拍可以commit完毕。

#### 并行

1. 源码

   ``` c
   a##N += b##N; b##N -= a##N;
   ```

2. 汇编

   实际反汇编与add并没有什么关系…… 使用mov/neg/sub指令集合进行操作的。

   ``` c
   // 单并发
   mov     w0, w2
   mov     w2, w1
   neg     w1, w1
   sub     w1, w1, w0
   
   // 两并发
   mov     w4, w3
   mov     w3, w2
   mov     w0, w20
   mov     w20, w19
   neg     w2, w2
   neg     w19, w19
   sub     w2, w2, w4
   sub     w19, w19, w0
   ```

3. 分析

   实测Intel的并发度接近2， 688的并发度为2.6.

   因为按照这种pattern，单并发的时延为0.67ns（2拍)，应该是neg一拍，sub一拍。

   多并发的时候，最多4个neg同时为1拍，最多4个sub同时为1拍。但由于6发射的限制（mov也要算上），这时候的IPC已经到了5.x了。所以并发度2.6是合理的。

 	 如果是按照串行的那个pattern，跟梁永祥确认，4条pipe理论并发度可以达到4。

​		在1630V200上，实测并发度为3.16. 单并发的时延应该是2cycle，IPC接近2.多并发场景，最多有6个neg或sub同时执行（neg和，在3并发场景，有6mov + 3neg + 3sub，可以在2拍完成，此时理论IPC=6；在4并发场景，有8mov+4neg+4sub，可以在2拍完成，理论IPC可以达到8。考虑到8发射的限制，V200上实测并发度3.16是合理的，对应的IPC约6.3.



### integer mul

#### 串行

1. 源码

``` c
r *= s;
```

2. 汇编

``` c
mul w2, w2, w1
```

3. 分析

   实测688为0.68ns，也就是2拍commit 1条mul指令，符合设计规格。

#### 并行

1. 源码

``` c

```

2. 汇编

``` c
mul w2, w2, w1
mul w4, w4, w3
mul w6, w6, w5
mul w8, w8, w7
```

3. 分析

   实测688并发度为4.2，688设计规格有2条mul的执行单元。
   
   实测V200并发度为6.25，设计规格有3条mul的并发pipeline。

### integer div

32位整型除法：
被除数（s）： 38<<20
除数（r）： 37
r = s/r;
循环执行： 39845888   ÷   1076915/39845888   ÷37

#### 串行

1. 源码

``` c
int r, s;
r = s / r;
```

2. 汇编

``` c
sdiv    w0, w1, w0
```

3. 分析

   实测688为2.83ns，即8拍commit 1条指令。 实际时延与数据的值相关。
   
   688设计规范为32bit div：6 - 12 cycle， 64bit div：6 - 20 cycle。 除法得到的结果值越小，则计算速度越快。


#### 并行

1. 源码

``` c

```

2. 汇编

``` c

```

3. 分析

   实测688并行度为1，（1620为1.13，6248为3.86）。

   关于比1620性能差的原因分析：

   1. DIV DP（除法运算单元数据通路），在1620CS (TaishanV110PG2) 和 P688 (LinxCore910V100)具有相同的 latency 和 throughput，都是：
        \- throughput = 1，即1个DIV部件
        \- latency 最小值为 6 cycles，即 E1->E2-> ... -> E6，E6对W1, write back to regfile.

   2. DIV 控制通路对1620CS (TaishanV110PG2) 和 P688 (LinxCore910V100) 是不同的：
        \- 1620CS (V110PG2)：DIV 指令由 LS ISSQ 发射；作为4-cycle指令reflow 
        \- P688 (LC910V100)：DIV 指令由 ALU ISSQ 发射；作为5-cycle指令reflow

   3. 问题根本原因在于，LS ISSQ 和 ALU ISSQ 的 ready/blocking -> pick 的实现机制有所不同
        \- LS ISSQ：P1 ready 当拍 pick，即 ready/block 对 P1 stage
        \- ALU ISSQ：P0 ready 下一拍(P1) pick，即 ready/block 对 P0 stage （这是为了解 ALU ISSQ wkup LS ISSQ timing critical path 而做的机制）
        \- 这会使最快的 连续 back-to-back DIV 有不同的时序，详见附件的时序图
        \- 从时序图上看出，虽然 div done都是同一拍（E2）给出，但 V110PG2下一拍 div active 拉低，ready同拍 pick，对P1 stage；而 LC910V100下一拍 div busy block 撤离，ready 对P0，下一拍才P1 (pick). 这样 V110PG2 fastest P1-to-P1 是 5拍，而 LC910V100 fastest P1-to-P1 是6拍。

   关于比Intel差的原因分析：

   ​	div单元数少于Intel。

### integer mod

与integer div类似。

初始值：

``` c
register int r = pState->N + iterations;
register int s = pState->N + 62;
// 688使用"-N=1"（不加-N参数则使用N=11）
// r的值与iterations相关，这个是统计出来的，每次测试不完全相同。含义是：enough=5000 us下能跑多少次iterations。
```

#### 串行

1. 源码

``` c
  r %= s; r |= s;
```

上述操作认为是1条指令。

2. 汇编

``` c
sdiv    w2, w0, w1   // 8 cycle
msub    w2, w2, w1, w0  // 2cycle
orr     w2, w1, w2         //  1cycle,可以被下一条sdiv吃掉（w2无依赖，可以rename）。
```

在1630V200上，汇编有修改，因为指令依赖导致理论值增加了orr的 1 cycle。

``` c
sdiv    w2, w0, w1   // 8 cycle
msub    w0, w2, w1, w0 // 2cycle
orr     w0, w1, w0     // 1cycle， 且与下一条指令存在指令依赖
sdiv    w2, w0, w1
msub    w2, w2, w1, w0
orr     w2, w1, w2
sdiv    w0, w2, w1
msub    w0, w0, w1, w2
orr w2, w1, w0
sdiv    w0, w2, w1
msub    w0, w0, w1, w2
orr w0, w1, w0
```



1. 分析

实测688为3.12ns，即9.4拍作为上述3条操作，其中sdiv为8拍。

实测1630V200为4.81ns，即12拍作为上述3条操作，其中sdiv为8拍。

#### 并行

初始值同上。

1. 源码

``` c
r##N %= s##N; r##N |= s##N;
```

2. 汇编

``` c
// 单并发
sdiv    w2, w0, w1
msub    w0, w2, w1, w0
orr     w2, w1, w0
    或者
sdiv    w0, w2, w1
msub    w0, w0, w1, w2
orr     w0, w1, w0
    或者
sdiv    w2, w0, w1
msub    w2, w2, w1, w0
orr     w0, w1, w2
// 4并发
sdiv    w0, w8, w4
msub    w0, w0, w4, w8
orr     w0, w4, w0
sdiv    w21, w7, w3
msub    w21, w21, w3, w7
orr     w7, w3, w21
sdiv    w20, w6, w2
msub    w20, w20, w2, w6
orr     w6, w2, w20
sdiv    w19, w5, w1
msub    w19, w19, w1, w5
orr     w5, w1, w19
```

3. 分析

   并发过程中，msub和orr的时延都可以被吃掉，sdiv只能单指令串行执行，并发程度完全由sdiv执行时间决定。
   
   实测688并发度为1.49（1620为2，6248为4.13），详见div的分析。
   
   1630V200：
   
   ​	实测1630V200并发度为1.67。



### int64 bit

#### 串行

1. 源码

``` c
int64 r, s, i;
while (iterations-- > 0) {
	r ^= i; s ^= r; r |= s;
}
```

2. 汇编

``` c
eor x0, x0, x1
eor x2, x0, x3
orr x0, x0, x3
```

与int bit一致，只是寄存器从w变成了x而已。并且发现，这个pattern没有被编译器优化掉，可能是因为有3个变量。

3. 分析

实测688为0.22ns，，即IPC = 1/（0.22*3） = 1.5， 与integer bit一致。

#### 并行

1. 源码

``` c
register int64 r##N, s##N, i##N;
r##N ^= i##N; s##N ^= r##N; r##N |= s##N;
```

2. 汇编

``` c
// 单并发（多并发行为一致）
eor x0, x1, x0
eor x2, x0, x3
orr x0, x0, x3
```

3. 分析

   实测688并发度为2.47，比integer bit高。原因在于，int64 bit使用了3个寄存器，因此汇编指令中没有被优化掉，因此指令于int bit完全不同，没有可比性。那么，理论IPC = 4，因为eor/orr全部都在ALU做，ALU有4个。由于串行IPC=1.5，并行IPC=4，则理论并发度为4/1.5 = 2.67，差不多。
   
   1630V200中，3*2个ALU pipeline可以并发执行，所以最大IPC=6，理论并发度为6/1.5=4，目前实测为3.46.
   
   ​	

### int64 add

#### 串行

1. 汇编

``` c
add x0, x1, x0, lsl #1
```

与int add行为一致。

2. 分析

实测688为0.17ns（每拍commit 2条指令），与int add行为一致。

#### 并行

1. 源码

``` c
a##N += b##N; b##N -= a##N;
```

2. 汇编

``` c
// 1并发
mov     x0, x2 
mov     x2, x1
neg     x1, x1 
sub     x1, x1, x0
// 4并发
mov     x7, x3
mov     x3, x2
mov     x6, x24
mov     x24, x21
mov     x5, x23
mov     x23, x20
mov     x4, x22
mov     x22, x19
neg     x2, x2
neg     x21, x21
neg     x20, x20
neg     x19, x19
sub     x2, x2, x7
sub     x21, x21, x6
sub     x20, x20, x5
sub     x19, x19, x4
```

代码进行了优化，原理大致是：

​		 a1 = a0 + b0;  

​		 b1 = b0 - a1 = b0 - (a0 + b0) = - a0

​		理论并发度待分析。

3. 分析

实测688并发度为2.63，与int add行为一致。



### int64 mul

#### 串行

1. 源码

``` c
 TEN(r *= s;); r -= t;
```

2. 汇编

``` c
mul x1, x1, x2
```

3. 分析

   实测688为1.01ns，即3拍完成。**INT64乘法的latency是3拍.**

#### 并行

1. 源码

``` c
---
---
```

2. 汇编

``` c
mul     x0, x4, x0
mul     x21, x3, x21
mul     x20, x2, x20
mul     x19, x1, x19
mul     x0, x4, x0
mul     x21, x3, x21
mul     x20, x2, x20
mul     x19, x1, x19
```

3. 分析

   实测688并发度为6.18，算下来IPC=2（并行16的时延为1.68ns），刚好有2个 mul单元，与理论值匹配。



### int64 div

64位整型除法：
被除数（s）：  (r + 17) << 13 = 262,967,996,992,372,736‬
除数（r）： 37 + 37 <<33 = 32,100,585,570,341‬
r = s/r;

#### 串行

1. 源码

``` c
r = s / r;
```

2. 汇编

``` c
sdiv    x0, x1, x0
```

3. 分析

   实测688为2.33ns，就是说7拍完成。而integer div为8拍完成。
   
   688设计规范为32bit div：6 - 12 cycle， 64bit div：6 - 20 cycle。 除法之后的结果值越小，则计算速度越快。

#### 并行

3. 分析

   实测688并发度为1，与int div一致，比1620低（1.17），原因见int div的分析。



### int64 mod

#### 串行

2. 汇编

``` c
sdiv    x0, x2, x1
msub    x0, x0, x1, x2
orr x2, x1, x0
```

3. 分析

   实测688为3.45ns，即10拍完成，比int mod多了1拍，其中sdiv为7拍比int mod少了1拍，orr是一致的，msub为乘减，是和sdiv有依赖的，需要3拍。

#### 并行

1. 源码

``` c

```

2. 汇编

``` c

```

3. 分析

   实测688并发度为1.82(即IPC=0.17)。

   - 比1620（2.2）低，原因同int div。
   - 比int mod高（1.49），因为串行的时延多了1拍，如果看IPC的话，是和int mod的IPC一致。认为是合理的，因为div单元只有1个，完成一条指令需要5 cycle。因此理论IPC约为1/5=0.2。



### float add

#### 串行

1. 源码

``` c
 while (iterations-- > 0) {
     TEN(f += (float)f;) f += (float)g;
     TEN(f += (float)f;) f += (float)g;
 }
```

2. 汇编

``` c
fadd    s0, s0, s0
```

3. 分析

   实测688为0.67ns，即2拍完成1个fadd，单精度add的lattency为2 cycle，符合设计规格。

#### 并行

1. 源码

``` c

```

2. 汇编

``` c

```

3. 分析

   实测688并行度为7.49，合理。因为FSU有4条pipe，流水线可以排满，因此每周期可以commit 4个。

### float mul

#### 串行

1. 源码

``` c
while (iterations-- > 0) {
    TEN(f *= f; f *= g;);
    TEN(f *= f; f *= g;);
}
```

2. 汇编

``` c
fmul    s0, s0, s0
fmul    s0, s8, s0
```

3. 分析

   实测688为1 ns，即3拍完成1个float mul，比int mul多1拍，与int64 mul相同。

#### 并行

1. 源码

``` c
r##N *= r##N; r##N *= s##N;
```

2. 汇编

``` c
// 4并发
fmul    s1, s1, s1     // 1cycle commit
fmul    s2, s25, s1    // 依赖于上一条完成后再执行
fmul    s20, s20, s20
fmul    s1, s26, s20
fmul    s0, s0, s0
fmul    s0, s15, s0
fmul    s19, s19, s19
fmul    s19, s14, s19
```

3. 分析

   实测688并发度为11.19，可以认为fmul能够全流水，FSU的4条pipe，每周期都可以commit 4条mul指令。

### float div

#### 串行

1. 源码

``` c
 while (iterations-- > 0) {
     FIVE(TEN(f = g / f;) TEN(g = f / g;))
 }
```

2. 汇编

``` c
fdiv    s0, s1, s0
```

3. 分析

   实测688为2.33ns，即7拍完成1条div。

#### 并行

1. 源码

``` c
r##N = s##N / r##N;
```

2. 汇编

``` c
// 12并发
fdiv    s1, s23, s1
fdiv    s0, s14, s0
fdiv    s19, s15, s19
fdiv    s18, s13, s18
fdiv    s17, s12, s17
fdiv    s16, s11, s16
fdiv    s7, s10, s7
fdiv    s6, s9, s6
fdiv    s5, s8, s5
fdiv    s4, s20, s4
fdiv    s3, s21, s3
fdiv    s2, s22, s2
```

3. 分析

   实测688并发度为9.17，（FSU unit的宽度为256bit）
   
   时延是7 cycle，迭代为3 cycle，因此16并发的话，4条pipe，每条pipe有4条独立数据流，data0在commit后又可以重新上pipeq，且没有指令依赖，因此每3 cycle可以commit 4条指令，IPC = 4/3。理论并发度为$(4/3) / (1/7) = 9.33$， 符合预期。
   
   
   
   ```
   对每一条pipe来说：
   cycle: 1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16
   p0-d0: ---------------7------------
   p0-d1:              ---------------7-----------
   p0-d2                          ---------------7-----------
   p0-d3                                      --------------7------------
   p0-d0:                                                 --------7-----------         
                    
   ```
   
   

### double add

#### 串行

1. 源码

``` c
 while (iterations-- > 0) {
     TEN(f += (double)f;) f += (double)g;
     TEN(f += (double)f;) f += (double)g;
 }
```

2. 汇编

``` c
fadd    d0, d0, d0
```

3. 分析

   实测688为0.67ns，即2拍完成，与float add一致。

#### 并行

3. 分析

   实测688为7.5，符合预期。分析详见float add并行部分。

   

### double mul

#### 串行

1. 源码

``` c

```

2. 汇编

``` c

```

3. 分析

   实测688为1ns，即3拍完成，与float mul一致。

#### 并行

1. 源码

``` c

```

2. 汇编

``` c

```

3. 分析

   实测688并发度为11.18，即4条FSU pipe全流水并发，符合预期。

   

### double div

#### 串行

1. 源码

``` c

```

2. 汇编

``` c

```

3. 分析

   实测688为3.34ns，即10拍完成，符合预期，比float多的3拍是在迭代运算上。

#### 并行

1. 源码

``` c

```

2. 汇编

``` c
// 单并发
fdiv    d0, d8, d0
// 8并发
fdiv    d1, d8, d1
fdiv    d0, d14, d0
fdiv    d7, d15, d7
fdiv    d6, d13, d6
fdiv    d5, d12, d5
fdiv    d4, d11, d4
fdiv    d3, d10, d3
fdiv    d2, d9, d2
```

3. 分析

   实测688并发度为6.66，总共4条pipe， 每条pipe的时延10cycle，迭代6cycle，因此可以认为6 cycle commit 4条指令， IPC=0.66， 理论并行度为0.66/0.1 = 6.66，完美契合。

### float bogomflops

#### 串行

1. 源码

``` c
while (iterations-- > 0) {
    register float *x = (float*)pState->data;
    for (i = 0; i < M; ++i) {
        x[0] = (1.0f + x[0]) * (1.5f - x[0]) / x[0];
        x[1] = (1.0f + x[1]) * (1.5f - x[1]) / x[1];
        x[2] = (1.0f + x[2]) * (1.5f - x[2]) / x[2];
        x[3] = (1.0f + x[3]) * (1.5f - x[3]) / x[3];
        x[4] = (1.0f + x[4]) * (1.5f - x[4]) / x[4];
        x[5] = (1.0f + x[5]) * (1.5f - x[5]) / x[5];
        x[6] = (1.0f + x[6]) * (1.5f - x[6]) / x[6];
        x[7] = (1.0f + x[7]) * (1.5f - x[7]) / x[7];
        x[8] = (1.0f + x[8]) * (1.5f - x[8]) / x[8];
        x[9] = (1.0f + x[9]) * (1.5f - x[9]) / x[9];
        x += 10;
    }
}
```

每一行的+-*/算作一次操作。

1. 汇编

``` c
ldr s3, [x0]
fadd    s2, s3, s1
fsub    s4, s0, s3
fmul    s2, s2, s4
fdiv    s2, s2, s3
str s2, [x0]
ldr s3, [x0, #4]
fadd    s2, s3, s1
fsub    s4, s0, s3
fmul    s2, s2, s4
fdiv    s2, s2, s3
str s2, [x0, #4]
ldr s3, [x0, #8]
fadd    s2, s3, s1
fsub    s4, s0, s3
fmul    s2, s2, s4
fdiv    s2, s2, s3
str s2, [x0, #8]
```

每次loop都是从内存中ldr上来，再str回去，因此loop间是无依赖的，可以通过rename来并发。每个loop的时延为：ldr 4cycle + fadd/fsub（无依赖，可rename同时执行） 2cycle + fmul 3cycle + fdiv 7cycle + str(算到resolve即可，此时已释放资源) 1cycle= 17 cycle。 单个loop内使用了6个寄存器，看物理寄存器的个数决定最多并发多少个loop。



1. 分析

   实测688为0.67ns。因为代码块之间没有依赖，可以通过rename寄存器实现全并发，相比1620的收益有2方面，一是指令时延降低，二是并发单元增多。


### double bogomflops

#### 串行

1. 源码

``` c

```

2. 汇编

``` c

```

1. 分析

   实测688为0.7ns，相比单精度来说除法时延多了一点，符合预期。



# Hi1630V200的变种

|                          | ns   | cycle             |                                                              |                                  |
| ------------------------ | ---- | ----------------- | ------------------------------------------------------------ | -------------------------------- |
| integer  div(单位：ns)   | 3.21 | 8                 | sdiv  w0, w1, w0                                             | 符合预期                         |
| integer  mod(单位：ns)   | 4.81 | 12                | sdiv  w2, w0, w1  msub  w0, w2, w1, w0  orr  w0, w1, w0  sdiv  w2, w0, w1  msub  w2, w2, w1, w0  orr  w2, w1, w2  sdiv  w0, w2, w1  msub  w0, w0, w1, w2  orr  w2, w1, w0  sdiv  w0, w2, w1  msub  w0, w0, w1, w2  orr  w0, w1, w0 | 3条指令全依赖：  8 + 3 + 1 =  12 |
| int64  div(单位：ns)     | 4.41 | 11                | sdiv  x0, x1, x0                                             | 符合预期                         |
| int64  mod(单位：ns)     | 6.41 | 16                | sdiv  x0, x2, x1  msub  x0, x0, x1, x2  orr  x2, x1, x0      | 11 + 4 +  1=16                   |
| integer  div parallelism | 8.01 | 每cycle 1条指令   | sdiv  w0, w4, w0     sdiv  w21, w3, w21    sdiv  w20, w2, w20    sdiv  w19, w1, w19 | 符合预期                         |
| integer  mod parallelism | 8.95 | 12 cycle  9条指令 | sdiv  w6, w0, w3        msub  w0, w6, w3, w0      orr   w6, w3, w0          sdiv  w5, w20, w2       msub  w20, w5, w2, w20     orr   w5, w2, w20         sdiv  w4, w19, w1       msub  w19, w4, w1, w19     orr   w4, w1, w19 | ???                              |
| int64  div parallelism   | 5.5  | 6cycle  3条指令   | sdiv  x0, x3, x0    sdiv  x20, x2, x20   sdiv  x19, x1, x19  | 符合预期                         |
| int64  mod parallelism   | 7.93 | 16 cycle  8条指令 |                                                              | 符合预期                         |




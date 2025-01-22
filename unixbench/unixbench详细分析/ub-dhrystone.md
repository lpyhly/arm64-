[TOC]

# 性能优化

1. 编译器

   默认是采用gcc编译，如果采用icc编译性能，可以显著提升。dry2stone整数运算，性能可以提升7.9%。

    -- from https://developer.aliyun.com/article/674732

2. 编译选项

   在1620中添加 `-mtune=tsv110`，可以提升性能。原理见下面。

   ### 分数计算方式
   
   `score = Run_Index / 总耗时（us） * 1000000 / 116700 * 10`
   
   `单次循环耗时（us） = 总耗时/Run_Index = 1000000 / 116700 * 10 /score  `
   
   其中，Run_Index为循环次数。
   
   

# 源码解读

## ub测试内容分析

主要分为3大块，比例分别为：

``` c
 assignments                  52 (51.0 %)
 control statements           33 (32.4 %)
 procedure, function calls    17 (16.7 %)
 // 共103条statement（不同编译器之间略有差别）
```

### 1. 表达式角度解析

具体指令为：

``` c
/*  1. Statement Type:
*  -----------------             number
*
*     V1 = V2                     9
*       (incl. V1 = F(..)
*     V = Constant               12
*     Assignment,                 7
*       with array element
*     Assignment,                 6
*       with record component
*                                --
*                                34       34
*
*     X = Y +|-|"&&"|"|" Z        5
*     X = Y +|-|"==" Constant     6
*     X = X +|- 1                 3
*     X = Y *|/ Z                 2
*     X = Expression,             1
*           two operators
*     X = Expression,             1
*           three operators
*                                --
*                                18       18
*
*     if ....                    14
*       with "else"      7
*       without "else"   7
*           executed        3
*           not executed    4
*     for ...                     7  |  counted every time
*     while ...                   4  |  the loop condition
*     do ... while                1  |  is evaluated
*     switch ...                  1
*     break                       1
*     declaration with            1
*       initialization
*                                --
*                                34       34
*
*     P (...)  procedure call    11
*       user procedure      10
*       library procedure    1
*     X = F (...)
*             function  call      6
*       user function        5
*       library function     1
*                                --
*                                17       17
*                                        ---
*                                        103
*
*    The average number of parameters in procedure or function calls
*    is 1.82 (not counting the function values as implicit parameters).
*/
```

### 2. 操作符角度解析

``` c
/*  2. Operators
*  ------------
*                          number    approximate
*                                    percentage
*
*    Arithmetic             32          50.8
*
*       +                     21          33.3
*       -                      7          11.1
*       *                      3           4.8
*       / (int div)            1           1.6
*
*    Comparison             27           42.8
*
*       ==                     9           14.3
*       /=                     4            6.3
*       >                      1            1.6
*       <                      3            4.8
*       >=                     1            1.6
*       <=                     9           14.3
*
*    Logic                   4            6.3
*
*       && (AND-THEN)          1            1.6
*       |  (OR)                1            1.6
*       !  (NOT)               2            3.2
*
*                           --          -----
*                           63          100.1
*/
```

### 3. 操作类型解析

``` c
                        number    approximate
                                  percentage

   Integer               175        72.3 %
   Character              45        18.6 %
   Pointer                12         5.0 %
   String30                6         2.5 %
   Array                   2         0.8 %
   Record                  2         0.8 %
                         ---       -------
                         242       100.0 %
```

When there is an access path leading to the final operand (e.g. a record component), only the final data type on the access path is counted.

### 4. 逻辑关系解析

```c
                            number    approximate
                                      percentage

 local variable              114        47.1 %
 global variable              22         9.1 %
 parameter                    45        18.6 %
    value                        23         9.5 %
    reference                    22         9.1 %
 function result               6         2.5 %
 constant                     55        22.7 %
                             ---       -------
                             242       100.0 %
```



## ub源码分析

### 编译选项

``` makefile
# 实际编译指令：
cd ./src; gcc -c -DTIME -Wall -pedantic -ansi -DHZ= -O2 -fomit-frame-pointer -fforce-addr -ffast-math -Wall dhry_1.c
cd ./src; gcc -c -DTIME -Wall -pedantic -ansi -DHZ= -O2 -fomit-frame-pointer -fforce-addr -ffast-math -Wall dhry_2.c
gcc -o ./pgms/dhry2 -DTIME -Wall -pedantic -ansi -O2 -fomit-frame-pointer -fforce-addr -ffast-math -Wall ./src/dhry_1.o ./src/dhry_2.o
```

在`dhry.h`中，

```c
#define TIMES
#define Mic_secs_Per_Second     1000000.0
#define structassign(d, s)      d = s
typedef int     One_Thirty;
typedef int     One_Fifty;
typedef char    Capital_Letter;
typedef int     Boolean;
typedef char    Str_30 [31];
typedef int     Arr_1_Dim [50];
typedef int     Arr_2_Dim [50] [50];
```



### 初始化代码分析

```assembly
0000000000400790 <main>:
  400790:   a9b653f3    stp x19, x20, [sp, #-160]!
  400794:   d0000094    adrp    x20, 412000 <exit@GLIBC_2.17>
  400798:   91078294    add x20, x20, #0x1e0
  40079c:   a9015bf5    stp x21, x22, [sp, #16]
  4007a0:   d0000095    adrp    x21, 412000 <exit@GLIBC_2.17>
  4007a4:   910302b3    add x19, x21, #0xc0
  4007a8:   a90263f7    stp x23, x24, [sp, #32]
  4007ac:   aa0103f7    mov x23, x1
  4007b0:   2a0003f8    mov w24, w0
  4007b4:   d2800700    mov x0, #0x38                   // #56
  4007b8:   f90027fe    str x30, [sp, #72]
  4007bc:   97ffffc9    bl  4006e0 <malloc@plt>
  4007c0:   aa0003f6    mov x22, x0
  4007c4:   d2800700    mov x0, #0x38                   // #56
  4007c8:   f9001276    str x22, [x19, #32]
  4007cc:   97ffffc5    bl  4006e0 <malloc@plt>
  4007d0:   91005001    add x1, x0, #0x14
  4007d4:   b0000004    adrp    x4, 401000 <_IO_stdin_used+0x28>
  4007d8:   91028084    add x4, x4, #0xa0
  4007dc:   b0000003    adrp    x3, 401000 <_IO_stdin_used+0x28>
  4007e0:   91038063    add x3, x3, #0xe0
  # Ptr_Glob->variant.var_1.Int_Comp      = 40; 全局变量，需要str写回。
  4007e4:   52800505    mov w5, #0x28                   // #40
  4007e8:   b9001005    str w5, [x0, #16]
  4007ec:   f9400885    ldr x5, [x4, #16]
  4007f0:   d2c00042    mov x2, #0x200000000            // #8589934592
  4007f4:   a9000816    stp x22, x2, [x0]
  # Arr_2_Glob [8][7] = 10;
  4007f8:   52800142    mov w2, #0xa                    // #10
  4007fc:   f8024005    stur    x5, [x0, #36]
  # strcpy() 全局变量 Ptr_Glob->variant.var_1.Str_Comp
  400800:   a9401c86    ldp x6, x7, [x4]
  400804:   a9001c26    stp x6, x7, [x1]
  400808:   f9400865    ldr x5, [x3, #16]
  40080c:   f9000a60    str x0, [x19, #16]
  400810:   f8417084    ldur    x4, [x4, #23]
  400814:   f8017024    stur    x4, [x1, #23]
  400818:   f9003be5    str x5, [sp, #112]
  
  # strcpy() 局部变量 Str_1_Loc
  40081c:   a9401c66    ldp x6, x7, [x3]
  400820:   a9036bf9    stp x25, x26, [sp, #48]
  400824:   f8417060    ldur    x0, [x3, #23]
  400828:   f90023fb    str x27, [sp, #64]
  40082c:   a9061fe6    stp x6, x7, [sp, #96]
  400830:   f80773e0    stur    x0, [sp, #119]
  400834:   b9065e82    str w2, [x20, #1628]
  
  # if (argc != 2) { fprintf+exit}
  400838:   71000b1f    cmp w24, #0x2
  40083c:   54000120    b.eq    400860 <main+0xd0>  // b.none
  400840:   d0000080    adrp    x0, 412000 <exit@GLIBC_2.17>
  400844:   b0000001    adrp    x1, 401000 <_IO_stdin_used+0x28>
  400848:   f94002e2    ldr x2, [x23]
  40084c:   91030021    add x1, x1, #0xc0
  400850:   f9405800    ldr x0, [x0, #176]
  400854:   97ffffcb    bl  400780 <fprintf@plt>
  400858:   52800505    mov w5, #0x28                   // #40
  40085c:   91005004    add x4, x0, #0x14
  
  # if not
  400860:   b9001005    str w5, [x0, #16]
  400864:   d2c00042    mov x2, #0x200000000            // #8589934592
  400868:   f9400865    ldr x5, [x3, #16]
  40086c:   f8024005    stur    x5, [x0, #36]
  400870:   f9400825    ldr x5, [x1, #16]
  400874:   f903faa0    str x0, [x21, #2032]
  400878:   a9000814    stp x20, x2, [x0]
  40087c:   52800142    mov w2, #0xa                    // #10
  400880:   71000aff    cmp w23, #0x2
  400884:   f90053e5    str x5, [sp, #160]
  400888:   a9402468    ldp x8, x9, [x3]
  40088c:   a9002488    stp x8, x9, [x4]
  400890:   a9401c26    ldp x6, x7, [x1]
  400894:   a9091fe6    stp x6, x7, [sp, #144]
  400898:   f8417063    ldur    x3, [x3, #23]
  40089c:   f8017083    stur    x3, [x4, #23]
  4008a0:   f8417020    ldur    x0, [x1, #23]
  4008a4:   f80a73e0    stur    x0, [sp, #167]
  4008a8:   b9065ec2    str w2, [x22, #1628]
  4008ac:   54000120    b.eq    4008d0 <main+0xd0>  // b.none
  4008b0:   90000100    adrp    x0, 420000 <exit@GLIBC_2.17>
  4008b4:   b0000001    adrp    x1, 401000 <Func_2+0x50>
  4008b8:   f9400262    ldr x2, [x19]
  4008bc:   91064021    add x1, x1, #0x190
  4008c0:   f9405800    ldr x0, [x0, #176]
  4008c4:   97ffffcb    bl  4007f0 <fprintf@plt>
  4008c8:   52800020    mov w0, #0x1                    // #1
  4008cc:   97ffff99    bl  400730 <exit@plt>
  4008d0:   f9400660    ldr x0, [x19, #8]
  4008d4:   d0000103    adrp    x3, 422000 <Arr_2_Glob+0x1f38>
  4008d8:   d2800001    mov x1, #0x0                    // #0 
  ………………
```

### loop主循环代码分析

在`dhry_1.c`中定义了`main`入口函数。会先进行一些初始化动作，真正的循环体为：

```assembly
for (Run_Index = 1; ; ++Run_Index)
{
  Proc_5();
  Proc_4();
    /* Ch_1_Glob == 'A', Ch_2_Glob == 'B', Bool_Glob == true */
  Int_1_Loc = 2;
  Int_2_Loc = 3;
  strcpy (Str_2_Loc, "DHRYSTONE PROGRAM, 2'ND STRING");
  Enum_Loc = Ident_2;
  Bool_Glob = ! Func_2 (Str_1_Loc, Str_2_Loc);
    /* Bool_Glob == 1 */
  while (Int_1_Loc < Int_2_Loc)  /* loop body executed once */
  {
    Int_3_Loc = 5 * Int_1_Loc - Int_2_Loc;
      /* Int_3_Loc == 7 */
    Proc_7 (Int_1_Loc, Int_2_Loc, &Int_3_Loc);
      /* Int_3_Loc == 7 */
    Int_1_Loc += 1;
  } /* while */
    /* Int_1_Loc == 3, Int_2_Loc == 3, Int_3_Loc == 7 */
  Proc_8 (Arr_1_Glob, Arr_2_Glob, Int_1_Loc, Int_3_Loc);
    /* Int_Glob == 5 */
  Proc_1 (Ptr_Glob);
  for (Ch_Index = 'A'; Ch_Index <= Ch_2_Glob; ++Ch_Index)
                           /* loop body executed twice */
  {
    if (Enum_Loc == Func_1 (Ch_Index, 'C'))
        /* then, not executed */
      {
      Proc_6 (Ident_1, &Enum_Loc);
      strcpy (Str_2_Loc, "DHRYSTONE PROGRAM, 3'RD STRING");
      Int_2_Loc = Run_Index;
      Int_Glob = Run_Index;
      }
  }
    /* Int_1_Loc == 3, Int_2_Loc == 3, Int_3_Loc == 7 */
  Int_2_Loc = Int_2_Loc * Int_1_Loc;
  Int_1_Loc = Int_2_Loc / Int_3_Loc;
  Int_2_Loc = 7 * (Int_2_Loc - Int_3_Loc) - Int_1_Loc;
    /* Int_1_Loc == 1, Int_2_Loc == 13, Int_3_Loc == 7 */
  Proc_2 (&Int_1_Loc);
    /* Int_1_Loc == 5 */

} /* loop "for Run_Index" */
```

#### loop_start->strcpy->Func_2

```assembly
400960:   14000004    b   400970 <main+0x170>
# for (Run_Index = 1; ; ++Run_Index)  全局变量，因此需要ldr+add+str操作
400964:   f9400300    ldr x0, [x24]
400968:   91000400    add x0, x0, #0x1
40096c:   f9000300    str x0, [x24]
# Proc_4()： Bool_Loc = Ch_1_Glob == 'A';
400970:   52800824    mov w4, #0x41                   // #65
400974:   39000344    strb    w4, [x26]   # Ch_1_Glob为全局变量，因此要写回内存。
# Proc_4()：	Bool_Glob = Bool_Loc | Bool_Glob;
# Proc_4()： 	Ch_2_Glob = 'B'; (w3)   mov+strb			
400978:   a94617e4    ldp x4, x5, [sp, #96]
40097c:   52800022    mov w2, #0x1                    // #1
400980:   52800843    mov w3, #0x42                   // #66
# main： strcpy (Str_2_Loc, "DHRYSTONE PROGRAM, 2'ND STRING"); part 1
400984:   f90063fc    str x28, [sp, #192]
400988:   9102c3e1    add x1, sp, #0xb0
40098c:   910243e0    add x0, sp, #0x90
400990:   39000263    strb    w3, [x19]
# main： Enum_Loc = Ident_2;  局部enum变量，Ident_2=1
400994:   b9000282    str w2, [x20]
400998:   b9008fe2    str w2, [sp, #140]
# main： strcpy (Str_2_Loc, "DHRYSTONE PROGRAM, 2'ND STRING"); part 2
40099c:   a90b17e4    stp x4, x5, [sp, #176]
4009a0:   f80c73fb    stur    x27, [sp, #199]
# Bool_Glob = ! Func_2 (Str_1_Loc, Str_2_Loc);
4009a4:   94000183    bl  400fb0 <Func_2>
4009a8:   7100001f    cmp w0, #0x0
```

#### Func_2->strcmp

```assembly
# strcmp的glibc源码实现
ENTRY_ALIGN(strcmp, 6)

    DELOUSE (0)
    DELOUSE (1)
    eor tmp1, src1, src2
    mov zeroones, #REP8_01
    tst tmp1, #7
    b.ne    L(misaligned8)     # 不跳转
    ands    tmp1, src1, #7
    b.ne    L(mutual_align)        # 不跳转
    /* NUL detection works on the principle that (X - 1) & (~X) & 0x80
       (=> (X - 1) & ~(X | 0x7f)) is non-zero iff a byte is zero, and
       can be done in parallel across the entire word.  */
L(loop_aligned):
    ldr data1, [src1], #8
    ldr data2, [src2], #8
L(start_realigned):
    sub tmp1, data1, zeroones
    orr tmp2, data1, #REP8_7f
    eor diff, data1, data2  /* Non-zero if differences found.  */
    bic has_nul, tmp1, tmp2 /* Non-zero if NUL terminator.  */
    orr syndrome, diff, has_nul
    cbz syndrome, L(loop_aligned)       # 前2次会跳转，最后一次不跳转。
    /* End of performance-critical section  -- one 64B cache line.  */

L(end):
```



``` asm
# strcmp的指令流trace分析
		# 前面的指令均可以用满8发射。
......
bl 400790<strcmp@plt>
adrp x16, 420000
br x17 (到strcmp的函数起始点) 
   7e680:    eor x7, x0, x1  <strcmp@@GLIBC_2.17>:
   
   		# cycle 1: 8发射用满(88,90,98,0c,a4,ac,b4,98)， mov+tst=1拍， bne+ands=1c, bne+ldr=1c,	 ldr=1c, sub+orr=1c, eor+bic=1c, orr+cbz=1c，跳转回去ldr=1c
   7e684:    mov x10, #0x101010101010101         // #72340172838076673
   7e688:    tst x7, #0x7
   7e68c:    b.ne    7e708 <strcmp@@GLIBC_2.17+0x88>  // b.any
   7e690:    ands    x7, x0, #0x7
   7e694:    b.ne    7e6dc <strcmp@@GLIBC_2.17+0x5c>  // b.any
# loop 1
   7e698:    ldr x2, [x0], #8
   7e69c:    ldr x3, [x1], #8
   7e6a0:    sub x7, x2, x10
   7e6a4:    orr x8, x2, #0x7f7f7f7f7f7f7f7f
   7e6a8:    eor x5, x2, x3
   7e6ac:    bic x4, x7, x8
   7e6b0:    orr x6, x5, x4
   7e6b4:    cbz x6, 7e698 <strcmp@@GLIBC_2.17+0x18> # 跳转
# loop 2
   7e698:    ldr x2, [x0], #8
   
      	# cycle 2：5发射（9c,a4,ac,b4,98)
   7e69c:    ldr x3, [x1], #8
   7e6a0:    sub x7, x2, x10
   7e6a4:    orr x8, x2, #0x7f7f7f7f7f7f7f7f
   7e6a8:    eor x5, x2, x3
   7e6ac:    bic x4, x7, x8
   7e6b0:    orr x6, x5, x4
   7e6b4:    cbz x6, 7e698 <strcmp@@GLIBC_2.17+0x18>
# loop 3
   7e698:    ldr x2, [x0], #8
   		
   		# cycle 15: 2发射（9c,a4)，这里等了很久。
   7e69c:    ldr x3, [x1], #8     # 发生了multihit。地址为0xfb40
   7e6a0:    sub x7, x2, x10
   7e6a4:    orr x8, x2, #0x7f7f7f7f7f7f7f7f
   7e6a8:    eor x5, x2, x3
   		# cycle 16： 1发射
   7e6ac:    bic x4, x7, x8   # 依赖orr和sub的结果
   		# cycle 18： 3发射（b4, b8, c0）
   7e6b0:    orr x6, x5, x4   # 依赖bic和eor的结果
   7e6b4:    cbz x6, 7e698 <strcmp@@GLIBC_2.17+0x18>  # 不跳转 
# 不再loop了，继续执行。
   7e6b8:    rev x6, x6
   7e6bc:    rev x2, x2
   7e6c0:    clz x11, x6
   		# cycle 19: 1发射（c8）
   7e6c4:    rev x3, x3      
   7e6c8:    lsl x2, x2, x11  # 依赖rev结果
   		# cycle 21：2发射（d0，d8）
   7e6cc:    lsl x3, x3, x11   # 依赖rev结果
   7e6d0:    lsr x2, x2, #56   # 依赖lsl结果
   7e6d4:    sub x0, x2, x3, lsr #56
   7e6d8:    ret
		# cycle 23： 1发射
#返回到了用户态。
   400fdc:  mov
   		# cycle 25: 8发射
   400fe0:  cmp
   400fe4:  b.le
   
```



## 指令流分析

### ESL指令流分析

从esl抓的trace流来看，总共209条指令，254条uops（其中stp/ldp会拆uops）， commit耗时 39cycle。

``` c
r:358291     fbid:152960 00004008AC: ldr x0, [x19, #0]	[LDR  D: X0  S: X19  I: 0x0, 0x0] [vlid:191921] 
r:358291     fbid:152960 00004008B0: add x0, x0, #0x1	[ADDI  D: X0  S: X0  I: 0x1]
r:358292     fbid:152960 00004008B4: str x0, [x19, #0]	[STRA  S: X19  I: 0x0] [vsid:150080] 
r:358292     fbid:152960 00004008B4: str x0, [x19, #0]	[STRD  S: X0] [vsid:150080] 
r:358292     fbid:152960 00004008B8: mov w2, #0x1	[MOVI  D: X2  I: 0x1]
r:358292     fbid:152960 00004008BC: mov w3, #0x42	[MOVI  D: X3  I: 0x42]
r:358292     fbid:152961 00004008C0: str x24, [sp, #144]	[STRA  S: SP_EL3  I: 0x90] [vsid:150081] 
r:358292     fbid:152961 00004008C0: str x24, [sp, #144]	[STRD  S: X24] [vsid:150081] 
r:358292     fbid:152961 00004008C4: add x1, sp, #0x80	[ADDI  D: X1  S: SP_EL3  I: 0x80]
r:358292     fbid:152961 00004008C8: add x0, sp, #0x60	[ADDI  D: X0  S: SP_EL3  I: 0x60]
r:358292     fbid:152961 00004008CC: strb x25, [x19, #40]	[STRA  S: X19  I: 0x28] [vsid:150082] 
r:358292     fbid:152961 00004008CC: strb x25, [x19, #40]	[STRD  S: X25] [vsid:150082] 
r:358292     fbid:152961 00004008D0: str w2, [x19, #44]	[STRA  S: X19  I: 0x2c] [vsid:150083] 
r:358292     fbid:152961 00004008D0: str w2, [x19, #44]	[STRD  S: X2] [vsid:150083] 
r:358292     fbid:152961 00004008D4: strb x3, [x19, #48]	[STRA  S: X19  I: 0x30] [vsid:150084] 
r:358292     fbid:152961 00004008D4: strb x3, [x19, #48]	[STRD  S: X3] [vsid:150084] 
r:358292     fbid:152961 00004008D8: str w2, [sp, #92]	[STRA  S: SP_EL3  I: 0x5c] [vsid:150085] 
r:358292     fbid:152961 00004008D8: str w2, [sp, #92]	[STRD  S: X2] [vsid:150085] 
r:358293     fbid:152961 00004008DC: stp x22, x23, [sp, #128]	[STPA  S: SP_EL3  I: 0x80] [vsid:150086] 
r:358293     fbid:152961 00004008DC: stp x22, x23, [sp, #128]	[STPD  S: X22, X23  I: 0x80] [vsid:150086] 
r:358293     fbid:152961 00004008E0: str x21, [sp, #151]	[STRA  S: SP_EL3  I: 0x97] [vsid:150087] 
r:358293     fbid:152961 00004008E0: str x21, [sp, #151]	[STRD  S: X21] [vsid:150087] 
r:358293     fbid:152961 00004008E4: bl 400ed4	[BL  D: X30  I: 0x400ed4] [TAKEN]
r:358293     fbid:152962 0000400ED4: adrp x6, 412000	[ADR  D: X6  I: 0x412000]
r:358293     fbid:152962 0000400ED8: str x30, [sp, #-16]!	[STRA  S: SP_EL3  I: 0xfffffffffffffff0] [vsid:150088] 
r:358293     fbid:152962 0000400ED8: str x30, [sp, #-16]!	[STRD  S: X30  I: 0xfffffffffffffff0] [vsid:150088] 
r:358293     fbid:152962 0000400ED8: str x30, [sp, #-16]!	[ADDI  D: SP_EL3  S: SP_EL3  I: 0xfffffffffffffff0]
r:358293     fbid:152962 0000400EDC: mov w4, #0x0	[MOVI  D: X4  I: 0x0]
r:358293     fbid:152962 0000400EE0: ldrb x2, [x0, #2]	[LDR  D: X2  S: X0  I: 0x2, 0x0] [vlid:191922] 
r:358297     fbid:152962 0000400EE4: ldrb x3, [x1, #3]	[LDR  D: X3  S: X1  I: 0x3, 0x0] [vlid:191923] 
r:358297     fbid:152962 0000400EE8: ldrb x5, [x6, #232]	[LDR  D: X5  S: X6  I: 0xe8, 0x0] [vlid:191924] 
r:358297     fbid:152962 0000400EEC: subs wzr, w2, w3	[ADD  D: SINK, PS_NZCV  S: X2, X3  I: 0x0, 0x4]
r:358298     fbid:152962 0000400EF0: b.eq 400f28	[BCOND  S: PS_NZCV  I: 0x400f28, 0x0]
r:358298     fbid:152962 0000400EF4: cbz w4, 400efc	[CBZ  S: X4  I: 0x400efc] [TAKEN]
r:358298     fbid:152963 0000400EFC: bl 400720	[BL  D: X30  I: 0x400720] [TAKEN]
r:358298     fbid:152964 0000400720: adrp x16, 412000	[ADR  D: X16  I: 0x412000]
r:358298     fbid:152964 0000400724: ldr x17, [x16, #48]	[LDR  D: X17  S: X16  I: 0x30, 0x0] [vlid:191925] 
r:358298     fbid:152964 0000400728: add x16, x16, #0x30	[ADDI  D: X16  S: X16  I: 0x30]
r:358298     fbid:152964 000040072C: br x17	[BR  S: X17] [TAKEN]
r:358298     fbid:152965 7F8824E680: eor x7, x0, x1	[LOP  D: X7  S: X0, X1  I: 0x0, 0x4]
r:358298     fbid:152965 7F8824E684: orr x10, xzr, #0x101010101010101	[LOPI  D: X10  S: ZERO  I: 0x101010101010101]
r:358298     fbid:152965 7F8824E688: ands xzr, x7, #0x7	[LOPI  D: SINK, PS_NZCV  S: X7  I: 0x7]
r:358298     fbid:152965 7F8824E68C: b.ne 7f8824e708	[BCOND  S: PS_NZCV  I: 0x7f8824e708, 0x1]
r:358299     fbid:152965 7F8824E690: ands x7, x0, #0x7	[LOPI  D: X7, PS_NZCV  S: X0  I: 0x7]
r:358299     fbid:152965 7F8824E694: b.ne 7f8824e6dc	[BCOND  S: PS_NZCV  I: 0x7f8824e6dc, 0x1]
r:358299     fbid:152965 7F8824E698: ldr x2, [x0], #8	[LDR  D: X2  S: X0  I: 0x0, 0x0] [vlid:191926] 
r:358299     fbid:152965 7F8824E698: ldr x2, [x0], #8	[ADDI  D: X0  S: X0  I: 0x8]
r:358299     fbid:152965 7F8824E69C: ldr x3, [x1], #8	[LDR  D: X3  S: X1  I: 0x0, 0x0] [vlid:191927] 
r:358299     fbid:152965 7F8824E69C: ldr x3, [x1], #8	[ADDI  D: X1  S: X1  I: 0x8]
r:358299     fbid:152965 7F8824E6A0: sub x7, x2, x10	[ADD  D: X7  S: X2, X10  I: 0x0, 0x4]
r:358299     fbid:152965 7F8824E6A4: orr x8, x2, #0x7f7f7f7f7f7f7f7f	[LOPI  D: X8  S: X2  I: 0x7f7f7f7f7f7f7f7f]
r:358299     fbid:152965 7F8824E6A8: eor x5, x2, x3	[LOP  D: X5  S: X2, X3  I: 0x0, 0x4]
r:358299     fbid:152965 7F8824E6AC: bic x4, x7, x8	[LOP  D: X4  S: X7, X8  I: 0x0, 0x4]
r:358299     fbid:152965 7F8824E6B0: orr x6, x5, x4	[LOP  D: X6  S: X5, X4  I: 0x0, 0x4]
r:358299     fbid:152965 7F8824E6B4: cbz x6, 7f8824e698	[CBZ  S: X6  I: 0x7f8824e698] [TAKEN]
r:358299     fbid:152966 7F8824E698: ldr x2, [x0], #8	[LDR  D: X2  S: X0  I: 0x0, 0x0] [vlid:191928] 
r:358299     fbid:152966 7F8824E698: ldr x2, [x0], #8	[ADDI  D: X0  S: X0  I: 0x8]
r:358299     fbid:152966 7F8824E69C: ldr x3, [x1], #8	[LDR  D: X3  S: X1  I: 0x0, 0x0] [vlid:191929] 
r:358299     fbid:152966 7F8824E69C: ldr x3, [x1], #8	[ADDI  D: X1  S: X1  I: 0x8]
r:358299     fbid:152966 7F8824E6A0: sub x7, x2, x10	[ADD  D: X7  S: X2, X10  I: 0x0, 0x4]
r:358300     fbid:152966 7F8824E6A4: orr x8, x2, #0x7f7f7f7f7f7f7f7f	[LOPI  D: X8  S: X2  I: 0x7f7f7f7f7f7f7f7f]
r:358300     fbid:152966 7F8824E6A8: eor x5, x2, x3	[LOP  D: X5  S: X2, X3  I: 0x0, 0x4]
r:358300     fbid:152966 7F8824E6AC: bic x4, x7, x8	[LOP  D: X4  S: X7, X8  I: 0x0, 0x4]
r:358300     fbid:152966 7F8824E6B0: orr x6, x5, x4	[LOP  D: X6  S: X5, X4  I: 0x0, 0x4]
r:358301     fbid:152966 7F8824E6B4: cbz x6, 7f8824e698	[CBZ  S: X6  I: 0x7f8824e698] [TAKEN]
r:358301     fbid:152967 7F8824E698: ldr x2, [x0], #8	[LDR  D: X2  S: X0  I: 0x0, 0x0] [vlid:191930] 
r:358301     fbid:152967 7F8824E698: ldr x2, [x0], #8	[ADDI  D: X0  S: X0  I: 0x8]
r:358305     fbid:152967 7F8824E69C: ldr x3, [x1], #8	[LDR  D: X3  S: X1  I: 0x0, 0x0] [vlid:191931] 
r:358305     fbid:152967 7F8824E69C: ldr x3, [x1], #8	[ADDI  D: X1  S: X1  I: 0x8]
r:358305     fbid:152967 7F8824E6A0: sub x7, x2, x10	[ADD  D: X7  S: X2, X10  I: 0x0, 0x4]
r:358305     fbid:152967 7F8824E6A4: orr x8, x2, #0x7f7f7f7f7f7f7f7f	[LOPI  D: X8  S: X2  I: 0x7f7f7f7f7f7f7f7f]
r:358305     fbid:152967 7F8824E6A8: eor x5, x2, x3	[LOP  D: X5  S: X2, X3  I: 0x0, 0x4]
r:358306     fbid:152967 7F8824E6AC: bic x4, x7, x8	[LOP  D: X4  S: X7, X8  I: 0x0, 0x4]
r:358306     fbid:152967 7F8824E6B0: orr x6, x5, x4	[LOP  D: X6  S: X5, X4  I: 0x0, 0x4]
r:358308     fbid:152967 7F8824E6B4: cbz x6, 7f8824e698	[CBZ  S: X6  I: 0x7f8824e698]
r:358308     fbid:152967 7F8824E6B8: rev x6, x6	[REV  D: X6  S: X6]
r:358308     fbid:152967 7F8824E6BC: rev x2, x2	[REV  D: X2  S: X2]
r:358308     fbid:152968 7F8824E6C0: clz x11, x6	[CLX  D: X11  S: X6]
r:358308     fbid:152968 7F8824E6C4: rev x3, x3	[REV  D: X3  S: X3]
r:358309     fbid:152968 7F8824E6C8: lsl x2, x2, x11	[LS  D: X2  S: X2, X11]
r:358309     fbid:152968 7F8824E6CC: lsl x3, x3, x11	[LS  D: X3  S: X3, X11]
r:358312     fbid:152968 7F8824E6D0: lsr x2, x2, #56	[BFM  D: X2  S: X2, ZERO  I: 0x1e3f]
r:358312     fbid:152968 7F8824E6D4: sub x0, x2, x3, lsr #56	[ADD  D: X0  S: X2, X3  I: 0x38, 0x1]
r:358312     fbid:152968 7F8824E6D8: ret	[RET  S: X30] [TAKEN]
r:358313     fbid:152969 0000400F00: mov w1, #0x0	[MOVI  D: X1  I: 0x0]
r:358313     fbid:152969 0000400F04: cmp w0, #0x0	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x0]
r:358314     fbid:152969 0000400F08: b.le 400f1c	[BCOND  S: PS_NZCV  I: 0x400f1c, 0xd] [TAKEN]
r:358314     fbid:152970 0000400F1C: mov w0, w1	[MOV  D: X0  S: X1]
r:358314     fbid:152970 0000400F20: ldr x30, [sp], #16	[LDR  D: X30  S: SP_EL3  I: 0x0, 0x0] [vlid:191932] 
r:358314     fbid:152970 0000400F20: ldr x30, [sp], #16	[ADDI  D: SP_EL3  S: SP_EL3  I: 0x10]
r:358314     fbid:152970 0000400F24: ret	[RET  S: X30] [TAKEN]
r:358314     fbid:152971 00004008E8: cmp w0, #0x0	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x0]
r:358314     fbid:152971 00004008EC: mov w4, #0x7	[MOVI  D: X4  I: 0x7]
r:358314     fbid:152971 00004008F0: csinc w3, wzr, wzr, ne	[CSEL  D: X3  S: ZERO, ZERO, PS_NZCV  I: 0x1]
r:358314     fbid:152971 00004008F4: add x2, sp, #0x58	[ADDI  D: X2  S: SP_EL3  I: 0x58]
r:358314     fbid:152971 00004008F8: mov w1, #0x3	[MOVI  D: X1  I: 0x3]
r:358314     fbid:152971 00004008FC: mov w0, #0x2	[MOVI  D: X0  I: 0x2]
r:358314     fbid:152971 0000400900: str w3, [x19, #44]	[STRA  S: X19  I: 0x2c] [vsid:150089] 
r:358314     fbid:152971 0000400900: str w3, [x19, #44]	[STRD  S: X3] [vsid:150089] 
r:358314     fbid:152971 0000400904: str w4, [sp, #88]	[STRA  S: SP_EL3  I: 0x58] [vsid:150090] 
r:358314     fbid:152971 0000400904: str w4, [sp, #88]	[STRD  S: X4] [vsid:150090] 
r:358314     fbid:152971 0000400908: bl 400e44	[BL  D: X30  I: 0x400e44] [TAKEN]
r:358315     fbid:152972 0000400E44: add w0, w0, #0x2	[ADDI  D: X0  S: X0  I: 0x2]
r:358315     fbid:152972 0000400E48: add w0, w0, w1	[ADD  D: X0  S: X0, X1  I: 0x0, 0x4]
r:358315     fbid:152972 0000400E4C: str w0, [x2, #0]	[STRA  S: X2  I: 0x0] [vsid:150091] 
r:358315     fbid:152972 0000400E4C: str w0, [x2, #0]	[STRD  S: X0] [vsid:150091] 
r:358315     fbid:152972 0000400E50: ret	[RET  S: X30] [TAKEN]
r:358315     fbid:152973 000040090C: ldr w3, [sp, #88]	[LDR  D: X3  S: SP_EL3  I: 0x58, 0x0] [vlid:191933] 
r:358315     fbid:152973 0000400910: mov x1, x20	[MOV  D: X1  S: X20]
r:358315     fbid:152973 0000400914: add x0, x19, #0x38	[ADDI  D: X0  S: X19  I: 0x38]
r:358315     fbid:152973 0000400918: mov w2, #0x3	[MOVI  D: X2  I: 0x3]
r:358315     fbid:152973 000040091C: bl 400e54	[BL  D: X30  I: 0x400e54] [TAKEN]
r:358315     fbid:152974 0000400E54: add w5, w2, #0x5	[ADDI  D: X5  S: X2  I: 0x5]
r:358315     fbid:152974 0000400E58: mov w6, #0xc8	[MOVI  D: X6  I: 0xc8]
r:358315     fbid:152974 0000400E5C: sbfiz x2, x2, #2, #32	[BFM  D: X2  S: X2, ZERO  I: 0x1f9f]
r:358315     fbid:152974 0000400E60: sxtw x7, w5	[BFM  D: X7  S: X5, ZERO  I: 0x101f]
r:358315     fbid:152974 0000400E64: add x8, x0, x5, sxtw #2	[ADDE  D: X8  S: X0, X5  I: 0x2, 0x6]
r:358315     fbid:152974 0000400E68: smull x6, w5, w6	[MULL  D: X6  S: X5, X6, ZERO]
r:358316     fbid:152974 0000400E6C: add x4, x6, x2	[ADD  D: X4  S: X6, X2  I: 0x0, 0x4]
r:358316     fbid:152974 0000400E70: str w3, [x0, x7, uxtx/lsl #2]	[STRA_IND  S: X0, X7  I: 0x2] [vsid:150092] 
r:358316     fbid:152974 0000400E70: str w3, [x0, x7, uxtx/lsl #2]	[STRD  S: X3] [vsid:150092] 
r:358316     fbid:152974 0000400E74: add x4, x1, x4	[ADD  D: X4  S: X1, X4  I: 0x0, 0x4]
r:358316     fbid:152974 0000400E78: str w3, [x8, #4]	[STRA  S: X8  I: 0x4] [vsid:150093] 
r:358316     fbid:152974 0000400E78: str w3, [x8, #4]	[STRD  S: X3] [vsid:150093] 
r:358316     fbid:152974 0000400E7C: str w5, [x8, #120]	[STRA  S: X8  I: 0x78] [vsid:150094] 
r:358316     fbid:152974 0000400E7C: str w5, [x8, #120]	[STRD  S: X5] [vsid:150094] 
r:358316     fbid:152975 0000400E80: add x1, x1, x6	[ADD  D: X1  S: X1, X6  I: 0x0, 0x4]
r:358316     fbid:152975 0000400E84: add x1, x1, x2	[ADD  D: X1  S: X1, X2  I: 0x0, 0x4]
r:358316     fbid:152975 0000400E88: adrp x3, 412000	[ADR  D: X3  I: 0x412000]
r:358316     fbid:152975 0000400E8C: ldr w2, [x4, #16]	[LDR  D: X2  S: X4  I: 0x10, 0x0] [vlid:191934] 
r:358316     fbid:152975 0000400E90: mov w6, #0x5	[MOVI  D: X6  I: 0x5]
r:358316     fbid:152975 0000400E94: str w5, [x4, #24]	[STRA  S: X4  I: 0x18] [vsid:150095] 
r:358316     fbid:152975 0000400E94: str w5, [x4, #24]	[STRD  S: X5] [vsid:150095] 
r:358316     fbid:152975 0000400E98: add w2, w2, #0x1	[ADDI  D: X2  S: X2  I: 0x1]
r:358316     fbid:152975 0000400E9C: stp w2, w5, [x4, #16]	[STPA  S: X4  I: 0x10] [vsid:150096] 
r:358316     fbid:152975 0000400E9C: stp w2, w5, [x4, #16]	[STPD  S: X2, X5  I: 0x10] [vsid:150096] 
r:358317     fbid:152975 0000400EA0: ldr w0, [x0, x7, uxtx/lsl #2]	[LDRIND_IMM0  D: X0  S: X0, X7  I: 0x2, 0x0] [vlid:191935] 
r:358317     fbid:152975 0000400EA4: str w6, [x3, #216]	[STRA  S: X3  I: 0xd8] [vsid:150097] 
r:358317     fbid:152975 0000400EA4: str w6, [x3, #216]	[STRD  S: X6] [vsid:150097] 
r:358317     fbid:152975 0000400EA8: str w0, [x1, #4020]	[STRA  S: X1  I: 0xfb4] [vsid:150098] 
r:358317     fbid:152975 0000400EA8: str w0, [x1, #4020]	[STRD  S: X0] [vsid:150098] 
r:358317     fbid:152975 0000400EAC: ret	[RET  S: X30] [TAKEN]
r:358317     fbid:152976 0000400920: ldr x0, [x19, #16]	[LDR  D: X0  S: X19  I: 0x10, 0x0] [vlid:191936] 
r:358317     fbid:152976 0000400924: bl 400c60	[BL  D: X30  I: 0x400c60] [TAKEN]
r:358317     fbid:152977 0000400C60: stp x19, x20, [sp, #-32]!	[STPA  S: SP_EL3  I: 0xffffffffffffffe0] [vsid:150099] 
r:358317     fbid:152977 0000400C60: stp x19, x20, [sp, #-32]!	[STPD  S: X19, X20  I: 0xffffffffffffffe0] [vsid:150099] 
r:358317     fbid:152977 0000400C60: stp x19, x20, [sp, #-32]!	[ADDI  D: SP_EL3  S: SP_EL3  I: 0xffffffffffffffe0]
r:358317     fbid:152977 0000400C64: mov x20, x0	[MOV  D: X20  S: X0]
r:358317     fbid:152977 0000400C68: mov w3, #0x5	[MOVI  D: X3  I: 0x5]
r:358317     fbid:152977 0000400C6C: stp x21, x30, [sp, #16]	[STPA  S: SP_EL3  I: 0x10] [vsid:150100] 
r:358317     fbid:152977 0000400C6C: stp x21, x30, [sp, #16]	[STPD  S: X21, X30  I: 0x10] [vsid:150100] 
r:358317     fbid:152977 0000400C70: adrp x21, 412000	[ADR  D: X21  I: 0x412000]
r:358317     fbid:152977 0000400C74: add x21, x21, #0xc0	[ADDI  D: X21  S: X21  I: 0xc0]
r:358318     fbid:152977 0000400C78: ldr x19, [x20, #0]	[LDR  D: X19  S: X20  I: 0x0, 0x0] [vlid:191937] 
r:358318     fbid:152977 0000400C7C: mov w0, #0xa	[MOVI  D: X0  I: 0xa]
r:358318     fbid:152977 0000400C80: ldr x2, [x21, #16]	[LDR  D: X2  S: X21  I: 0x10, 0x0] [vlid:191938] 
r:358318     fbid:152977 0000400C84: ldr w1, [x21, #24]	[LDR  D: X1  S: X21  I: 0x18, 0x0] [vlid:191939] 
r:358318     fbid:152977 0000400C88: ldp x4, x5, [x2, #0]	[LDP  D: X4, X5  S: X2  I: 0x0, 0x0] [vlid:191940] 
r:358318     fbid:152977 0000400C8C: stp x4, x5, [x19, #0]	[STPA  S: X19  I: 0x0] [vsid:150101] 
r:358318     fbid:152977 0000400C8C: stp x4, x5, [x19, #0]	[STPD  S: X4, X5  I: 0x0] [vsid:150101] 
r:358318     fbid:152977 0000400C90: ldp x4, x5, [x2, #16]	[LDP  D: X4, X5  S: X2  I: 0x10, 0x0] [vlid:191941] 
r:358318     fbid:152977 0000400C94: stp x4, x5, [x19, #16]	[STPA  S: X19  I: 0x10] [vsid:150102] 
r:358318     fbid:152977 0000400C94: stp x4, x5, [x19, #16]	[STPD  S: X4, X5  I: 0x10] [vsid:150102] 
r:358318     fbid:152977 0000400C98: ldp x4, x5, [x2, #32]	[LDP  D: X4, X5  S: X2  I: 0x20, 0x0] [vlid:191942] 
r:358319     fbid:152977 0000400C9C: stp x4, x5, [x19, #32]	[STPA  S: X19  I: 0x20] [vsid:150103] 
r:358319     fbid:152977 0000400C9C: stp x4, x5, [x19, #32]	[STPD  S: X4, X5  I: 0x20] [vsid:150103] 
r:358319     fbid:152978 0000400CA0: ldr x4, [x2, #48]	[LDR  D: X4  S: X2  I: 0x30, 0x0] [vlid:191943] 
r:358319     fbid:152978 0000400CA4: str x4, [x19, #48]	[STRA  S: X19  I: 0x30] [vsid:150104] 
r:358319     fbid:152978 0000400CA4: str x4, [x19, #48]	[STRD  S: X4] [vsid:150104] 
r:358319     fbid:152978 0000400CA8: str w3, [x20, #16]	[STRA  S: X20  I: 0x10] [vsid:150105] 
r:358319     fbid:152978 0000400CA8: str w3, [x20, #16]	[STRD  S: X3] [vsid:150105] 
r:358319     fbid:152978 0000400CAC: ldr x4, [x20, #0]	[LDR  D: X4  S: X20  I: 0x0, 0x0] [vlid:191944] 
r:358319     fbid:152978 0000400CB0: str x4, [x19, #0]	[STRA  S: X19  I: 0x0] [vsid:150106] 
r:358319     fbid:152978 0000400CB0: str x4, [x19, #0]	[STRD  S: X4] [vsid:150106] 
r:358319     fbid:152978 0000400CB4: str w3, [x19, #16]	[STRA  S: X19  I: 0x10] [vsid:150107] 
r:358319     fbid:152978 0000400CB4: str w3, [x19, #16]	[STRD  S: X3] [vsid:150107] 
r:358319     fbid:152978 0000400CB8: ldr x2, [x2, #0]	[LDR  D: X2  S: X2  I: 0x0, 0x0] [vlid:191945] 
r:358320     fbid:152978 0000400CBC: str x2, [x19, #0]	[STRA  S: X19  I: 0x0] [vsid:150108] 
r:358320     fbid:152978 0000400CBC: str x2, [x19, #0]	[STRD  S: X2] [vsid:150108] 
r:358320     fbid:152978 0000400CC0: ldr x2, [x21, #16]	[LDR  D: X2  S: X21  I: 0x10, 0x0] [vlid:191946] 
r:358320     fbid:152978 0000400CC4: add x2, x2, #0x10	[ADDI  D: X2  S: X2  I: 0x10]
r:358320     fbid:152978 0000400CC8: bl 400e44	[BL  D: X30  I: 0x400e44] [TAKEN]
r:358320     fbid:152979 0000400E44: add w0, w0, #0x2	[ADDI  D: X0  S: X0  I: 0x2]
r:358320     fbid:152979 0000400E48: add w0, w0, w1	[ADD  D: X0  S: X0, X1  I: 0x0, 0x4]
r:358321     fbid:152979 0000400E4C: str w0, [x2, #0]	[STRA  S: X2  I: 0x0] [vsid:150109] 
r:358321     fbid:152979 0000400E4C: str w0, [x2, #0]	[STRD  S: X0] [vsid:150109] 
r:358321     fbid:152979 0000400E50: ret	[RET  S: X30] [TAKEN]
r:358323     fbid:152980 0000400CCC: ldr w0, [x19, #8]	[LDR  D: X0  S: X19  I: 0x8, 0x0] [vlid:191947] 
r:358323     fbid:152980 0000400CD0: cbz w0, 400d04	[CBZ  S: X0  I: 0x400d04] [TAKEN]
r:358323     fbid:152981 0000400D04: ldr w0, [x20, #12]	[LDR  D: X0  S: X20  I: 0xc, 0x0] [vlid:191948] 
r:358323     fbid:152981 0000400D08: mov w1, #0x6	[MOVI  D: X1  I: 0x6]
r:358323     fbid:152981 0000400D0C: str w1, [x19, #16]	[STRA  S: X19  I: 0x10] [vsid:150110] 
r:358323     fbid:152981 0000400D0C: str w1, [x19, #16]	[STRD  S: X1] [vsid:150110] 
r:358323     fbid:152981 0000400D10: add x1, x19, #0xc	[ADDI  D: X1  S: X19  I: 0xc]
r:358323     fbid:152981 0000400D14: bl 400df0	[BL  D: X30  I: 0x400df0] [TAKEN]
r:358323     fbid:152982 0000400DF0: cmp w0, #0x2	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x2]
r:358323     fbid:152982 0000400DF4: b.eq 400e38	[BCOND  S: PS_NZCV  I: 0x400e38, 0x0] [TAKEN]
r:358323     fbid:152983 0000400E38: mov w0, #0x1	[MOVI  D: X0  I: 0x1]
r:358323     fbid:152983 0000400E3C: str w0, [x1, #0]	[STRA  S: X1  I: 0x0] [vsid:150111] 
r:358323     fbid:152983 0000400E3C: str w0, [x1, #0]	[STRD  S: X0] [vsid:150111] 
r:358323     fbid:152983 0000400E40: ret	[RET  S: X30] [TAKEN]
r:358324     fbid:152984 0000400D18: ldr x3, [x21, #16]	[LDR  D: X3  S: X21  I: 0x10, 0x0] [vlid:191949] 
r:358324     fbid:152984 0000400D1C: mov x2, x19	[MOV  D: X2  S: X19]
r:358325     fbid:152984 0000400D20: ldr w0, [x19, #16]	[LDR  D: X0  S: X19  I: 0x10, 0x0] [vlid:191950] 
r:358325     fbid:152984 0000400D24: mov w1, #0xa	[MOVI  D: X1  I: 0xa]
r:358325     fbid:152984 0000400D28: ldp x21, x30, [sp, #16]	[LDP  D: X21, X30  S: SP_EL3  I: 0x10, 0x0] [vlid:191951] 
r:358325     fbid:152984 0000400D2C: ldr x3, [x3, #0]	[LDR  D: X3  S: X3  I: 0x0, 0x0] [vlid:191952] 
r:358325     fbid:152984 0000400D30: str x3, [x2], #16	[STRA  S: X2  I: 0x0] [vsid:150112] 
r:358325     fbid:152984 0000400D30: str x3, [x2], #16	[STRD  S: X3  I: 0x10] [vsid:150112] 
r:358325     fbid:152984 0000400D30: str x3, [x2], #16	[ADDI  D: X2  S: X2  I: 0x10]
r:358325     fbid:152984 0000400D34: ldp x19, x20, [sp], #32	[LDP  D: X19, X20  S: SP_EL3  I: 0x0, 0x0] [vlid:191953] 
r:358325     fbid:152984 0000400D34: ldp x19, x20, [sp], #32	[ADDI  D: SP_EL3  S: SP_EL3  I: 0x20]
r:358325     fbid:152984 0000400D38: b 400e44	[B  I: 0x400e44] [TAKEN]
r:358326     fbid:152985 0000400E44: add w0, w0, #0x2	[ADDI  D: X0  S: X0  I: 0x2]
r:358326     fbid:152985 0000400E48: add w0, w0, w1	[ADD  D: X0  S: X0, X1  I: 0x0, 0x4]
r:358328     fbid:152985 0000400E4C: str w0, [x2, #0]	[STRA  S: X2  I: 0x0] [vsid:150113] 
r:358328     fbid:152985 0000400E4C: str w0, [x2, #0]	[STRD  S: X0] [vsid:150113] 
r:358328     fbid:152985 0000400E50: ret	[RET  S: X30] [TAKEN]
r:358328     fbid:152986 0000400928: ldrb x0, [x19, #48]	[LDR  D: X0  S: X19  I: 0x30, 0x0] [vlid:191954] 
r:358328     fbid:152986 000040092C: cmp w0, #0x40	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x40]
r:358328     fbid:152986 0000400930: b.ls 4008ac	[BCOND  S: PS_NZCV  I: 0x4008ac, 0x9]
r:358328     fbid:152986 0000400934: mov w1, #0x43	[MOVI  D: X1  I: 0x43]
r:358328     fbid:152986 0000400938: mov w27, #0x41	[MOVI  D: X27  I: 0x41]
r:358328     fbid:152986 000040093C: mov w0, w27	[MOV  D: X0  S: X27]
r:358328     fbid:152986 0000400940: bl 400eb0	[BL  D: X30  I: 0x400eb0] [TAKEN]
r:358328     fbid:152987 0000400EB0: and w2, w0, #0xff	[LOPI  D: X2  S: X0  I: 0xff]
r:358328     fbid:152987 0000400EB4: mov w0, #0x0	[MOVI  D: X0  I: 0x0]
r:358328     fbid:152987 0000400EB8: subs wzr, w2, w1, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X2, X1  I: 0x0, 0x0]
r:358328     fbid:152987 0000400EBC: b.eq 400ec4	[BCOND  S: PS_NZCV  I: 0x400ec4, 0x0]
r:358328     fbid:152987 0000400EC0: ret	[RET  S: X30] [TAKEN]
r:358329     fbid:152988 0000400944: ldr w1, [sp, #92]	[LDR  D: X1  S: SP_EL3  I: 0x5c, 0x0] [vlid:191955] 
r:358329     fbid:152988 0000400948: subs wzr, w0, w1	[ADD  D: SINK, PS_NZCV  S: X0, X1  I: 0x0, 0x4]
r:358329     fbid:152988 000040094C: b.eq 40097c	[BCOND  S: PS_NZCV  I: 0x40097c, 0x0]
r:358329     fbid:152988 0000400950: ldrb x1, [x19, #48]	[LDR  D: X1  S: X19  I: 0x30, 0x0] [vlid:191956] 
r:358329     fbid:152988 0000400954: add w0, w27, #0x1	[ADDI  D: X0  S: X27  I: 0x1]
r:358329     fbid:152988 0000400958: and w27, w0, #0xff	[LOPI  D: X27  S: X0  I: 0xff]
r:358329     fbid:152988 000040095C: subs wzr, w1, w0, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X1, X0  I: 0x0, 0x0]
r:358329     fbid:152988 0000400960: b.cc 4008ac	[BCOND  S: PS_NZCV  I: 0x4008ac, 0x3]
r:358329     fbid:152988 0000400964: mov w1, #0x43	[MOVI  D: X1  I: 0x43]
r:358329     fbid:152988 0000400968: mov w0, w27	[MOV  D: X0  S: X27]
r:358329     fbid:152988 000040096C: bl 400eb0	[BL  D: X30  I: 0x400eb0] [TAKEN]
r:358329     fbid:152989 0000400EB0: and w2, w0, #0xff	[LOPI  D: X2  S: X0  I: 0xff]
r:358329     fbid:152989 0000400EB4: mov w0, #0x0	[MOVI  D: X0  I: 0x0]
r:358330     fbid:152989 0000400EB8: subs wzr, w2, w1, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X2, X1  I: 0x0, 0x0]
r:358330     fbid:152989 0000400EBC: b.eq 400ec4	[BCOND  S: PS_NZCV  I: 0x400ec4, 0x0]
r:358330     fbid:152989 0000400EC0: ret	[RET  S: X30] [TAKEN]
r:358330     fbid:152990 0000400970: ldr w1, [sp, #92]	[LDR  D: X1  S: SP_EL3  I: 0x5c, 0x0] [vlid:191957] 
r:358330     fbid:152990 0000400974: subs wzr, w0, w1	[ADD  D: SINK, PS_NZCV  S: X0, X1  I: 0x0, 0x4]
r:358330     fbid:152990 0000400978: b.ne 400950	[BCOND  S: PS_NZCV  I: 0x400950, 0x1] [TAKEN]
r:358330     fbid:152991 0000400950: ldrb x1, [x19, #48]	[LDR  D: X1  S: X19  I: 0x30, 0x0] [vlid:191958] 
r:358330     fbid:152991 0000400954: add w0, w27, #0x1	[ADDI  D: X0  S: X27  I: 0x1]
r:358330     fbid:152991 0000400958: and w27, w0, #0xff	[LOPI  D: X27  S: X0  I: 0xff]
r:358330     fbid:152991 000040095C: subs wzr, w1, w0, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X1, X0  I: 0x0, 0x0]
r:358330     fbid:152991 0000400960: b.cc 4008ac	[BCOND  S: PS_NZCV  I: 0x4008ac, 0x3] [TAKEN]
```

### EMU指令流分析

#### 波形：1630v200

![image-20221207170314117](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221207170314117.png)

####指令流： 

​       共38cycle/loop， 15.2ns，折合分数为5637分。与实测基本匹配。

![image-20221208154133098](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221208154133098.png)

![image-20221208154159405](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221208154159405.png)

![image-20221208154228429](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221208154228429.png)



#### 长时延load产生的原因

波形中，有一个长时延load操作产生的原因：

- ldr之前有同地址的str，因此ldr变慢了。
- ldr正常来说是可以passthrough去STQ中取，目前onepro相比910来说有修改，每条ldr可以根据data的位宽拆成ldr0和ldr1（比如8B的ldr，可以拆成4B+4B的两个ldrx，这里与uops的概念不同，是passthrough取stq时会做的事情），能各自对应一条STQ的操作，也就是最多可以对应到两条STQ的操作（比如两个stq操作的地址都与ldr操作有重叠的），但是若一条ldr0对应到了两条stp的操作（即当前的ldr操作地址，在前面有2条涵盖该地址的str都在stq中未allocate到scb）时，就会发生multihit，会引发fail，如图所示。当fail时，就无法直接从stq返回数据了，需要等data allocate到scb后才能取到。但是，这里有两个因素制约：

  - B500版本的SCB分了奇偶两组，奇数组负责cacheline的前32B的merge，偶数组负责后32B；在波形中看到，奇数组的8个scb都满了，而偶数组仍有空缺。在B301不分奇偶时可以从stq到scb，但是现在就不行了，需要等待空闲scb。
  -  出于功耗考虑，scb较空时，去到DC的速度比之前慢，merge window时间会长一些，从而导致奇数scb在后续更容易出现满的场景。
- 长时延的load指令（0x40)命中了multihit的store指令(st1 0x40 & st2 0x47)，因此不能直接passthrough从STP取值，必须等到deallocate到SCB后才能取，（onepro可以passthrough两条，而910只能passthrough一条）。而st2(rid 94)前面有一堆store指令在等着retire（rid38->94)，用了一段时间才轮到了st2，而最前面的rid38的str指令retire慢的原因，是前面有一堆指令在等待commit，no_flush_ptr一直没有走到rid38。

![image-20221208161516710](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20221208161516710.png)

- 对应的软件实现逻辑是：

  ``` c
  for (Run_Index = 1; ; ++Run_Index)
  {
  
    ......
    strcpy (Str_2_Loc, "DHRYSTONE PROGRAM, 2'ND STRING");
    Enum_Loc = Ident_2;
    Bool_Glob = ! Func_2 (Str_1_Loc, Str_2_Loc); // Func_2 -> 调用strcmp
      /* Bool_Glob == 1 */
    ......
    for (Ch_Index = 'A'; Ch_Index <= Ch_2_Glob; ++Ch_Index)
                             /* loop body executed twice */
    {
      if (Enum_Loc == Func_1 (Ch_Index, 'C'))
          /* then, not executed */
        {
        Proc_6 (Ident_1, &Enum_Loc);
        strcpy (Str_2_Loc, "DHRYSTONE PROGRAM, 3'RD STRING");
        Int_2_Loc = Run_Index;
        Int_Glob = Run_Index;
        }
    }
    .....
  
  } /* loop "for Run_Index" */
  ```



#### 无法满IPC运行的原因 

​	主要是strcmp函数中存在指令依赖导致的。 `strcmp`的两个字符串首地址给的是`0xfb10`和`0xfb30`，因此是对齐的，两个b.ne都不跳转。 取指的时候，两个参数对应的是strcmp(a,b)的两个字符串地址，字符串为`31Byte`长度的char型字符。a的首地址为0xfb10，b的首地址为0xfb30。

​	每次对比8Byte，若相同则继续对比，这里会对比3次，对应的地址分别为：

- src1=0xfb10, src2=0xfb30; （同64B）
- src1=0xfb18, src2=0xfb38; （同64B）
- src1=0xfb20, src2=0xfb40; （同128Byte的不同64B）



在`strcmp`之前，有多次全局变量的写操作（mov+str)， 以及1次同地址的`builtin_strcpy`调用（stp+str+stur指令）。`strcpy`写的就是src2地址，即(stp 0xfb30, str 0xfb40, stur 0xfb47)。



- 首先，strcmp中指令依赖导致低IPC运行，以及noflush指针走的比取指慢很多。

- 其次，大量的store指令们，在retire到SCB时会造成排队，同时noflush更新慢也会让retire动作延后进行。
- 当遇到multihit场景时，就会有叠加影响，从而出现大时延。



## 1650优化

ESL tag 0728总体性能： 30cycle/loop（折合分数为7141分）， 2.5GHz，静态编译，共246uops。

相比1630v200

### topdown

​	![image-20230816151642070](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230816151642070.png)

![image-20230812102646509](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230812102646509.png)

​		`exe ports util`对应的是`iex/iex_pipe.cpp`中的`issued_cnt_per_cyc`统计，memdata对应的是load/strore相关指令的data的uop，is_store应该是STA？应该是说，如果不是STD/STA/DATA过程，其他的指令都算在这里。

​		<img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230812105205413.png" alt="image-20230812105205413"  />



### 1650 EMU数据

B503版本上，开SMT性能为6385分，对应topdown中retiring = 0.76，对应IPC=8*0.76=6，比esl要低。

topdown对比：

<img src="D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20240330163409546.png" alt="image-20240330163409546" style="zoom:200%;" />

### 重点函数优化strcmp

esl trace上看（静态编译下），strcmp()的commit（从pc 400D1C->400D20,  52672->52685）共13cycle；

1630v200 EMU（动态编译下），strcmp()耗时为9.3ns = 23cycle。



# 编译选项

``` c
// 首先编译dhry2
gcc -c -DTIME -Wall -pedantic -ansi -static -DHZ= -O2 -mtune=tsv110 -fomit-frame-pointer -fforce-addr -ffast-math -Wall dhry_1.c
gcc -c -DTIME -Wall -pedantic -ansi -static -DHZ= -O2 -mtune=tsv110 -fomit-frame-pointer -fforce-addr -ffast-math -Wall dhry_2.c
gcc -o ./pgms/dhry2 -DTIME -Wall -pedantic -ansi -static -O2 -mtune=tsv110 -fomit-frame-pointer -fforce-addr -ffast-math -Wall ./src/dhry_1.o ./src/dhry_2.o
// 然后编译dhry2reg，相比上面只增加了-DREG=register
gcc -c -DTIME -Wall -pedantic -ansi -static -DREG=register -DHZ= -O2 -mtune=tsv110 -fomit-frame-pointer -fforce-addr -ffast-math -Wall dhry_1.c -o dhry_1_reg.o
cd ./src; gcc -c -DTIME -Wall -pedantic -ansi -static -DREG=register -DHZ= -O2 -mtune=tsv110 -fomit-frame-pointer -fforce-addr -ffast-math -Wall dhry_2.c -o dhry_2_reg.o
gcc -o ./pgms/dhry2reg -DTIME -Wall -pedantic -ansi -static -O2 -mtune=tsv110 -fomit-frame-pointer -fforce-addr -ffast-math -Wall ./src/dhry_1_reg.o ./src/dhry_2_reg.o
```

## 关于tsv110

gcc中，如果指定-mtune=tsv100，则可以针对性的进行编译优化，原理是在gcc编译时对于硬件特性进行了针对性的配置。

``` c
 static const struct tune_params tsv110_tunings =
 {
   &tsv110_extra_costs,
   &tsv110_addrcost_table,
   &tsv110_regmove_cost,
   &tsv110_vector_cost,
   &generic_branch_cost,
   &generic_approx_modes,
   SVE_NOT_IMPLEMENTED, /* sve_width  */
   4,    /* memmov_cost  */
   4,    /* issue_rate  */
   (AARCH64_FUSE_AES_AESMC | AARCH64_FUSE_ALU_BRANCH
    | AARCH64_FUSE_ALU_CBZ), /* fusible_ops  */
   "16", /* function_align.  */
   "4",  /* jump_align.  */
   "8",  /* loop_align.  */
   2,    /* int_reassoc_width.  */
   4,    /* fp_reassoc_width.  */
   1,    /* vec_reassoc_width.  */
   2,    /* min_div_recip_mul_sf.  */
   2,    /* min_div_recip_mul_df.  */
   0,    /* max_case_values.  */
   tune_params::AUTOPREFETCHER_WEAK,     /* autoprefetcher_model.  */
   (AARCH64_EXTRA_TUNE_NONE),     /* tune_flags.  */
   &tsv110_prefetch_tune
 };
```





# 附录

## 1. esl 完整trace流。（1loop）

```c
c0 currEL:0 2019410  sq:1471981    f:358269     b:358258     d1:358271     d2:358272     a:358273     q:358274     e:358278     ed:358281     w:358282     ep:7          rid:160        r:358291     fbid:152960 T:0 00000000004008AC: f9400260    ldr x0, [x19, #0]	[LDR  D: X0  S: X19  I: 0x0, 0x0] [vlid:191921] 
c0 currEL:0 2019412  sq:1471982    f:358269     b:358258     d1:358271     d2:358272     a:358273     q:358274     e:358282     ed:0          w:358283     ep:22         rid:160        r:358291     fbid:152960 T:0 00000000004008B0: 91000400    add x0, x0, #0x1	[ADDI  D: X0  S: X0  I: 0x1]
c0 currEL:0 2019414  sq:1471983    f:358269     b:358258     d1:358271     d2:358272     a:358273     q:358274     e:358278     ed:0          w:358282     ep:4          rid:161        r:358292     fbid:152960 T:0 00000000004008B4: f9000260    str x0, [x19, #0]	[STRA  S: X19  I: 0x0] [vsid:150080] 
c0 currEL:0 2019414  sq:1471984    f:358269     b:358258     d1:358271     d2:358272     a:358273     q:358274     e:358283     ed:0          w:358285     ep:15         rid:161        r:358292     fbid:152960 T:0 00000000004008B4: f9000260    str x0, [x19, #0]	[STRD  S: X0] [vsid:150080] 
c0 currEL:0 2019416  sq:1471985    f:358269     b:358258     d1:358271     d2:358272     a:358273     q:0          e:0          ed:0          w:0          ep:4294967295 rid:161        r:358292     fbid:152960 T:0 00000000004008B8: 52800022    mov w2, #0x1	[MOVI  D: X2  I: 0x1]
c0 currEL:0 2019418  sq:1471986    f:358269     b:358258     d1:358272     d2:358273     a:358274     q:0          e:0          ed:0          w:0          ep:4294967295 rid:162        r:358292     fbid:152960 T:0 00000000004008BC: 52800843    mov w3, #0x42	[MOVI  D: X3  I: 0x42]
c0 currEL:0 2019420  sq:1471987    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358283     ed:0          w:358287     ep:0          rid:163        r:358292     fbid:152961 T:0 00000000004008C0: f9004bf8    str x24, [sp, #144]	[STRA  S: SP_EL3  I: 0x90] [vsid:150081] 
c0 currEL:0 2019420  sq:1471988    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358283     ed:0          w:358285     ep:17         rid:163        r:358292     fbid:152961 T:0 00000000004008C0: f9004bf8    str x24, [sp, #144]	[STRD  S: X24] [vsid:150081] 
c0 currEL:0 2019422  sq:1471989    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358283     ed:0          w:358284     ep:20         rid:163        r:358292     fbid:152961 T:0 00000000004008C4: 910203e1    add x1, sp, #0x80	[ADDI  D: X1  S: SP_EL3  I: 0x80]
c0 currEL:0 2019424  sq:1471990    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358283     ed:0          w:358284     ep:22         rid:164        r:358292     fbid:152961 T:0 00000000004008C8: 910183e0    add x0, sp, #0x60	[ADDI  D: X0  S: SP_EL3  I: 0x60]
c0 currEL:0 2019426  sq:1471991    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358283     ed:0          w:358287     ep:4          rid:165        r:358292     fbid:152961 T:0 00000000004008CC: 3900a279    strb x25, [x19, #40]	[STRA  S: X19  I: 0x28] [vsid:150082] 
c0 currEL:0 2019426  sq:1471992    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358284     ed:0          w:358286     ep:15         rid:165        r:358292     fbid:152961 T:0 00000000004008CC: 3900a279    strb x25, [x19, #40]	[STRD  S: X25] [vsid:150082] 
c0 currEL:0 2019428  sq:1471993    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358284     ed:0          w:358288     ep:0          rid:166        r:358292     fbid:152961 T:0 00000000004008D0: b9002e62    str w2, [x19, #44]	[STRA  S: X19  I: 0x2c] [vsid:150083] 
c0 currEL:0 2019428  sq:1471994    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358284     ed:0          w:358286     ep:17         rid:166        r:358292     fbid:152961 T:0 00000000004008D0: b9002e62    str w2, [x19, #44]	[STRD  S: X2] [vsid:150083] 
c0 currEL:0 2019430  sq:1471995    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358284     ed:0          w:358288     ep:4          rid:167        r:358292     fbid:152961 T:0 00000000004008D4: 3900c263    strb x3, [x19, #48]	[STRA  S: X19  I: 0x30] [vsid:150084] 
c0 currEL:0 2019430  sq:1471996    f:358275     b:358274     d1:358276     d2:358277     a:358278     q:358279     e:358285     ed:0          w:358287     ep:15         rid:167        r:358292     fbid:152961 T:0 00000000004008D4: 3900c263    strb x3, [x19, #48]	[STRD  S: X3] [vsid:150084] 
c0 currEL:0 2019432  sq:1471997    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358285     ed:0          w:358289     ep:0          rid:168        r:358292     fbid:152961 T:0 00000000004008D8: b9005fe2    str w2, [sp, #92]	[STRA  S: SP_EL3  I: 0x5c] [vsid:150085] 
c0 currEL:0 2019432  sq:1471998    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358286     ed:0          w:358288     ep:17         rid:168        r:358292     fbid:152961 T:0 00000000004008D8: b9005fe2    str w2, [sp, #92]	[STRD  S: X2] [vsid:150085] 
c0 currEL:0 2019434  sq:1471999    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358285     ed:0          w:358289     ep:4          rid:169        r:358293     fbid:152961 T:0 00000000004008DC: a9085ff6    stp x22, x23, [sp, #128]	[STPA  S: SP_EL3  I: 0x80] [vsid:150086] 
c0 currEL:0 2019434  sq:1472000    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358286     ed:0          w:358288     ep:15         rid:169        r:358293     fbid:152961 T:0 00000000004008DC: a9085ff6    stp x22, x23, [sp, #128]	[STPD  S: X22, X23  I: 0x80] [vsid:150086] 
c0 currEL:0 2019436  sq:1472001    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358286     ed:0          w:358290     ep:0          rid:170        r:358293     fbid:152961 T:0 00000000004008E0: f80973f5    str x21, [sp, #151]	[STRA  S: SP_EL3  I: 0x97] [vsid:150087] 
c0 currEL:0 2019436  sq:1472002    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358287     ed:0          w:358289     ep:17         rid:170        r:358293     fbid:152961 T:0 00000000004008E0: f80973f5    str x21, [sp, #151]	[STRD  S: X21] [vsid:150087] 
c0 currEL:0 2019438  sq:1472003    f:358275     b:358274     d1:358277     d2:358278     a:358279     q:358280     e:358284     ed:0          w:358285     ep:19         rid:170        r:358293     fbid:152961 T:0 00000000004008E4: 9400017c    bl 400ed4	[BL  D: X30  I: 0x400ed4] [TAKEN]
c0 currEL:0 2019440  sq:1472004    f:358276     b:358275     d1:358277     d2:358278     a:358279     q:0          e:0          ed:0          w:0          ep:4294967295 rid:171        r:358293     fbid:152962 T:0 0000000000400ED4: d0000086    adrp x6, 412000	[ADR  D: X6  I: 0x412000]
c0 currEL:0 2019442  sq:1472005    f:358276     b:358275     d1:358277     d2:358278     a:358279     q:358280     e:358286     ed:0          w:358290     ep:4          rid:172        r:358293     fbid:152962 T:0 0000000000400ED8: f81f0ffe    str x30, [sp, #-16]!	[STRA  S: SP_EL3  I: 0xfffffffffffffff0] [vsid:150088] 
c0 currEL:0 2019442  sq:1472006    f:358276     b:358275     d1:358277     d2:358278     a:358279     q:358280     e:358287     ed:0          w:358289     ep:15         rid:172        r:358293     fbid:152962 T:0 0000000000400ED8: f81f0ffe    str x30, [sp, #-16]!	[STRD  S: X30  I: 0xfffffffffffffff0] [vsid:150088] 
c0 currEL:0 2019442  sq:1472007    f:358276     b:358275     d1:358277     d2:358278     a:358279     q:358280     e:358284     ed:0          w:358285     ep:22         rid:172        r:358293     fbid:152962 T:0 0000000000400ED8: f81f0ffe    str x30, [sp, #-16]!	[ADDI  D: SP_EL3  S: SP_EL3  I: 0xfffffffffffffff0]
c0 currEL:0 2019444  sq:1472008    f:358276     b:358275     d1:358277     d2:358278     a:358279     q:0          e:0          ed:0          w:0          ep:4294967295 rid:172        r:358293     fbid:152962 T:0 0000000000400EDC: 52800004    mov w4, #0x0	[MOVI  D: X4  I: 0x0]
c0 currEL:0 2019446  sq:1472009    f:358276     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358285     ed:358288     w:358289     ep:5          rid:173        r:358293     fbid:152962 T:0 0000000000400EE0: 39400802    ldrb x2, [x0, #2]	[LDR  D: X2  S: X0  I: 0x2, 0x0] [vlid:191922] 
c0 currEL:0 2019448  sq:1472010    f:358276     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358285     ed:358293     w:358294     ep:6          rid:174        r:358297     fbid:152962 T:0 0000000000400EE4: 39400c23    ldrb x3, [x1, #3]	[LDR  D: X3  S: X1  I: 0x3, 0x0] [vlid:191923] 
c0 currEL:0 2019450  sq:1472011    f:358276     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358285     ed:358288     w:358289     ep:7          rid:175        r:358297     fbid:152962 T:0 0000000000400EE8: 3943a0c5    ldrb x5, [x6, #232]	[LDR  D: X5  S: X6  I: 0xe8, 0x0] [vlid:191924] 
c0 currEL:0 2019452  sq:1472012    f:358276     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358294     ed:0          w:358295     ep:21         rid:175        r:358297     fbid:152962 T:0 0000000000400EEC: 6b03005f    subs wzr, w2, w3	[ADD  D: SINK, PS_NZCV  S: X2, X3  I: 0x0, 0x4]
c0 currEL:0 2019454  sq:1472013    f:358276     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358294     ed:0          w:358295     ep:21         rid:176        r:358298     fbid:152962 T:0 0000000000400EF0: 540001c0    b.eq 400f28	[BCOND  S: PS_NZCV  I: 0x400f28, 0x0]
c0 currEL:0 2019456  sq:1472014    f:358276     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358285     ed:0          w:358286     ep:19         rid:177        r:358298     fbid:152962 T:0 0000000000400EF4: 34000044    cbz w4, 400efc	[CBZ  S: X4  I: 0x400efc] [TAKEN]
c0 currEL:0 2019458  sq:1472015    f:358277     b:358275     d1:358278     d2:358279     a:358280     q:358281     e:358285     ed:0          w:358286     ep:21         rid:178        r:358298     fbid:152963 T:0 0000000000400EFC: 97fffe09    bl 400720	[BL  D: X30  I: 0x400720] [TAKEN]
c0 currEL:0 2019460  sq:1472016    f:358278     b:358276     d1:358279     d2:358280     a:358281     q:0          e:0          ed:0          w:0          ep:4294967295 rid:179        r:358298     fbid:152964 T:0 0000000000400720: d0000090    adrp x16, 412000	[ADR  D: X16  I: 0x412000]
c0 currEL:0 2019462  sq:1472017    f:358278     b:358276     d1:358279     d2:358280     a:358281     q:358282     e:358286     ed:358289     w:358290     ep:5          rid:180        r:358298     fbid:152964 T:0 0000000000400724: f9401a11    ldr x17, [x16, #48]	[LDR  D: X17  S: X16  I: 0x30, 0x0] [vlid:191925] 
c0 currEL:0 2019464  sq:1472018    f:358278     b:358276     d1:358279     d2:358280     a:358281     q:358282     e:358286     ed:0          w:358287     ep:18         rid:180        r:358298     fbid:152964 T:0 0000000000400728: 9100c210    add x16, x16, #0x30	[ADDI  D: X16  S: X16  I: 0x30]
c0 currEL:0 2019466  sq:1472019    f:358278     b:358276     d1:358279     d2:358280     a:358281     q:358282     e:358290     ed:0          w:358291     ep:21         rid:181        r:358298     fbid:152964 T:0 000000000040072C: d61f0220    br x17	[BR  S: X17] [TAKEN]
c0 currEL:0 2019468  sq:1472020    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358288     ed:0          w:358289     ep:20         rid:182        r:358298     fbid:152965 T:0 0000007F8824E680: ca010007    eor x7, x0, x1	[LOP  D: X7  S: X0, X1  I: 0x0, 0x4]
c0 currEL:0 2019470  sq:1472021    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358289     ed:0          w:358290     ep:23         rid:182        r:358298     fbid:152965 T:0 0000007F8824E684: b200c3ea    orr x10, xzr, #0x101010101010101	[LOPI  D: X10  S: ZERO  I: 0x101010101010101]
c0 currEL:0 2019472  sq:1472022    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358289     ed:0          w:358290     ep:18         rid:183        r:358298     fbid:152965 T:0 0000007F8824E688: f24008ff    ands xzr, x7, #0x7	[LOPI  D: SINK, PS_NZCV  S: X7  I: 0x7]
c0 currEL:0 2019474  sq:1472023    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358291     ed:0          w:358292     ep:21         rid:183        r:358298     fbid:152965 T:0 0000007F8824E68C: 540003e1    b.ne 7f8824e708	[BCOND  S: PS_NZCV  I: 0x7f8824e708, 0x1]
c0 currEL:0 2019476  sq:1472024    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358290     ed:0          w:358291     ep:23         rid:184        r:358299     fbid:152965 T:0 0000007F8824E690: f2400807    ands x7, x0, #0x7	[LOPI  D: X7, PS_NZCV  S: X0  I: 0x7]
c0 currEL:0 2019478  sq:1472025    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358291     ed:0          w:358292     ep:19         rid:184        r:358299     fbid:152965 T:0 0000007F8824E694: 54000241    b.ne 7f8824e6dc	[BCOND  S: PS_NZCV  I: 0x7f8824e6dc, 0x1]
c0 currEL:0 2019480  sq:1472026    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358288     ed:358291     w:358292     ep:6          rid:185        r:358299     fbid:152965 T:0 0000007F8824E698: f8408402    ldr x2, [x0], #8	[LDR  D: X2  S: X0  I: 0x0, 0x0] [vlid:191926] 
c0 currEL:0 2019480  sq:1472027    f:358280     b:358279     d1:358281     d2:358282     a:358283     q:358284     e:358288     ed:0          w:358289     ep:23         rid:185        r:358299     fbid:152965 T:0 0000007F8824E698: f8408402    ldr x2, [x0], #8	[ADDI  D: X0  S: X0  I: 0x8]
c0 currEL:0 2019482  sq:1472028    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358289     ed:358292     w:358293     ep:7          rid:186        r:358299     fbid:152965 T:0 0000007F8824E69C: f8408423    ldr x3, [x1], #8	[LDR  D: X3  S: X1  I: 0x0, 0x0] [vlid:191927] 
c0 currEL:0 2019482  sq:1472029    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358289     ed:0          w:358290     ep:19         rid:186        r:358299     fbid:152965 T:0 0000007F8824E69C: f8408423    ldr x3, [x1], #8	[ADDI  D: X1  S: X1  I: 0x8]
c0 currEL:0 2019484  sq:1472030    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358292     ed:0          w:358293     ep:20         rid:186        r:358299     fbid:152965 T:0 0000007F8824E6A0: cb0a0047    sub x7, x2, x10	[ADD  D: X7  S: X2, X10  I: 0x0, 0x4]
c0 currEL:0 2019486  sq:1472031    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358292     ed:0          w:358293     ep:23         rid:187        r:358299     fbid:152965 T:0 0000007F8824E6A4: b200d848    orr x8, x2, #0x7f7f7f7f7f7f7f7f	[LOPI  D: X8  S: X2  I: 0x7f7f7f7f7f7f7f7f]
c0 currEL:0 2019488  sq:1472032    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358293     ed:0          w:358294     ep:19         rid:187        r:358299     fbid:152965 T:0 0000007F8824E6A8: ca030045    eor x5, x2, x3	[LOP  D: X5  S: X2, X3  I: 0x0, 0x4]
c0 currEL:0 2019490  sq:1472033    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358293     ed:0          w:358294     ep:20         rid:188        r:358299     fbid:152965 T:0 0000007F8824E6AC: 8a2800e4    bic x4, x7, x8	[LOP  D: X4  S: X7, X8  I: 0x0, 0x4]
c0 currEL:0 2019492  sq:1472034    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358294     ed:0          w:358295     ep:23         rid:188        r:358299     fbid:152965 T:0 0000007F8824E6B0: aa0400a6    orr x6, x5, x4	[LOP  D: X6  S: X5, X4  I: 0x0, 0x4]
c0 currEL:0 2019494  sq:1472035    f:358280     b:358279     d1:358282     d2:358283     a:358284     q:358285     e:358295     ed:0          w:358296     ep:19         rid:189        r:358299     fbid:152965 T:0 0000007F8824E6B4: b4ffff26    cbz x6, 7f8824e698	[CBZ  S: X6  I: 0x7f8824e698] [TAKEN]
c0 currEL:0 2019496  sq:1472036    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358290     ed:358293     w:358294     ep:5          rid:190        r:358299     fbid:152966 T:0 0000007F8824E698: f8408402    ldr x2, [x0], #8	[LDR  D: X2  S: X0  I: 0x0, 0x0] [vlid:191928] 
c0 currEL:0 2019496  sq:1472037    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358290     ed:0          w:358291     ep:22         rid:190        r:358299     fbid:152966 T:0 0000007F8824E698: f8408402    ldr x2, [x0], #8	[ADDI  D: X0  S: X0  I: 0x8]
c0 currEL:0 2019498  sq:1472038    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358290     ed:358294     w:358295     ep:6          rid:191        r:358299     fbid:152966 T:0 0000007F8824E69C: f8408423    ldr x3, [x1], #8	[LDR  D: X3  S: X1  I: 0x0, 0x0] [vlid:191929] 
c0 currEL:0 2019498  sq:1472039    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358290     ed:0          w:358291     ep:20         rid:191        r:358299     fbid:152966 T:0 0000007F8824E69C: f8408423    ldr x3, [x1], #8	[ADDI  D: X1  S: X1  I: 0x8]
c0 currEL:0 2019500  sq:1472040    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358294     ed:0          w:358295     ep:22         rid:191        r:358299     fbid:152966 T:0 0000007F8824E6A0: cb0a0047    sub x7, x2, x10	[ADD  D: X7  S: X2, X10  I: 0x0, 0x4]
c0 currEL:0 2019502  sq:1472041    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358294     ed:0          w:358295     ep:18         rid:192        r:358300     fbid:152966 T:0 0000007F8824E6A4: b200d848    orr x8, x2, #0x7f7f7f7f7f7f7f7f	[LOPI  D: X8  S: X2  I: 0x7f7f7f7f7f7f7f7f]
c0 currEL:0 2019504  sq:1472042    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358295     ed:0          w:358296     ep:20         rid:192        r:358300     fbid:152966 T:0 0000007F8824E6A8: ca030045    eor x5, x2, x3	[LOP  D: X5  S: X2, X3  I: 0x0, 0x4]
c0 currEL:0 2019506  sq:1472043    f:358281     b:358280     d1:358283     d2:358284     a:358285     q:358286     e:358295     ed:0          w:358296     ep:22         rid:193        r:358300     fbid:152966 T:0 0000007F8824E6AC: 8a2800e4    bic x4, x7, x8	[LOP  D: X4  S: X7, X8  I: 0x0, 0x4]
c0 currEL:0 2019508  sq:1472044    f:358281     b:358280     d1:358284     d2:358285     a:358286     q:358287     e:358296     ed:0          w:358297     ep:18         rid:193        r:358300     fbid:152966 T:0 0000007F8824E6B0: aa0400a6    orr x6, x5, x4	[LOP  D: X6  S: X5, X4  I: 0x0, 0x4]
c0 currEL:0 2019510  sq:1472045    f:358281     b:358280     d1:358284     d2:358285     a:358286     q:358287     e:358297     ed:0          w:358298     ep:21         rid:194        r:358301     fbid:152966 T:0 0000007F8824E6B4: b4ffff26    cbz x6, 7f8824e698	[CBZ  S: X6  I: 0x7f8824e698] [TAKEN]
c0 currEL:0 2019512  sq:1472046    f:358282     b:358281     d1:358284     d2:358285     a:358286     q:358287     e:358291     ed:358294     w:358295     ep:7          rid:195        r:358301     fbid:152967 T:0 0000007F8824E698: f8408402    ldr x2, [x0], #8	[LDR  D: X2  S: X0  I: 0x0, 0x0] [vlid:191930] 
c0 currEL:0 2019512  sq:1472047    f:358282     b:358281     d1:358284     d2:358285     a:358286     q:358287     e:358292     ed:0          w:358293     ep:18         rid:195        r:358301     fbid:152967 T:0 0000007F8824E698: f8408402    ldr x2, [x0], #8	[ADDI  D: X0  S: X0  I: 0x8]
c0 currEL:0 2019514  sq:1472048    f:358282     b:358281     d1:358284     d2:358285     a:358286     q:358287     e:358291     ed:358301     w:358302     ep:5          rid:196        r:358305     fbid:152967 T:0 0000007F8824E69C: f8408423    ldr x3, [x1], #8	[LDR  D: X3  S: X1  I: 0x0, 0x0] [vlid:191931] 
c0 currEL:0 2019514  sq:1472049    f:358282     b:358281     d1:358284     d2:358285     a:358286     q:358287     e:358291     ed:0          w:358292     ep:22         rid:196        r:358305     fbid:152967 T:0 0000007F8824E69C: f8408423    ldr x3, [x1], #8	[ADDI  D: X1  S: X1  I: 0x8]
c0 currEL:0 2019516  sq:1472050    f:358282     b:358281     d1:358284     d2:358285     a:358286     q:358287     e:358295     ed:0          w:358296     ep:18         rid:196        r:358305     fbid:152967 T:0 0000007F8824E6A0: cb0a0047    sub x7, x2, x10	[ADD  D: X7  S: X2, X10  I: 0x0, 0x4]
c0 currEL:0 2019518  sq:1472051    f:358282     b:358281     d1:358284     d2:358285     a:358286     q:358287     e:358296     ed:0          w:358297     ep:20         rid:197        r:358305     fbid:152967 T:0 0000007F8824E6A4: b200d848    orr x8, x2, #0x7f7f7f7f7f7f7f7f	[LOPI  D: X8  S: X2  I: 0x7f7f7f7f7f7f7f7f]
c0 currEL:0 2019520  sq:1472052    f:358282     b:358281     d1:358285     d2:358286     a:358287     q:358288     e:358302     ed:0          w:358303     ep:22         rid:197        r:358305     fbid:152967 T:0 0000007F8824E6A8: ca030045    eor x5, x2, x3	[LOP  D: X5  S: X2, X3  I: 0x0, 0x4]
c0 currEL:0 2019522  sq:1472053    f:358282     b:358281     d1:358285     d2:358286     a:358287     q:358288     e:358297     ed:0          w:358298     ep:18         rid:198        r:358306     fbid:152967 T:0 0000007F8824E6AC: 8a2800e4    bic x4, x7, x8	[LOP  D: X4  S: X7, X8  I: 0x0, 0x4]
c0 currEL:0 2019524  sq:1472054    f:358282     b:358281     d1:358285     d2:358286     a:358287     q:358288     e:358303     ed:0          w:358304     ep:21         rid:198        r:358306     fbid:152967 T:0 0000007F8824E6B0: aa0400a6    orr x6, x5, x4	[LOP  D: X6  S: X5, X4  I: 0x0, 0x4]
c0 currEL:0 2019526  sq:1472055    f:358282     b:358281     d1:358285     d2:358286     a:358287     q:358288     e:358304     ed:0          w:358305     ep:23         rid:199        r:358308     fbid:152967 T:0 0000007F8824E6B4: b4ffff26    cbz x6, 7f8824e698	[CBZ  S: X6  I: 0x7f8824e698]
c0 currEL:0 2019528  sq:1472056    f:358282     b:358281     d1:358285     d2:358286     a:358287     q:358288     e:358304     ed:0          w:358305     ep:19         rid:200        r:358308     fbid:152967 T:0 0000007F8824E6B8: dac00cc6    rev x6, x6	[REV  D: X6  S: X6]
c0 currEL:0 2019530  sq:1472057    f:358282     b:358281     d1:358285     d2:358286     a:358287     q:358288     e:358295     ed:0          w:358296     ep:21         rid:200        r:358308     fbid:152967 T:0 0000007F8824E6BC: dac00c42    rev x2, x2	[REV  D: X2  S: X2]
c0 currEL:0 2019532  sq:1472058    f:358283     b:358282     d1:358285     d2:358286     a:358287     q:358288     e:358305     ed:0          w:358306     ep:22         rid:201        r:358308     fbid:152968 T:0 0000007F8824E6C0: dac010cb    clz x11, x6	[CLX  D: X11  S: X6]
c0 currEL:0 2019534  sq:1472059    f:358283     b:358282     d1:358285     d2:358286     a:358287     q:358288     e:358302     ed:0          w:358303     ep:19         rid:201        r:358308     fbid:152968 T:0 0000007F8824E6C4: dac00c63    rev x3, x3	[REV  D: X3  S: X3]
c0 currEL:0 2019536  sq:1472060    f:358283     b:358282     d1:358286     d2:358287     a:358288     q:358289     e:358306     ed:0          w:358307     ep:18         rid:202        r:358309     fbid:152968 T:0 0000007F8824E6C8: 9acb2042    lsl x2, x2, x11	[LS  D: X2  S: X2, X11]
c0 currEL:0 2019538  sq:1472061    f:358283     b:358282     d1:358286     d2:358287     a:358288     q:358289     e:358306     ed:0          w:358307     ep:20         rid:202        r:358309     fbid:152968 T:0 0000007F8824E6CC: 9acb2063    lsl x3, x3, x11	[LS  D: X3  S: X3, X11]
c0 currEL:0 2019540  sq:1472062    f:358283     b:358282     d1:358286     d2:358287     a:358288     q:358289     e:358307     ed:0          w:358308     ep:22         rid:203        r:358312     fbid:152968 T:0 0000007F8824E6D0: d378fc42    lsr x2, x2, #56	[BFM  D: X2  S: X2, ZERO  I: 0x1e3f]
c0 currEL:0 2019542  sq:1472063    f:358283     b:358282     d1:358286     d2:358287     a:358288     q:358289     e:358308     ed:0          w:358310     ep:18         rid:203        r:358312     fbid:152968 T:0 0000007F8824E6D4: cb43e040    sub x0, x2, x3, lsr #56	[ADD  D: X0  S: X2, X3  I: 0x38, 0x1]
c0 currEL:0 2019544  sq:1472064    f:358283     b:358282     d1:358286     d2:358287     a:358288     q:358289     e:358293     ed:0          w:358294     ep:21         rid:204        r:358312     fbid:152968 T:0 0000007F8824E6D8: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019546  sq:1472065    f:358284     b:358283     d1:358286     d2:358287     a:358288     q:0          e:0          ed:0          w:0          ep:4294967295 rid:205        r:358313     fbid:152969 T:0 0000000000400F00: 52800001    mov w1, #0x0	[MOVI  D: X1  I: 0x0]
c0 currEL:0 2019548  sq:1472066    f:358284     b:358283     d1:358286     d2:358287     a:358288     q:358289     e:358310     ed:0          w:358311     ep:19         rid:205        r:358313     fbid:152969 T:0 0000000000400F04: 7100001f    cmp w0, #0x0	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x0]
c0 currEL:0 2019550  sq:1472067    f:358284     b:358283     d1:358286     d2:358287     a:358288     q:358289     e:358310     ed:0          w:358311     ep:19         rid:206        r:358314     fbid:152969 T:0 0000000000400F08: 540000ad    b.le 400f1c	[BCOND  S: PS_NZCV  I: 0x400f1c, 0xd] [TAKEN]
c0 currEL:0 2019552  sq:1472068    f:358286     b:358283     d1:358287     d2:358288     a:358289     q:0          e:0          ed:0          w:0          ep:4294967295 rid:207        r:358314     fbid:152970 T:0 0000000000400F1C: 2a0103e0    mov w0, w1	[MOV  D: X0  S: X1]
c0 currEL:0 2019554  sq:1472069    f:358286     b:358283     d1:358287     d2:358288     a:358289     q:358290     e:358294     ed:358297     w:358298     ep:6          rid:208        r:358314     fbid:152970 T:0 0000000000400F20: f84107fe    ldr x30, [sp], #16	[LDR  D: X30  S: SP_EL3  I: 0x0, 0x0] [vlid:191932] 
c0 currEL:0 2019554  sq:1472070    f:358286     b:358283     d1:358287     d2:358288     a:358289     q:358290     e:358296     ed:0          w:358297     ep:21         rid:208        r:358314     fbid:152970 T:0 0000000000400F20: f84107fe    ldr x30, [sp], #16	[ADDI  D: SP_EL3  S: SP_EL3  I: 0x10]
c0 currEL:0 2019556  sq:1472071    f:358286     b:358283     d1:358287     d2:358288     a:358289     q:358290     e:358298     ed:0          w:358299     ep:23         rid:208        r:358314     fbid:152970 T:0 0000000000400F24: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019558  sq:1472072    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358297     ed:0          w:358298     ep:20         rid:209        r:358314     fbid:152971 T:0 00000000004008E8: 7100001f    cmp w0, #0x0	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x0]
c0 currEL:0 2019560  sq:1472073    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:0          e:0          ed:0          w:0          ep:4294967295 rid:209        r:358314     fbid:152971 T:0 00000000004008EC: 528000e4    mov w4, #0x7	[MOVI  D: X4  I: 0x7]
c0 currEL:0 2019562  sq:1472074    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358298     ed:0          w:358299     ep:19         rid:210        r:358314     fbid:152971 T:0 00000000004008F0: 1a9f17e3    csinc w3, wzr, wzr, ne	[CSEL  D: X3  S: ZERO, ZERO, PS_NZCV  I: 0x1]
c0 currEL:0 2019564  sq:1472075    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358298     ed:0          w:358299     ep:20         rid:210        r:358314     fbid:152971 T:0 00000000004008F4: 910163e2    add x2, sp, #0x58	[ADDI  D: X2  S: SP_EL3  I: 0x58]
c0 currEL:0 2019566  sq:1472076    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358295     ed:0          w:358296     ep:23         rid:211        r:358314     fbid:152971 T:0 00000000004008F8: 52800061    mov w1, #0x3	[MOVI  D: X1  I: 0x3]
c0 currEL:0 2019568  sq:1472077    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:0          e:0          ed:0          w:0          ep:4294967295 rid:211        r:358314     fbid:152971 T:0 00000000004008FC: 52800040    mov w0, #0x2	[MOVI  D: X0  I: 0x2]
c0 currEL:0 2019570  sq:1472078    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358295     ed:0          w:358299     ep:0          rid:212        r:358314     fbid:152971 T:0 0000000000400900: b9002e63    str w3, [x19, #44]	[STRA  S: X19  I: 0x2c] [vsid:150089] 
c0 currEL:0 2019570  sq:1472079    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358299     ed:0          w:358301     ep:17         rid:212        r:358314     fbid:152971 T:0 0000000000400900: b9002e63    str w3, [x19, #44]	[STRD  S: X3] [vsid:150089] 
c0 currEL:0 2019572  sq:1472080    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358297     ed:0          w:358301     ep:4          rid:213        r:358314     fbid:152971 T:0 0000000000400904: b9005be4    str w4, [sp, #88]	[STRA  S: SP_EL3  I: 0x58] [vsid:150090] 
c0 currEL:0 2019572  sq:1472081    f:358287     b:358284     d1:358288     d2:358289     a:358290     q:358291     e:358295     ed:0          w:358297     ep:15         rid:213        r:358314     fbid:152971 T:0 0000000000400904: b9005be4    str w4, [sp, #88]	[STRD  S: X4] [vsid:150090] 
c0 currEL:0 2019574  sq:1472082    f:358287     b:358284     d1:358289     d2:358290     a:358291     q:358292     e:358296     ed:0          w:358297     ep:23         rid:213        r:358314     fbid:152971 T:0 0000000000400908: 9400014f    bl 400e44	[BL  D: X30  I: 0x400e44] [TAKEN]
c0 currEL:0 2019576  sq:1472083    f:358288     b:358284     d1:358289     d2:358290     a:358291     q:358292     e:358299     ed:0          w:358300     ep:19         rid:214        r:358315     fbid:152972 T:0 0000000000400E44: 11000800    add w0, w0, #0x2	[ADDI  D: X0  S: X0  I: 0x2]
c0 currEL:0 2019578  sq:1472084    f:358288     b:358284     d1:358289     d2:358290     a:358291     q:358292     e:358300     ed:0          w:358301     ep:20         rid:214        r:358315     fbid:152972 T:0 0000000000400E48: 0b010000    add w0, w0, w1	[ADD  D: X0  S: X0, X1  I: 0x0, 0x4]
c0 currEL:0 2019580  sq:1472085    f:358288     b:358284     d1:358289     d2:358290     a:358291     q:358292     e:358299     ed:0          w:358303     ep:0          rid:215        r:358315     fbid:152972 T:0 0000000000400E4C: b9000040    str w0, [x2, #0]	[STRA  S: X2  I: 0x0] [vsid:150091] 
c0 currEL:0 2019580  sq:1472086    f:358288     b:358284     d1:358289     d2:358290     a:358291     q:358292     e:358301     ed:0          w:358303     ep:17         rid:215        r:358315     fbid:152972 T:0 0000000000400E4C: b9000040    str w0, [x2, #0]	[STRD  S: X0] [vsid:150091] 
c0 currEL:0 2019582  sq:1472087    f:358288     b:358284     d1:358289     d2:358290     a:358291     q:358292     e:358297     ed:0          w:358298     ep:19         rid:215        r:358315     fbid:152972 T:0 0000000000400E50: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019584  sq:1472088    f:358289     b:358285     d1:358290     d2:358291     a:358292     q:358293     e:358297     ed:358306     w:358307     ep:7          rid:216        r:358315     fbid:152973 T:0 000000000040090C: b9405be3    ldr w3, [sp, #88]	[LDR  D: X3  S: SP_EL3  I: 0x58, 0x0] [vlid:191933] 
c0 currEL:0 2019586  sq:1472089    f:358289     b:358285     d1:358290     d2:358291     a:358292     q:0          e:0          ed:0          w:0          ep:4294967295 rid:216        r:358315     fbid:152973 T:0 0000000000400910: aa1403e1    mov x1, x20	[MOV  D: X1  S: X20]
c0 currEL:0 2019588  sq:1472090    f:358289     b:358285     d1:358290     d2:358291     a:358292     q:358293     e:358297     ed:0          w:358298     ep:23         rid:217        r:358315     fbid:152973 T:0 0000000000400914: 9100e260    add x0, x19, #0x38	[ADDI  D: X0  S: X19  I: 0x38]
c0 currEL:0 2019590  sq:1472091    f:358289     b:358285     d1:358290     d2:358291     a:358292     q:0          e:0          ed:0          w:0          ep:4294967295 rid:217        r:358315     fbid:152973 T:0 0000000000400918: 52800062    mov w2, #0x3	[MOVI  D: X2  I: 0x3]
c0 currEL:0 2019592  sq:1472092    f:358289     b:358285     d1:358290     d2:358291     a:358292     q:358293     e:358298     ed:0          w:358299     ep:21         rid:218        r:358315     fbid:152973 T:0 000000000040091C: 9400014e    bl 400e54	[BL  D: X30  I: 0x400e54] [TAKEN]
c0 currEL:0 2019594  sq:1472093    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:358294     e:358299     ed:0          w:358300     ep:21         rid:219        r:358315     fbid:152974 T:0 0000000000400E54: 11001445    add w5, w2, #0x5	[ADDI  D: X5  S: X2  I: 0x5]
c0 currEL:0 2019596  sq:1472094    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:0          e:0          ed:0          w:0          ep:4294967295 rid:219        r:358315     fbid:152974 T:0 0000000000400E58: 52801906    mov w6, #0xc8	[MOVI  D: X6  I: 0xc8]
c0 currEL:0 2019598  sq:1472095    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:358294     e:358298     ed:0          w:358299     ep:18         rid:220        r:358315     fbid:152974 T:0 0000000000400E5C: 937e7c42    sbfiz x2, x2, #2, #32	[BFM  D: X2  S: X2, ZERO  I: 0x1f9f]
c0 currEL:0 2019600  sq:1472096    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:358294     e:358301     ed:0          w:358302     ep:20         rid:220        r:358315     fbid:152974 T:0 0000000000400E60: 93407ca7    sxtw x7, w5	[BFM  D: X7  S: X5, ZERO  I: 0x101f]
c0 currEL:0 2019602  sq:1472097    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:358294     e:358300     ed:0          w:358302     ep:22         rid:221        r:358315     fbid:152974 T:0 0000000000400E64: 8b25c808    add x8, x0, x5, sxtw #2	[ADDE  D: X8  S: X0, X5  I: 0x2, 0x6]
c0 currEL:0 2019604  sq:1472098    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:358294     e:358300     ed:0          w:358302     ep:18         rid:221        r:358315     fbid:152974 T:0 0000000000400E68: 9b267ca6    smull x6, w5, w6	[MULL  D: X6  S: X5, X6, ZERO]
c0 currEL:0 2019606  sq:1472099    f:358290     b:358285     d1:358291     d2:358292     a:358293     q:358294     e:358302     ed:0          w:358303     ep:21         rid:222        r:358316     fbid:152974 T:0 0000000000400E6C: 8b0200c4    add x4, x6, x2	[ADD  D: X4  S: X6, X2  I: 0x0, 0x4]
c0 currEL:0 2019608  sq:1472100    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358304     ed:0          w:358308     ep:4          rid:223        r:358316     fbid:152974 T:0 0000000000400E70: b8277803    str w3, [x0, x7, uxtx/lsl #2]	[STRA_IND  S: X0, X7  I: 0x2] [vsid:150092] 
c0 currEL:0 2019608  sq:1472101    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358307     ed:0          w:358309     ep:15         rid:223        r:358316     fbid:152974 T:0 0000000000400E70: b8277803    str w3, [x0, x7, uxtx/lsl #2]	[STRD  S: X3] [vsid:150092] 
c0 currEL:0 2019610  sq:1472102    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358304     ed:0          w:358305     ep:21         rid:223        r:358316     fbid:152974 T:0 0000000000400E74: 8b040024    add x4, x1, x4	[ADD  D: X4  S: X1, X4  I: 0x0, 0x4]
c0 currEL:0 2019612  sq:1472103    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358302     ed:0          w:358306     ep:0          rid:0          r:358316     fbid:152974 T:0 0000000000400E78: b9000503    str w3, [x8, #4]	[STRA  S: X8  I: 0x4] [vsid:150093] 
c0 currEL:0 2019612  sq:1472104    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358307     ed:0          w:358309     ep:17         rid:0          r:358316     fbid:152974 T:0 0000000000400E78: b9000503    str w3, [x8, #4]	[STRD  S: X3] [vsid:150093] 
c0 currEL:0 2019614  sq:1472105    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358303     ed:0          w:358307     ep:4          rid:1          r:358316     fbid:152974 T:0 0000000000400E7C: b9007905    str w5, [x8, #120]	[STRA  S: X8  I: 0x78] [vsid:150094] 
c0 currEL:0 2019614  sq:1472106    f:358290     b:358285     d1:358292     d2:358293     a:358294     q:358295     e:358300     ed:0          w:358302     ep:15         rid:1          r:358316     fbid:152974 T:0 0000000000400E7C: b9007905    str w5, [x8, #120]	[STRD  S: X5] [vsid:150094] 
c0 currEL:0 2019616  sq:1472107    f:358291     b:358286     d1:358292     d2:358293     a:358294     q:358295     e:358305     ed:0          w:358306     ep:21         rid:1          r:358316     fbid:152975 T:0 0000000000400E80: 8b060021    add x1, x1, x6	[ADD  D: X1  S: X1, X6  I: 0x0, 0x4]
c0 currEL:0 2019618  sq:1472108    f:358291     b:358286     d1:358292     d2:358293     a:358294     q:358295     e:358306     ed:0          w:358307     ep:22         rid:2          r:358316     fbid:152975 T:0 0000000000400E84: 8b020021    add x1, x1, x2	[ADD  D: X1  S: X1, X2  I: 0x0, 0x4]
c0 currEL:0 2019620  sq:1472109    f:358291     b:358286     d1:358292     d2:358293     a:358294     q:0          e:0          ed:0          w:0          ep:4294967295 rid:2          r:358316     fbid:152975 T:0 0000000000400E88: d0000083    adrp x3, 412000	[ADR  D: X3  I: 0x412000]
c0 currEL:0 2019622  sq:1472110    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358305     ed:358308     w:358309     ep:5          rid:3          r:358316     fbid:152975 T:0 0000000000400E8C: b9401082    ldr w2, [x4, #16]	[LDR  D: X2  S: X4  I: 0x10, 0x0] [vlid:191934] 
c0 currEL:0 2019624  sq:1472111    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:0          e:0          ed:0          w:0          ep:4294967295 rid:3          r:358316     fbid:152975 T:0 0000000000400E90: 528000a6    mov w6, #0x5	[MOVI  D: X6  I: 0x5]
c0 currEL:0 2019626  sq:1472112    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358305     ed:0          w:358309     ep:0          rid:4          r:358316     fbid:152975 T:0 0000000000400E94: b9001885    str w5, [x4, #24]	[STRA  S: X4  I: 0x18] [vsid:150095] 
c0 currEL:0 2019626  sq:1472113    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358300     ed:0          w:358302     ep:17         rid:4          r:358316     fbid:152975 T:0 0000000000400E94: b9001885    str w5, [x4, #24]	[STRD  S: X5] [vsid:150095] 
c0 currEL:0 2019628  sq:1472114    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358309     ed:0          w:358310     ep:19         rid:4          r:358316     fbid:152975 T:0 0000000000400E98: 11000442    add w2, w2, #0x1	[ADDI  D: X2  S: X2  I: 0x1]
c0 currEL:0 2019630  sq:1472115    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358305     ed:0          w:358309     ep:4          rid:5          r:358316     fbid:152975 T:0 0000000000400E9C: 29021482    stp w2, w5, [x4, #16]	[STPA  S: X4  I: 0x10] [vsid:150096] 
c0 currEL:0 2019630  sq:1472116    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358310     ed:0          w:358312     ep:15         rid:5          r:358316     fbid:152975 T:0 0000000000400E9C: 29021482    stp w2, w5, [x4, #16]	[STPD  S: X2, X5  I: 0x10] [vsid:150096] 
c0 currEL:0 2019632  sq:1472117    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358304     ed:358312     w:358313     ep:6          rid:6          r:358317     fbid:152975 T:0 0000000000400EA0: b8677800    ldr w0, [x0, x7, uxtx/lsl #2]	[LDRIND_IMM0  D: X0  S: X0, X7  I: 0x2, 0x0] [vlid:191935] 
c0 currEL:0 2019634  sq:1472118    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358301     ed:0          w:358305     ep:0          rid:7          r:358317     fbid:152975 T:0 0000000000400EA4: b900d866    str w6, [x3, #216]	[STRA  S: X3  I: 0xd8] [vsid:150097] 
c0 currEL:0 2019634  sq:1472119    f:358291     b:358286     d1:358293     d2:358294     a:358295     q:358296     e:358303     ed:0          w:358305     ep:17         rid:7          r:358317     fbid:152975 T:0 0000000000400EA4: b900d866    str w6, [x3, #216]	[STRD  S: X6] [vsid:150097] 
c0 currEL:0 2019636  sq:1472120    f:358291     b:358286     d1:358294     d2:358295     a:358296     q:358297     e:358307     ed:0          w:358311     ep:4          rid:8          r:358317     fbid:152975 T:0 0000000000400EA8: b90fb420    str w0, [x1, #4020]	[STRA  S: X1  I: 0xfb4] [vsid:150098] 
c0 currEL:0 2019636  sq:1472121    f:358291     b:358286     d1:358294     d2:358295     a:358296     q:358297     e:358313     ed:0          w:358315     ep:15         rid:8          r:358317     fbid:152975 T:0 0000000000400EA8: b90fb420    str w0, [x1, #4020]	[STRD  S: X0] [vsid:150098] 
c0 currEL:0 2019638  sq:1472122    f:358291     b:358286     d1:358294     d2:358295     a:358296     q:358297     e:358301     ed:0          w:358302     ep:23         rid:8          r:358317     fbid:152975 T:0 0000000000400EAC: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019640  sq:1472123    f:358292     b:358287     d1:358294     d2:358295     a:358296     q:358297     e:358301     ed:358304     w:358305     ep:7          rid:9          r:358317     fbid:152976 T:0 0000000000400920: f9400a60    ldr x0, [x19, #16]	[LDR  D: X0  S: X19  I: 0x10, 0x0] [vlid:191936] 
c0 currEL:0 2019642  sq:1472124    f:358292     b:358287     d1:358294     d2:358295     a:358296     q:358297     e:358301     ed:0          w:358302     ep:21         rid:9          r:358317     fbid:152976 T:0 0000000000400924: 940000cf    bl 400c60	[BL  D: X30  I: 0x400c60] [TAKEN]
c0 currEL:0 2019644  sq:1472125    f:358293     b:358287     d1:358294     d2:358295     a:358296     q:358297     e:358303     ed:0          w:358307     ep:0          rid:10         r:358317     fbid:152977 T:0 0000000000400C60: a9be53f3    stp x19, x20, [sp, #-32]!	[STPA  S: SP_EL3  I: 0xffffffffffffffe0] [vsid:150099] 
c0 currEL:0 2019644  sq:1472126    f:358293     b:358287     d1:358294     d2:358295     a:358296     q:358297     e:358304     ed:0          w:358306     ep:17         rid:10         r:358317     fbid:152977 T:0 0000000000400C60: a9be53f3    stp x19, x20, [sp, #-32]!	[STPD  S: X19, X20  I: 0xffffffffffffffe0] [vsid:150099] 
c0 currEL:0 2019644  sq:1472127    f:358293     b:358287     d1:358294     d2:358295     a:358296     q:358297     e:358301     ed:0          w:358302     ep:19         rid:10         r:358317     fbid:152977 T:0 0000000000400C60: a9be53f3    stp x19, x20, [sp, #-32]!	[ADDI  D: SP_EL3  S: SP_EL3  I: 0xffffffffffffffe0]
c0 currEL:0 2019646  sq:1472128    f:358293     b:358287     d1:358294     d2:358295     a:358296     q:0          e:0          ed:0          w:0          ep:4294967295 rid:10         r:358317     fbid:152977 T:0 0000000000400C64: aa0003f4    mov x20, x0	[MOV  D: X20  S: X0]
c0 currEL:0 2019648  sq:1472129    f:358293     b:358287     d1:358294     d2:358295     a:358296     q:0          e:0          ed:0          w:0          ep:4294967295 rid:11         r:358317     fbid:152977 T:0 0000000000400C68: 528000a3    mov w3, #0x5	[MOVI  D: X3  I: 0x5]
c0 currEL:0 2019650  sq:1472130    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358306     ed:0          w:358310     ep:4          rid:12         r:358317     fbid:152977 T:0 0000000000400C6C: a9017bf5    stp x21, x30, [sp, #16]	[STPA  S: SP_EL3  I: 0x10] [vsid:150100] 
c0 currEL:0 2019650  sq:1472131    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358302     ed:0          w:358304     ep:15         rid:12         r:358317     fbid:152977 T:0 0000000000400C6C: a9017bf5    stp x21, x30, [sp, #16]	[STPD  S: X21, X30  I: 0x10] [vsid:150100] 
c0 currEL:0 2019652  sq:1472132    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:0          e:0          ed:0          w:0          ep:4294967295 rid:12         r:358317     fbid:152977 T:0 0000000000400C70: d0000095    adrp x21, 412000	[ADR  D: X21  I: 0x412000]
c0 currEL:0 2019654  sq:1472133    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358302     ed:0          w:358303     ep:18         rid:13         r:358317     fbid:152977 T:0 0000000000400C74: 910302b5    add x21, x21, #0xc0	[ADDI  D: X21  S: X21  I: 0xc0]
c0 currEL:0 2019656  sq:1472134    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358306     ed:358309     w:358310     ep:5          rid:14         r:358318     fbid:152977 T:0 0000000000400C78: f9400293    ldr x19, [x20, #0]	[LDR  D: X19  S: X20  I: 0x0, 0x0] [vlid:191937] 
c0 currEL:0 2019658  sq:1472135    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358302     ed:0          w:358303     ep:23         rid:14         r:358318     fbid:152977 T:0 0000000000400C7C: 52800140    mov w0, #0xa	[MOVI  D: X0  I: 0xa]
c0 currEL:0 2019660  sq:1472136    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358303     ed:358306     w:358307     ep:6          rid:15         r:358318     fbid:152977 T:0 0000000000400C80: f9400aa2    ldr x2, [x21, #16]	[LDR  D: X2  S: X21  I: 0x10, 0x0] [vlid:191938] 
c0 currEL:0 2019662  sq:1472137    f:358293     b:358287     d1:358295     d2:358296     a:358297     q:358298     e:358303     ed:358307     w:358308     ep:7          rid:16         r:358318     fbid:152977 T:0 0000000000400C84: b9401aa1    ldr w1, [x21, #24]	[LDR  D: X1  S: X21  I: 0x18, 0x0] [vlid:191939] 
c0 currEL:0 2019664  sq:1472138    f:358293     b:358287     d1:358296     d2:358297     a:358298     q:358299     e:358307     ed:358310     w:358311     ep:5          rid:17         r:358318     fbid:152977 T:0 0000000000400C88: a9401444    ldp x4, x5, [x2, #0]	[LDP  D: X4, X5  S: X2  I: 0x0, 0x0] [vlid:191940] 
c0 currEL:0 2019666  sq:1472139    f:358293     b:358287     d1:358296     d2:358297     a:358298     q:358299     e:358310     ed:0          w:358314     ep:0          rid:18         r:358318     fbid:152977 T:0 0000000000400C8C: a9001664    stp x4, x5, [x19, #0]	[STPA  S: X19  I: 0x0] [vsid:150101] 
c0 currEL:0 2019666  sq:1472140    f:358293     b:358287     d1:358296     d2:358297     a:358298     q:358299     e:358311     ed:0          w:358313     ep:17         rid:18         r:358318     fbid:152977 T:0 0000000000400C8C: a9001664    stp x4, x5, [x19, #0]	[STPD  S: X4, X5  I: 0x0] [vsid:150101] 
c0 currEL:0 2019668  sq:1472141    f:358293     b:358287     d1:358296     d2:358297     a:358298     q:358299     e:358307     ed:358310     w:358311     ep:6          rid:19         r:358318     fbid:152977 T:0 0000000000400C90: a9411444    ldp x4, x5, [x2, #16]	[LDP  D: X4, X5  S: X2  I: 0x10, 0x0] [vlid:191941] 
c0 currEL:0 2019670  sq:1472142    f:358293     b:358287     d1:358296     d2:358297     a:358298     q:358299     e:358310     ed:0          w:358314     ep:4          rid:20         r:358318     fbid:152977 T:0 0000000000400C94: a9011664    stp x4, x5, [x19, #16]	[STPA  S: X19  I: 0x10] [vsid:150102] 
c0 currEL:0 2019670  sq:1472143    f:358293     b:358287     d1:358296     d2:358297     a:358298     q:358299     e:358311     ed:0          w:358313     ep:15         rid:20         r:358318     fbid:152977 T:0 0000000000400C94: a9011664    stp x4, x5, [x19, #16]	[STPD  S: X4, X5  I: 0x10] [vsid:150102] 
c0 currEL:0 2019672  sq:1472144    f:358293     b:358287     d1:358297     d2:358298     a:358299     q:358300     e:358307     ed:358310     w:358311     ep:7          rid:21         r:358318     fbid:152977 T:0 0000000000400C98: a9421444    ldp x4, x5, [x2, #32]	[LDP  D: X4, X5  S: X2  I: 0x20, 0x0] [vlid:191942] 
c0 currEL:0 2019674  sq:1472145    f:358293     b:358287     d1:358297     d2:358298     a:358299     q:358300     e:358311     ed:0          w:358315     ep:0          rid:22         r:358319     fbid:152977 T:0 0000000000400C9C: a9021664    stp x4, x5, [x19, #32]	[STPA  S: X19  I: 0x20] [vsid:150103] 
c0 currEL:0 2019674  sq:1472146    f:358293     b:358287     d1:358297     d2:358298     a:358299     q:358300     e:358312     ed:0          w:358314     ep:17         rid:22         r:358319     fbid:152977 T:0 0000000000400C9C: a9021664    stp x4, x5, [x19, #32]	[STPD  S: X4, X5  I: 0x20] [vsid:150103] 
c0 currEL:0 2019676  sq:1472147    f:358294     b:358289     d1:358297     d2:358298     a:358299     q:358300     e:358308     ed:358311     w:358312     ep:5          rid:23         r:358319     fbid:152978 T:0 0000000000400CA0: f9401844    ldr x4, [x2, #48]	[LDR  D: X4  S: X2  I: 0x30, 0x0] [vlid:191943] 
c0 currEL:0 2019678  sq:1472148    f:358294     b:358289     d1:358297     d2:358298     a:358299     q:358300     e:358311     ed:0          w:358315     ep:4          rid:24         r:358319     fbid:152978 T:0 0000000000400CA4: f9001a64    str x4, [x19, #48]	[STRA  S: X19  I: 0x30] [vsid:150104] 
c0 currEL:0 2019678  sq:1472149    f:358294     b:358289     d1:358297     d2:358298     a:358299     q:358300     e:358312     ed:0          w:358314     ep:15         rid:24         r:358319     fbid:152978 T:0 0000000000400CA4: f9001a64    str x4, [x19, #48]	[STRD  S: X4] [vsid:150104] 
c0 currEL:0 2019680  sq:1472150    f:358294     b:358289     d1:358297     d2:358298     a:358299     q:358300     e:358306     ed:0          w:358310     ep:0          rid:25         r:358319     fbid:152978 T:0 0000000000400CA8: b9001283    str w3, [x20, #16]	[STRA  S: X20  I: 0x10] [vsid:150105] 
c0 currEL:0 2019680  sq:1472151    f:358294     b:358289     d1:358297     d2:358298     a:358299     q:358300     e:358305     ed:0          w:358307     ep:17         rid:25         r:358319     fbid:152978 T:0 0000000000400CA8: b9001283    str w3, [x20, #16]	[STRD  S: X3] [vsid:150105] 
c0 currEL:0 2019682  sq:1472152    f:358294     b:358289     d1:358297     d2:358298     a:358299     q:358300     e:358305     ed:358308     w:358309     ep:6          rid:26         r:358319     fbid:152978 T:0 0000000000400CAC: f9400284    ldr x4, [x20, #0]	[LDR  D: X4  S: X20  I: 0x0, 0x0] [vlid:191944] 
c0 currEL:0 2019684  sq:1472153    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358312     ed:0          w:358316     ep:4          rid:27         r:358319     fbid:152978 T:0 0000000000400CB0: f9000264    str x4, [x19, #0]	[STRA  S: X19  I: 0x0] [vsid:150106] 
c0 currEL:0 2019684  sq:1472154    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358309     ed:0          w:358311     ep:15         rid:27         r:358319     fbid:152978 T:0 0000000000400CB0: f9000264    str x4, [x19, #0]	[STRD  S: X4] [vsid:150106] 
c0 currEL:0 2019686  sq:1472155    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358312     ed:0          w:358316     ep:0          rid:28         r:358319     fbid:152978 T:0 0000000000400CB4: b9001263    str w3, [x19, #16]	[STRA  S: X19  I: 0x10] [vsid:150107] 
c0 currEL:0 2019686  sq:1472156    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358306     ed:0          w:358308     ep:17         rid:28         r:358319     fbid:152978 T:0 0000000000400CB4: b9001263    str w3, [x19, #16]	[STRD  S: X3] [vsid:150107] 
c0 currEL:0 2019688  sq:1472157    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358308     ed:358311     w:358312     ep:7          rid:29         r:358319     fbid:152978 T:0 0000000000400CB8: f9400042    ldr x2, [x2, #0]	[LDR  D: X2  S: X2  I: 0x0, 0x0] [vlid:191945] 
c0 currEL:0 2019690  sq:1472158    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358313     ed:0          w:358317     ep:4          rid:30         r:358320     fbid:152978 T:0 0000000000400CBC: f9000262    str x2, [x19, #0]	[STRA  S: X19  I: 0x0] [vsid:150108] 
c0 currEL:0 2019690  sq:1472159    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358314     ed:0          w:358316     ep:15         rid:30         r:358320     fbid:152978 T:0 0000000000400CBC: f9000262    str x2, [x19, #0]	[STRD  S: X2] [vsid:150108] 
c0 currEL:0 2019692  sq:1472160    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358309     ed:358312     w:358313     ep:5          rid:31         r:358320     fbid:152978 T:0 0000000000400CC0: f9400aa2    ldr x2, [x21, #16]	[LDR  D: X2  S: X21  I: 0x10, 0x0] [vlid:191946] 
c0 currEL:0 2019694  sq:1472161    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358313     ed:0          w:358314     ep:23         rid:31         r:358320     fbid:152978 T:0 0000000000400CC4: 91004042    add x2, x2, #0x10	[ADDI  D: X2  S: X2  I: 0x10]
c0 currEL:0 2019696  sq:1472162    f:358294     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358305     ed:0          w:358306     ep:19         rid:32         r:358320     fbid:152978 T:0 0000000000400CC8: 9400005f    bl 400e44	[BL  D: X30  I: 0x400e44] [TAKEN]
c0 currEL:0 2019698  sq:1472163    f:358295     b:358289     d1:358298     d2:358299     a:358300     q:358301     e:358305     ed:0          w:358306     ep:20         rid:33         r:358320     fbid:152979 T:0 0000000000400E44: 11000800    add w0, w0, #0x2	[ADDI  D: X0  S: X0  I: 0x2]
c0 currEL:0 2019700  sq:1472164    f:358295     b:358289     d1:358299     d2:358300     a:358301     q:358302     e:358308     ed:0          w:358309     ep:20         rid:33         r:358320     fbid:152979 T:0 0000000000400E48: 0b010000    add w0, w0, w1	[ADD  D: X0  S: X0, X1  I: 0x0, 0x4]
c0 currEL:0 2019702  sq:1472165    f:358295     b:358289     d1:358299     d2:358300     a:358301     q:358302     e:358314     ed:0          w:358318     ep:0          rid:34         r:358321     fbid:152979 T:0 0000000000400E4C: b9000040    str w0, [x2, #0]	[STRA  S: X2  I: 0x0] [vsid:150109] 
c0 currEL:0 2019702  sq:1472166    f:358295     b:358289     d1:358299     d2:358300     a:358301     q:358302     e:358309     ed:0          w:358311     ep:17         rid:34         r:358321     fbid:152979 T:0 0000000000400E4C: b9000040    str w0, [x2, #0]	[STRD  S: X0] [vsid:150109] 
c0 currEL:0 2019704  sq:1472167    f:358295     b:358289     d1:358299     d2:358300     a:358301     q:358302     e:358306     ed:0          w:358307     ep:19         rid:34         r:358321     fbid:152979 T:0 0000000000400E50: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019706  sq:1472168    f:358296     b:358290     d1:358299     d2:358300     a:358301     q:358302     e:358310     ed:358318     w:358319     ep:6          rid:35         r:358323     fbid:152980 T:0 0000000000400CCC: b9400a60    ldr w0, [x19, #8]	[LDR  D: X0  S: X19  I: 0x8, 0x0] [vlid:191947] 
c0 currEL:0 2019708  sq:1472169    f:358296     b:358290     d1:358299     d2:358300     a:358301     q:358302     e:358319     ed:0          w:358320     ep:23         rid:35         r:358323     fbid:152980 T:0 0000000000400CD0: 340001a0    cbz w0, 400d04	[CBZ  S: X0  I: 0x400d04] [TAKEN]
c0 currEL:0 2019710  sq:1472170    f:358297     b:358290     d1:358299     d2:358300     a:358301     q:358302     e:358306     ed:358309     w:358310     ep:7          rid:36         r:358323     fbid:152981 T:0 0000000000400D04: b9400e80    ldr w0, [x20, #12]	[LDR  D: X0  S: X20  I: 0xc, 0x0] [vlid:191948] 
c0 currEL:0 2019712  sq:1472171    f:358297     b:358290     d1:358299     d2:358300     a:358301     q:0          e:0          ed:0          w:0          ep:4294967295 rid:36         r:358323     fbid:152981 T:0 0000000000400D08: 528000c1    mov w1, #0x6	[MOVI  D: X1  I: 0x6]
c0 currEL:0 2019714  sq:1472172    f:358297     b:358290     d1:358299     d2:358300     a:358301     q:358306     e:358314     ed:0          w:358318     ep:4          rid:37         r:358323     fbid:152981 T:0 0000000000400D0C: b9001261    str w1, [x19, #16]	[STRA  S: X19  I: 0x10] [vsid:150110] 
c0 currEL:0 2019714  sq:1472173    f:358297     b:358290     d1:358299     d2:358300     a:358301     q:358306     e:358315     ed:0          w:358317     ep:15         rid:37         r:358323     fbid:152981 T:0 0000000000400D0C: b9001261    str w1, [x19, #16]	[STRD  S: X1] [vsid:150110] 
c0 currEL:0 2019716  sq:1472174    f:358297     b:358290     d1:358300     d2:358301     a:358306     q:358307     e:358311     ed:0          w:358312     ep:23         rid:37         r:358323     fbid:152981 T:0 0000000000400D10: 91003261    add x1, x19, #0xc	[ADDI  D: X1  S: X19  I: 0xc]
c0 currEL:0 2019718  sq:1472175    f:358297     b:358290     d1:358300     d2:358301     a:358306     q:358307     e:358311     ed:0          w:358312     ep:19         rid:38         r:358323     fbid:152981 T:0 0000000000400D14: 94000037    bl 400df0	[BL  D: X30  I: 0x400df0] [TAKEN]
c0 currEL:0 2019720  sq:1472176    f:358298     b:358291     d1:358300     d2:358301     a:358306     q:358307     e:358311     ed:0          w:358312     ep:21         rid:39         r:358323     fbid:152982 T:0 0000000000400DF0: 7100081f    cmp w0, #0x2	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x2]
c0 currEL:0 2019722  sq:1472177    f:358298     b:358291     d1:358300     d2:358301     a:358306     q:358307     e:358311     ed:0          w:358312     ep:21         rid:39         r:358323     fbid:152982 T:0 0000000000400DF4: 54000220    b.eq 400e38	[BCOND  S: PS_NZCV  I: 0x400e38, 0x0] [TAKEN]
c0 currEL:0 2019724  sq:1472178    f:358299     b:358291     d1:358300     d2:358301     a:358306     q:0          e:0          ed:0          w:0          ep:4294967295 rid:40         r:358323     fbid:152983 T:0 0000000000400E38: 52800020    mov w0, #0x1	[MOVI  D: X0  I: 0x1]
c0 currEL:0 2019726  sq:1472179    f:358299     b:358291     d1:358300     d2:358301     a:358306     q:358307     e:358313     ed:0          w:358317     ep:0          rid:41         r:358323     fbid:152983 T:0 0000000000400E3C: b9000020    str w0, [x1, #0]	[STRA  S: X1  I: 0x0] [vsid:150111] 
c0 currEL:0 2019726  sq:1472180    f:358299     b:358291     d1:358300     d2:358301     a:358306     q:358307     e:358313     ed:0          w:358315     ep:17         rid:41         r:358323     fbid:152983 T:0 0000000000400E3C: b9000020    str w0, [x1, #0]	[STRD  S: X0] [vsid:150111] 
c0 currEL:0 2019728  sq:1472181    f:358299     b:358291     d1:358300     d2:358301     a:358306     q:358307     e:358312     ed:0          w:358313     ep:23         rid:41         r:358323     fbid:152983 T:0 0000000000400E40: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019730  sq:1472182    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358312     ed:358315     w:358316     ep:5          rid:42         r:358324     fbid:152984 T:0 0000000000400D18: f9400aa3    ldr x3, [x21, #16]	[LDR  D: X3  S: X21  I: 0x10, 0x0] [vlid:191949] 
c0 currEL:0 2019732  sq:1472183    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:0          e:0          ed:0          w:0          ep:4294967295 rid:42         r:358324     fbid:152984 T:0 0000000000400D1C: aa1303e2    mov x2, x19	[MOV  D: X2  S: X19]
c0 currEL:0 2019734  sq:1472184    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358312     ed:358321     w:358322     ep:6          rid:43         r:358325     fbid:152984 T:0 0000000000400D20: b9401260    ldr w0, [x19, #16]	[LDR  D: X0  S: X19  I: 0x10, 0x0] [vlid:191950] 
c0 currEL:0 2019736  sq:1472185    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:0          e:0          ed:0          w:0          ep:4294967295 rid:43         r:358325     fbid:152984 T:0 0000000000400D24: 52800141    mov w1, #0xa	[MOVI  D: X1  I: 0xa]
c0 currEL:0 2019738  sq:1472186    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358312     ed:358315     w:358316     ep:7          rid:44         r:358325     fbid:152984 T:0 0000000000400D28: a9417bf5    ldp x21, x30, [sp, #16]	[LDP  D: X21, X30  S: SP_EL3  I: 0x10, 0x0] [vlid:191951] 
c0 currEL:0 2019740  sq:1472187    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358316     ed:358319     w:358320     ep:5          rid:45         r:358325     fbid:152984 T:0 0000000000400D2C: f9400063    ldr x3, [x3, #0]	[LDR  D: X3  S: X3  I: 0x0, 0x0] [vlid:191952] 
c0 currEL:0 2019742  sq:1472188    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358315     ed:0          w:358319     ep:4          rid:46         r:358325     fbid:152984 T:0 0000000000400D30: f8010443    str x3, [x2], #16	[STRA  S: X2  I: 0x0] [vsid:150112] 
c0 currEL:0 2019742  sq:1472189    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358320     ed:0          w:358322     ep:15         rid:46         r:358325     fbid:152984 T:0 0000000000400D30: f8010443    str x3, [x2], #16	[STRD  S: X3  I: 0x10] [vsid:150112] 
c0 currEL:0 2019742  sq:1472190    f:358300     b:358292     d1:358301     d2:358306     a:358307     q:358308     e:358315     ed:0          w:358316     ep:23         rid:46         r:358325     fbid:152984 T:0 0000000000400D30: f8010443    str x3, [x2], #16	[ADDI  D: X2  S: X2  I: 0x10]
c0 currEL:0 2019744  sq:1472191    f:358300     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358313     ed:358316     w:358317     ep:6          rid:47         r:358325     fbid:152984 T:0 0000000000400D34: a8c253f3    ldp x19, x20, [sp], #32	[LDP  D: X19, X20  S: SP_EL3  I: 0x0, 0x0] [vlid:191953] 
c0 currEL:0 2019744  sq:1472192    f:358300     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358313     ed:0          w:358314     ep:18         rid:47         r:358325     fbid:152984 T:0 0000000000400D34: a8c253f3    ldp x19, x20, [sp], #32	[ADDI  D: SP_EL3  S: SP_EL3  I: 0x20]
c0 currEL:0 2019746  sq:1472193    f:358300     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358313     ed:0          w:358314     ep:21         rid:47         r:358325     fbid:152984 T:0 0000000000400D38: 14000043    b 400e44	[B  I: 0x400e44] [TAKEN]
c0 currEL:0 2019748  sq:1472194    f:358301     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358322     ed:0          w:358323     ep:22         rid:48         r:358326     fbid:152985 T:0 0000000000400E44: 11000800    add w0, w0, #0x2	[ADDI  D: X0  S: X0  I: 0x2]
c0 currEL:0 2019750  sq:1472195    f:358301     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358323     ed:0          w:358324     ep:18         rid:48         r:358326     fbid:152985 T:0 0000000000400E48: 0b010000    add w0, w0, w1	[ADD  D: X0  S: X0, X1  I: 0x0, 0x4]
c0 currEL:0 2019752  sq:1472196    f:358301     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358316     ed:0          w:358320     ep:0          rid:49         r:358328     fbid:152985 T:0 0000000000400E4C: b9000040    str w0, [x2, #0]	[STRA  S: X2  I: 0x0] [vsid:150113] 
c0 currEL:0 2019752  sq:1472197    f:358301     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358324     ed:0          w:358326     ep:17         rid:49         r:358328     fbid:152985 T:0 0000000000400E4C: b9000040    str w0, [x2, #0]	[STRD  S: X0] [vsid:150113] 
c0 currEL:0 2019754  sq:1472198    f:358301     b:358292     d1:358306     d2:358307     a:358308     q:358309     e:358316     ed:0          w:358317     ep:23         rid:49         r:358328     fbid:152985 T:0 0000000000400E50: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019756  sq:1472199    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:358310     e:358317     ed:358320     w:358321     ep:7          rid:50         r:358328     fbid:152986 T:0 0000000000400928: 3940c260    ldrb x0, [x19, #48]	[LDR  D: X0  S: X19  I: 0x30, 0x0] [vlid:191954] 
c0 currEL:0 2019758  sq:1472200    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:358310     e:358321     ed:0          w:358322     ep:19         rid:50         r:358328     fbid:152986 T:0 000000000040092C: 7101001f    cmp w0, #0x40	[ADDI  D: SINK, PS_NZCV  S: X0  I: 0x40]
c0 currEL:0 2019760  sq:1472201    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:358310     e:358321     ed:0          w:358322     ep:19         rid:51         r:358328     fbid:152986 T:0 0000000000400930: 54fffbe9    b.ls 4008ac	[BCOND  S: PS_NZCV  I: 0x4008ac, 0x9]
c0 currEL:0 2019762  sq:1472202    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:0          e:0          ed:0          w:0          ep:4294967295 rid:52         r:358328     fbid:152986 T:0 0000000000400934: 52800861    mov w1, #0x43	[MOVI  D: X1  I: 0x43]
c0 currEL:0 2019764  sq:1472203    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:0          e:0          ed:0          w:0          ep:4294967295 rid:52         r:358328     fbid:152986 T:0 0000000000400938: 5280083b    mov w27, #0x41	[MOVI  D: X27  I: 0x41]
c0 currEL:0 2019766  sq:1472204    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:0          e:0          ed:0          w:0          ep:4294967295 rid:53         r:358328     fbid:152986 T:0 000000000040093C: 2a1b03e0    mov w0, w27	[MOV  D: X0  S: X27]
c0 currEL:0 2019768  sq:1472205    f:358302     b:358293     d1:358307     d2:358308     a:358309     q:358310     e:358317     ed:0          w:358318     ep:23         rid:53         r:358328     fbid:152986 T:0 0000000000400940: 9400015c    bl 400eb0	[BL  D: X30  I: 0x400eb0] [TAKEN]
c0 currEL:0 2019770  sq:1472206    f:358303     b:358293     d1:358307     d2:358308     a:358309     q:358310     e:358314     ed:0          w:358315     ep:18         rid:54         r:358328     fbid:152987 T:0 0000000000400EB0: 12001c02    and w2, w0, #0xff	[LOPI  D: X2  S: X0  I: 0xff]
c0 currEL:0 2019772  sq:1472207    f:358303     b:358293     d1:358308     d2:358309     a:358310     q:0          e:0          ed:0          w:0          ep:4294967295 rid:54         r:358328     fbid:152987 T:0 0000000000400EB4: 52800000    mov w0, #0x0	[MOVI  D: X0  I: 0x0]
c0 currEL:0 2019774  sq:1472208    f:358303     b:358293     d1:358308     d2:358309     a:358310     q:358311     e:358315     ed:0          w:358316     ep:19         rid:55         r:358328     fbid:152987 T:0 0000000000400EB8: 6b21005f    subs wzr, w2, w1, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X2, X1  I: 0x0, 0x0]
c0 currEL:0 2019776  sq:1472209    f:358303     b:358293     d1:358308     d2:358309     a:358310     q:358311     e:358315     ed:0          w:358316     ep:19         rid:55         r:358328     fbid:152987 T:0 0000000000400EBC: 54000040    b.eq 400ec4	[BCOND  S: PS_NZCV  I: 0x400ec4, 0x0]
c0 currEL:0 2019778  sq:1472210    f:358303     b:358293     d1:358308     d2:358309     a:358310     q:358311     e:358318     ed:0          w:358319     ep:23         rid:56         r:358328     fbid:152987 T:0 0000000000400EC0: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019780  sq:1472211    f:358304     b:358294     d1:358308     d2:358309     a:358310     q:358311     e:358315     ed:358318     w:358319     ep:5          rid:57         r:358329     fbid:152988 T:0 0000000000400944: b9405fe1    ldr w1, [sp, #92]	[LDR  D: X1  S: SP_EL3  I: 0x5c, 0x0] [vlid:191955] 
c0 currEL:0 2019782  sq:1472212    f:358304     b:358294     d1:358308     d2:358309     a:358310     q:358311     e:358319     ed:0          w:358320     ep:21         rid:57         r:358329     fbid:152988 T:0 0000000000400948: 6b01001f    subs wzr, w0, w1	[ADD  D: SINK, PS_NZCV  S: X0, X1  I: 0x0, 0x4]
c0 currEL:0 2019784  sq:1472213    f:358304     b:358294     d1:358308     d2:358309     a:358310     q:358311     e:358319     ed:0          w:358320     ep:21         rid:58         r:358329     fbid:152988 T:0 000000000040094C: 54000180    b.eq 40097c	[BCOND  S: PS_NZCV  I: 0x40097c, 0x0]
c0 currEL:0 2019786  sq:1472214    f:358304     b:358294     d1:358308     d2:358309     a:358310     q:358311     e:358317     ed:358320     w:358321     ep:6          rid:59         r:358329     fbid:152988 T:0 0000000000400950: 3940c261    ldrb x1, [x19, #48]	[LDR  D: X1  S: X19  I: 0x30, 0x0] [vlid:191956] 
c0 currEL:0 2019788  sq:1472215    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:358312     e:358317     ed:0          w:358318     ep:22         rid:59         r:358329     fbid:152988 T:0 0000000000400954: 11000760    add w0, w27, #0x1	[ADDI  D: X0  S: X27  I: 0x1]
c0 currEL:0 2019790  sq:1472216    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:358312     e:358318     ed:0          w:358319     ep:19         rid:60         r:358329     fbid:152988 T:0 0000000000400958: 12001c1b    and w27, w0, #0xff	[LOPI  D: X27  S: X0  I: 0xff]
c0 currEL:0 2019792  sq:1472217    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:358312     e:358321     ed:0          w:358322     ep:21         rid:60         r:358329     fbid:152988 T:0 000000000040095C: 6b20003f    subs wzr, w1, w0, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X1, X0  I: 0x0, 0x0]
c0 currEL:0 2019794  sq:1472218    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:358312     e:358321     ed:0          w:358322     ep:21         rid:61         r:358329     fbid:152988 T:0 0000000000400960: 54fffa63    b.cc 4008ac	[BCOND  S: PS_NZCV  I: 0x4008ac, 0x3]
c0 currEL:0 2019796  sq:1472219    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:0          e:0          ed:0          w:0          ep:4294967295 rid:62         r:358329     fbid:152988 T:0 0000000000400964: 52800861    mov w1, #0x43	[MOVI  D: X1  I: 0x43]
c0 currEL:0 2019798  sq:1472220    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:0          e:0          ed:0          w:0          ep:4294967295 rid:62         r:358329     fbid:152988 T:0 0000000000400968: 2a1b03e0    mov w0, w27	[MOV  D: X0  S: X27]
c0 currEL:0 2019800  sq:1472221    f:358304     b:358294     d1:358309     d2:358310     a:358311     q:358312     e:358320     ed:0          w:358321     ep:23         rid:63         r:358329     fbid:152988 T:0 000000000040096C: 94000151    bl 400eb0	[BL  D: X30  I: 0x400eb0] [TAKEN]
c0 currEL:0 2019802  sq:1472222    f:358305     b:358294     d1:358309     d2:358310     a:358311     q:358312     e:358319     ed:0          w:358320     ep:19         rid:64         r:358329     fbid:152989 T:0 0000000000400EB0: 12001c02    and w2, w0, #0xff	[LOPI  D: X2  S: X0  I: 0xff]
c0 currEL:0 2019804  sq:1472223    f:358305     b:358294     d1:358310     d2:358311     a:358312     q:0          e:0          ed:0          w:0          ep:4294967295 rid:64         r:358329     fbid:152989 T:0 0000000000400EB4: 52800000    mov w0, #0x0	[MOVI  D: X0  I: 0x0]
c0 currEL:0 2019806  sq:1472224    f:358305     b:358294     d1:358310     d2:358311     a:358312     q:358313     e:358320     ed:0          w:358321     ep:19         rid:65         r:358330     fbid:152989 T:0 0000000000400EB8: 6b21005f    subs wzr, w2, w1, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X2, X1  I: 0x0, 0x0]
c0 currEL:0 2019808  sq:1472225    f:358305     b:358294     d1:358310     d2:358311     a:358312     q:358313     e:358320     ed:0          w:358321     ep:19         rid:65         r:358330     fbid:152989 T:0 0000000000400EBC: 54000040    b.eq 400ec4	[BCOND  S: PS_NZCV  I: 0x400ec4, 0x0]
c0 currEL:0 2019810  sq:1472226    f:358305     b:358294     d1:358310     d2:358311     a:358312     q:358313     e:358321     ed:0          w:358322     ep:23         rid:66         r:358330     fbid:152989 T:0 0000000000400EC0: d65f03c0    ret	[RET  S: X30] [TAKEN]
c0 currEL:0 2019812  sq:1472227    f:358306     b:358295     d1:358310     d2:358311     a:358312     q:358313     e:358318     ed:358321     w:358322     ep:7          rid:67         r:358330     fbid:152990 T:0 0000000000400970: b9405fe1    ldr w1, [sp, #92]	[LDR  D: X1  S: SP_EL3  I: 0x5c, 0x0] [vlid:191957] 
c0 currEL:0 2019814  sq:1472228    f:358306     b:358295     d1:358310     d2:358311     a:358312     q:358313     e:358322     ed:0          w:358323     ep:21         rid:67         r:358330     fbid:152990 T:0 0000000000400974: 6b01001f    subs wzr, w0, w1	[ADD  D: SINK, PS_NZCV  S: X0, X1  I: 0x0, 0x4]
c0 currEL:0 2019816  sq:1472229    f:358306     b:358295     d1:358310     d2:358311     a:358312     q:358313     e:358322     ed:0          w:358323     ep:21         rid:68         r:358330     fbid:152990 T:0 0000000000400978: 54fffec1    b.ne 400950	[BCOND  S: PS_NZCV  I: 0x400950, 0x1] [TAKEN]
c0 currEL:0 2019818  sq:1472230    f:358307     b:358296     d1:358310     d2:358311     a:358312     q:358313     e:358317     ed:358320     w:358321     ep:5          rid:69         r:358330     fbid:152991 T:0 0000000000400950: 3940c261    ldrb x1, [x19, #48]	[LDR  D: X1  S: X19  I: 0x30, 0x0] [vlid:191958] 
c0 currEL:0 2019820  sq:1472231    f:358307     b:358296     d1:358311     d2:358312     a:358313     q:358314     e:358319     ed:0          w:358320     ep:20         rid:69         r:358330     fbid:152991 T:0 0000000000400954: 11000760    add w0, w27, #0x1	[ADDI  D: X0  S: X27  I: 0x1]
c0 currEL:0 2019822  sq:1472232    f:358307     b:358296     d1:358311     d2:358312     a:358313     q:358314     e:358320     ed:0          w:358321     ep:22         rid:70         r:358330     fbid:152991 T:0 0000000000400958: 12001c1b    and w27, w0, #0xff	[LOPI  D: X27  S: X0  I: 0xff]
c0 currEL:0 2019824  sq:1472233    f:358307     b:358296     d1:358311     d2:358312     a:358313     q:358314     e:358322     ed:0          w:358323     ep:19         rid:70         r:358330     fbid:152991 T:0 000000000040095C: 6b20003f    subs wzr, w1, w0, uxtb #0	[ADDE  D: SINK, PS_NZCV  S: X1, X0  I: 0x0, 0x0]
c0 currEL:0 2019826  sq:1472234    f:358307     b:358296     d1:358311     d2:358312     a:358313     q:358314     e:358322     ed:0          w:358323     ep:19         rid:71         r:358330     fbid:152991 T:0 0000000000400960: 54fffa63    b.cc 4008ac	[BCOND  S: PS_NZCV  I: 0x4008ac, 0x3] [TAKEN]

```


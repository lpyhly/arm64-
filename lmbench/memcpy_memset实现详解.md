[TOC]

#memcpy深度解读

## memcpy实测性能分析

### 1290 2.5GHz B503





## 源码解读

在glibc库中可以直接调用memcpy()函数进行操作，对于arm64， 源码在glibc-2.29/sysdeps/aarch64/memcpy.S。

主要是调用了ldp/stp指令，主体操作分为3种场景：

1. 小于16bytes的；
2. 小于96bytes的；
3. 大于96bytes的，会先对目的地址进行对齐操作，然后以 **64** byte为单位来loop。

show me the code：

``` c
add srcend, src, count // 确定结束地址
add dstend, dstin, count
    
cmp count, 16
b.ls    L(copy16)  // 小于16 byte的copy操作
cmp count, 96
b.hi    L(copy_long) // 大于 96 byte的copy操作
// 不符合这两个条件的，就是中间size的处理。
```

### 17 - 96 byte

首先分析中间size的操作， 17 - 96 byte。分为两大类：

1. 大于64 byte的：调用copy96

   操作为： 先copy前64byte，再copy最后的32byte。即先是[src, src + 63]  -> [dst, dst + 63]，然后是[srcend - 32, srcend - 1] -> [dstend - 32, dstend - 1]. 这样，对于大于64byte但小于96byte的字节，会重复复制中间几个byte，但是代码就会比较简单。

2. 小于64 byte，大于32byte

   操作为：先copy前32byte，再copy后32byte。

3. 小于32 byte，大于16byte

   操作为：先copy前16byte，再copy最后的16byte。

``` c
 /* Medium copies: 17..96 bytes.  */
 sub tmp1, count, 1
 ldp A_l, A_h, [src]		 // 首先从src复制16byte
 tbnz    tmp1, 6, L(copy96)  // 函数copy96是用来实现64-96 bytes的copy操作的。
 ldp D_l, D_h, [srcend, -16]
 tbz tmp1, 5, 1f
 ldp B_l, B_h, [src, 16]   // byte size: [32 - 64]
 ldp C_l, C_h, [srcend, -32]
 stp B_l, B_h, [dstin, 16]
 stp C_l, C_h, [dstend, -32]
1：
 stp A_l, A_h, [dstin]  // byte size: [16 - 32]
 stp D_l, D_h, [dstend, -16]
 ret
```

### 1 - 15 byte

然后分析小size，是小于16byte的。

与上面原理一致，先从前往后copy，再从后往前copy。

``` c
L(copy16): // 小于16byte的会走到这里（不包含等于）
    cmp count, 8  // 区分是否大于8 byte
    b.lo    1f
    ldr A_l, [src]      // 大于 8 byte的操作
    ldr A_h, [srcend, -8]
    str A_l, [dstin]
    str A_h, [dstend, -8]
    ret
    .p2align 4
1:
    tbz count, 2, 1f  // 若小于8byte，继续区分是否大于2 byte
    ldr A_lw, [src]
    ldr A_hw, [srcend, -4]
    str A_lw, [dstin]
    str A_hw, [dstend, -4]
    ret
 ...
```

### 大于96 byte

最后分析大size，大于96 byte的。

1. 首先处理目的不对齐的问题，让目的地址以16byte对齐。
2. 以64 byte为单位进行loop。

``` c
L(copy_long):
    and tmp1, dstin, 15
    bic dst, dstin, 15 // 位清零
    ldp D_l, D_h, [src]  // 将src的前16byte copy到dst，那么不对齐的部分一定全部被copy过去了。
    sub src, src, tmp1 // 目的地址不对齐的字节数，在源地址中对应删除。
    add count, count, tmp1  // 若dstin=0x3，则前13byte是不对齐的，第14byte一定是对齐的。
        					// 这里tmp1=3，为了补齐16byte，方便后面直接sub 16.
    ldp A_l, A_h, [src, 16] // 这时的src已经是修改后的了。连续ldp 64 byte 
    stp D_l, D_h, [dstin]
    ldp B_l, B_h, [src, 32]
    ldp C_l, C_h, [src, 48]
    ldp D_l, D_h, [src, 64]!
    subs    count, count, 128 + 16  // 16byte是补齐后的。
    b.ls    L(last64)
L(loop64):
    stp A_l, A_h, [dst, 16]
    ldp A_l, A_h, [src, 16]
    stp B_l, B_h, [dst, 32]
    ldp B_l, B_h, [src, 32]
    stp C_l, C_h, [dst, 48]
    ldp C_l, C_h, [src, 48]
    stp D_l, D_h, [dst, 64]!
    ldp D_l, D_h, [src, 64]!
    subs    count, count, 64
    b.hi    L(loop64)
        
     /* Write the last full set of 64 bytes.  The remainder is at most 64
        bytes, so it is safe to always copy 64 bytes from the end even if
        there is just 1 byte left.  */
 L(last64):							// 最后的字节处理，一定不超过64byte，从后往前处理。
     ldp E_l, E_h, [srcend, -64]
     stp A_l, A_h, [dst, 16]
     ldp A_l, A_h, [srcend, -48]
     stp B_l, B_h, [dst, 32]
     ldp B_l, B_h, [srcend, -32]
     stp C_l, C_h, [dst, 48]
     ldp C_l, C_h, [srcend, -16]
     stp D_l, D_h, [dst, 64]
     stp E_l, E_h, [dstend, -64]
     stp A_l, A_h, [dstend, -48]
     stp B_l, B_h, [dstend, -32]
     stp C_l, C_h, [dstend, -16]
     ret
 
```



## 理论性能分析

跑的是openEuler 21.03（glibc 2.29）版本，若在1620上则调用`__memcpy_falkor`函数，否则调用`__memcpy_generic`函数。

``` c
libc_ifunc (__libc_memcpy,
            (IS_THUNDERX (midr)
             ? __memcpy_thunderx
             : (IS_FALKOR (midr) || IS_PHECDA (midr) || IS_ARES (midr) || IS_KUNPENG920 (midr)
                ? __memcpy_falkor
                : (IS_THUNDERX2 (midr) || IS_THUNDERX2PA (midr)
                  ? __memcpy_thunderx2
                  : __memcpy_generic))));
```

其中，generic的完整memcpy代码如下：

``` c
ENTRY (MEMCPY)

        DELOUSE (0)
        DELOUSE (1)
        DELOUSE (2)

        prfm    PLDL1KEEP, [src]
        add     srcend, src, count
        add     dstend, dstin, count
        cmp     count, 32
        b.ls    L(copy32)
        cmp     count, 128
        b.hi    L(copy_long)

        /* Medium copies: 33..128 bytes.  */
        ldp     A_l, A_h, [src]
        ldp     B_l, B_h, [src, 16]
        ldp     C_l, C_h, [srcend, -32]
        ldp     D_l, D_h, [srcend, -16]
        cmp     count, 64
        b.hi    L(copy128)
        stp     A_l, A_h, [dstin]
        stp     B_l, B_h, [dstin, 16]
        stp     C_l, C_h, [dstend, -32]
        stp     D_l, D_h, [dstend, -16]
        ret

        .p2align 4
        /* Small copies: 0..32 bytes.  */
L(copy32):
        /* 16-32 bytes.  */
        cmp     count, 16
        b.lo    1f
        ldp     A_l, A_h, [src]
        ldp     B_l, B_h, [srcend, -16]
        stp     A_l, A_h, [dstin]
        stp     B_l, B_h, [dstend, -16]
        ret
        .p2align 4
1:
        /* 8-15 bytes.  */
        tbz     count, 3, 1f
        ldr     A_l, [src]
        ldr     A_h, [srcend, -8]
        str     A_l, [dstin]
        str     A_h, [dstend, -8]
        ret
        .p2align 4
1:
        /* 4-7 bytes.  */
        tbz     count, 2, 1f
        ldr     A_lw, [src]
        ldr     A_hw, [srcend, -4]
        str     A_lw, [dstin]
        str     A_hw, [dstend, -4]
        ret

        /* Copy 0..3 bytes.  Use a branchless sequence that copies the same
           byte 3 times if count==1, or the 2nd byte twice if count==2.  */
1:
        cbz     count, 2f
        lsr     tmp1, count, 1
        ldrb    A_lw, [src]
        ldrb    A_hw, [srcend, -1]
        ldrb    B_lw, [src, tmp1]
        strb    A_lw, [dstin]
        strb    B_lw, [dstin, tmp1]
        strb    A_hw, [dstend, -1]
2:      ret

        .p2align 4
        /* Copy 65..128 bytes.  Copy 64 bytes from the start and
           64 bytes from the end.  */
L(copy128):
        ldp     E_l, E_h, [src, 32]
        ldp     F_l, F_h, [src, 48]
        ldp     G_l, G_h, [srcend, -64]
        ldp     H_l, H_h, [srcend, -48]
        stp     A_l, A_h, [dstin]
        stp     B_l, B_h, [dstin, 16]
        stp     E_l, E_h, [dstin, 32]
        stp     F_l, F_h, [dstin, 48]
        stp     G_l, G_h, [dstend, -64]
        stp     H_l, H_h, [dstend, -48]
        stp     C_l, C_h, [dstend, -32]
        stp     D_l, D_h, [dstend, -16]
        ret

        /* Align DST to 16 byte alignment so that we don't cross cache line
           boundaries on both loads and stores.  There are at least 128 bytes
           to copy, so copy 16 bytes unaligned and then align.  The loop
           copies 64 bytes per iteration and prefetches one iteration ahead.  */

        .p2align 4
L(copy_long):
        and     tmp1, dstin, 15
        bic     dst, dstin, 15
        ldp     D_l, D_h, [src]
        sub     src, src, tmp1
        add     count, count, tmp1      /* Count is now 16 too large.  */
        ldp     A_l, A_h, [src, 16]
        stp     D_l, D_h, [dstin]
        ldp     B_l, B_h, [src, 32]
        ldp     C_l, C_h, [src, 48]
        ldp     D_l, D_h, [src, 64]!
        subs    count, count, 128 + 16  /* Test and readjust count.  */
        b.ls    L(last64)
L(loop64):
        stp     A_l, A_h, [dst, 16]
        ldp     A_l, A_h, [src, 16]
        stp     B_l, B_h, [dst, 32]
        ldp     B_l, B_h, [src, 32]
        stp     C_l, C_h, [dst, 48]
        ldp     C_l, C_h, [src, 48]
        stp     D_l, D_h, [dst, 64]!
        ldp     D_l, D_h, [src, 64]!
        subs    count, count, 64
        b.hi    L(loop64)

        /* Write the last full set of 64 bytes.  The remainder is at most 64
           bytes, so it is safe to always copy 64 bytes from the end even if
           there is just 1 byte left.  */
L(last64):
        ldp     E_l, E_h, [srcend, -64]
        stp     A_l, A_h, [dst, 16]
        ldp     A_l, A_h, [srcend, -48]
        stp     B_l, B_h, [dst, 32]
        ldp     B_l, B_h, [srcend, -32]
        stp     C_l, C_h, [dst, 48]
        ldp     C_l, C_h, [srcend, -16]
        stp     D_l, D_h, [dst, 64]
        stp     E_l, E_h, [dstend, -64]
        stp     A_l, A_h, [dstend, -48]
        stp     B_l, B_h, [dstend, -32]
        stp     C_l, C_h, [dstend, -16]
        ret

        .p2align 4
L(move_long):
        cbz     tmp1, 3f

        add     srcend, src, count
        add     dstend, dstin, count

        /* Align dstend to 16 byte alignment so that we don't cross cache line
           boundaries on both loads and stores.  There are at least 128 bytes
           to copy, so copy 16 bytes unaligned and then align.  The loop
           copies 64 bytes per iteration and prefetches one iteration ahead.  */

        and     tmp1, dstend, 15
        ldp     D_l, D_h, [srcend, -16]
        sub     srcend, srcend, tmp1
        sub     count, count, tmp1
        ldp     A_l, A_h, [srcend, -16]
        stp     D_l, D_h, [dstend, -16]
        ldp     B_l, B_h, [srcend, -32]
        ldp     C_l, C_h, [srcend, -48]
        ldp     D_l, D_h, [srcend, -64]!
        sub     dstend, dstend, tmp1
        subs    count, count, 128
        b.ls    2f

        nop
1:
        stp     A_l, A_h, [dstend, -16]
        ldp     A_l, A_h, [srcend, -16]
        stp     B_l, B_h, [dstend, -32]
        ldp     B_l, B_h, [srcend, -32]
        stp     C_l, C_h, [dstend, -48]
        ldp     C_l, C_h, [srcend, -48]
        stp     D_l, D_h, [dstend, -64]!
        ldp     D_l, D_h, [srcend, -64]!
        subs    count, count, 64
        b.hi    1b

        /* Write the last full set of 64 bytes.  The remainder is at most 64
           bytes, so it is safe to always copy 64 bytes from the start even if
           there is just 1 byte left.  */
2:
        ldp     G_l, G_h, [src, 48]
        stp     A_l, A_h, [dstend, -16]
        ldp     A_l, A_h, [src, 32]
        stp     B_l, B_h, [dstend, -32]
        ldp     B_l, B_h, [src, 16]
        stp     C_l, C_h, [dstend, -48]
        ldp     C_l, C_h, [src]
        stp     D_l, D_h, [dstend, -64]
        stp     G_l, G_h, [dstin, 48]
        stp     A_l, A_h, [dstin, 32]
        stp     B_l, B_h, [dstin, 16]
        stp     C_l, C_h, [dstin]
3:      ret

END (MEMCPY)
libc_hidden_builtin_def (MEMCPY)

```



### 总表

| size(B) | 理论时延（cycle） | 实测时延（cycle） | 备注                                    |
| ------- | ----------------- | ----------------- | --------------------------------------- |
| 0       | 9                 |                   | 框架 3c + 循环内 6c(已在scalesim上验证) |
| 1-3     | 8                 | 8（1B）           | 框架 3c + 循环内 5c(已在scalesim上验证) |
| 4-7     | 7                 |                   | 框架 3c + 循环内 4c(已在scalesim上验证) |
| 8-64    | 6                 | 6（64B）         | 已在scalesim/EMU上验证过                 |
| 65-128  | 7-8               | 7（128B）         | 已在scalesim上验证，软件不做对齐处理    |
| 129-144 | 8                 |                   |                                         |
| 145-208 | 9-10              |                   |                                         |
| 209-272 | 12-13             |                   |                                         |
| 273-336 | 15                |                   |                                         |
| 337-400 | 17-18             |                   |                                         |
| 401-464 | 20-21             |                   |                                         |
| 465-528 | 23                | 23.2  (512B)      | 已在scalesim上验证。 |
| 640 | 28 | 28 | 512B+64\*2 |
| 1k      | 43            | 43.2      | 512B+64\*8 |
| 2k      | 83                           | 83.3       | 512B+64\*24 |
| 4k      | 163                       | 163      | 512B+64\*56 |
| 4160    | 165.5                     | 167   |  4k+64B，开始跨4k边界了，              |
| 8k      | 323                             | 343      | 跨4k边界，有FE bound和bad specu了 |
| 8320    | 328                        | 347.5    |   8k+128B                             |
|         |                                |                                         ||

注： ldp会拆成 2 uops， stp 拆成 2 uops（sta+std），stp！拆成2uops（sta&addi+std）， 只有 ldp! 会拆成 3 uops。

### 框架性能分析

中间的调用流程为：

``` c
// C语言调用
		while (iterations-- > 0) {
                memcpy(dst,p,N);
        }
// 汇编
00000000004018f8 <loop_memcpy>:
...
   401914:   d1000413    sub x19, x0, #0x1
   401918:   b4000100    cbz x0, 401938 <loop_memcpy+0x40>
   40191c:   aa1403e2    mov x2, x20
   401920:   aa1603e1    mov x1, x22
   401924:   aa1503e0    mov x0, x21
   401928:   97fffdba    bl  401010 <memcpy@plt>
   40192c:   d1000673    sub x19, x19, #0x1
   401930:   b100067f    cmn x19, #0x1
   401934:   54ffff41    b.ne    40191c <loop_memcpy+0x24>  // b.any
       
0000000000401010 <memcpy@plt>:
  401010:   f00000f0    adrp    x16, 420000 <memcpy@GLIBC_2.17>
  401014:   f9400211    ldr x17, [x16]
  401018:   91000210    add x16, x16, #0x0
  40101c:   d61f0220    br  x17
    
// 对应的trace流： 从memcpy@plt返回后开始（scalesim仿真）
S1:
    sub x19, x19, #0x1
	cmn x19, #0x1    
	b.ne 40191c      // 除最后一次，均跳转
S2：        
	mov x2, x20      
	mov x1, x22      
	mov x0, x21      
	bl 401010       // 一定跳转
S3：
	adrp x16, 420000 
	ldr x17, [x16, #0]
	mov x16, x16     
	br x17      // 绝对跳转
```

因此，框架每次增加 3 cycle。

### 0B 性能分析

``` c
ENTRY_ALIGN (__memcpy_falkor, 6)
    	prfm    PLDL1KEEP, [src]
        cmp     count, 32
        add     srcend, src, count
        add     dstend, dstin, count
        b.ls    L(copy32)  // 小于32B的，走copy32函数
    ......
   
L(copy32):
        /* 16-32 bytes.  */
        cmp     count, 16 // 小于16B的，跳转到1f：
        b.lo    1f
	......
1:
        /* 8-15 */
        tbz     count, 3, 1f
1:
        /* 4-7 */
        tbz     count, 2, 1f
1:
        /* 0 */
        cbz     count, 2f
2:
        ret
```

性能分析：

在Hi1630V200上，从取指角度上看，上述流程需要 6 cycle（在Scalesim上仿真过），加上框架3c，总共9c，而且是每个loop稳定9c的。

``` c
s1: // 无依赖，可以全并发。 cmp+** + b.ls也是可以1拍完成的。
		prfm    PLDL1KEEP, [src]
		cmp     count, 32
        add     srcend, src, count
        add     dstend, dstin, count
        b.ls    L(copy32)
s2: // cmp + b.lo 可以一拍完成。这里是一定会跳转的，所以单拍内不能再执行后续操作；若不跳转，还可以在同拍内继续后续操作的取指。
		cmp     count, 16
        b.lo    1f
s3-s5： // 每个tbz都是会跳转的，所以每拍1个。
            tbz     count, 3, 1f
            tbz     count, 2, 1f
            cbz     count, 2f
s6： // ret 占1拍。
            ret
```



### 1-3B性能分析

``` c
ENTRY_ALIGN (__memcpy_falkor, 6)
    	prfm    PLDL1KEEP, [src]
    	cmp     count, 32            
        add     srcend, src, count
        add     dstend, dstin, count
        b.ls    L(copy32)  // 小于32B的，走copy32函数
    ......
   
L(copy32):
        /* 16-32 bytes.  */
        cmp     count, 16 // 小于16B的，跳转到1f：     //   S2
        b.lo    1f
	......
1:
        /* 8-15 */
        tbz     count, 3, 1f     //   S3
1:
        /* 4-7 */
        tbz     count, 2, 1f     //   S4
1:
        /* 0-3 */
        cbz     count, 1, 2f     //   S5 不跳转，直到ret指令均可以同拍取指（8发射）。
        lsr     tmp1, count, 1
        ldrb    A_lw, [src]         // 1B会重复copy 3遍； 2B会重复copy 2遍
        ldrb    A_hw, [srcend, -1]  
        ldrb    B_lw, [src, tmp1]
            // trace流为：
//          ldrb x6, [x1, #0]   
//			ldrb x7, [x4, #-1]  
//			ldrb x8, [x1, x14]  
        strb    A_lw, [dstin]       //   有依赖，但是依然可以同拍发出去？ 
        strb    B_lw, [dstin, tmp1]   // 3个strb到同一个地址，需要顺序完成
        strb    A_hw, [dstend, -1]     // 
2:      ret                      // 这个ret与前面一条strb是同一个rid，所以可以同拍取指

```

那么，前面4cycle与0B是一致的， 第5拍`cbz`判断不跳转，后面几条指令全部可以单拍发射出去。

从trace流看（r:xx是commit的cycle计数， fbid可以理解为取指的cycle计数）ldrb + strb虽然是有依赖的，但是可以同拍取指，在不同时候commit。

``` c
rid:223        r:37531      fbid:1987   sub x19, x19, #0x1
rid:223        r:37531      fbid:1987   cmn x19, #0x1    
rid:0          r:37531      fbid:1987   b.ne 40191c      
rid:1          r:37531      fbid:1988   mov x2, x20      
rid:1          r:37531      fbid:1988   mov x1, x22      
rid:2          r:37531      fbid:1988   mov x0, x21      
rid:2          r:37531      fbid:1988   bl 401010        
rid:3          r:37531      fbid:1989   adrp x16, 420000 
rid:4          r:37531      fbid:1989   ldr x17, [x16, #0]
rid:4          r:37531      fbid:1989   mov x16, x16     
rid:5          r:37532      fbid:1989   br x17   
rid:6          r:37532      fbid:1990   PREFETCH (imm offset)
rid:6          r:37532      fbid:1990   add x4, x1, x2   
rid:7          r:37532      fbid:1990   add x5, x0, x2   
rid:7          r:37532      fbid:1990   cmp x2, #0x20    
rid:8          r:37532      fbid:1990   b.ls 7fa2b86320  
rid:9          r:37532      fbid:1991   cmp x2, #0x10    
rid:9          r:37532      fbid:1991   b.cc 7fa2b86340  
rid:10         r:37532      fbid:1992   tbz w2, #3, 7fa2b8
rid:11         r:37533      fbid:1993   tbz w2, #2, 7fa2b8
rid:12         r:37533      fbid:1994   cbz x2, 7fa2b86398
rid:13         r:37533      fbid:1994   lsr x14, x2, #1  
rid:14         r:37535      fbid:1994   ldrb x6, [x1, #0]  // 两条ldrb同拍commit
rid:15         r:37535      fbid:1994   ldrb x7, [x4, #-1]
rid:16         r:37538      fbid:1994   ldrb x8, [x1, x14] // 需要x14的值，会滞后
rid:17         r:37538      fbid:1994   strb x6, [x0, #0]  // 第一条strb（重复了2次，按照STA、STD分开写的），可以和上一条ldrb同拍commit，因为没有依赖
rid:17         r:37538      fbid:1994   strb x6, [x0, #0]  
rid:18         r:37539      fbid:1994   strb x8, [x0, x14] // 两条strb同拍commit
rid:18         r:37539      fbid:1994   strb x8, [x0, x14]
rid:19         r:37539      fbid:1994   strb x7, [x5, #-1]
rid:19         r:37539      fbid:1994   strb x7, [x5, #-1]
rid:19         r:37539      fbid:1994   ret     
```

从trace流可以看到，全流程需要8cycle。 2、3 byte性能与1byte是一致的，代码路径也是相同的，都是8 cycle完成。

​	单个loop共29条指令（trace中有3个strb，每个strb占了2行，但inst只计算为1），ipc=3.59，那么单个loop耗时为8.07 cycle。

``` c
// 14cycle 情况
r:41775      fbid:6347 T:0 000000000040191C: aa1403e2    mov x2, x20      
r:41775      fbid:6347 T:0 0000000000401920: aa1603e1    mov x1, x22      
r:41775      fbid:6347 T:0 0000000000401924: aa1503e0    mov x0, x21      
r:41775      fbid:6347 T:0 0000000000401928: 97fffdba    bl 401010        
r:41775      fbid:6348 T:0 0000000000401010: f00000f0    adrp x16, 420000 
r:41779      fbid:6348 T:0 0000000000401014: f9400211    ldr x17, [x16, #0]
r:41779      fbid:6348 T:0 0000000000401018: 91000210    mov x16, x16     
r:41780      fbid:6348 T:0 000000000040101C: d61f0220    br x17   [BR  S: 
r:41782      fbid:6349 T:0 0000007F916862D0: f9800020    PREFETCH (imm offset)
r:41782      fbid:6349 T:0 0000007F916862D4: 8b020024    add x4, x1, x2   
r:41782      fbid:6349 T:0 0000007F916862D8: 8b020005    add x5, x0, x2   
r:41782      fbid:6349 T:0 0000007F916862DC: f100805f    cmp x2, #0x20    
// 8cycle 情况
r:41816      fbid:6379 T:0 000000000040191C: aa1403e2    mov x2, x20      
r:41816      fbid:6379 T:0 0000000000401920: aa1603e1    mov x1, x22      
r:41816      fbid:6379 T:0 0000000000401924: aa1503e0    mov x0, x21      
r:41816      fbid:6379 T:0 0000000000401928: 97fffdba    bl 401010        
r:41816      fbid:6380 T:0 0000000000401010: f00000f0    adrp x16, 420000 
r:41816      fbid:6380 T:0 0000000000401014: f9400211    ldr x17, [x16, #0]
r:41816      fbid:6380 T:0 0000000000401018: 91000210    mov x16, x16     
r:41817      fbid:6380 T:0 000000000040101C: d61f0220    br x17   [BR  S: 
r:41817      fbid:6381 T:0 0000007F916862D0: f9800020    PREFETCH (imm offset)
r:41817      fbid:6381 T:0 0000007F916862D4: 8b020024    add x4, x1, x2   
r:41817      fbid:6381 T:0 0000007F916862D8: 8b020005    add x5, x0, x2   
r:41817      fbid:6381 T:0 0000007F916862DC: f100805f    cmp x2, #0x20    
r:41817      fbid:6381 T:0 0000007F916862E0: 54000209    b.ls 7f91686320  
r:41817      fbid:6382 T:0 0000007F91686320: f100405f    cmp x2, #0x10    
r:41817      fbid:6382 T:0 0000007F91686324: 540000e3    b.cc 7f91686340  
r:41817      fbid:6383 T:0 0000007F91686340: 36180102    tbz w2, #3, 7f91686360           
```



``` c
=======================core0 PM STATISTIC=======================
IPC..................................................:    3.5912
TOTAL CYCLE..........................................:      4760
TOTAL INST...........................................:     17094
========================Top Down Metrics========================
Retiring.............................................:     44.89%
Front End Bound......................................:     53.57%
 |--Fetch Bandwidth Bound............................:     52.84%
   |--Small FB Write to Empty instQ..................:     48.12%
Back End Bound.......................................:      1.54%
 |--Resource Bound...................................:      0.00%
 |--Core Bound.......................................:     36.79%
   |--Divider........................................:      0.00%
   |--FSU Stall......................................:      0.00%
   |--Exe Ports Util.................................:     36.79%
     |--0_ports_serialize............................:      0.00%
     |--0_ports_non_serialize........................:      0.59%
     |--1_ports......................................:     33.00%
     |--2_ports......................................:     16.20%
     |--3_ports......................................:      8.89%
     |--4_ports......................................:     16.74%

```

#### 1B 波形分析

从commit角度分析：

1650ES波形：10cycle

![image-20240525105717834](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20240525105717834.png)

1630v200波形： 10 cycle

​	![image-20230530101711871](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230530101711871.png)

1290 TotemV6波形：10 cycle

![image-20230530101805840](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230530101805840.png)



```c
//1650ES
cycle 1:   401034,ffff_9603_1e80-1eb8   
cycle 2:   ffff_9603_1ed0
cycle 3:   ffff_9603_1ee8  
cycle 4:   ffff_9603_1f08
cycle 5:   NULL
cycle 6:   NULL
cycle 7:   401b30,401b34,401b44
cycle 8:   NULL
cycle 9:   NULL
cycle 10:  NULL
cycle 11:  同cycle 1.
// 对应代码
  401030:       f00000f0        adrp    x16, 420000 <memcpy@GLIBC_2.17>
  401034:       f9400211        ldr     x17, [x16]
  401038:       91000210        add     x16, x16, #0x0
  40103c:       d61f0220        br      x17

0000000000401b10 <loop_memcpy>:
........
  401b30:       b4000100        cbz     x0, 401b50 <loop_memcpy+0x40>
  401b34:       aa1403e2        mov     x2, x20
  401b38:       aa1603e1        mov     x1, x22
  401b3c:       aa1503e0        mov     x0, x21
  401b40:       97fffd3c        bl      401030 <memcpy@plt>
........

    
//1630v200
cycle 1: 	ffff_f7e8_6390,ffff_f7e8_6394, 40192c - 401014
cycle 2:	40101c;
cycle 3:	ffff_f7e8_62d0->6360;
cycle 4:	ffff_f7e8_6378/7c;
cycle 5:	NULL;
cycle 6:	NULL;
cycle 7:	ffff_f7e8_6380/84;
cycle 8:	NULL;
cycle 9:	NULL;
cycle 10:	ffff_f7e8_6388/8c;
cycle 11:	同cycle 1.

// 1290
cycle 1: 	ffff_f7e8_6390,ffff_f7e8_6394, 40194c - 401010
cycle 2:	401014;
cycle 3:	ffff_f7e8_62d0->6340;
cycle 4:	ffff_f7e8_6360;
cycle 5:	ffff_f7e8_6378/7c;
cycle 6:	NULL;
cycle 7:	NULL;
cycle 8:	ffff_f7e8_6380/84;
cycle 9:	NULL;
cycle 10:	ffff_f7e8_6388/8c;
cycle 11:	同cycle 1.
```

从IFU角度分析：

1630v200：

![image-20230530111533223](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230530111533223.png)





### 4-7B性能分析

``` c
ENTRY_ALIGN (__memcpy_falkor, 6)
    	prfm    PLDL1KEEP, [src]     //   S1
    	cmp     count, 32            
        add     srcend, src, count
        add     dstend, dstin, count
        b.ls    L(copy32)  // 小于32B的，走copy32函数
    ......
   
L(copy32):
        /* 16-32 bytes.  */
        cmp     count, 16 // 小于16B的，跳转到1f：     //   S2
        b.lo    1f
	......
1:
        /* 8-15 */
        tbz     count, 3, 1f     //   S3
1:
        /* 4-7 */
        tbz     count, 2, 1f     //   S4， 不跳转，一直到ret都可以同拍取指。
        ldr     A_lw, [src]     // ldr/str的时延可以被其他指令吃掉
        ldr     A_hw, [srcend, -4]
        str     A_lw, [dstin]
        str     A_hw, [dstend, -4]
        ret
```

综上，4-7 B性能为 4 cycle + 框架 3 cycle = 7c，ldr/str时延可以被前端取指全部吃掉。

单个loop共25条指令（trace中有2个str，每个str占了2行，但inst只计算为1），ipc=3.54，那么单个loop耗时为7.06 cycle。

``` shell
=======================core0 PM STATISTIC=======================
IPC..................................................:    3.5400
TOTAL CYCLE..........................................:      2911
TOTAL INST...........................................:     10305
Retiring.............................................:     44.25%
Front End Bound......................................:     55.80%
 |--Fetch Bandwidth Bound............................:     54.87%
   |--Small FB Write to Empty instQ..................:     54.83%

```



### 8-15B性能分析

``` c
ENTRY_ALIGN (__memcpy_falkor, 6)
    	PREFETCH (imm offset)     //   S1
    	cmp     count, 32            
        add     srcend, src, count
        add     dstend, dstin, count
        b.ls    L(copy32)  // 小于32B的，走copy32函数
    ......
   
L(copy32):
        /* 16-32 bytes.  */
        cmp     count, 16 // 小于16B的，跳转到1f：     //   S2
        b.lo    1f
	......
1:
        /* 8-15 */
        tbz     count, 3, 1f     //   S3,不跳转，一直到ret都可以同拍取指。
        ldr     A_l, [src]
        ldr     A_h, [srcend, -8]
        str     A_l, [dstin]
        str     A_h, [dstend, -8]
        ret
```

同上，   8-11B性能为 3 cycle + 框架 3 cycle = 6c，ldr/str时延可以被前端取指全部吃掉。从topdown可以看到，全部bound在前端，后端完全没有bound。

单次loop共24条指令（trace中有2个str，每个str占了2行，但inst只计算为1），ipc=3.97，算下来每个loop耗时6.04。 从trace中看到也是全部为6cycle完成，稳定的。

``` shell
IPC..................................................:    3.9747
TOTAL CYCLE..........................................:      1305
TOTAL INST...........................................:      5187

========================Top Down Metrics========================
Retiring.............................................:     49.68%
Front End Bound......................................:     50.36%
 |--Fetch Bandwidth Bound............................:     49.67%
   |--Small FB Write to Empty instQ..................:     49.68%
FE bound level 3 :
|--INTRA B2 FLUSH NUM................................:         1,    0.077%
|--INTRA B3 FLUSH NUM ...............................:       217,   16.744%
   |--INTRA B3 BR FLUSH NUM .........................:         1,    0.077%
   |--INTRA B3 IBR FLUSH NUM ........................:       216,   16.667%
   |--INTRA B5 IBR FLUSH NUM(FOR CORRELATION) .......:         0,    0.000%
   |--SC B3 FLUSH CNT(FOR CORRELATION)...............:         0,    0.000%
FE bound level 4 :
|--Bside Pipe #0.....................................:       866,   66.821%
|--Bside Pipe #1.....................................:       430,   33.179%
INSTQ EMPTY BUT DECODE VALID.........................:      1297
L1 RECEIVE SNOOP NUM.................................:         0
FETCH BLOCK NUM......................................:      1729
TOTAL INST...........................................:      5187
|--FETCH BLOCK NUM...................................:      1729
|--FETCH BLOCK RATE..................................:     33.33%
FILLQ FULL CNT.......................................:         0
32B FETCH BLOCK NUM..................................:      1728
|--32B FETCH BLOCK SIZE 1 or 2 INST RATIO............:       648,   37.500%
|--32B FETCH BLOCK SIZE 3 or 4 INST RATIO............:       864,   50.000%
|--32B FETCH BLOCK SIZE 5 or 6 INST RATIO............:       216,   12.500%
|--32B FETCH BLOCK SIZE 7 or 8 INST RATIO............:         0,    0.000%
|--32B FETCH BLOCK SIZE  >   8 INST RATIO............:         0,    0.000%
64B FETCH BLOCK NUM..................................:      1296
|--64B FETCH BLOCK SIZE 1  or 2  INST RATIO..........:       216,   16.667%
|--64B FETCH BLOCK SIZE 3  or 4  INST RATIO..........:       648,   50.000%
|--64B FETCH BLOCK SIZE 5  or 6  INST RATIO..........:       431,   33.256%
|--64B FETCH BLOCK SIZE 7  or 8  INST RATIO..........:         0,    0.000%
|--64B FETCH BLOCK SIZE 9  or 10 INST RATIO..........:         0,    0.000%
|--64B FETCH BLOCK SIZE 11 or 12 INST RATIO..........:         0,    0.000%
|--64B FETCH BLOCK SIZE 13 or 14 INST RATIO..........:         0,    0.000%
|--64B FETCH BLOCK SIZE 15 or 16 INST RATIO..........:         1,    0.077%
|--64B FETCH BLOCK SIZE < 8 INST RATIO...............:      1295,   99.923%
2TAKEN FETCH BLOCK NUM...............................:       430
|--2TAKEN FETCH BLOCK SIZE 1  or 2  INST RATIO.......:         0,    0.000%
|--2TAKEN FETCH BLOCK SIZE 3  or 4  INST RATIO.......:         0,    0.000%
|--2TAKEN FETCH BLOCK SIZE 5  or 6  INST RATIO.......:         0,    0.000%
|--2TAKEN FETCH BLOCK SIZE 7  or 8  INST RATIO.......:       429,   99.767%
|--AVERAGE 2TAKEN FETCH BLOCK SIZE...................:      7.52
64B FETCH BLOCK AVERAGE SIZE.........................:      4.01
TOTAL DECODE INST....................................:      5184
|--FETCH BLOCK CNT...................................:      1728
|--AVERAGE FETCH BLOCK SIZE..........................:      3.00

```



### 16-32B性能分析

从这里开始，topdown中有了后端bound了。在此之前，都是全部bound在前端。

``` c
ENTRY_ALIGN (__memcpy_falkor, 6)
    	prfm    PLDL1KEEP, [src]     //   S1
    	cmp     count, 32            
        add     srcend, src, count
        add     dstend, dstin, count
        b.ls    L(copy32)  // 小于等于32B的，走copy32函数(ls: lower or same)
    ......
   
L(copy32):
        /* 16-32 bytes.  */
        cmp     count, 16        //   S2， 不跳转
        b.lo    1f
        ldp     A_l, A_h, [src]            
        ldp     B_l, B_h, [srcend, -16]
        stp     A_l, A_h, [dstin]
        stp     B_l, B_h, [dstend, -16]
        ret
```

16-32B性能为为6c，与上面不同的是，ldp/stp时延无法被前端取指全部吃掉。前端取指： 2 cycle + 框架 3 cycle = 5c。但commit的时候， ldp/stp的时延没有被全部吃掉，所以用了6 cycle才全部完成。从前端`f:`阶段看，在br指令后面等了一拍，才开始下一轮的取指，这个bubble应该就是被lsu阻塞了。（为什么等？？？）



​	stp需要等ldp的数据写回到register file后才能继续做，中间间隔了4c（ldp和stpd之间）

``` c
// e: 即e1阶段，真正上到LSU的pipe了； w： e2阶段。
e:36001      w:36005    r:36008      fbid:1711   ldp x6, x7, [x1, #0]  
e:36001      w:36005    r:36008      fbid:1711   ldp x8, x9, [x4, #-16]
e:36001      w:36005    r:36009      fbid:1711   stp x6, x7, [x0, #0]   [STPA 
e:36005      w:36007    r:36009      fbid:1711   stp x6, x7, [x0, #0]   [STPD 
```



单次loop共23条指令（trace中有2个stp，每个stp占了2行，但inst只计算为1），ipc=3.8340，算下来每个loop耗时6.00。 从trace中看到也是全部为6cycle完成，稳定的。

``` c
// trace流：
r:36686	fbid:1789	sub x19, x19, #0x1   // C1： 一拍可以同时commit很多指令，还包含上面的指令。
r:36686	fbid:1789	cmn x19, #0x1
r:36686	fbid:1789	b.ne 40191c
r:36686	fbid:1790	mov x2, x20
r:36686	fbid:1790	mov x1, x22
r:36686	fbid:1790	mov x0, x21
r:36686	fbid:1790	bl 401010
r:36686	fbid:1791	adrp x16, 420000
r:36688	fbid:1791	ldr x17, [x16, #0]   // C3: 这条ldr需要间隔 2 cycle才commit。
    		// e1: 998,   004
r:36688	fbid:1791	mov x16, x16
r:36689	fbid:1791	br x17				 // C4: 跳转指令，依赖x17的值，隔了1cycle
r:36690	fbid:1792	PREFETCH (imm offset) // C1: 
r:36690	fbid:1792	add x4, x1, x2
r:36690	fbid:1792	add x5, x0, x2
r:36690	fbid:1792	cmp x2, #0x20
r:36690	fbid:1792	b.ls 7fb5386320
r:36690	fbid:1793	cmp x2, #0x10
r:36690	fbid:1793	b.cc 7fb5386340
r:36691	fbid:1793	ldp x6, x7, [x1, #0]  // C2: 两条ldp同时commit
    		// e1: 001
r:36691	fbid:1793	ldp x8, x9, [x4, #-16]
    		// e1: 001
r:36692	fbid:1793	stp x6, x7, [x0, #0]    // C3： 2条stp同时commit，还可以带着之后的一堆
r:36692	fbid:1793	stp x6, x7, [x0, #0]   
    		// e1: 005
r:36692	fbid:1793	stp x8, x9, [x5, #-16]
r:36692	fbid:1793	stp x8, x9, [x5, #-16]
    		// e1: 005
r:36692	fbid:1793	ret
    =========================================================
r:36692	fbid:1794	sub x19, x19, #0x1
r:36692	fbid:1794	cmn x19, #0x1
r:36692	fbid:1794	b.ne 40191c
r:36692	fbid:1795	mov x2, x20
r:36692	fbid:1795	mov x1, x22
r:36692	fbid:1795	mov x0, x21
r:36692	fbid:1795	bl 401010
r:36692	fbid:1796	adrp x16, 420000

```

```c
=======================core0 PM STATISTIC=======================
IPC..................................................:    3.8340
TOTAL CYCLE..........................................:     10446
TOTAL INST...........................................:     40050
========================Top Down Metrics========================
Retiring.............................................:     47.93%
Front End Bound......................................:     43.75%
 |--Fetch Latency Bound..............................:     16.67%
 |--Fetch Bandwidth Bound............................:     27.09%
   |--Small FB Write to Empty instQ..................:     25.00%   // 不准，先忽略。
Back End Bound.......................................:      8.32%
 |--Memory Bound.....................................:    100.00%
   |--L1 Bound.......................................:    100.00%
     |--Pipeline.....................................:    100.00%  // Load Cancel
```

### 33 - 64B性能分析

``` c
ENTRY (MEMCPY)
        prfm    PLDL1KEEP, [src]   // S1： 连续预测了12条。
        add     srcend, src, count
        add     dstend, dstin, count
        cmp     count, 32
        b.ls    L(copy32)    // 不跳转
        cmp     count, 128
        b.hi    L(copy_long)  // 不满足>128，不跳转

        /* Medium copies: 33..128 bytes.  */
        ldp     A_l, A_h, [src]   //前32B， 后32B，都进行ldp + stp，可以覆盖33-64B范围。
        ldp     B_l, B_h, [src, 16]     //  连续取指8条。
        ldp     C_l, C_h, [srcend, -32]
        ldp     D_l, D_h, [srcend, -16]
        cmp     count, 64
        b.hi    L(copy128)    // S2： 不满足>64， 不跳转。 
        stp     A_l, A_h, [dstin]
        stp     B_l, B_h, [dstin, 16]
        stp     C_l, C_h, [dstend, -32]
        stp     D_l, D_h, [dstend, -16]   
        ret
```

取指方面，相比16B少了一个跳转，但多了4条ldp/stp，综合来看取指时间是和16B一致的。scalesim上check，上述指令只需要2 cycle就可以取完，再加上框架的3c，只要5c就可以完成取指。

那么，就有更多的bound在后端了。  其中，4条ldp，4条stp 执行时间为：

- c1： 前三条ldp； c2： 后1条ldp+前两条stp； c3：后两条stp。

从commit角度看： 

c1:  prfm-> ldp之前的bhi，同拍commit；   // **为什么？**

c2：3条ldp

c3： ldp+cmp+stp+stp

c4:   stp+stp+ret+框架前到ldr之前的adrp指令（共13条uops，stp拆成2uops）

c5：框架中的ldr + mov 

c6：框架中的br

单次loop共29条指令（trace中有4个stp，每个stp占了2行，但inst只计算为1），ipc=4.83，算下来每个loop耗时6.00。 从trace中看到也基本都是6cycle完成。

``` c
=======================core0 PM STATISTIC=======================
IPC..................................................:    4.8334
TOTAL CYCLE..........................................:     14802
TOTAL INST...........................................:     71544
========================Top Down Metrics========================
Retiring.............................................:     60.42%
Front End Bound......................................:     20.83%
 |--Fetch Latency Bound..............................:     16.67%
 |--Fetch Bandwidth Bound............................:      4.17%
Back End Bound.......................................:     18.75%
 |--Resource Bound...................................:      0.00%
 |--Core Bound.......................................:     33.20%
   |--Divider........................................:      0.00%
   |--FSU Stall......................................:      0.00%
   |--Exe Ports Util.................................:     33.20%
     |--0_ports_serialize............................:      0.00%
     |--0_ports_non_serialize........................:     10.48%
     |--1_ports......................................:      6.11%
     |--2_ports......................................:      4.88%
     |--3_ports......................................:     25.56%
     |--4_ports......................................:     24.06%
     |--5_ports......................................:     28.91%
     |--6_ports......................................:      0.00%
     |--7_ports......................................:      0.00%
     |--8_ports......................................:      0.00%
 |--Memory Bound.....................................:     66.80%
   |--L1 Bound.......................................:     66.80%
     |--Pipeline.....................................:     66.80%

```



### 65-128B性能分析

没有进行对齐处理，src和dst都保持原始地址pattern，即非对齐场景下则一直做非对齐。

``` c
ENTRY (MEMCPY)
        prfm    PLDL1KEEP, [src]      //S1： 取指
        add     srcend, src, count
        add     dstend, dstin, count
        cmp     count, 32
        b.ls    L(copy32)    // 不跳转
        cmp     count, 128
        b.hi    L(copy_long)  // 不满足>128，不跳转

        /* Medium copies: 33..128 bytes.  */
        ldp     A_l, A_h, [src]   //前32B， 后32B，都进行ldp + stp，可以覆盖33-64B范围。
        ldp     B_l, B_h, [src, 16]
        ldp     C_l, C_h, [srcend, -32]
        ldp     D_l, D_h, [srcend, -16]
        cmp     count, 64
        b.hi    L(copy128)    // >64， 跳转
        ......
L(copy128):
        ldp     E_l, E_h, [src, 32]     // S2: 取指
        ldp     F_l, F_h, [src, 48]
        ldp     G_l, G_h, [srcend, -64]
        ldp     H_l, H_h, [srcend, -48]
        stp     A_l, A_h, [dstin]
        stp     B_l, B_h, [dstin, 16]
        stp     E_l, E_h, [dstin, 32]
        stp     F_l, F_h, [dstin, 48]
        stp     G_l, G_h, [dstend, -64]
        stp     H_l, H_h, [dstend, -48]
        stp     C_l, C_h, [dstend, -32]
        stp     D_l, D_h, [dstend, -16]
        ret
```

性能上分析，加上框架，每个loop共37条指令（trace中有8个stp，每个stp占了2行，但inst只计算为1），IPC=5.44，则单个loop用时6.8 cycle。在scalesim上实际运行也可以看到，不同loop间commit的用时分别为：6,9,7,6,7,5,8,7,5,8 cycle。还是和流水线前后关系相关。  **？？？**(是否也和cancel有关)

从LSU角度看：

- c1:  ldp*3
- c2:  ldp+cmp+b.hi+copy128中的ldp*2
- c3:  ldp\*2 + stp\*2
- c4: stp*2
- c5: stp*2
- c6: stp*2 + 框架中的ldr

TOPDOWN分析：已经完全没有FE，全部bound在backend上了。

``` shell
=======================core0 PM STATISTIC=======================
IPC..................................................:    5.4475
TOTAL CYCLE..........................................:      8444
TOTAL INST...........................................:     45999
========================Top Down Metrics========================
Retiring.............................................:     68.09%
Front End Bound......................................:      0.61%
Bad Speculation......................................:      0.00%
Back End Bound.......................................:     31.30%
 |--Core Bound.......................................:     64.85%
   |--Divider........................................:      0.00%
   |--FSU Stall......................................:      0.00%
   |--Exe Ports Util.................................:     64.85%
     |--0_ports_serialize............................:      0.00%
     |--0_ports_non_serialize........................:      0.31%
     |--1_ports......................................:      0.11%
     |--2_ports......................................:     10.79%
     |--3_ports......................................:     13.43%
     |--4_ports......................................:     36.05%
     |--5_ports......................................:     23.01%
     |--6_ports......................................:     10.62%
     |--7_ports......................................:      3.67%
     |--8_ports......................................:      1.98%
 |--Memory Bound.....................................:     20.82%
   |--L1 Bound.......................................:     20.82%
     |--Pipeline.....................................:     20.82%
```

#### 128B 波形分析

1650ES： 8cycle	![image-20240615170646869](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20240615170646869.png)

```c
C1: ffffXXX   + 401b34/b44, 401030
C2: Null
C3: Null
C4: 401034, ffffXXXXXXXX
C5-C8: ffffXXXXXXX

```

1630V200： 7cycle

![image-20240617154618625](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20240617154618625.png)

```c
C1: ffffXXXXXXX +  40191c/24/2c/34
C2: 401010/14
C3: 40101c+ffffXXXXX
C4-C7: ffffXXXXXXXX
```



### 大于128B性能分析

​	非对齐场景，首先对src/dst地址做前向补齐，count计数也相应增加。

- 非对齐场景下：

  - src一直是非对齐的。
  - 优先保障dst对齐，但前16B，以及尾部64B的stp，都不做对齐处理，按实际地址去stp。即dst首尾地址都不对齐时，最多存在5条非对齐的stp指令。
  - dst的其余size，一定是对齐的。

- 对于前向补齐后的count，若>144B，则会进入到loop64去循环。 到了loop64后，dst地址一定是对齐的。

- 为了保障性能，会优先ldp前64B，因此在loop64中ldp和stp的地址不是同一个64B地址块的，没有地址依赖。为了少用寄存器，先stp，然后释放出来一对寄存器，ldp立即使用。但因为地址是不同的，因此rename阶段可以直接解依赖。

  

``` c
ENTRY (MEMCPY)
        prfm    PLDL1KEEP, [src]
        add     srcend, src, count
        add     dstend, dstin, count
        cmp     count, 32 
        b.ls    L(copy32) // 不跳转
        cmp     count, 128
        b.hi    L(copy_long)  // >128，跳转
L(copy_long):
        and     tmp1, dstin, 15
        bic     dst, dstin, 15    // 位清零
        ldp     D_l, D_h, [src] // 将src的前16B copy到dst，那么不对齐的部分一定全部被copy过去了。
        sub     src, src, tmp1 // 目的地址不对齐的字节数，在源地址中对应删除。
        add     count, count, tmp1  // 若dstin=0x3，则前13byte是不对齐的，第14byte一定是对齐的。
        					// 这里tmp1=3，为了补齐16byte，方便后面直接sub 16.
        ldp A_l, A_h, [src, 16] // 这时的src已经是修改后的了。连续ldp 64 byte，这里多load了tmp1 B
        stp     D_l, D_h, [dstin]
        ldp     B_l, B_h, [src, 32]
        ldp     C_l, C_h, [src, 48]
        ldp     D_l, D_h, [src, 64]! // 此时src已经更新。
        subs    count, count, 128 + 16  // 这里减多了，看stp：16B是对应非对齐的已经ldp/stp过了。 
            				  // 剩余的对齐size，都在loop64 + last64中去stp，
            				  // 这里率先减去了last64中要stp的128B， 以及copy_long中stp的16B
                              // 至此，ldp的size数为（64,64+16]，对齐场景实际ldp了80B。
        b.ls    L(last64)     // 前向补齐后，若count <= 144B，则跳转，只需要copy最后64B即可。
L(loop64):
        stp     A_l, A_h, [dst, 16]
        ldp     A_l, A_h, [src, 16]
        stp     B_l, B_h, [dst, 32]
        ldp     B_l, B_h, [src, 32]
        stp     C_l, C_h, [dst, 48]
        ldp     C_l, C_h, [src, 48]
        stp     D_l, D_h, [dst, 64]!
        ldp     D_l, D_h, [src, 64]!
        subs    count, count, 64
        b.hi    L(loop64)

        /* Write the last full set of 64 bytes.  The remainder is at most 64
           bytes, so it is safe to always copy 64 bytes from the end even if
           there is just 1 byte left.  */
L(last64):									// copy最后的
        ldp     E_l, E_h, [srcend, -64]		// 最后64B，按src地址从后往前找64B。
        stp     A_l, A_h, [dst, 16]			// 上一轮的64B先stp下来。
        ldp     A_l, A_h, [srcend, -48]
        stp     B_l, B_h, [dst, 32]
        ldp     B_l, B_h, [srcend, -32]
        stp     C_l, C_h, [dst, 48]
        ldp     C_l, C_h, [srcend, -16]
        stp     D_l, D_h, [dst, 64]
        stp     E_l, E_h, [dstend, -64]		// 最后64B， 按dst地址从后往前stp
        stp     A_l, A_h, [dstend, -48]     //    若尾部非对齐，那这64B全部都是非对齐stp指令。
        stp     B_l, B_h, [dstend, -32]
        stp     C_l, C_h, [dstend, -16]
        ret
```

#### loop次数与size对照表

```c
// 计算公式：
// C: copy的size数， B： dst偏移量
loop64次数n = 1 + int((C+B-144)/64)  // 这里为截断取整
对固定n，  C+B ∈ (64n+80, 64n+144]
// 说明：
//    144 = 128 + 16， 即开头的16B，以及last64中的128B
//    C+B为新的count
//    loop64内每次循环会减去64B，然后判断是否大于0.

对应表：
    C+B   (128,144]  (144, 208] (208, 272]  ...  (464,528]  (528,592]  (592,656] ... 
  loop次数       0       1           2       ...      6          7          8     ...
```



#### 144B以下性能分析

性能上看，先看地址补齐后仍小于等于144 B的场景：

- 从ldp/stp角度看，ldp * 2 + stp + ldp * 3;
  - 前两条ldp在非对齐场景下是有地址重叠的， stp与第一条ldp是有数据依赖的。后面3条ldp与ldp/stp均无依赖。

- last64中：共4个ldp，以及8个stp。
  - 第一条ldp在非对齐场景下，与上一块中的最后一条ldp存在地址重叠； 最后一条stp在非对齐场景下，与第4条stp存在地址重叠。

从LSU角度看：

- c1:  ldp*2
- c2:  stp，ldp*3 (stp在e1晚了3拍，刚好可以晚1拍拿到ldp的data)
- c3:  ldp * 3，stp * 2
- c4:  ldp *1， stp * 2
- c5: stp * 2 （非对齐场景下，跨64B时stp会拆成2 uops，这里就只能stp * 1了）
- c6: stp * 2（非对齐场景下，跨64B时stp会拆成2 uops，这里就只能stp * 1了）

以140 B对齐地址（尾部不对齐）为例进行scalesim仿真：性能上分析，加上框架，每个loop共43条指令（trace中有9个stp，每个stp占了2行，但inst只计算为1；一个ldp x12,x13,[x1,#64]! 占了2行，但inst只计算为1），IPC=5.37，则单个loop用时8.00 cycle。

```shell
=======================core0 PM STATISTIC=======================
IPC..................................................:    5.3749
TOTAL CYCLE..........................................:      1139
TOTAL INST...........................................:      6122
========================Top Down Metrics========================
Retiring.............................................:     67.19%
Front End Bound......................................:      0.00%
Bad Speculation......................................:      0.00%
Back End Bound.......................................:     32.81%
|--Core Bound.......................................:     78.93%
  |--Exe Ports Util.................................:     78.93%
    |--0_ports_serialize............................:      0.00%
    |--0_ports_non_serialize........................:      0.00%
    |--1_ports......................................:      0.00%
    |--2_ports......................................:      0.97%
    |--3_ports......................................:     19.05%
    |--4_ports......................................:     35.29%
    |--5_ports......................................:     31.69%
    |--6_ports......................................:     12.03%
    |--7_ports......................................:      0.97%
    |--8_ports......................................:      0.00%
|--Memory Bound.....................................:     19.75%
  |--L1 Bound.......................................:     19.75%
    |--Pipeline.....................................:     19.75%
```

#### 144B以上性能分析

再看大于144B size的场景。则需要加入loop64的性能分析。以512B size为例，分析issueQ -> LSU/ALU的时间：

- c1:  ldp*2
- c2:  stp，ldp*3 (stp在e1晚了3拍，刚好可以晚1拍拿到ldp的data)
- c3:  ldp * 2/3（64B非对齐地址则只能\*2，其他情况可以\*3，但是受限于stp的地址依赖，后续只能ldp2条），stp * 2
- c4:  ldp *2/3， stp * 2
- 再跳到c3，共重复6次loop64，共16cycle完成。（详见下述分析）
- c18：ldp * 2, stp * 2
- c19:   ldp * 2, stp * 2 
- c20：stp * 2
- c21:   stp * 2
- c22：框架中的cmn, b.ne, bl, ldr 
- c23:  br, PREFETCH, add,add, cmp,b.ls, cmp

以512 B对齐地址（尾部不对齐）为例进行scalesim仿真：性能上分析，加上框架，每个loop共103条指令（trace中，普通stp占了2行，但inst只计算为1；一个ldp x12,x13,[x1,#64]! 占了2行，但inst只计算为1； 一个带！的stp占了3行，但inst只计算为1），IPC=4.47，则单个loop用时23.00 cycle。

512B非对齐地址，共103条指令，IPC=4.41，单个loop用时23.33 cycle。

``` c
// cycle计算
            ENTRY到loop64前        loop64      last64     框架     合计
// 512B（相同场景有：对齐的512B->527B, 非16B对齐，dst偏移B，则无论对齐还是非对齐。
实际inst数       19                   60          13        11     103
traced打印数     21                   96          21        11     149
// 144B以下（不进入loop64）
实际inst数       19                   0           13        11      43
traced打印数     21                   0           21        11      53                
// 4kB，对齐
实际inst数       19                   620         13        11     663
```

理论分析loop64循环体的性能：

​		5 cycle 完成2次 loop64，理论带宽为 64GB/s = 25.6 B/cycle.

``` c
L(loop64):
        stp     A_l, A_h, [dst, 16]   // S1:  2uops                        
        ldp     A_l, A_h, [src, 16]   //      2uops      S4: 2+2+2+2                  S9
        stp     B_l, B_h, [dst, 32]   //      2uops                      
        ldp     B_l, B_h, [src, 32]   //      2uops                      S7: 2+2+2+2
        stp     C_l, C_h, [dst, 48]   // S2:  2uops
        ldp     C_l, C_h, [src, 48]   //      2uops      S5: 2+2+3+1
        stp     D_l, D_h, [dst, 64]!  //      2uops                      
        ldp     D_l, D_h, [src, 64]!  // S3:  3uops                      S8: 3+1+1+2
        subs    count, count, 64      //      1uops       
        b.hi    L(loop64)             //      1uops      S6: 1+2+2+2
            						  // （预测带宽还够，所以可以继续取指）       

```



``` shell
# 512B对齐
=======================core0 PM STATISTIC=======================
IPC..................................................:    4.4769
TOTAL CYCLE..........................................:      2338
TOTAL INST...........................................:     10467
========================Top Down Metrics========================
Retiring.............................................:     55.96%
Front End Bound......................................:      0.00%
Back End Bound.......................................:     44.04%
 |--Resource Bound...................................:      0.00%
 |--Core Bound.......................................:     70.53%
   |--Exe Ports Util.................................:     70.53%
     |--0_ports_serialize............................:      0.00%
     |--0_ports_non_serialize........................:      0.00%
     |--1_ports......................................:      0.00%
     |--2_ports......................................:      5.13%
     |--3_ports......................................:     19.46%
     |--4_ports......................................:     29.47%
     |--5_ports......................................:     26.13%
     |--6_ports......................................:     16.77%
     |--7_ports......................................:      2.95%
     |--8_ports......................................:      0.09%
 |--Memory Bound.....................................:     17.28%
   |--L1 Bound.......................................:     17.28%
     |--Pipeline.....................................:     17.28%

# 512B非对齐， -B5
=======================core0 PM STATISTIC=======================
IPC..................................................:    4.4141
TOTAL CYCLE..........................................:       821
TOTAL INST...........................................:      3624    
========================Top Down Metrics========================
Retiring.............................................:     55.18%    
Back End Bound.......................................:     44.82%
 |--Core Bound.......................................:     72.47%
   |--Exe Ports Util.................................:     72.47%
     |--0_ports_serialize............................:      0.00%
     |--0_ports_non_serialize........................:      0.00%
     |--1_ports......................................:      0.49%
     |--2_ports......................................:      4.87%
     |--3_ports......................................:     17.66%
     |--4_ports......................................:     29.11%
     |--5_ports......................................:     27.53%
     |--6_ports......................................:     18.39%
     |--7_ports......................................:      1.95%
     |--8_ports......................................:      0.00%
 |--Memory Bound.....................................:     12.30%
   |--L1 Bound.......................................:     12.30
     |--Pipeline.....................................:     12.30%
```

 ===   ldr因为bank冲突等，可能会被cancel掉。(刷)

``` shell
 =====================Load Cancel break down=====================
 load cancel..........................................:       985
 load tlb miss cancel.................................:         0
 load lose tlb arb cancel.............................:         0
 load lose tag port cancel............................:       186
 |--load lose tag port conflict fill..................:         0
 load lose result pipe cancel.........................:       321
 load_lose_dc_data_arb_cancel.........................:       146
 |--force_cancel_by_scb...............................:       146
 load_dc_miss_cancel..................................:         0
 load bank conflict cancel............................:        77
 load bank first ma occupy............................:         0
```

#### 4k性能分析

单个loop的指令数共663条，在scalesim上运行时间为163 cycle， IPC=4.0680。数据和emu实测的刚好能对的上。 loop64循环了62次，每次5/2 cycle，则已经有155cycle了。

``` shell
 =======================core0 PM STATISTIC=======================
 IPC..................................................:    4.0680
 TOTAL CYCLE..........................................:      7158
 TOTAL INST...........................................:     29119
 ========================Top Down Metrics========================
 Retiring.............................................:     50.85%
  Back End Bound.......................................:    49.15%
  |--Resource Bound...................................:      0.00%
  |--Core Bound.......................................:     80.16%
    |--Exe Ports Util.................................:     80.16%
      |--0_ports_serialize............................:      0.00%
      |--0_ports_non_serialize........................:      0.00%
      |--1_ports......................................:      0.00%
      |--2_ports......................................:      5.91%
      |--3_ports......................................:     14.71%
      |--4_ports......................................:     33.33%
      |--5_ports......................................:     26.15%
      |--6_ports......................................:     19.47%
      |--7_ports......................................:      0.41%
      |--8_ports......................................:      0.01%
  |--Memory Bound.....................................:      7.63%
    |--L1 Bound.......................................:      7.63%
      |--Pipeline.....................................:      7.63%
```

#### 8k性能分析



``` shell
=======================core0 PM STATISTIC=======================
IPC..................................................:    3.6691
TOTAL CYCLE..........................................:      4449
TOTAL INST...........................................:     16324
========================Top Down Metrics========================
Retiring.............................................:     45.86%
Front End Bound......................................:      5.25%
 |--Fetch Latency Bound..............................:      4.56%
   |--SP Flush Bubble................................:      1.24%
   |--Bru Flush Bubble...............................:      2.32%
   |--B5 Flush Bubble................................:      0.00%
   |--B3 Flush Bubble................................:      0.70%
   |--B2 Flush Bubble................................:      0.00%
 |--Fetch Bandwidth Bound............................:      0.68%
   |--Small FB Write to Empty instQ..................:      0.34%
Bad Speculation......................................:      3.11%
 |--Branch Mispredicts...............................:      3.11%
   |--Other Branch...................................:      3.11%
 |--Machine Clears...................................:      0.00%
Back End Bound.......................................:     45.78%
 |--Resource Bound...................................:      0.00%
 |--Core Bound.......................................:     74.33%
   |--Exe Ports Util.................................:     74.33%
     |--0_ports_serialize............................:      0.00%
     |--0_ports_non_serialize........................:      6.23%
     |--1_ports......................................:      0.31%
     |--2_ports......................................:      6.11%
     |--3_ports......................................:     13.58%
     |--4_ports......................................:     31.78%
     |--5_ports......................................:     24.03%
     |--6_ports......................................:     17.96%
     |--7_ports......................................:      0.00%
     |--8_ports......................................:      0.00%
 |--Memory Bound.....................................:     10.97%
   |--L1 Bound.......................................:     10.97%
     |--Pipeline.....................................:     10.97%
```







## x86源码解读

### vmovdqu/vmovdqu64/vmovdqa指令

vmovdqu和vmovdqa指令集的区别在于对齐方式： vmovdqa表示地址是aligned对齐格式，vmovdqu表示是unaligned非对齐的。对齐的定义是：  16 (EVEX.128,VEX.128, SSE)/32(EVEX.256、VEX.256)/64(EVEX.512) 字节边界。

首先看intel中的size定义方式： word = 16 Byte

<img src="D:\at_work\Documents\我的总结文档\images\image-20220829202844816.png" alt="image-20220829202844816" style="zoom:50%;" />

然后，vmovdqu64就是一次性copy 64B（quadword）， vmovdqu32一次性copy 32B， vmovdqu16一次性copy 16B， vmovdqu8/vmovdqu一次性copy 8B（其中vmovdqu没有掩码）。

操作具体方式：     将 128、256 或 512 位的压缩字节/字/双字/四字整数值从源操作数（第二个操作数）移动到目标操作数（第一个操作数）。该指令可用于从内存位置加载向量寄存器，将向量寄存器的内容存储到内存位置，或在两个向量寄存器之间移动数据。

掩码：   目标操作数根据写掩码以 8 位 (VMOVDQU8)、16 位 (VMOVDQU16)、32 位 (VMOVDQU32) 或 64 位 (VMOVDQU64) 粒度更新。

### glibc如何选择当前环境使用哪个版本的memcpy

在"ifunc-memmove.h"文件中，会对cpu feature进行判断，然后选择不同的实现方式。判断cpu_feature的顺序为：（2022年glibc主线版本）

| 优先级 | cpu feature                                                  | 对应函数实现                             |
| ------ | ------------------------------------------------------------ | ---------------------------------------- |
| 1      | Prefer_ERMS/Prefer_FSRM                                      | _erms                                    |
| 2      | AVX512F & !Prefer_No_AVX512 & AVX512VL & ERMS                | avx512_unaligned_erms                    |
| 3      | AVX512F& !Prefer_No_AVX512 & AVX512VL & !ERMS                | avx512_unaligned                         |
| 4      | AVX512F& !Prefer_No_AVX512 & !AVX512VL                       | avx512_no_vzeroupper                     |
| 5      | AVX_Fast_Unaligned_Load & AVX512VL & ERMS                    | evex_unaligned_erms                      |
| 6      | AVX_Fast_Unaligned_Load & !AVX512VL & !ERMS                  | evex_unaligned                           |
| 7      | AVX_Fast_Unaligned_Load & !AVX512VL & RTM                    | avx_unaligned_erms_rtm/avx_unaligned_rtm |
| 8      | AVX_Fast_Unaligned_Load & !AVX512VL & Prefer_No_VZEROUPPER   | avx_unaligned_erms/avx_unaligned         |
| 9      | !AVX_Fast_Unaligned_Load & (!SSSE3 \|\| Fast_Unaligned_Copy) | sse2_unaligned_erms/sse2_unaligned       |
| 10     | SSSE3 & Fast_Copy_Backward                                   | ssse3_back                               |
| 11     | SSSE3 & !Fast_Copy_Backward                                  | ssse3                                    |

（openEuler 21.03- glibc 2.31版本）

|      | cpu feature                                                | 对应函数实现                           |
| ---- | ---------------------------------------------------------- | -------------------------------------- |
| 1    | Prefer_ERMS/Prefer_FSRM                                    | _erms                                  |
| 2    | AVX512F_Usable & !Prefer_No_AVX512 & Prefer_No_VZEROUPPER  | avx512_no_vzeroupper                   |
| 3    | AVX512F_Usable & !Prefer_No_AVX512 & !Prefer_No_VZEROUPPER | avx512_unaligned_erms/avx512_unaligned |
| 4    | AVX_Fast_Unaligned_Load                                    | avx_unaligned_erms/avx_unaligned       |
| 5    | !SSSE3 \|\| Fast_Unaligned_Copy                            | sse2_unaligned_erms/sse2_unaligned     |
| 6    | Fast_Copy_Backward                                         | ssse3_back                             |
| 7    | others                                                     | ssse3                                  |

openEuler21.03的glibc2.31默认场景下，为`+Prefer_No_AVX512 ` & `+AVX_Fast_Unaligned_Load` & `+ERMS`：

- 选择prefer_no_avx512的原因：当AVX512ER特性未使能时，默认会令cpu_feature的prefer_no_avx512使能，是为了防止AVX512ER不使能时cpu频率过低。

  ``` c
  "sysdeps/x86/cpu-features.c"
      /* Since AVX512ER is unique to Xeon Phi, set Prefer_No_VZEROUPPER
           if AVX512ER is available.  Don't use AVX512 to avoid lower CPU
           frequency if AVX512ER isn't available.  */
  ```

- openEuler22.03的glibc2.34默认场景下，则使能了AVX512功能 &  `+ERMS`，看代码应该是改为判断`AVX_VNNI`特性是否使能，而6248已经使能了该特性。

  ``` c
  /* Processors with AVX512 and AVX-VNNI won't lower CPU frequency
     when ZMM load and store instructions are used.  */
  if (!CPU_FEATURES_CPU_P (cpu_features, AVX_VNNI))
    cpu_features->preferred[index_arch_Prefer_No_AVX512]
      |= bit_arch_Prefer_No_AVX512;
  ```

  



可以人工干预cpu feature，方法是使用`GLIBC_TUNABLES=glibc.cpu.hwcaps=`宏来对不想使能的特性进行屏蔽（-表示去掉）：

```c
// 去掉Prefer_No_AVX512, AVX2_Usable, AVX_Fast_Unaligned_Load，此时顺位为使用AVX512来实现memcpy。
GLIBC_TUNABLES=glibc.cpu.hwcaps=-AVX2_Usable,-AVX_Fast_Unaligned_Load,-Prefer_No_AVX512 ./simple_memcpy -S 16m -l 9999999
// 不使用rep movsb指令。
GLIBC_TUNABLES=glibc.cpu.hwcaps=-ERMS ./simple_memcpy -S 16m -l 909999    
```



### x86 memcpy整体实现思路

1. 使用重叠的load/store来避免分支。
2. 将所有src都load进寄存器后再store，从而避免overlap
3. 若dst>src，则后向拷贝，每次4 \* VEC_SIZE，load不对齐，store对齐（主loop）。loop循环前先load最初的4*vec和最后的vec，然后loop循环后再store它们。（其余的均可以在loop内完成）
4. 若dst<=src，则前向拷贝，同样是load不对齐，store对齐。loop循环前先load最后的4*vec和最初的vec，然后loop循环后再store它们。
5. **支持ERM特性的硬件中，若size>=\_\_x86_rep_movsb_threshold且size<\_\_x86_rep_movsb_stop_threshold，则使用REP MOVSB指令。**
6. 若size>=__x86_shared_non_temporal_threshold，且src和dst无overlap，则使用non-temporal的store替代对齐的sotre，每次copy 2或4个page。
7. 若size<16 * __x86_shared_non_temporal_threshold且src和dst为page对齐，则使用non-temp store每次拷贝2 page；若src_page_alignment – dst_page_alignment < 8*VEC_SIZE，也认为是非page对齐。
8. 若size >= 16 * __x86_shared_non_temporal_threshold，则使用non-temp store每次拷贝4 page



*注： 6248上check， VEC_SIZE = 32 Byte*

### AVX512实现

AVX512包含多个指令子集，比如：

- AVX512-F： 基本指令集，含向量化算术运算、比较、类型转换、数据移动、数据排列、向量和掩码的按位逻辑运算，以及诸如 min/max之类的其他数学函数。
- AVX512-CD： 冲突检测指令集。
- AVX512-PF： 预取指令集

  

### avx_unaligned_erms实现

调用的函数是：`__memmove_avx_unaligned_erms`，但是是否使用`rep movsb`指令，还需要看memcpy的具体size。：

在"sysdeps/x86_64/multiarch/memmove-vec-unaligned-erms.S"文件中，找到`ENTRY (MEMMOVE_SYMBOL (__memmove, unaligned_erms))`入口函数。其中，erms特性表示硬件支持movsb指令。

实现思路：

``` c
#define REP_MOVSB_THRESHOLD    (2048 * (VEC_SIZE / 16))

/* The large memcpy micro benchmark in glibc shows that 6 times of
     shared cache size is the approximate value above which non-temporal
     store becomes faster on a 8-core processor.  This is the 3/4 of the
     total shared cache size.  */
long int __x86_shared_cache_size attribute_hidden = 1024 * 1024;
__x86_shared_non_temporal_threshold
    = (cpu_features->non_temporal_threshold != 0
       ? cpu_features->non_temporal_threshold
       : __x86_shared_cache_size * threads * 3 / 4); // threads为1P的逻辑核数

// 备注： 在6248上check， VEC_SIZE = 32 Byte，那么
// VEC_SIZE = 32 Byte
// REP_MOVSB_THRESHOLD = 4k
// __x86_shared_non_temporal_threshold = 35-36m之间（实测出来的）  when SMT=off
```



	memmove_unaligned_erms -> 
	
	start_erms: 入口函数
	1. size若< VEC_SIZE，则进入less_vec函数；
	2. 若大于VEC_SIZE * 2， 则进入movsb_more_2x_vec函数。
	3. 若处于两者之间，直接用vmovu指令完成memcpy动作。
	
	movsb_more_2x_vec:
	1. 若超过REP_MOVSB_THRESHOLD，则跳到movsb函数；
	2. 若不超过，则去到more_2x_vec函数执行。(此分支不会使用rep movsb指令)
	
	movsb：
	1. 判断size是否超过__x86_shared_non_temporal_threshold，若超过则进入more_8x_vec
	2. 若不超过，再判断是否为forward，若是则直接调用rep movsb指令
	3. 若为backward，则进入more_8x_vec_backward函数。
	
	movsb_backward:
	1. 使用指令组合std + rep movsb + cld来进行backward类型的movsb操作。
	
	================== 以下为非rep movsb实现部分 ==============
	more_2x_vec：
	1. 若大于VEC_SIZE * 8，则进入more_8x_vec；
	2. 若小于VEC_SIZE * 4，则进入last_4x_vec;
	3. 处于两者之间，直接使用vmovu指令完成memcpy动作。
	
	more_8x_vec：
	1. 判断是否为backward，若是则进入more_8x_vec_backward
	2. 判断size是否超过__x86_shared_non_temporal_threshold， 若超过则跳到large_forward.
	3. 若不超过且为forward，则在loop_4x_vec_forward函数中使用vmovu/vmova指令进行主循环的memcpy动作。
	
	more_8x_vec_backward：
	1.  判断size是否超过__x86_shared_non_temporal_threshold， 若超过则跳到back_forward.
	2. 若不超过，则在loop_4x_vec_backward函数中使用vmovu/vmova指令进行主循环的memcpy动作。
	
	large_forward:
	1. 判断是否存在overlap，若存在则返回去使用loop_4x_vec_forward作为主函数；
	2. 若不存在overlap，则通过PREFETCHED + VMOVU + VMOVNT指令来进行主循环操作。
	
	large_backward: 同large_forward


​	
### avx_unaligned实现

 ``` c
// 源码文件： sysdeps/x86_64/multiarch/memmove-vec-unaligned-erms.S
__memmove_avx_unaligned的函数入口，进入到start函数

start:
1. size若< VEC_SIZE，则进入less_vec函数；
2. 若大于VEC_SIZE * 2， 则进入more_2x_vec函数。
3. 若处于两者之间，直接用vmovu指令完成memcpy动作。

less_vec函数&more_2x_vec函数实现详见上一节。
 ```







``` c
ENTRY (MEMMOVE_SYMBOL (__memmove, unaligned_erms))
        movq    %rdi, %rax
L(start_erms):
        cmp     $VEC_SIZE, %RDX_LP   // VEC_SIZE = 32 Byte, in Intel 6248.
        jb      L(less_vec)     
        cmp     $(VEC_SIZE * 2), %RDX_LP
        ja      L(movsb_more_2x_vec)
L(last_2x_vec):
        /* From VEC and to 2 * VEC.  No branch when size == VEC_SIZE. */
        VMOVU   (%rsi), %VEC(0)
        VMOVU   -VEC_SIZE(%rsi,%rdx), %VEC(1)
        VMOVU   %VEC(0), (%rdi)
        VMOVU   %VEC(1), -VEC_SIZE(%rdi,%rdx)
L(return):
        VZEROUPPER
        ret

L(movsb):
        cmp     __x86_shared_non_temporal_threshold(%rip), %RDX_LP
        jae     L(more_8x_vec)
        cmpq    %rsi, %rdi
        jb      1f
        /* Source == destination is less common.  */
        je      L(nop)
        leaq    (%rsi,%rdx), %r9
        cmpq    %r9, %rdi
        /* Avoid slow backward REP MOVSB.  */
# if REP_MOVSB_THRESHOLD <= (VEC_SIZE * 8)
#  error Unsupported REP_MOVSB_THRESHOLD and VEC_SIZE!
# endif
        jb      L(more_8x_vec_backward)
1:
        mov     %RDX_LP, %RCX_LP
        rep movsb
L(nop):
        ret
#endif
```







## 带宽计算

对于L1 hit的case，一般瓶颈在LSU上。

- 以Hi1630V200为例，芯片设计上LSU有 3条Load pipe + 2条store pipe，其中load pipe位宽64B， store pipe位宽32B，因此在neon/sve场景下某些可以用满带宽的指令pattern上，load的最大带宽为64B*3/cycle， store的最大带宽为32B\*2/cycle。

- 对于memcy的case，使用的是ldp/stp指令，因此pipe无法用满，只能用到16B。最大位宽上来看，因为ldp/stp存在数据依赖，因此瓶颈在store pipe上，store的最大带宽为 16B * 2pipe = 32 B/cycle * 2.5 GHz = 80 GB/s，也就是memcpy的理论最大带宽了。
- 实际带宽，还要考虑loop之外的指令损耗。





## QA

1. **为什么选择16 byte作为第一个分界线？**

   ​	因为16byte就可以ldp操作了。

   

2. **为什么选择96 byte而不是64 byte作为分界线?**

   ​	需要处理dst的对齐问题，因为ldp的操作为16byte， 因此地址的头尾都需要有16byte来作为裕量。那么最小保证的长度就是16 + 64 + 16 = 96.

   

3. **misalign的定义？**

   ​	软件上是以16byte作为边界的。会去保证16byte对齐。

   ​	硬件上根据LSU的长度来确认对齐边界。因此1620上是16Byte（128bit per LSU）， 688/intel 6248上是32 byte。



# `copy_{from/to}_user`深度解读

相比`memcpy`多了地址合法性校验，会先check一下from或者to的地址是不是合法的el0地址，以及size是否超限等。

### copy前的校验

```asm
copy_to_user  # 包含校验copy的大小，不能超过源地址指针指向的object的大小。 -- check_copy_size
->  _copy_to_use("uaccess.h") # 检查dst地址，是否在task的栈空间内。（arm64实现中定义了INLINE_COPY_TO_USER） -- access_ok; 同时对于kasan实现也做了些相关校验。
    -> raw_copy_to_user

#define raw_copy_to_user(to, from, n)                                   \
({                                                                      \
        unsigned long __actu_ret;                                       \
        uaccess_ttbr0_enable();                                         \
        __actu_ret = __arch_copy_to_user(__uaccess_mask_ptr(to),        \
                                    (from), (n));                       \
        uaccess_ttbr0_disable();                                        \
        __actu_ret;                                                     \
})
```

``` c
static __always_inline __must_check bool
check_copy_size(const void *addr, size_t bytes, bool is_source)
{   
        int sz = __builtin_object_size(addr, 0);
        if (unlikely(sz >= 0 && sz < bytes)) {
                if (!__builtin_constant_p(bytes))
                        copy_overflow(sz, bytes);
                else if (is_source)
                        __bad_copy_from();
                else
                        __bad_copy_to();
                return false;
        }
        if (WARN_ON_ONCE(bytes > INT_MAX))
                return false;
        check_object_size(addr, bytes, is_source);
        return true;
}
```

## copy过程源码分析

``` asm
// arch/arm64/lib/copy_to_user.S
/ * Parameters:
 *      x0 - to
 *      x1 - from
 *      x2 - n
 * Returns:
 *      x0 - bytes not copied
 */
......
        .macro ldp1 reg1, reg2, ptr, val
        ldp \reg1, \reg2, [\ptr], \val
        .endm

        .macro stp1 reg1, reg2, ptr, val
        user_stp 9997f, \reg1, \reg2, \ptr, \val
        .endm

end     .req    x5
srcin   .req    x15
SYM_FUNC_START(__arch_copy_to_user)
        add     end, x0, x2
        mov     srcin, x1
#include "copy_template.S"
        mov     x0, #0
        ret

        // Exception fixups
9997:   cmp     dst, dstin
        b.ne    9998f
        // Before being absolutely sure we couldn't copy anything, try harder
        ldrb    tmp1w, [srcin]
USER(9998f, sttrb tmp1w, [dst])
        add     dst, dst, #1
9998:   sub     x0, end, dst                    // bytes not copied
        ret
SYM_FUNC_END(__arch_copy_to_user)

```

### 关于sttr/ldtr指令

``` asm
        .macro user_ldp l, reg1, reg2, addr, post_inc
8888:           ldtr    \reg1, [\addr];
8889:           ldtr    \reg2, [\addr, #8];
                add     \addr, \addr, \post_inc;

                _asm_extable_uaccess    8888b, \l;
                _asm_extable_uaccess    8889b, \l;
        .endm

        .macro user_stp l, reg1, reg2, addr, post_inc
8888:           sttr    \reg1, [\addr];
8889:           sttr    \reg2, [\addr, #8];
                add     \addr, \addr, \post_inc;

                _asm_extable_uaccess    8888b,\l;
                _asm_extable_uaccess    8889b,\l;
        .endm
```

## 性能分析

其中，copy_from_user和copy_to_user都会调用copy_template.S这个公共文件，实现逻辑是一样的，只是指令有差别。大体逻辑和memcpy很像，根据不同size来决定不同的处理方案。会区分：

- 0 - 15B -> tiny15函数，会对4个bit分别判断，然后使用ldrb/ldrh/ldrw/ldr以及对应st*进行copy处理。
- 若大于15B，先判断**源地址**是否对齐（16Byte对齐），不对齐场景先搞成对齐场景。
  - 若对齐则走SrcAligned。
  - 若不对齐，则对源非对齐地址先取补（按位取反再加1），然后判断最后4个bit的值，从而计算出距离16B对齐还有多少Byte，并分别进行处理。若src[0] = 1，说明src地址为奇数，一定要做一个1B的copy；src[1] = 1，则需要做一个2B的copy……  比如，若src=0x1111，取补为1，说明还需要1B就实现了16B对齐，那么就只需要提前ldrb 1B + strb 1B 即可。
- 16B -> 63B，在SrcAligned域段中判断是否大于64B，若小于则走tail63，通过3个ldp去做处理。
- 64B-> 127B， 使用强依赖的ldp/stp做处理；
- 128B， 使用若依赖的ldp/stp循环作处理，与memcpy实现一致。

###  0->15B性能分析

``` asm
.Ltiny15:
    /*
    * Prefer to break one ldp/stp into several load/store to access
    * memory in an increasing address order,rather than to load/store 16
    * bytes from (src-16) to (dst-16) and to backward the src to aligned
    * address,which way is used in original cortex memcpy. If keeping
    * the original memcpy process here, memmove need to satisfy the
    * precondition that src address is at least 16 bytes bigger than dst
    * address,otherwise some source data will be overwritten when memove
    * call memcpy directly. To make memmove simpler and decouple the
    * memcpy's dependency on memmove, withdrew the original process.
    */
    tbz count, #3, 1f
    ldr1    tmp1, src, #8
    str1    tmp1, dst, #8
1:
    tbz count, #2, 2f
    ldr1    tmp1w, src, #4
    str1    tmp1w, dst, #4
2:
    tbz count, #1, 3f
    ldrh1   tmp1w, src, #2
    strh1   tmp1w, dst, #2
3:
    tbz count, #0, .Lexitfunc
    ldrb1   tmp1w, src, #1
    strb1   tmp1w, dst, #1

    b   .Lexitfunc
```



### 非对齐处理

``` asm
    mov dst, dstin
    cmp count, #16
    /*When memory length is less than 16, the accessed are not aligned.*/
    b.lo    .Ltiny15

    neg tmp2, src          /* 取补 */
    ands    tmp2, tmp2, #15/* Bytes to reach alignment. */
    b.eq    .LSrcAligned     /* 对齐场景 */
    sub count, count, tmp2
    /*
    * Copy the leading memory data from src to dst in an increasing
    * address order.By this way,the risk of overwriting the source
    * memory data is eliminated when the distance between src and
    * dst is less than 16. The memory accesses here are alignment.
    */
    tbz tmp2, #0, 1f      /* 非对齐场景  */
    ldrb1   tmp1w, src, #1    //从src取 1Byte， 然后src更新为src+1
    strb1   tmp1w, dst, #1
1:
    tbz tmp2, #1, 2f
    ldrh1   tmp1w, src, #2    //从src取 2 Byte， 然后src更新为src+2
    strh1   tmp1w, dst, #2
2:
    tbz tmp2, #2, 3f
    ldr1    tmp1w, src, #4
    str1    tmp1w, dst, #4
3:
    tbz tmp2, #3, .LSrcAligned
    ldr1    tmp1, src, #8
    str1    tmp1, dst, #8

.LSrcAligned:
    cmp count, #64
    b.ge    .Lcpy_over64
    /*
    * Deal with small copies quickly by dropping straight into the
    * exit block.
    */
```

### 64->127 B

这里的ldp/stp是强依赖的，也就是说stp依赖**上一条**的ldp。

```asm
.Lcpy_over64:
    subs    count, count, #128
    b.ge    .Lcpy_body_large
    /*
    * Less than 128 bytes to copy, so handle 64 here and then jump
    * to the tail.
    */
    ldp1    A_l, A_h, src, #16
    stp1    A_l, A_h, dst, #16
    ldp1    B_l, B_h, src, #16
    ldp1    C_l, C_h, src, #16
    stp1    B_l, B_h, dst, #16
    stp1    C_l, C_h, dst, #16
    ldp1    D_l, D_h, src, #16
    stp1    D_l, D_h, dst, #16

    tst count, #0x3f
    b.ne    .Ltail63
    b   .Lexitfunc
```

### 128B以上

这里的stp仅依赖**上一个循环**的ldp。

``` asm
.Lcpy_body_large:
    /* pre-get 64 bytes data. */
    ldp1    A_l, A_h, src, #16
    ldp1    B_l, B_h, src, #16
    ldp1    C_l, C_h, src, #16
    ldp1    D_l, D_h, src, #16
1:
    /*
    * interlace the load of next 64 bytes data block with store of the last
    * loaded 64 bytes data.
    */
    stp1    A_l, A_h, dst, #16
    ldp1    A_l, A_h, src, #16
    stp1    B_l, B_h, dst, #16
    ldp1    B_l, B_h, src, #16
    stp1    C_l, C_h, dst, #16
    ldp1    C_l, C_h, src, #16
    stp1    D_l, D_h, dst, #16
    ldp1    D_l, D_h, src, #16
    subs    count, count, #64
    b.ge    1b
    stp1    A_l, A_h, dst, #16
    stp1    B_l, B_h, dst, #16
    stp1    C_l, C_h, dst, #16
    stp1    D_l, D_h, dst, #16

    tst count, #0x3f
    b.ne    .Ltail63
.Lexitfunc:
```





# memset 深度解读

`memset`在`glibc 2.17`中使用`stp`方式实现，在`glibc 2.29`中使用`dc zva`方式实现。

在`glibc 2.31`版本，对于`kunpeng 920`有单独的实现方式，但是没有用到`dc zva`指令。



## x86源码解读

### glibc如何选择当前环境使用哪个版本的memset

x86上：

| 优先级 | cpu feature                                                  | 对应函数实现                               |
| ------ | ------------------------------------------------------------ | ------------------------------------------ |
| 1      | Prefer_ERMS                                                  | erms                                       |
| 2      | AVX512F & !Prefer_No_AVX512 & AVX512VL & AVX512BW & BMI2 &(ERMS/!ERMS) | avx512_unaligned_erms/avx512_unaligned     |
| 3      | AVX512F& !Prefer_No_AVX512 & 非上述情况                      | avx512_no_vzeroupper                       |
| 4      | (!AVX512F  \|\|Prefer_No_AVX512)  & AVX2 & AVX512VL & AVX512BW & BMI2 | evex_unaligned_erms/evex_unaligned         |
| 5      | (!AVX512F \|\| Prefer_No_AVX512)  & AVX2 & 非上述情况 & RTM  | avx2_unaligned_erms_rtm/avx2_unaligned_rtm |
| 6      | (!AVX512F \|\| Prefer_No_AVX512)  & AVX2 & 非情况4 & !RTM & Prefer_No_VZEROUPPER | avx2_unaligned_erms/avx2_unaligned         |
| 7      | (!AVX512F \|\| Prefer_No_AVX512) & !AVX2 & ERMS              | sse2_unaligned_erms                        |
| 8      | others                                                       | sse2_unaligned                             |

其中，rtm表示Restricted transaction memory， 在老版本的intel机器上有这个特性，后面的版本（6248及以后）就改成了tm & tm2特性。

aarch64上：

|      | cpu feature                              | 对应函数实现     |
| ---- | ---------------------------------------- | ---------------- |
| 1    | midr = kunpeng920                        | __memset_kunpeng |
| 2    | midr = falkor \|\| PHECDA，且zva_size=64 | __memset_falkor  |
| 3    | midr = EMAG 且zva_size=64                | __memset_emag    |
| 4    | midr = A64FX且sve使能                    | __memset_a64fx   |
| 5    | others                                   | __memset_generic |



## glibc aarch64 源码解读

### aarch64 kunpeng920版本

直接不使用dc zva指令。 

size > 128B， 则通过stp q0（NEON寄存器）实现。

size < 16B,      则str指令实现。

size 在[16,128] Byte之间，使用str + stp NEON寄存器实现。

 ### aarch64 generic版本

通过文件 `sysdeps/aarch64/multiarch/memset_generic.c`链接到文件`"sysdeps/aarch64/memset.S"`中，使用汇编方式实现。显示的热点函数名为`_GI__memset_generic`。

实现思路与`memcpy`类似，通过neon寄存器实现。

1. 小于16bytes的；str/strw/strb指令实现；
2. 小于96bytes的；stp指令实现；
3. 小于256bytes的： 去到函数set_long，使用stp指令实现。
4. 大于256byte的： 去到try_zva函数，使用dc zva指令实现。

``` assembly
#define dstin   x0
#define val x1
#define valw    w1
#define count   x2
#define dst x3
#define dstend  x4
#define tmp1    x5
#define tmp1w   w5
#define tmp2    x6
#define tmp2w   w6
#define zva_len x7
#define zva_lenw w7
ENTRY_ALIGN (MEMSET, 6)

    DELOUSE (0)    # LP64中为空定义
    DELOUSE (2)

    dup v0.16B, valw
    add dstend, dstin, count

    cmp count, 96          #  > 96
    b.hi    L(set_long)
    cmp count, 16
    b.hs    L(set_medium)  # >= 16, 即[16,96]
    ...      # [0,15]
	
```

###  【0-15】 Byte

- 先判断长度是否为[8,15]，若是，则使用x寄存器，从start开始str一个8B，再从end往前str一个8B，则一定能覆盖所有[8-15]长度的memset需求；
- 若长度在[4,7]，则使用w寄存器str两次，每次4B；
- 若长度在[0,3]，则使用strb/strh指令来进行适配。

``` c
    mov val, v0.D[0]             
    /* Set 0..15 bytes.  */
    tbz count, 3, 1f
    str val, [dstin]
    str val, [dstend, -8]
    ret
    nop
1:  tbz count, 2, 2f
    str valw, [dstin]
    str valw, [dstend, -4]
    ret
2:  cbz count, 3f
    strb    valw, [dstin]
    tbz count, 1, 3f
    strh    valw, [dstend, -2]
3:  ret
```

### 【16-96】Byte

通过stp指令实现，会分成[16, 63]， [64,96]两种情况来判断。

``` assembly
    /* Set 17..96 bytes.  */
L(set_medium):
    str q0, [dstin]
    tbnz    count, 6, L(set96)
    str q0, [dstend, -16]
    tbz count, 5, 1f
    str q0, [dstin, 16]
    str q0, [dstend, -32]
1:  ret

    .p2align 4
    /* Set 64..96 bytes.  Write 64 bytes from the start and
       32 bytes from the end.  */
L(set96):
    str q0, [dstin, 16]
    stp q0, q0, [dstin, 32]
    stp q0, q0, [dstend, -32]
    ret

```

### 【96，255） Byte

会区分是否大于255，若不大于，则使用stp指令实现。

``` assembly
    .p2align 3
    nop
L(set_long):
    and valw, valw, 255 # valw只取低8bit的数值，表示memset的值的低8bit是什么。
    bic dst, dstin, 15 
    str q0, [dstin]
    cmp count, 256   # 若<256，则标记位CF=1，有借位；否则CF=0
    # 判断是否大于256Byte
    ccmp    valw, 0, 0, cs # cs会判断CF位，若CF==0且valw==0，则标记ZF=1
    b.eq    L(try_zva)  # 仅当memset 0且长度>=256B时才会到try_zva。
L(no_zva):
    sub count, dstend, dst  /* Count is 16 too large.  */
    sub dst, dst, 16        /* Dst is biased by -32.  */
    sub count, count, 64 + 16   /* Adjust count and bias for loop.  */
1:  stp q0, q0, [dst, 32]
    stp q0, q0, [dst, 64]!
L(tail64):
    subs    count, count, 64
    b.hi    1b
2:  stp q0, q0, [dstend, -64]
    stp q0, q0, [dstend, -32]
    ret


```

### 【256，∞） Byte

根据硬件对zva size的支持粒度，是否为64Byte/128Byte/其他，分为`zva_64`和`zva_128`以及`zva_other`三种实现，区别主要在于单次zva的数值。

以`zva_64`为例，当size=256Byte时，

- 对最前的64 Byte通过str/stp来置0（在try_zva之前的set_long中已经对前16B置零了，所以这里只操作了16+32 Byte）
- 从第一个64Byte对齐地址开始，通过stp对64Byte对齐地址置0；（若已经是64B对齐，则选择下一个64B对齐地址）
- 第二个64B对齐地址开始使用dc zva来清零，每次64B；
- 最后的[64, 128)Byte用4条stp完成；（两条stp从前往后，两条stp从后往前，从而解决dstend非64B对齐问题）

以`zva_128`为例，当size=256Byte时，

- 对最前的128byte通过stp来置0；

- 从第一个128Byte对齐地址开始，通过dc zva以128 Byte为粒度来置0；

- 对最后的128byte通过stp来置0。

从而实现非对齐处理问题。所以，对于256Byte长度的对齐memset来说，

`stp q0* 4 + dc zva 128B + stp q0 *4`，相当于多做了128B的清零操作。



``` assembly
L(try_zva):
#ifdef ZVA_MACRO		# falkor架构才会用到。
    zva_macro
#else
    .p2align 3
    mrs tmp1, dczid_el0		# 判断架构中dczid能力
    tbnz    tmp1w, 4, L(no_zva) # dczid[4]表示是否架构支持dc zva指令
    and tmp1w, tmp1w, 15	# 低4bit表示支持的最大zva size的LOG2值，单位为word（4B）
    cmp tmp1w, 4    /* ZVA size is 64 bytes.  */
    b.ne     L(zva_128) # 2**4 = 16words=64B，若支持size>64，则跳到zva_128.

    /* Write the first and last 64 byte aligned block using stp rather
       than using DC ZVA.  This is faster on some cores.
     */
L(zva_64):
    str q0, [dst, 16]
    stp q0, q0, [dst, 32]
    bic dst, dst, 63
    stp q0, q0, [dst, 64]
    stp q0, q0, [dst, 96]
    sub count, dstend, dst   # dst前移到64B对齐地址后，计算此时的count，≥count_ori
    sub count, count, 128+64+64  
    	# 去掉前面几条stp（128 B)
    	# + 一条zva(64B)
    	# + 最后的2条stp（64B）后，剩下的字节数。
    add dst, dst, 128
    nop
1:  dc  zva, dst
    add dst, dst, 64
    subs    count, count, 64
    b.hi    1b
    stp q0, q0, [dst, 0]
    stp q0, q0, [dst, 32]
    stp q0, q0, [dstend, -64]
    stp q0, q0, [dstend, -32]
    ret

    .p2align 3
L(zva_128):
    cmp tmp1w, 5    /* ZVA size is 128 bytes.  */
    b.ne    L(zva_other)

    str q0, [dst, 16]
    stp q0, q0, [dst, 32]
    stp q0, q0, [dst, 64]
    stp q0, q0, [dst, 96]
    bic dst, dst, 127
    sub count, dstend, dst  /* Count is now 128 too large.  */
    sub count, count, 128+128   /* Adjust count and bias for loop.  */
    add dst, dst, 128
1:  dc  zva, dst
    add dst, dst, 128
    subs    count, count, 128
    b.hi    1b
    stp q0, q0, [dstend, -128]
    stp q0, q0, [dstend, -96]
    stp q0, q0, [dstend, -64]
    stp q0, q0, [dstend, -32]
    ret
L(zva_other):
    mov tmp2w, 4
    lsl zva_lenw, tmp2w, tmp1w
    add tmp1, zva_len, 64   /* Max alignment bytes written.  */
    cmp count, tmp1
    blo L(no_zva)

    sub tmp2, zva_len, 1
    add tmp1, dst, zva_len
    add dst, dst, 16
    subs    count, tmp1, dst    /* Actual alignment bytes to write.  */
    bic tmp1, tmp1, tmp2    /* Aligned dc zva start address.  */
    beq 2f
1:  stp q0, q0, [dst], 64
    stp q0, q0, [dst, -32]
    subs    count, count, 64
    b.hi    1b
2:  mov dst, tmp1
    sub count, dstend, tmp1 /* Remaining bytes to write.  */
    subs    count, count, zva_len
    b.lo    4f
3:  dc  zva, dst
    add dst, dst, zva_len
    subs    count, count, zva_len
    b.hs    3b
4:  add count, count, zva_len
    sub dst, dst, 32        /* Bias dst for tail loop.  */
    b   L(tail64)
#endif

END (MEMSET)
libc_hidden_builtin_def (MEMSET)
```

对于大于256Byte的场景，目前在1630V200上看，硬件支持到64Byte粒度，因此会走zva_64的路线。理论上分析，当cache hit时，可以每cycle执行一条指令。

- dc zva指令走st pipeline，1 cycle可以走一个，虽然有2个st pipeline，但是因为要依赖add的结果，所以只能串行做。
- 所以，理论上dc zva的带宽就是 64B/c = 160GB/s

``` c
1:  dc  zva, dst       // 依赖上一条add的结果。
    add dst, dst, 64   // 可以与上一条并行
    subs    count, count, 64  // 无依赖，可以与上一条并行
    b.hi    1b                // sub+b可以同拍完成。
```

### 框架性能分析

框架调用流程为：

``` c
// 对应的trace流： 从memset@plt返回后开始（scalesim仿真）
S1:
    sub x19, x19, #0x1
	cmn x19, #0x1    
	b.ne 4015dc      // 除最后一次，均跳转
S2：        
	mov x2, x20      
	mov w1, #0x0     
	mov x0, x21      
	bl 401230       // 一定跳转
S3：
	adrp x16, 418000 
	ldr x17, [x16, #112]
    add x16, x16, #0x70
	br x17      // 绝对跳转
// 若bound在取指上，则框架增加了3cycle。
```

然后进入memset的函数实现，这里的trace是512Byte的，也就是说从memset到try_zva函数的代码实现。

![image-20221018204817540](C:\Users\h00424781\AppData\Roaming\Typora\typora-user-images\image-20221018204817540.png)

然后是memset本身开销

``` asm
    dup v0.16B, valw
    add dstend, dstin, count
    cmp count, 96          #  > 96
    b.hi    L(set_long)
    cmp count, 16
    b.hs    L(set_medium)  # >= 16, 即[16,96]
```



### 实测值分析

| memset size | 1630V200 实测值 | 理论最大值（仅考虑zva_64的循环体内） |
| ----------- | --------------- | ------------------------------------ |
| 512         | 5ns = 12.5 c    | 8 c                                  |
| 4k          | 27ns = 67.5c    | 64c                                  |
| 512k        | 3.3 us = 8250c  | 8192c                                |


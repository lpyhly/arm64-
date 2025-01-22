# memcpy深度解读

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

## 17 - 96 byte

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

##1 - 15 byte

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

## 大于96 byte

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

## QA

1. **为什么选择16 byte作为第一个分界线？**

   ​	因为16byte就可以ldp操作了。

   

2. **为什么选择96 byte而不是64 byte作为分界线?**

   ​	需要处理dst的对齐问题，因为ldp的操作为16byte， 因此地址的头尾都需要有16byte来作为裕量。那么最小保证的长度就是16 + 64 + 16 = 96.

   

3. **misalign的定义？**

   ​	软件上是以16byte作为边界的。会去保证16byte对齐。

   ​	硬件上根据LSU的长度来确认对齐边界。因此1620上是16Byte（128bit per LSU）， 688/intel 6248上是32 byte。

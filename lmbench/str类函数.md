# str类函数

# strchr

该函数从微架构上看，可能会存在fsu瓶颈，即浮点运算过多。



### 波形分析

1630v200（onepro）上分析。

![image-20230524091033682](D:%5Cat_work%5CDocuments%5C%E6%88%91%E7%9A%84%E6%80%BB%E7%BB%93%E6%96%87%E6%A1%A3%5Cimages%5Cimage-20230524091033682.png)

### 汇编级分析（指令trace）

测试参数：`-s 100 -n 1 `

```c
1, 161, pc 4014b8, inst 97fffefa, bl #0x4010a0
1, 161, pc 4010a0, inst f00000f0, adrp x16, #0x420000
1, 161, pc 4010a4, inst f9403a11, ldr x17, [x16, #0x70], load, va 0x420070, pa 0x104463070, data 0xfffff7de4570, RAM
1, 161, pc 4010a8, inst 9101c210, add x16, x16, #0x70
1, 161, pc 4010ac, inst d61f0220, br x17
1, 161, pc fffff7de4570, inst 52808024, mov w4, #0x401
1, 161, pc fffff7de4574, inst 72a80204, movk w4, #0x4010, lsl #16
1, 161, pc fffff7de4578, inst 4e010c20, dup v0.16b, w1
1, 161, pc fffff7de457c, inst 927be802, and x2, x0, #0xffffffffffffffe0
1, 161, pc fffff7de4580, inst 4e040c90, dup v16.4s, w4
1, 161, pc fffff7de4584, inst f2401003, ands x3, x0, #0x1f
1, 161, pc fffff7de4588, inst 4eb08607, add v7.4s, v16.4s, v16.4s
1, 161, pc fffff7de458c, inst 540002a0, b.eq #0xfffff7de45e0
1, 161, pc fffff7de4590, inst 4cdfa041, ld1 {v1.16b, v2.16b}, [x2], #32, load, va 0x422000, pa 0x1044a3000, data 0x6975712065685400, RAM, load, va 0x422008, pa 0x1044a3008, data 0x6e776f7262206b63, RAM, load, va 0x422010, pa 0x1044a3010, data 0x6d756a20786f6620, RAM, load, va 0x422018, pa 0x1044a3018, data 0x207265766f207370, RAM
1, 161, pc fffff7de4594, inst cb0303e3, neg x3, x3
1, 161, pc fffff7de4598, inst 4e209823, cmeq v3.16b, v1.16b, #0
1, 161, pc fffff7de459c, inst 6e208c25, cmeq v5.16b, v1.16b, v0.16b
1, 161, pc fffff7de45a0, inst 4e209844, cmeq v4.16b, v2.16b, #0
1, 161, pc fffff7de45a4, inst 6e208c46, cmeq v6.16b, v2.16b, v0.16b
1, 161, pc fffff7de45a8, inst 4e271c63, and v3.16b, v3.16b, v7.16b
1, 161, pc fffff7de45ac, inst 4e271c84, and v4.16b, v4.16b, v7.16b
1, 161, pc fffff7de45b0, inst 4e301ca5, and v5.16b, v5.16b, v16.16b
1, 161, pc fffff7de45b4, inst 4e301cc6, and v6.16b, v6.16b, v16.16b
1, 161, pc fffff7de45b8, inst 4ea51c71, orr v17.16b, v3.16b, v5.16b
1, 161, pc fffff7de45bc, inst 4ea61c92, orr v18.16b, v4.16b, v6.16b
1, 161, pc fffff7de45c0, inst d37ff863, lsl x3, x3, #1
1, 161, pc fffff7de45c4, inst 4e32be31, addp v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de45c8, inst 92800005, mov x5, #-1
1, 161, pc fffff7de45cc, inst 4e32be31, addp v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de45d0, inst 9ac324a3, lsr x3, x5, x3
1, 161, pc fffff7de45d4, inst 4e083e25, mov x5, v17.d[0]
1, 161, pc fffff7de45d8, inst 8a2300a3, bic x3, x5, x3
1, 161, pc fffff7de45dc, inst b50002a3, cbnz x3, #0xfffff7de4630
1, 161, pc fffff7de45e0, inst 4cdfa041, ld1 {v1.16b, v2.16b}, [x2], #32, load, va 0x422020, pa 0x1044a3020, data 0x797a616c20656874, RAM, load, va 0x422028, pa 0x1044a3028, data 0x6568542e676f6420, RAM, load, va 0x422030, pa 0x1044a3030, data 0x62206b6369757120, RAM, load, va 0x422038, pa 0x1044a3038, data 0x786f66206e776f72, RAM
1, 161, pc fffff7de45e4, inst 4e209823, cmeq v3.16b, v1.16b, #0
1, 161, pc fffff7de45e8, inst 6e208c25, cmeq v5.16b, v1.16b, v0.16b
1, 161, pc fffff7de45ec, inst 4e209844, cmeq v4.16b, v2.16b, #0
1, 161, pc fffff7de45f0, inst 6e208c46, cmeq v6.16b, v2.16b, v0.16b
1, 161, pc fffff7de45f4, inst 4ea51c71, orr v17.16b, v3.16b, v5.16b
1, 161, pc fffff7de45f8, inst 4ea61c92, orr v18.16b, v4.16b, v6.16b
1, 161, pc fffff7de45fc, inst 4eb21e31, orr v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de4600, inst 4ef1be31, addp v17.2d, v17.2d, v17.2d
1, 161, pc fffff7de4604, inst 4e083e23, mov x3, v17.d[0]
1, 161, pc fffff7de4608, inst b4fffec3, cbz x3, #0xfffff7de45e0
1, 161, pc fffff7de45e0, inst 4cdfa041, ld1 {v1.16b, v2.16b}, [x2], #32, load, va 0x422040, pa 0x1044a3040, data 0x6f2073706d756a20, RAM, load, va 0x422048, pa 0x1044a3048, data 0x2065687420726576, RAM, load, va 0x422050, pa 0x1044a3050, data 0x676f6420797a616c, RAM, load, va 0x422058, pa 0x1044a3058, data 0x697571206568542e, RAM
1, 161, pc fffff7de45e4, inst 4e209823, cmeq v3.16b, v1.16b, #0
1, 161, pc fffff7de45e8, inst 6e208c25, cmeq v5.16b, v1.16b, v0.16b
1, 161, pc fffff7de45ec, inst 4e209844, cmeq v4.16b, v2.16b, #0
1, 161, pc fffff7de45f0, inst 6e208c46, cmeq v6.16b, v2.16b, v0.16b
1, 161, pc fffff7de45f4, inst 4ea51c71, orr v17.16b, v3.16b, v5.16b
1, 161, pc fffff7de45f8, inst 4ea61c92, orr v18.16b, v4.16b, v6.16b
1, 161, pc fffff7de45fc, inst 4eb21e31, orr v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de4600, inst 4ef1be31, addp v17.2d, v17.2d, v17.2d
1, 161, pc fffff7de4604, inst 4e083e23, mov x3, v17.d[0]
1, 161, pc fffff7de4608, inst b4fffec3, cbz x3, #0xfffff7de45e0
1, 161, pc fffff7de45e0, inst 4cdfa041, ld1 {v1.16b, v2.16b}, [x2], #32, load, va 0x422060, pa 0x1044a3060, data 0x7262206b63, RAM, load, va 0x422068, pa 0x1044a3068, data 0x1011, RAM, load, va 0x422070, pa 0x1044a3070, data 0x65745f6f636f633c, RAM, load, va 0x422078, pa 0x1044a3078, data 0x67676972745f7473, RAM
1, 161, pc fffff7de45e4, inst 4e209823, cmeq v3.16b, v1.16b, #0
1, 161, pc fffff7de45e8, inst 6e208c25, cmeq v5.16b, v1.16b, v0.16b
1, 161, pc fffff7de45ec, inst 4e209844, cmeq v4.16b, v2.16b, #0
1, 161, pc fffff7de45f0, inst 6e208c46, cmeq v6.16b, v2.16b, v0.16b
1, 161, pc fffff7de45f4, inst 4ea51c71, orr v17.16b, v3.16b, v5.16b
1, 161, pc fffff7de45f8, inst 4ea61c92, orr v18.16b, v4.16b, v6.16b
1, 161, pc fffff7de45fc, inst 4eb21e31, orr v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de4600, inst 4ef1be31, addp v17.2d, v17.2d, v17.2d
1, 161, pc fffff7de4604, inst 4e083e23, mov x3, v17.d[0]
1, 161, pc fffff7de4608, inst b4fffec3, cbz x3, #0xfffff7de45e0
1, 161, pc fffff7de460c, inst 4e271c63, and v3.16b, v3.16b, v7.16b
1, 161, pc fffff7de4610, inst 4e271c84, and v4.16b, v4.16b, v7.16b
1, 161, pc fffff7de4614, inst 4e301ca5, and v5.16b, v5.16b, v16.16b
1, 161, pc fffff7de4618, inst 4e301cc6, and v6.16b, v6.16b, v16.16b
1, 161, pc fffff7de461c, inst 4ea51c71, orr v17.16b, v3.16b, v5.16b
1, 161, pc fffff7de4620, inst 4ea61c92, orr v18.16b, v4.16b, v6.16b
1, 161, pc fffff7de4624, inst 4e32be31, addp v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de4628, inst 4e32be31, addp v17.16b, v17.16b, v18.16b
1, 161, pc fffff7de462c, inst 4e083e23, mov x3, v17.d[0]
1, 161, pc fffff7de4630, inst d1008042, sub x2, x2, #0x20
1, 161, pc fffff7de4634, inst dac00063, rbit x3, x3
1, 161, pc fffff7de4638, inst dac01063, clz x3, x3
1, 161, pc fffff7de463c, inst f240007f, tst x3, #1
1, 161, pc fffff7de4640, inst 8b430440, add x0, x2, x3, lsr #1
1, 161, pc fffff7de4644, inst 9a9f0000, csel x0, x0, xzr, eq
1, 161, pc fffff7de4648, inst d65f03c0, ret 
1, 161, pc 4014bc, inst 94000681, bl #0x402ec0
1, 161, pc 402ec0, inst d00000e1, adrp x1, #0x420000
1, 161, pc 402ec4, inst 9107c021, add x1, x1, #0x1f0
1, 161, pc 402ec8, inst f9415822, ldr x2, [x1, #0x2b0], load, va 0x4204a0, pa 0x1044634a0, data 0x46000005aa26a0, RAM
1, 161, pc 402ecc, inst 8b020000, add x0, x0, x2
1, 161, pc 402ed0, inst f9015820, str x0, [x1, #0x2b0], store, va 0x4204a0, pa 0x1044634a0, data 0x46000005aa26a0, RAM
1, 161, pc 402ed4, inst d65f03c0, ret 
1, 161, pc 4014c0, inst d1000673, sub x19, x19, #1
1, 161, pc 4014c4, inst b100067f, cmn x19, #1
1, 161, pc 4014c8, inst 54ffff41, b.ne #0x4014b0
1, 161, pc 4014b0, inst 2a1503e1, mov w1, w21
1, 161, pc 4014b4, inst aa1403e0, mov x0, x20
1, 161, pc 4014b8, inst 97fffefa, bl #0x4010a0

```


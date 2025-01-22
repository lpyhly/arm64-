# vma架构

![image-20200914155657354](D:\at_work\Documents\我的总结文档\images\image-20200914155541749.png)





# vma中获取pa

vma中存在pgd，也就是对应着TTBRx寄存器中的基地址。

如果页表已经建好，那么到时候就可以直接硬件翻译出来va->pa了，不需要软件参与。这里，硬件会去到pgd开始的一段地址空间去进行寻址。

如果页表不存在，则硬件触发pagefault，然后走pagefault的流程处理，软件会按照页表的格式去写对应pgd, p4d, pud, pmd, pte对应的地址空间的值（这里，arm64中，pud=p4d=pgd），也就是修改pte页表内容，从而让硬件可以直接识别到，具体的见《pagefault缺页异常处理流程.md》。


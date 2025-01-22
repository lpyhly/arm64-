# 附录

## 关于swap空间

磁盘的一段空间。读写swap空间实际是磁盘IO操作。

### swap的分类：

文件映射，将一个文件映射为swap file，当作swap空间；

磁盘空间预留。在重装系统的磁盘分区期间，直接划出一部分空间作为swap空间。

 

### swap与普通磁盘空间的区别：

匿名文件可以在ddr空间不够的时候放到swap，但不能放到普通磁盘。

对于file-back的文件，ddr找不到时，会先找swap空间再找普通磁盘么？或者swap找文件的速度更快？（反正是类似的意思）

 

### swap与ddr的区别：

在ddr中的一定有pa，pa与ddr的物理位置是一一对应的。在swap中的一定没有pa。当从ddr->swap时，会调用free_pages释放pa；反之调用alloc_pages重新申请。因此，当数据从ddr->swap->ddr走一圈，pa可能会发生变化。

### swap设置

关闭/开启swap空间：

swapoff/swapon

 

设置swap中匿名页/file-backend file的回收比例：

swapiness，越大则越倾向于回收匿名页。

swapiness=0，则ddr中的匿名页一定不会被回收。

 

 

## ddr寻址

设计时就需要考虑最大支持的dimm条容量，然后预留出相应大小的pa地址范围。如，1620设计每个die最大支持256G（4 channel * 2dimm/channel * 32GB/dim），则地址空间是这样给的。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

注意，这里是按照HHA为粒度的，但若是选择die内/片内/片间交织，则交织范围内的地址空间是共享的。

比如，当chip 0 TB上有4根32G 3200MHz的dimm条，选择die内交织，那么bios给这个die分配ddr地址空间时，会在chip 0 TB的ddr空间上拿出128G地址出来，即0x20_0000_0000 -> 0x40_0000_0000，供TB的HHA0/HHA1共享。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)
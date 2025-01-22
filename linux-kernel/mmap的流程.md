# 1      mmap的流程

## 1.1    mmap仅用于用户态

mmap可以映射文件 & 匿名空间，后者可以用于父子进程通信（fork时会将memory map空间一同映射过去），也可以认为是单纯的malloc一段空间。

映射文件时，可以给不同的参数，比如可读PROT_READ，可写PROT_WRITE等，若不标注可写则说明是只读空间，后续pagefault时对应的函数不同。

## 1.2    mmap以pagesize为单位

4k/16k/64k的粒度来映射，而且是pagesize对齐的。这个可以通过传入do_mmap的参数pgoff看出，这个值的单位是page。

## 1.3    mmap的常用flags

MAP_SHARED：多个进程mmap同一片地址或同一个file时，对当前file的写入时会被所有人感知到的；

MAP_PRIVATE：与上面相反，copy-on-write形式，所作的修改不会被其他人感知。

MAP_ANNOYMOUS：匿名映射。

## 1.4    mmap系统调用执行流程

1. 检查参数，并根据传入的映射类型设置vma的flags.

2. 进程查找其虚拟地址空间，找到一块空闲的满足要求的虚拟地址空间.

3. 根据找到的虚拟地址空间初始化vma.

4. 设置vma->vm_file.

5. 根据文件系统类型，将vma->vm_ops设为对应的file_operations.

6. 将vma插入mm的链表中.

前2步在do_mmap中处理；后4步在mmap_region中处理。

## 1.5    mmap源码分析

arch/arm64/kernel/sys.c 定义了sys_mmap的调用 

```c
-> ksys_mmap_pgoff. (mm/mmap.c) 判断是否为匿名页/大页，进行一些处理;
	-> vm_mmap_pgoff 安全审计，
		-> do_mmap_pgoff->do_mmap 终于开始有实际内容啦~~

```





### 1.5.1       do_mmap

就是拿到了va，并将需要映射的file+pgoff准备好。具体做了3件事：

1. 根据映射长度len获取到了未被分配过的一段len长度的va地址；

2. 将file_based/annoymous需要映射的pgoff值计算出来，前者需要映射的是file+pgoff，后者为pgoff（file=0）。

3. 根据传入的flags设置对应的vm_flags。



对应代码来看：

Ø 对flag属性进行区分处理，比如若为PROT_READ，则多数情况下再添加一个PROT_EXEC；

Ø 某些异常情况检测，比如非对齐/overflow等。

Ø 从地址空间获取一个va地址。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

Ø 添加vm_flags；

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image004.jpg)

Ø 对于file-backend的情况，先从struct file中获取inode，调用file_mmap_ok判断当前file及其offset是否可以映射。

Ø 对于匿名映射，判断传入的flag来设置不同的vm_flags。对于MAP_SHARED，则文件偏移量pgoff=0（这里没有文件，所以按照内存0地址来理解），同时设置vm_flags=VM_SHARED|VM_MAYSHRE， 对于MAP_PRIVATE则只设置pgoff=addr>>PAGE_SHIFT（pgoff的单位是page）。

Ø 调用mmap_region进行va<->file+pgoff的映射。

### 1.5.2       mmap_region

mmap_region：申请一个vma，然后进行填充

​    ![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image006.jpg)

对于file_based，再填充vm_file和vm_ops（vm_ops通过call_mmap填充）： 

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image008.jpg)

 

其中call_mmap调用file->f_op->mmap()钩子函数链接到具体fs，具体函数可以参见内核中的fs/xfs（或其他文件系统名）/xfs_file.c的struct file_operations结构体对应的实例的mmap钩子。也可以通过bpftrace来获取到：

``` shell
bpftrace -e 'kprobe:__mmap_region {@=count(); printf("%s\n", ksym(((struct file *)arg1)->f_op->mmap));}'  -c "./a.out 0xc6055e0200 8"
```



![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image010.jpg)

其中，xfs_file_mmap做的事情只是把vma->vm_ops填充，供后续pagefault时调用。

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image012.jpg)

![img](file:///C:\Users\H00424~1\AppData\Local\Temp\msohtmlclip1\01\clip_image014.jpg)

然后，将申请的新vma加入mm的vma链表，更新mm。



# 影响lat_mmap的因素

### 文件系统类型 

映射文件的文件系统不同，性能会有差异。在1630上实测不同fs下mmap 2m性能：

 2.87（tmpfs） 3.48（nonefs）   4.11（xfs）  5.53（ext4）  3.99（ext2）

默认使用的映射文件是./haha，因此和当前文件系统类型有关。

- 在emu上，如果是fileimg形式，则映射的是ext2的ramblock；如果是minifs形式，则为nonefs；

- 在样片上，默认是nonefs格式，即/不挂载文件系统，直接从内存中读取。




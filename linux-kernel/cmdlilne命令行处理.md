# cmdline的函数定义

这些函数会放到.init.setup段。

1. 通过`__setup()`函数，比如`__setup("nohz_full=", housekeeping_nohz_full_setup);`
2. 模块driver中的定义。

# cmdline的解析

start_kernel函数中，

1. 调用`setup_arch`解析tags获取cmdline，然后拷贝到boot_command_line中。

2. 调用`parse_early_param()`来进行高优先级的参数解析。do_early_param遍历.init.setup段，如果有`obs_kernel_param`的`early`为1，或cmdline中有`console`参数并且obs_kernel_param有`earlycon`参数，则会调用该obs_kernel_param的setup函数来解析参数。
3. 调用`parse_args`来进行所有parameter的参数解析。对应的处理函数在代码中使用`__setup`函数进行了注册。

# cmdline大全

官方参考：

​	https://www.kernel.org/doc/html/v4.14/admin-guide/kernel-parameters.html



| 特性名       |      |      |
| ------------ | ---- | ---- |
| apparmor 0/1 |      |      |
| selinux 0/1  |      |      |
|              |      |      |
|              |      |      |
|              |      |      |
|              |      |      |
|              |      |      |
|              |      |      |
|              |      |      |


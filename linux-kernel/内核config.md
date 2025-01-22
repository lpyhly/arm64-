# 性能相关

### auditd

1. 若auditd开启，则相关的操作会被audit跟踪记录，比如文件操作。

### security

1. 若CONFIG_SECURITY开启，会在inode操作（如getattr）时进行安全hook的检查，对每个安全相关的hook一一排查，看看是否有不符合的。
2. 安全相关的hook钩子函数详见"include/linux/lsm_hooks.h"
3. CONFIG_SECURITY一旦开启，这些hook钩子就会使用到，与selinux是否开启无关。

### trace

1. 应该也会起作用，会trace各种操作，比如security的操作也会进行记录。


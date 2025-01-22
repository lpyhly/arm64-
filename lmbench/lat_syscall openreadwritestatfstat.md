# lat_syscall open/read/write/stat/fstat


## 测试方式

```shell
taskset -c 1 ./lat_syscall -P1 -W0 -N30  null /dev/null
taskset -c 1 ./lat_syscall -P1 -W0 -N30  read /dev/null
taskset -c 1 ./lat_syscall -P1 -W0 -N30  write /dev/null
taskset -c 1 ./lat_syscall -P1 -W0 -N30  stat /dev/null
taskset -c 1 ./lat_syscall -P1 -W0 -N30  fstat /dev/null
taskset -c 1 ./lat_syscall -P1 -W0 -N30  open /dev/null
```


## open

### 测试代码解析

测试打开和关闭`/dev/null`这个文件描述符的性能，单次loop中含有两个系统调用，open & close。

``` c
void
do_openclose(iter_t iterations, void *cookie)
{
    struct _state *pState = (struct _state*)cookie;
    int fd;

    while (iterations-- > 0) {
        fd = open(pState->file, 0);
        if (fd == -1) {
            perror(pState->file);
            return;
        }
        close(fd);
    }
}
```

### 函数级别分析

在1620服务器上check热点（openEuler22.03， 4k pagesize），可以看到60% open + 40% close

```perl
- 60.61% __open                                                              
   - 56.60% el0_sync                                                         
      - el0_sync_handler                                                     
         - 56.55% el0_svc                                                    
            - do_el0_svc                                                     
               - 56.50% el0_svc_common.constprop.0                           
                  - 52.10% __arm64_sys_openat                                
                     - 51.53% do_sys_openat2                                 
                        - 35.98% do_filp_open                                
                           - 35.54% path_openat                              
                              - 14.21% do_open                               
                                 - 6.17% vfs_open                            
                                    - 5.77% do_dentry_open                   
                                       - 1.42% security_file_open            
                                          - 0.92% selinux_file_open          
                                               0.67% inode_has_perm          
                                         1.02% chrdev_open                   
                                       - 0.75% path_get                      
                                            lockref_get                      
                                         0.61% mntget                        
                                 - 4.09% complete_walk                       
                                    - 3.76% try_to_unlazy                    
                                       - 2.86% __legitimize_path             
                                            2.17% __legitimize_mnt           
                                 - 2.57% may_open                            
                                    - 2.17% inode_permission                 
                                       - 2.04% inode_permission.part.0       
                                          - 1.39% security_inode_permission  
                                               1.20% selinux_inode_permission
                                   0.77% ima_file_check                      
                              - 9.10% link_path_walk.part.0                  
                                 - 5.51% inode_permission.part.0             
                                    - 4.81% security_inode_permission        
                                       - 4.48% selinux_inode_permission      
                                            2.96% avc_lookup                 
                                 - 2.76% walk_component                      
                                    - 1.72% lookup_fast                      
                                         __d_lookup_rcu                      
                                      0.92% step_into                        
                              - 6.28% alloc_empty_file                       
                                 - 5.78% __alloc_file                        
                                    - 3.61% security_file_alloc              
                                         2.19% kmem_cache_alloc              
                                      1.47% kmem_cache_alloc                 
                              - 2.31% open_last_lookups                      
                                 - 1.78% lookup_fast                         
                                      __d_lookup_rcu                         
                              - 1.82% path_init                              
                                 - 1.15% nd_jump_root                        
                                      set_root                               
                              - 1.15% terminate_walk                         
                                   0.67% dput                                
                        - 8.52% get_unused_fd_flags                          
                           - 7.95% __alloc_fd                                
                              - 6.43% files_cgroup_alloc_fd                  
                                   6.26% page_counter_try_charge             
                        - 3.84% getname                                      
                           - 3.79% getname_flags                             
                              - 1.69% strncpy_from_user                      
                                 - 1.12% __check_object_size                 
                                      0.77% __check_object_size.part.0       
                                1.34% kmem_cache_alloc                       
                        - 1.02% fd_install                                   
                             __fd_install                                    
                        - 0.60% putname                                      
                             kmem_cache_free                                 
                  - 1.29% syscall_trace_exit                                 
                       0.97% __audit_syscall_exit                            
                    1.07% syscall_trace_enter                                
```






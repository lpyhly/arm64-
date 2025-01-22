# cmake作用

- 跨平台
- 编译
- 根据CMakeLists.txt来生成Makefile。

# cmake 架构

| CMakeLists.txt |                                                        |      |
| -------------- | ------------------------------------------------------ | ---- |
|                | add_subdirectory进入子目录，然后继续执行CMakeLists.txt |      |
|                | scripts： 脚本，可单独执行                             |      |
|                |                                                        |      |
|                |                                                        |      |
|                |                                                        |      |
|                |                                                        |      |
|                |                                                        |      |



# cmake基本语法

| 命令           | 含义           |
| -------------- | -------------- |
| add_executable | 添加可执行目标 |
| foreach...endforeach| 循环|
| MSVC| windows下进行编译|


## 变量定义和引用

没有=赋值操作，只能通过set，option（ON/OFF)来定义变量；

CMake中，变量的值要么是string，要么是strings组成的list

- 变量引用： ${variable_name}

- 环境变量： $ENV{VAR}，采用set()/unset()定义和取消，默认变量作用域为set的当前作用域，即：
  - 函数内set的，作用域在本函数；
  - CMakeLists.txt定义的，作用域在当前directory以及子directory中；
  - set(variable value **CACHE** <type> "**")方式定义的，会缓存到CMakeCache.txt中，下次运行时会直接使用该值。


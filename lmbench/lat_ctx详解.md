[TOC]



# lat_ctx源码分析

## 参数解析

```
[-P <parallelism>] [-W <warmup>] [-N <repetitions>] [-s kbytes] processes [processes ...]
```

其中，-s可以选择为0，表示不需要将过程中的数据进行读取累加。

processes表示pipe消息传递过程中的进程个数，与-P是不同的。

## 输出结果解析

已经排除掉pipe管道read/write，以及数据累加的耗时了。

结果为单次ctx的耗时（无论processes的取指），即一个进程切换到另一个进程的时间，单位us。

## 基本原理

以-P1， -s0， processes=3为例进行说明。

1. 一个主进程，先malloc出来3组int型的pState->p指针，用来盛放将来pipe的fd。

2. 主进程fork 2个子进程（合计processs - 1个子进程 + 一个主进程） c1-c2。其中，c1分配p[0]的读指针和p[1]的写指针； c2分配p[1]的读指针和p[2]的写指针；依次类推，c[i]分配p[i-1]的读指针和p[i]的写指针。 
3. 2个子进程分别死循环操作，其中c1一直从p[0]读数据，读到后写到p[1]去； c2读p[1]，写p[2].
4. 子进程都部署好后，进行正式的测试流程。

主进程的操作：

​	往p[0]写数据，从p[2]中读数据，往复循环。单次操作中，会唤醒所有procs个进程。





## 代码分解

首先benchmark测量出基础数据：

```c
benchmp(initialize_overhead, benchmark_overhead, cleanup_overhead,
    0, 1, warmup, repetitions, &state);
```

然后benchmark正式测试，这里会传入procs的个数，在后面计算时间时也会用总时间除以procs进行计算。

``` c
benchmp(initialize, benchmark, cleanup, 0, parallel,
    warmup, repetitions, &state);
```

最后将二者相减，输出lat_ctx的结果。

首先overhead的计算，分两步：初始化和benchmark。

## overhead - init

创建pipe管道。

``` c
128 void
129 initialize_overhead(iter_t iterations, void* cookie)
130 {
131     int i;
132     int procs;
133     int* p;
134     struct _state* pState = (struct _state*)cookie;
135
136     if (iterations) return;
137
138     pState->pids = NULL;
139     pState->p = (int**)malloc(pState->procs * (sizeof(int*) + 2 * sizeof(int)));
140     pState->data = (pState->process_size > 0) ? malloc(pState->process_size) : NULL;
141     if (!pState->p || (pState->process_size > 0 && !pState->data)) {
142         perror("malloc");
143         exit(1);
144     }
145     p = (int*)&pState->p[pState->procs];
146     for (i = 0; i < pState->procs; ++i) {
147         pState->p[i] = p;
148         p += 2;
149     }
150
151     if (pState->data)
152         bzero(pState->data, pState->process_size);
153
154     procs = create_pipes(pState->p, pState->procs);
155     if (procs < pState->procs) {
156         cleanup_overhead(0, cookie);
157         exit(1);
158     }
159 }
```

第139 - 150行：

先创建pState->p指针，用来存放pipe的fd指针。共procs组， 每组包含一个int *的指针和2个int型的fd空间（读+写），其中指针指向的就是后面int型空间的地址，方便索引。

第151 - 153行：

​	初始化全0的data空间，也是malloc出来的，大小由参数-s指定。

第154行：

​	将pState->p指针创建procs个pipe管道，每一组的`p[i][0]`和`p[i][1]`分别为一个管道的读写口。

### overhead - benchmark

``` c
178 void
179 benchmark_overhead(iter_t iterations, void* cookie)
180 {
181     struct _state* pState = (struct _state*)cookie;
182     int i = 0;
183     int msg = 1;
184
185     while (iterations-- > 0) {
186         if (write(pState->p[i][1], &msg, sizeof(msg)) != sizeof(msg)) {
187             /* perror("read/write on pipe"); */
188             exit(1);
189         }
190         if (read(pState->p[i][0], &msg, sizeof(msg)) != sizeof(msg)) {
191             /* perror("read/write on pipe"); */
192             exit(1);
193         }
194         if (++i == pState->procs) {
195             i = 0;
196         }
197         bread(pState->data, pState->process_size);
198     }
199 }
```

对管道依次串行操作，每次操作为：

- 进行一写一读，**读写数量为1个int型字符，即4 Byte；**
- 然后将所有data按byte为单位读到寄存器中。

这里只有1个进程，这个进程包含procs组管道，因此这里的测试是没有进程切换的。只是测试了pipe管道读写以及bread的损耗。

### 正式测试 - init

``` c
201 void
202 initialize(iter_t iterations, void* cookie)
203 {
204     int procs;
205     struct _state* pState = (struct _state*)cookie;
206
207     if (iterations) return;
208
209     initialize_overhead(iterations, cookie);
210
211     pState->pids = (pid_t*)malloc(pState->procs * sizeof(pid_t));
212     if (pState->pids == NULL)
213         exit(1);
214     bzero((void*)pState->pids, pState->procs * sizeof(pid_t));
215     procs = create_daemons(pState->p, pState->pids,
216                    pState->procs, pState->process_size);
217     if (procs < pState->procs) {
218         cleanup(0, cookie);
219         exit(1);
220     }
221 }

```

第209行：

​	先调用`initialize_overhead`创建好pipe管道。

第211-215行：

调用create_daemons进行操作 ， 其中pids用来存放子进程的pid号，方便cleanup时kill掉。

create_daemons做的事情为：

- fork出procs - 1个子进程。
- 每个子进程都只保留pipe管道的一读一写指针（不配对），规则为：c1分配p[0]的读指针和p[1]的写指针； c2分配p[1]的读指针和p[2]的写指针；依次类推，c[i]分配p[i-1]的读指针和p[i]的写指针；
- 对于每个子进程调用doit函数：
  - malloc出来一段data区域并全0初始化
  - **死循环**，循环内操作为： 阻塞从当前pipe管道中读数据；读到后去到bread处理（bread的是data数据，与pipe管道完全无关）；最后write到下一个pipe管道的写指针。

###　正式测试 - benchmark

``` c
246 void
247 benchmark(iter_t iterations, void* cookie)
248 {
249     struct _state* pState = (struct _state*)cookie;
250     int msg;
251
252     /*
253      * Main process - all others should be ready to roll, time the
254      * loop.
255      */
256     while (iterations-- > 0) {
257         if (write(pState->p[0][1], &msg, sizeof(msg)) !=
258             sizeof(msg)) {
259             /* perror("read/write on pipe"); */
260             exit(1);
261         }
262         if (read(pState->p[pState->procs-1][0], &msg, sizeof(msg)) != sizeof(msg)) {
263             /* perror("read/write on pipe"); */
264             exit(1);
265         }
266         bread(pState->data, pState->process_size);
267     }
268 }
```

均为主进程的操作，子进程就自己死循环着玩。

第257-261行：

​	往pipe[0]管道中写。此时可以激发子进程c1读到数据，然后写到pipe[1]管道；再激发c2，依次类推。

第262-265行：

​	从pipe[procs-1]管道中读数据，这个数据是由第procs-1个子进程写进来的。

第266行：

​	例行操作：读完数据后bread一大堆data，这个是进程独立的，是用来延缓ctx的时间的。



## 瓶颈点分析

### cache miss

overhead的时候，只有一个进程会去bread一个8k；而benchmark中，多个进程，每个进程都会去bread一个8k。（每个进程都是write->read->bread）

因此，多进程下，性能中是包含有cache miss带来的影响的。
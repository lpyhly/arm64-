# google benchmark

 https://github.com/google/benchmark

## 编译运行

| 需求               | 命令                                                         |
| ------------------ | ------------------------------------------------------------ |
| 生成buildsystem    | cmake -E chdir "archbench_ln/build" cmake -DBENCHMARK_ENABLE_GTEST_TESTS=off  -DCMAKE_BUILD_TYPE=Release -DBENCHMARK_ENABLE_TESTING=off ../ |
| 编译测试套         | cmake -E chdir "build" ctest --build-config Release          |
| 自定义编译         | g++ benchmark/test/hly_test1.cc -std=c++11 -isystem benchmark/include  -Lbenchmark/build/src -lbenchmark -lpthread -o mybenchmark |
| 显示所有测试套list | ./benchmark_test  --benchmark_list_tests=true                |
|                    |                                                              |

## 运行参数

``` c++
          [--benchmark_list_tests={true|false}]\n"
          [--benchmark_filter=<regex>]\n"
          [--benchmark_min_time=<min_time>]\n"
          [--benchmark_repetitions=<num_repetitions>]\n"
          [--benchmark_enable_random_interleaving={true|false}]\n"
          [--benchmark_report_aggregates_only={true|false}]\n"
          [--benchmark_display_aggregates_only={true|false}]\n"
          [--benchmark_format=<console|json|csv>]\n"
          [--benchmark_out=<filename>]\n"
          [--benchmark_out_format=<json|console|csv>]\n"
          [--benchmark_color={auto|true|false}]\n"
          [--benchmark_counters_tabular={true|false}]\n"
          [--benchmark_perf_counters=<counter>,...]\n"
          [--benchmark_context=<key>=<value>,...]\n" // 自定义参数
          [--v=<verbosity>]\n");
```

## 测试套参数

| 传参型函数 |说明|
| ------------------ | ------------ |
| BENCHMARK(foo)->Iterations(次数) | 指定iterations次数 |
| BENCHMARK(foo)->Arg(1) | 传参，只能是int型。 |
| BENCHMARK(foo)->Range(10,80) | 一次性传多个参数，幂次增长 |
| BENCHMARK(foo)->Args({90}) | 传递格式化的参数 |
| BENCHMARK(BM_SetInsert)->Apply(CustomArguments); | 给BM_SetInser传多个参数，参数产生方式由CustomArguments函数生成。 |
| BENCHMARK(BM_SetInsert)->ArgsProduct({      benchmark::CreateRange(8, 128, /*multi=*/2),      benchmark::CreateDenseRange(1, 4, /*step=*/1)    }) | 复杂组合型传参 |
|  |  |
|  |  |
|  |  |
|  |  |

| 自定义函数组件                                         | 说明            |
| ------------------------------------------------------ | --------------- |
| BENCHMARK(BM_memcpy)->Name("memcpy")->Range(8, 8<<10); | 重命名benchmark |
|                                                        |                 |
|                                                        |                 |



## 测试套编写

使用方式举例： https://www.cnblogs.com/apocelipes/p/11067594.html

### 多线程编程

架构保证多个线程同时从auto开始，并且在所有线程都运行完毕后再退出循环。

``` c
static void BM_MultiThreaded(benchmark::State& state) {
  if (state.thread_index() == 0) {
    // Setup code here.
  }
  for (auto _ : state) {
    // Run the test as normal.
  }
  if (state.thread_index() == 0) {
    // Teardown code here.
  }
}
BENCHMARK(BM_MultiThreaded)->Threads(2);
```

相关参数：

| 命令                                                 | 含义                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| BENCHMARK(BM_CalculatePi)->Threads(8);               | 8线程运行                                                    |
| BENCHMARK(BM_CalculatePi)->ThreadRange(1, 32);       | 1,2,4,8,16,32线程运行                                        |
| BENCHMARK(BM_CalculatePi)->ThreadPerCpu();           | 有多少个核，就启动多少个线程，含义同->Threads(benchmark::CPUInfo::Get().num_cpus) |
| BENCHMARK(BM_CalculatePi)->DenseThreadRange(1, 8, 3) | 运行1, 4, 7 and 8 threads.                                   |






# 已有测试套list



 ## 源码解读

### 函数定义流程？

在include/benchmark/benchmark.h中，BENCHMARK()宏定义函数，本质是调用RegisterBenchmarkInternal来例化了一个函数。



### 主循环的调用流程？

在include/benchmark/benchmark.h中，BENCHMARK_MAIN() 中定义了一个main函数，分3步走： 

	1. Initialize
 	2. RunSpecifiedBenchmarks
 	3. Shutdown

``` c++
#define BENCHMARK_MAIN()                                                \
  int main(int argc, char** argv) {                                     \
    ::benchmark::Initialize(&argc, argv);                               \
    if (::benchmark::ReportUnrecognizedArguments(argc, argv)) return 1; \
    ::benchmark::RunSpecifiedBenchmarks();                              \
    ::benchmark::Shutdown();                                            \
    return 0;                                                           \
  }                                                                     \
  int main(int, char**)
```

然后进入到src/benchmark.cc函数，其中Initialize()通过ParseCommandLineFlags（）去解析传入的参数，同时设置log等级为FLAGS_v（通过-v设置）， shutdown()删除internal::global_context（存放用户自定义的参数，通过--benchmark_context=<key>=<value>来设置）。

``` c++
void Initialize(int* argc, char** argv) {
  internal::ParseCommandLineFlags(argc, argv);
  internal::LogLevel() = FLAGS_v;
}

void Shutdown() {
  delete internal::global_context;
}
```

再看主要运行语句：RunSpecifiedBenchmarks，可以有最多两个参数：display_reporter和file_reporter，前者是显示reporter，默认为default_display_reporter， 后者是说输出写到哪个文件，默认写到console串口，由（--benchmark_out指定）。做的事情有：

1. 打开或创建两个reporter
2. FindBenchmarksInternal去根据传入的filter参数来寻找匹配的benchmark名字。
3. RunBenchmarks运行。
4. 返回benchmarks.size()。

RunBenchmarks函数做的事情为：

1. 把所有需要运行的cases放到repetition_indices向量中，包含其repeat次数，若指定了random interleaving，则将该向量打乱重排。
2. 对向量遍历，调用BenchmarkRunner.DoOneRepetition做一次。若repetition不为1，则做repetition次，最后通过runner.GetResults()函数把结果写到RunResults中，再调用Report把报告打印出来。

DoOneRepetition函数做的事情为：

1. 调用DoNIterations()真正跑测试他，边跑边计算iterations次数；
2. 若规定次数后依然不满足收敛条件，则重新估计iterations，重跑，直到满足停止条件为止。
   1. **注意，若不指定Iterations且为第一次运行时，则iter=1，然后results_are_significant = false，重新估计iterations。 即测试套至少运行2次。**
3. memory_result目前没有实现。
4. 创建report，并++num_repetitions_done。

``` c
void BenchmarkRunner::DoOneRepetition() {
  const bool is_the_first_repetition = num_repetitions_done == 0;
  IterationResults i;

  for (;;) {
    i = DoNIterations();

    const bool results_are_significant = !is_the_first_repetition ||
                                         has_explicit_iteration_count ||
                                         ShouldReportIterationResults(i);

    if (results_are_significant) break;  // Good, let's report them!
      iters = PredictNumItersNeeded(i);
  }
   ...
}

```



### iterations

``` c
const bool results_are_significant = !is_the_first_repetition ||
                                     has_explicit_iteration_count ||
                                     ShouldReportIterationResults(i);
```

iterations的计算有3种方式：

1. 若repetitions >1且当次不是第一次运行，则直接取上一次的iteration值；
2. 若定义benchmark时通过->Iterations()指定了迭代次数；
3. 是否符合结束条件：error了，或迭代太多次了，或运行时间到达计时时间了。

``` c
static constexpr IterationCount kMaxIterations = 1000000000;
return i.results.has_error_ ||
       i.iters >= kMaxIterations ||  // Too many iterations already.
       i.seconds >= min_time ||      // The elapsed time is large enough.
       // CPU time is specified but the elapsed real time greatly exceeds
       // the minimum time.
       // Note that user provided timers are except from this sanity check.
       ((i.results.real_time_used >= 5 * min_time) && !b.use_manual_time());
```

关于iterations的值的计算：PredictNumItersNeeded函数中可以看到，

1. 当第一次运行时，iter=1，计算出此时的运行时间。

2. 计算乘子：看一下 iter为1的运行时间是否达到了最小运行时间的10%， 

   - 若达到了则 multiplier = 最少运行时间 * 1.4 / iter为1的运行时间，差不多为之前iter值的14倍。

   - 若没有达到，则按部就班，multiplier=10（即每次测试iterations增长为之前的10倍）

3. 最后返回multiplier * iters.



### 多线程

在"src/benchmark_runner.cc"的DoNIterations（）函数中进行多线程的创建与执行。

```c++
BenchmarkRunner::IterationResults BenchmarkRunner::DoNIterations() {
  std::unique_ptr<internal::ThreadManager> manager;

  // Run all but one thread in separate threads
  for (std::size_t ti = 0; ti < pool.size(); ++ti) {
    pool[ti] = std::thread(&RunInThread, &b, iters, static_cast<int>(ti + 1),
                           manager.get(), perf_counters_measurement_ptr);
  } // 创建并运行多个线程，线程数由pool指定（传入参数为threads-1），运行的函数为RunInThread，后面的都是函数参数。
  RunInThread(&b, iters, 0, manager.get(), perf_counters_measurement_ptr);
    // 当前线程（主线程）也运行同样的任务。

  // The main thread has finished. Now let's wait for the other threads.
  manager->WaitForAllThreads();
  for (std::thread& thread : pool) thread.join();
    // 主线程阻塞在这里，等所有子线程完成。
```

RunInThread函数会调用BenchmarkInstance->Run()运行，设置timer并收集results。

``` c
b->Run(iters, thread_id, &timer, manager, perf_counters_measurement);
```



#### StartKeepRunning 开始计时

####FinishKeepRunning 结束计时

``` c
void State::StartKeepRunning() {
  BM_CHECK(!started_ && !finished_);
  started_ = true;
  total_iterations_ = error_occurred_ ? 0 : max_iterations;
  manager_->StartStopBarrier();
  if (!error_occurred_) ResumeTiming();
}

void State::FinishKeepRunning() {
  BM_CHECK(started_ && (!finished_ || error_occurred_));
  if (!error_occurred_) {
    PauseTiming();
  }
  // Total iterations has now wrapped around past 0. Fix this.
  total_iterations_ = 0;
  finished_ = true;
  manager_->StartStopBarrier();
}
```

#### 



###　timer

timer有两个，real time和CPU time，使用的是c++的chrono库，调用的接口是：

``` c++
start_real_time_ = ChronoClockNow();
start_cpu_time_ = ReadCpuTimerOfChoice();
```

#### real time

选择high_resolution_clock，通过反汇编看到实际调用的是cntvct_el0，因此精度可以达到CPU Cycle级别。

``` c
struct ChooseClockType {
#if defined(HAVE_STEADY_CLOCK)
  typedef ChooseSteadyClock<>::type type;
#else
  typedef std::chrono::high_resolution_clock type;
#endif
};

inline double ChronoClockNow() {
  typedef ChooseClockType::type ClockType;
  using FpSeconds = std::chrono::duration<double, std::chrono::seconds::period>;
  return FpSeconds(ClockType::now().time_since_epoch()).count();
}
```

####　cpu time

当前进程/线程的实际运行时间，即处于RUNNING状态的时间。

``` c++
  double ReadCpuTimerOfChoice() const {
    if (measure_process_cpu_time) return ProcessCPUUsage();
    return ThreadCPUUsage();
  }
...
    
  double ProcessCPUUsage() {
#if defined(BENCHMARK_OS_WINDOWS) ...
#elif defined(BENCHMARK_OS_EMSCRIPTEN) ...
#elif defined(CLOCK_PROCESS_CPUTIME_ID) && !defined(BENCHMARK_OS_MACOSX)
  // FIXME We want to use clock_gettime, but its not available in MacOS 10.11. See
  // https://github.com/google/benchmark/pull/292
  struct timespec spec;
  if (clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &spec) == 0)
    return MakeTime(spec);
  DiagnoseAndExit("clock_gettime(CLOCK_PROCESS_CPUTIME_ID, ...) failed");
#else
  struct rusage ru;
  if (getrusage(RUSAGE_SELF, &ru) == 0) return MakeTime(ru);
  DiagnoseAndExit("getrusage(RUSAGE_SELF, ...) failed");
#endif
}

```


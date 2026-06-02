#	性能分析工具 perf

perf 是 Linux 平台的一款性能分析工具，能够对程序的全程运行或部分运行时段进行监控，实现函数级甚至指令级的性能统计和热点查找，从而帮助评估并定位程序性能瓶颈。perf 可以监控多类指标，例如程序运行总时间、程序执行总指令数、CPU 周期数、程序中分支指令总数和分支预测失败率、Cache 命中率、程序触发缺页中断数量等，这些指标都称为事件。使用命令 `perf list` 可以列出 perf 支持的全部事件，实际工作中可以根据需要选择相应事件。部分常见事件如下：
``` shell
$ perf list
branch-instructions OR branches                [Hardware event]
branch-misses                                  [Hardware event]
bus-cycles                                     [Hardware event]
cache-misses                                   [Hardware event]
cache-references                               [Hardware event]
cpu-cycles OR cycles                           [Hardware event]
instructions                                   [Hardware event]
ref-cycles                                     [Hardware event]
cpu-clock                                      [Software event]
cpu-migrations OR migrations                   [Software event]
page-faults OR faults                          [Software event]
task-clock                                     [Software event]
L1-dcache-load-misses                          [Hardware cache event]
L1-dcache-loads                                [Hardware cache event]
L1-dcache-prefetch-misses                      [Hardware cache event]
L1-dcache-prefetches                           [Hardware cache event]
L1-dcache-store-misses                         [Hardware cache event]
L1-dcache-stores                               [Hardware cache event]
L1-icache-load-misses                          [Hardware cache event]
L1-icache-loads                                [Hardware cache event]
L1-icache-prefetch-misses                      [Hardware cache event]
L1-icache-prefetches                           [Hardware cache event]
branch-load-misses                             [Hardware cache event]
branch-loads                                   [Hardware cache event]
...
```

perf 工具支持的子命令较多，可以通过执行 perf 命令查看全部子命令。本节只介绍以下3种常用 perf 子命令。

-	perf stat：在程序开始运行时，对特定事件计数器进行统计，并在程序运行结束时将默认或指定事件的统计结果汇总显示到标准输出。

-	perf top：实时显示系统/进程的性能统计信息。

-	perf record/perf report：perf record 用于记录一段时间内或程序全过程的性能事件，并将结果保存在 perf.data 文件中；perf report 用于读取 perf record 生成的 perf.data 文件，并显示分析数据。

##	perf stat 的使用

perf stat 用于对程序进行概览性性能统计。它的常用命令格式为：
``` shell
perf stat [ -e <event> | --event=EVENT] [ -p <pid> | start_command ]
```
其中，参数 `-e` 或 `--event` 用于指定要监测的具体事件。参数 `-p` 用于监测一个已经在运行的程序，后面跟随的 pid 为该程序进程号。例如，要统计进程号为17223的程序在监测时间内的分支预测失败情况，具体命令如下：
``` shell
perf stat -e branches -e branch-misses -p 17223
```
如果不指定具体监控事件，perf stat 默认监测的事件包括 task-clock、context-switches、cpu-migrations、page-faults、cycles、instructions、branches、branch-misses。例如，使用 perf stat 全程监控名为 hot 的程序，其运行命令和结果信息如下：
``` shell
$ perf stat ./hot
Performance counter stats for './hot':
4,736.17      msec task-clock:u        #   0.090 CPUs utilized
412,675           context-switches:u       #   0.087 M/sec
0      cpu-migrations:u         #   0.000 K/sec
45      page-faults:u            #   0.010 K/sec
707,087,126      cycles:u                 #   0.149 GHz
517,319,871      instructions:u           #   0.73  insn per cycle
113,415,431      branches:u               #   23.947 M/sec
4,710,708      branch-misses:u          #   4.15% of all branches
52.778699178 seconds time elapsed
6.000671000 seconds user
0.000000000 seconds sys
```

这里显示了执行 `perf stat` 后默认事件的信息统计。第一列显示每个事件的占用时间或执行次数统计值，第二列显示事件名称，第三列为事件备注信息。评价程序性能最直观的指标是总执行时间，即数据 `52.778699178 seconds time elapsed`，它表示程序执行消耗的实际时间（从程序开始执行到完成所经历的时间）。最后两行数据 `6.000671000 seconds user` 和 `0.000000000 seconds sys` 分别统计该程序消耗的用户态 CPU 时间和内核态 CPU 时间。

事件 task-clock 统计程序真正占用的处理器时间，单位为毫秒。这里统计结果为 `4,736.17` 毫秒。该值与程序总执行时间的比值即为 CPU 占用率，也就是第三列显示的 `0.090 CPUs utilized`。比值越高，说明程序越多时间花费在 CPU 计算上，而不是 I/O 上。对于密集计算型多线程程序，如果单线程执行，此值可以接近1；如果多线程执行，此值可接近当前处理器使用的核心数。

事件 context-switches 统计程序执行过程中的上下文切换总次数。这里统计结果为 `412,675` 次。

如果程序中执行了系统调用、进程切换等操作，都会触发上下文切换。该值与事件 task-clock 统计结果的比值为单位时间内上下文切换次数，即第三列显示的 `0.087 M/sec`。

perf 支持的部分事件需要 root 权限，例如 context-switches。当权限不足时，获取到的事件信息可能为0。建议使用 perf 之前将 `/proc/sys/kernel/perf_event_paranoid` 的值设置为 -1，或者以 root 用户或管理员身份（即使用 sudo 命令）执行 perf 命令。

事件 cpu-migrations 统计程序执行过程中处理器核心迁移次数。这里统计结果为 `0` 次，说明程序执行过程中一直在同一个处理器核心上运行，没有发生迁移。通常系统为了维护多个处理器核心之间的负载均衡，在达到一定条件后可能会将任务从一个处理器核心迁移到另一个处理器核心。

事件 page-faults 统计程序执行过程中缺页异常发生的总次数。这里统计结果为 `45` 次。该值与事件 task-clock 统计结果的比值为单位时间内发生的缺页次数，即第三列显示的 `0.010 K/sec`。

事件 cycles 统计程序执行占用的处理器周期数。这里统计结果为 `707,087,126` 次。此值与事件 task-clock 统计结果的比值为有效主频，即第三列显示的 `0.149 GHz`，远小于当前程序运行主机的处理器主频 2.3 GHz。

事件 instructions 统计程序执行的总指令数量。这里统计结果为 `517,319,871` 次。此值与 cycles 值的比值为 IPC（insn per cycle），表示平均每个 CPU 周期内执行的指令数，这里 IPC 值为第三列显示的 `0.73`。通常 IPC 值越高越好，值越高说明程序越充分地利用处理器。当前龙芯处理器为四发射结构，理论上 IPC 值最高可以接近4。前面介绍指令重排优化时，完全可以通过 IPC 值变化判断重排效果。

事件 branches 和 branch-misses 分别统计程序执行过程中的分支指令数量和分支预测失败的指令数量。这里显示的结果分别为 `113,415,431` 次和 `4,710,708` 次。branch-misses 值与 branches 值的比值为分支预测失败率，即 branch-misses 后面显示的 `4.15% of all branches`。分支预测失败率越高，对程序执行性能的影响越大。前面提到的循环展开技术可以减少分支指令的执行。

如果默认事件不能满足要求，可以使用 `perf stat -e event_name` 指定具体事件进行统计。例如，要查看一个程序执行过程中的一级数据缓存情况，可以使用命令 `perf stat -e L1-dcache-load-misses,L1-dcache-loads`。
``` shell
$ perf stat -e L1-dcache-load-misses,L1-dcache-loads ./a.out
1,647,471      L1-dcache-load-misses:u  #    0.01% of all L1-dcache hits
11,981,683,574      L1-dcache-loads:u
5.642483054 seconds time elapsed  
```
LoongArch 架构处理器支持硬件预取功能，因此一般程序的 Cache 未命中率不会很高。这里显示当前程序的数据 Cache 未命中率为0.01%，说明当前程序中数据加载指令具有较好的命中率。如果用户程序出现较高的 Cache 未命中率，可以进一步使用 perf record 定位问题函数，并尝试调整函数实现逻辑，或使用 LoongArch 数据预取指令进行优化。

另外，perf stat 还可以按线程监测某个程序的性能，示例命令如下：
``` shell
//监测进程ID为1318程序的指令数量
perf stat --per-thread -e branch-misses  -p 1318
```
##	perf top 的使用

可以看到，perf stat 能对程序进行概括性的统计分析，但不能精确到函数和汇编指令级别。perf top 不仅可以精确到函数和汇编指令级别进行事件性能统计，还可以实时显示性能统计结果。perf top 的命令格式如下：
``` shell
perf top [ -e <event> | --event=EVENT] [ -p <pid> ]
```
其中，参数 `-e` 或 `--event` 用于指定待监测的性能事件；如果不指定，默认监测的性能事件类型为 cycles。若要实时监测进程号为17223的程序性能，可以使用如下命令，结果如图10-2所示。
``` shell
$ perf top -p 17223
```

```{image} ../../img/cha/pic_a_2.png
:alt: perf top的实时数据
:class: bg-primary
:scale: 80 %
:align: center
```

从图10-2可以看出，perf top 以函数为单位，按照性能占比从高到低排列，占比较高（超过5.00%）的函数会被标记为红色。函数名称和函数所在库分别在第三列和第二列展示。随着程序持续运行，perf top 也会实时更新热点函数及其占比。

如果想继续查看热点函数内的热点汇编指令分布情况，可以使用鼠标单击该函数所在行进入 annotate 模式（等价于 perf 子命令 `perf annotate method_name`）。图10-3显示了最热函数 thread_f 被进一步展开后的信息。

```{image} ../../img/cha/pic_a_3.png
:alt: perf annotate后的结果数据
:class: bg-primary
:scale: 80 %
:align: center
```

##	perf record/report 的使用

perf top 实时展示系统性能信息，但不保存数据，因此无法用于后续总体性能分析。perf record 解决了这一问题。perf record 用于对一段时间内或程序全过程的性能事件做统计记录，并将结果保存在名为 perf.data 的文件中。该文件不能直接查看，需要使用 perf report 读取 perf.data 文件内容，并将分析数据显示到输出终端。它们的命令格式如下：
``` shell
perf record [ -e <event> | --event=EVENT] [ -p <pid> | start_command ]
perf report [ file_name ]
```

perf record 的默认监控事件类型也是 cycles。如需指定其他事件类型，可以使用参数 `-e` 或 `--event`。监控可以在程序启动之前开始，也可以在程序运行过程中开始（使用参数 `-p` 指定进程号）。perf report 默认加载当前目录下名为 perf.data 的性能文件；如果文件不在当前目录，或名称不是 perf.data（可能被重命名），可以通过指定文件路径和文件名加载。对现有进程（进程号为17223）进行一段时间性能监控的命令如下：
``` shell
perf record -p 17223
```
采样时间可以适当长一些，这样能更精确地定位热点函数和热点指令。采样完成后（使用 Ctrl+C 停止采样），使用命令 `perf report` 加载 perf.data 并查看其中信息，展示信息与 perf top 在图10-2中展示的内容基本一致。

对于复杂服务程序的性能定位，perf 工具能够显著提高排查效率。同样，`perf record` 后面也可以通过参数 `-e event_name` 指定事件，通过参数 `-p pid` 对已经运行的程序做监测记录。这里不再详细列举，读者可以根据自己的程序自行测试。更详细的使用方法可以通过命令 `perf record -h` 查看。

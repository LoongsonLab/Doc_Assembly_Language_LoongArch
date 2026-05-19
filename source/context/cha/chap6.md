#	性能分析工具perf

perf 是Linux平台的一款性能分析工具，能够对一个程序进行全程或者部分运行时段进行监控，实现函数级甚至指令级的性能统计和热点查找，从而帮助我们评估和定位程序的性能瓶颈。监控也是多方面的，比如程序运行的总时间、程序执行总指令数、CPU周期数、程序中分支指令总数和分支预测率、Cache命中率、程序触发缺页中断数量等，这些都被称为事件。使用命令perf list可以统计出perf支持的全部事件，实际工作中可以根据需要来选择相应的事件，部分常见事件如下：
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

perf工具支持的子命令也很多，可以通过执行perf命令查看全部子命令。本节只介绍以下3种常用的perf子命令。

-	perf stat：在程序开始时，对特定的事件计数器进行计算，在程序运行结束时把默认或者指定的事件统计结果简单地汇总并显示在标准输出上。

-	perf top：实时显示系统/进程的性能统计信息。

-	perf record/perf report：perf record用于记录一段时间内或程序全过程的性能事件，并将结果保存在perf.data文件中；而perf report用于读取perfrecord生成的perf.data文件，并显示分析数据。

##	 perf stat的使用

perf stat用于对程序进行一个概览性的性能统计。它的常用命令格式为
``` shell
perf stat [ -e <event> | --event=EVENT] [ -p <pid> | start_command ]
```
其中参数“-e”或“--event”用来指定要监测的具体事件。参数“-p”用于监测一个已经在运行的程序，后面跟的pid为此程序进程号。例如，要统计程序进程号为17223的程序在监测时间内的分支预测率，具体命令如下：
``` shell
perf stat -e branches -e branch-misses -p 17223
```
如果不指定具体监控事件，perf stat的默认监测的事件有task-clock、context-switches、cpu-migrations、page-faults、cycles、instructions、branches、branch-misses。例如使用perf stat全程监控一个名为hot的程序的性能，运行命令和结果信息如下：
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

这里显示了执行“perf stat”后默认事件的信息统计，第一列显示了每个事件的占用时间或执行次数的统计值，第二列显示了每个事件的名称，第三列为每个事件的备注信息。评价程序性能好坏最直观的就是总执行时间，即数据“52.778699178seconds time elapsed”，这代表了程序执行消耗的实际时间（从程序开始执行到完成所经历的时间）。最后两行数据“6.000671000 seconds user”和“0.000000000 seconds sys”分别统计的是此程序消耗的用户态CPU时间和内核态CPU时间。

事件task-clock统计的是此程序真正占用的处理器时间，单位为毫秒。这里统计的结果是“4,736.17”毫秒。该值与程序的总执行时间的比值即为CPU占用率，即第三列显示的“0.090 CPUs utilized”，比值越高说明程序的更多时间花费在 CPU 计算上而非 I/O上。对于密集计算型多线程程序，如果是单线程执行，此值可以接近1；如果是多线程执行，此值可接近当前处理器所用核数。

事件context-switches统计的是程序执行过程中上下文切换总次数。这里统计的结果是“412,675”次。

如果程序中执行了系统调用、进程切换等，都会触发上下文切换。该值与事件task-clock统计结果比值为单位时间内上下文切换次数，即第三列显示的“0.087 M/sec”。

perf支持的事件中有些是需要root权限的，例如事件context-switches，当权限不足时获取到的事件信息将为0。建议使用perf之前将proc/sys/kernel/perf_event_paranoid的值设置为-1，或者以root用户或管理者的身份（即sudo命令）执行perf命令。

事件cpu-migrations统计的是程序执行过程中处理器核的迁移次数。这里统计的结果是“0”次，说明程序执行过程一直在一个处理器核运行，没有发生过迁移。通常系统为了维护多个处理器之间的负载均衡，在达到一定条件后可能会将一个任务从一个处理器核迁移到另一个处理器核上。

事件page-faults统计的是程序执行过程中缺页异常发生的总次数。这里统计的结果是“45”次。该值与事件task-clock统计结果比值为单位时间内发生的缺页次数，即第三列显示的“0.010 K/sec”。

事件cycles统计的是程序执行占用的处理器周期数。这里统计的结果是“707,087,126”次。此值与事件task-clock统计结果的比值为有效主频，即第三列显示的“0.149 GHz”，远小于当前程序运行主机的处理器主频2.3GHz。

事件instructions统计的是程序执行的总指令数量。这里统计的结果是“517,319,871”次。此值和事件cycles值的比值为IPC(insn per cycle)，代表平均一个CPU周期内执行的指令数，这里IPC值为第三列显示的“0.73”。通常IPC值越高越好，值越高说明程序更充分的利用处理器。当前龙芯处理器为四发射结构，那么理论上IPC值最高可以接近4。前面在介绍指令重排优化时，完全可以通过IPC值变化来判断重排效果好坏。

事件branches和branch-misses分别统计程序执行过程中的分支指令数量和分支预测失败的指令数量。这里显示分别为“113,415,431”次和“4,710,708”次。branch-misses值与branches的比值为分支预测率，即branch-misses后面显示的“ 4.15% of allbranches ”。分支预测率越高，越影响程序的执行性能。前面提到的循环展开技术可以减少分支指令的执行。

如果默认的事件不能满足要求，可以使用“pert stat-e event_name”来指定具体事件的统计。例如要查看一个程序执行过程中的一级数据缓存情况，可以使用命令“perf stat -e L1-dcache-load-misses，L1-dcache-loads”。
``` shell
$ perf stat -e L1-dcache-load-misses,L1-dcache-loads ./a.out
1,647,471      L1-dcache-load-misses:u  #    0.01% of all L1-dcache hits
11,981,683,574      L1-dcache-loads:u
5.642483054 seconds time elapsed  
```
LoongArch架构的处理器支持硬件预取功能，故一般正常的程序的Cache 未命中率都不会很高。这里显示当前程序的数据Cache未命中率为0.01%，说明当前程序中数据加载指令都有较好的命中率，导致Cache的命中率很高。如果用户程序出现Cache 未命中率很高的情况，可以进一步使用perfrecord来定位问题函数，并尝试调整函数实现逻辑或使用LoongArch的数据预取指令尝试对其进行优化。

另外perf stat还可以按线程来监测某个程序的性能，示例命令如下：
``` shell
//监测进程ID为1318程序的指令数量
perf stat --per-thread -e branch-misses  -p 1318
```
##	perf top的使用

可以看到perf stat能对程序进行概括性的统计分析，但是不能精确到函数和汇编指令级别。perftop不仅能精确到函数和汇编指令级别的事件性能统计，还可以实时显示出性能统计结果。perf top的命令格式如下：
``` shell
perf top [ -e <event> | --event=EVENT] [ -p <pid> ]
```
其中参数“-e”或“--event”用来指定待监测的性能事件，如果不指定，默认监测的性能事件类型为cycles。如要实时监测进程号为17223的程序的性能，则使用如下命令，结果如图10-2所示。
``` shell
$perf top -p 17223
```

```{image} ../../img/cha/pic_a_2.png
:alt: perf top的实时数据
:class: bg-primary
:scale: 80 %
:align: center
```

从图10-2可以看出，perf top以函数为单位，按其性能占比情况从高到低排列，占比较高（超过5.00%）的函数会被标记为红色。函数名称和函数所在库分别在第三列和第二列展示。随着程序的持续运行，perf top也会实时更新热点函数和其占比。

如果想继续查看这个热点函数内的热点汇编指令分布情况，可以使用鼠标继续单击此函数所在行进入annotate模式（同perf子命令perf annotatemethod_name），图10-3显示了最热函数thread_f被进一步展开后的信息。

```{image} ../../img/cha/pic_a_3.png
:alt: perf annotate后的结果数据
:class: bg-primary
:scale: 80 %
:align: center
```

##	perf record/report的使用

perf top 实时展示了系统的性能信息，但它并不保存数据，所以无法用于后续总体的性能分析，perfrecord解决了这一问题。perf record用于对一段时间内或程序全过程的性能事件做统计记录，并将结果保存在名为perf.data的文件中，这个文件不能直接查看，需要使用perf report来帮助读取perf.data文件内容，并显示分析数据到输出终端。它们的命令格式如下：
``` shell
perf record [ -e <event> | --event=EVENT] [ -p <pid> | start_command ]
perf report [ file_name ]
```

perf record的默认监控事件类型也是cycles，如需指定其他事件类型，可以使用参数“-e”或者“--event”。监控可以在程序启动之前开始，也可以在程序运行过程中（使用参数“-p”指定进程号）。perf report默认加载当前目录下的名为perf.data的性能文件，如果文件不在当前目录或者名称不是perf.data（可能被重命名），可以通过指定文件路径和文件名来加载。对现有进程（进程号为17223）进行一段时间性能监控的命令如下：
``` shell
perf record -p 17223
```
采样时间可以尽可能长一些，这样能更精准定位到热点函数和热点指令。采样完成后（使用Ctrl+C停止采样）使用命令“perf report”来加载perf.data并查看其中信息，展示的信息和perf top展示的图10-2基本一致。

对于复杂的服务程序的性能定位，perf工具将起到事半功倍的效果。同样“perf record”后面也可以通过参数“-e event_name”来指定事件，通过参数“-p pid”来对已经运行的程序做监测记录，这里不详细列举，读者可以根据自己的程序自行测试。更详细的使用方法可以通过命令“perf record -h”来查看。
#	GDB 调试器的常用命令

GDB 调试器（GNU Debugger）是 Linux 平台下最常用的程序调试器之一，目前可以为 C、C++、Go、Objective-C 等多种程序设计语言提供调试支持。Linux 平台下许多带有调试功能的 C、C++ 集成开发环境（IDE），其调试核心都源自 GDB。GDB 通常以 gdb 命令的形式在终端（Shell）中使用。gdb 命令本身提供了许多选项（参数），可以帮助用户快速定位程序异常点，或监控程序执行的细节，例如异常点或断点处的寄存器值、函数调用栈信息、线程调度情况等。

##	GDB 的启动和退出

GDB 既可以以程序二进制文件名作为参数随程序一同启动，也可以以进程号为参数动态附加到正在运行的程序。启动时可以指定程序运行参数、配置文件，也可以附带 core 文件。下面列举几种 GDB 常用的启动方式。
``` shell
gdb program              //启动gdb并执行程序program
gdb program core         //启动gdb并停止到core文件中的异常位置
gdb -p 1234              //启动并绑定gdb到进程号为1234的程序上
gdb attach -p 1234       //同gdb -p 1234
gdb --args program       //同gdb program，program后面可以带命令行参数
gdb -x gdbinit program   //同gdb program，同时指定gdb配置文件
```
除上面列举的常用启动参数外，还可以使用 `gdb -h` 或 `gdb --help` 查看更详细的 GDB 参数说明。

为了更好地使用 GDB 调试程序，通常希望被调试程序的二进制文件及其依赖的动态库文件中包含符号表信息。通常情况下，为了尽量减小程序占用空间，已经发布的产品级二进制程序文件会经过瘦身处理，即剥离文件中的符号信息和调试信息（使用 Linux 命令 `file 文件名` 查看时会显示 `stripped`）。在这种情况下，GDB 调试过程中将无法看到函数名、变量名和行号等直观信息。使用 gcc/g++ 编译源码时，添加 `-g` 选项可以生成带有符号信息和调试信息的二进制文件。

一个简单的带调试信息的程序编译和 GDB 运行示例如下：
``` shell
$ gcc -g gdbtest.c -o gdbtest
$ gdb -q gdbtest
Reading symbols from gdbtest...done.
(gdb) r
Starting program: /home/sunguoyun/LABook/gdbtest
Hello World !
[Inferior 1 (process 28806) exited normally]
(gdb) q
```

这里在编译 gdbtest.c 文件时，使用 `-g` 参数通知编译器保留调试信息。GDB 启动时默认会输出版本信息、版权说明、帮助提示等内容，这里通过添加参数 `-q` 屏蔽这部分信息。GDB 启动后，可以在 `(gdb)` 后键入任何 GDB 支持的命令来控制程序执行。例如，这里键入的两个命令 `r` 和 `q` 分别表示运行程序（run 命令的缩写形式）和退出 GDB（quit 命令的缩写形式）。更多 GDB 命令可以通过在 `(gdb)` 后键入 `help` 或 `help all` 查看。GDB 支持的命令很多，本章仅围绕工作中常用的 gdb 调试功能展开，例如程序断点设置、单步调试、查看堆栈信息、查看寄存器信息等。

##	断点设置

通常在 GDB 启动后进行断点设置。程序断点可以让 GDB 在程序执行到指定位置（如某行、某个函数、某个地址）时暂停程序，等待用户进一步处理。GDB 支持在程序中设置3种类型的断点：break 断点（又称程序断点）、watch 断点（又称数据断点）和 catch 断点（又称事件断点）。break 断点可以让程序执行到指定行或指定函数位置时暂停，是最常用的断点类型。watch 断点用于监视某个数据变量的变化，当指定数据变量或内存地址单元被修改时，程序暂停。catch 断点用于捕获程序执行期间产生的指定事件，例如 assert、exception、syscall、signal、fork 等。下面分别介绍这3种断点的使用方法。

1.	break 断点设置

GDB 中 break 断点设置的相关命令如表9-1所示。

```{image} ../../img/ch9/t2p_9_1.png
:alt: break 断点设置的相关命令
:class: bg-primary
:scale: 80 %
:align: center
```

表9-1中列出了5个 break 断点设置相关命令。其中，break、tbreak 和 rbreak 属于软件断点，用于一般程序的断点设置；hbreak 和 thbreak 属于硬件断点，主要用于调试位于 EPROM/ROM 上的代码。break 命令的缩写为 `b`。参数 LOCATION 可以是行号、函数名或具体内存地址。如果没有指定 LOCATION，则默认为当前栈帧的 PC 值。使用选项 `thread THREADNUM` 可以将断点设置到某一个线程，其中线程号 THREADNUM 可以通过命令 `info threads` 查看并获得。选项 `if CONDITION` 用于设置条件断点，即当条件表达式 CONDITION 的值为真时，断点才会生效。这对调试某个变量为特定值，或调试循环到指定次数的情况很有用。下面列举几种常用的 break 设置命令：
``` shell
b a.c:4                //在源C语言文件a.c的第4行设置断点
b main                 //在函数main入口处设置断点
b a.c:add              //在源C语言文件a.c的函数add入口处设置断点
b *0x120000774         //在地址0x120000774处设置断点
b a.c:21 if out == 20  //条件断点，即当变量等于20时，程序在a.c中的21行处暂停
b a.c:21 thread 1      //在文件a.c的21行设置断点，仅对Num为1的线程起效
```
命令 tbreak（缩写为 tb）和 rbreak（缩写为 rb）的用法与 break 类似。区别在于，tbreak 表示临时断点，即该断点只生效一次；rbreak 用于对满足匹配规则的所有函数设置断点。使用示例如下：
``` shell
tbreak a.c:21     //在a.c中的21行设置断点，此断点只生效一次
ignore 1 10       //跳过（忽略）1号断点的前10次执行。1为断点号
rbreak .          //对程序中所有函数设置断点
rbreak a.c::.     //仅对a.c文件中的所有函数设置断点
rbreak add*       //对程序中所有以add为前缀的函数设置断点
```
硬件断点 hbreak（缩写为 hb）和 thbreak（缩写为 thb）的用法也与 break 类似，这里不再举例。thbreak 也称硬件临时断点，即只生效一次。

断点设置后，可以使用命令 `info break` 或 `info b` 查看当前程序已经设置的所有断点信息。下面通过一个具体示例演示 break 的使用。具体的C语言程序如下：
``` text
/* gdbtest.c
*  gcc -g gdbtest.c -o gdbtest
*/
1  #include <stdio.h>
2  int main (int argc, char *argv[])
3  {
4     printf("Hello World ! argc=%d\n", argc);
5     for(int i=0; i<argc;i++){
6        printf("%s\n",argv[i]);
7     }
8     return 0;
9 }
```
对这个程序使用 break 调试的信息如下：
``` shell
$ gdb gdbtest -q
Reading symbols from gdbtest...done.
(gdb)         -->设置断点为源码的第4行
Breakpoint 1 at 0x120000728: file gdbtest.c, line 4.
(gdb) info b  -->查看断点设置信息
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000120000728 in main at gdbtest.c:4
(gdb) r       -->运行程序
Starting program: /home/sunguoyun/LABook/gdbtest
Breakpoint 1, main (argc=1, argv=0xffffff3428) at gdbtest.c:4
4     printf("Hello World ! argc=%d\n", argc);-->程序运行到第4行时暂停
(gdb) c       -->继续程序的执行
Continuing.
Hello World ! argc=1
/home/sunguoyun/LABook/gdbtest
[Inferior 1 (process 31481) exited normally]-->程序执行完毕
(gdb) q       -->退出gdb
```

clear 命令可以删除指定位置的所有断点，参数 location 通常为某一行代码的行号或某个具体函数名。当参数 location 为某个函数名时，表示删除位于该函数入口处的所有断点。

delete 命令（缩写形式为 d）可以删除指定编号的断点或全部断点，其参数 num 为指定断点的编号。当未指定 num 时，delete 命令会删除当前程序中存在的所有断点。

禁用断点可以使用 disable 命令，其参数 `num1 num2 ...` 表示一次可以禁用多个断点。例如，`disable 1` 表示禁用编号为1的断点，`disable 1 2 3` 表示禁用编号分别为1、2和3的断点；当没有指定编号值时，disable 表示禁用当前程序的所有断点。对于被禁用的断点，可以使用 enable 命令重新启用，其使用方式与 disable 相同。删除和禁用断点的示例如下：
``` shell
(gdb) info b           -->显示当前共有3个断点
Num     Type           Disp Enb           What
1       breakpoint     keep y   0x0000000120000774 in main at gdbtest.c:4
2       breakpoint     keep y   0x0000000120000788 in main at gdbtest.c:5
3       breakpoint     keep y   0x0000000120000790 in main at gdbtest.c:6
(gdb) disable 2      -->禁用编号为2的断点，对应的Enb显示为n
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000120000774 in main at gdbtest.c:4
2       breakpoint     keep n   0x0000000120000788 in main at gdbtest.c:5
3       breakpoint     keep y   0x0000000120000790 in main at gdbtest.c:6
(gdb) delete 1       -->删除编号为1的断点
(gdb) info b
Num     Type           Disp Enb Address            What
2       breakpoint     keep n   0x0000000120000788 in main at gdbtest.c:5
3       breakpoint     keep y   0x0000000120000790 in main at gdbtest.c:6
(gdb) enable 2       -->重新启用编号为2的断点
(gdb) info b
Num     Type           Disp Enb Address            What
2       breakpoint     keep y   0x0000000120000788 in main at gdbtest.c:5
3       breakpoint     keep y   0x0000000120000790 in main at gdbtest.c:6
(gdb) 
```

在使用 GDB 调试程序的过程中，可以借助 watch 断点监控程序中某个变量或表达式的值。只要该值发生改变，程序就会停止执行。这对于定位某个变量或内存单元遭到非法篡改的问题很有帮助。与 watch 断点设置相关的命令如下：
``` shell
watch a                  //对变量a设置断点。仅当a发生写变化（被修改）时，程序暂停
watch *(int*)0x120008064 //对地址0x120008064设置断点，当此地址内的4字节发生写变化时，程序暂停
watch a thread 2         //对变量a设置断点，仅当a在线程2中发生写变化时，程序暂停
rwatch a                 //对变量a设置断点。仅当a发生读变化时，程序暂停
awatch a                 //对变量a设置断点。当a发生读或者写变化时，程序暂停
info watch               //查看当前程序设置的所有watch断点
info b                   //查看当前程序设置的所有break断点和watch断点
info thread              //查看当前程序的所有线程信息
```
watch 断点和 break 断点使用相同的删除命令 clear 或 delete。下面通过一个C语言示例演示 watch 断点的使用。
``` c
/* gdbtest.c
*  gcc -g gdbtest.c -o gdbtest
*/
#include <stdio.h>
int tt;
int main (int argc, char *argv[])
{
	for(int i=0; i<3; i++){
		tt = i;
	}
	return 0;
}
```
使用 watch 断点观测变量 tt 值变化的方式如下：
``` shell
$ gdb gdbtest -q
Reading symbols from gdbtest...done.
(gdb) watch tt   -->设置观测点
Hardware watchpoint 1: tt
(gdb) info watch
Num     Type           Disp Enb Address    What
1       hw watchpoint  keep y              tt
(gdb) r
Starting program: /home/sunguoyun/LABook/gdbtest
Hello World ! argc=1
/home/sunguoyun/LABook/gdbtest
Hardware watchpoint 1: tt-->变量tt的值发生变化
Old value = 0
New value = 1
main (argc=1, argv=0xffffff3428) at gdbtest.c:9
9     for(int i=0; i<3; i++){
(gdb) c
Continuing.
Hardware watchpoint 1: tt-->变量tt的值再次发生变化
Old value = 1
New value = 2
main (argc=1, argv=0xffffff3428) at gdbtest.c:9
9     for(int i=0; i<3; i++){
(gdb)
```
在程序运行之前或运行过程中，都可以设置 watch 断点。这里是在程序运行之前对变量 tt 设置 watch 断点。通过 `info watch` 可以查看当前程序已经设置的 watch 断点信息。watch 的实现一般需要处理器硬件支持。从上面的信息可以看出，龙芯处理器硬件支持 watch 断点。

与 watch 相似的另外两个观察断点命令为 rwatch 和 awatch。区别在于，watch 用于观察某个变量或内存值的写变化（即其值被修改），rwatch 用于观察某个变量或内存值的读变化（即其值被使用但未被修改），而 awatch 用于观察某个变量或内存值的读/写变化（即其值被使用或被修改都会被捕获）。

3.	catch 断点设置

catch 断点的作用是监控程序中某一事件的发生，例如程序发生某种异常、某一动态库被加载等。一旦目标事件发生，程序就会暂停执行。catch 断点的设置方式如下：
``` shell
tcatch event
```
参数 event 表示要监控的具体事件。catch 常用的 event 事件类型如表9-2所示。

```{image} ../../img/ch9/t2p_9_2.png
:alt: Catch常用的event事件类型
:class: bg-primary
:scale: 80 %
:align: center
```

下面列举几种 catch 断点的设置方式：
``` shell
catch signal SIGBUS   //捕获SIGBUS事件，当此事件发生时程序暂停
tcatch signal SIGBUS  //仅捕获SIGBUS事件一次
catch signal all      //捕获所有信号事件，当此事件发生时程序暂停
catch syscall chroot  //捕获系统调用chroot，当此接口被调用时程序暂停
catch syscall         //捕获所有系统调用
info break            //查看所有break、watch和catch断点信息
delete 1              //删除Num为1的断点。此断点可以是break、watch或catch断点
```
例如，要捕获程序运行时动态库加载的事件，具体示例如下：
``` shell
(gdb) catch load      -->捕获动态库加载事件的断点设置
Catchpoint 4 (load)
(gdb) r              -->启动程序
Starting program: /home/sunguoyun/c-test/gdbtest
Catchpoint 4         -->捕获动态库加载事件，程序暂停
Inferior loaded /lib/loongarch64-linux-gnu/libc.so.6
0x000000fff7fe0050 in _dl_debug_state () from /lib64/ld.so.1
(gdb) 
```

##	查看变量、内存数据和寄存器信息

1.	print/display命令

当程序执行被 GDB 暂停到某个断点处时，可以通过 print 命令或 display 命令查看某个变量或表达式的值。其中，print 命令可以缩写为 `p`。print 和 display 命令的常用格式如下：
``` shell
p variable
p file::variable
print function::variable
display variable
display file::variable
display function::variable
```
参数 variable 用于指示要查看或修改的目标变量。当程序中包含多个作用域不同但名称相同的变量或表达式时，可以在变量前面添加文件名（file::variable）或函数名（function::variable）。

display 命令也用于在调试阶段查看某个变量或表达式的值。它与 print 命令的区别在于，使用 display 命令查看变量或表达式的值后，每当程序暂停执行（例如单步执行）时，GDB 都会自动输出该值。

2.	info register命令

此命令可以在程序暂停在某个断点时，查看一个、多个或所有寄存器的信息。下面列出的命令都是查看寄存器信息的有效方式。
``` shell
info register r4            //查看寄存器r4的值
info register r4  r5        //查看寄存器r4和r5的值
info all-registers          //查看所有通用寄存器、浮点寄存器、向量寄存器的值
i r r4                      //查看寄存器r4的值
i r a0                      //查看寄存器a0（即r4）的值
i r r4 r5                   //查看寄存器r4和r5的值
i r f0                      //查看浮点寄存器f0的值
i r                         //查看所有通用寄存器、pc、badvaddr的值
i all-r                     //查看所有通用寄存器、浮点寄存器、向量寄存器的值
```

下面以一个具体示例来介绍查看寄存器信息的方法。使用的C语言程序如下：
``` text
1  /* gdbtest.c */
2   #include <stdio.h>
3   int add( int a, int b) {
4     return a+b;
5  }
6
7  int main (int argc, char *argv[]) {
8    add(1, 2);
9    return 0;
10}
```
调试命令信息如下：
``` shell
$ gdb gdbtest -q
Reading symbols from gdbtest...done.
(gdb) b add    -->设置断点到函数add入口处
Breakpoint 1 at 0x120000674: file gdbtest.c, line 4.
(gdb) r
Starting program: /home/sunguoyun/LABook/gdbtest
Breakpoint 1, add (a=1, b=2) at gdbtest.c:4
4     return a+b;
(gdb) i r a0   -->查看寄存器a0的值，分别显示十六进制和十进制
a0             0x1                 1
(gdb) i r a1   -->查看寄存器a1的值
a1             0x2                 2
(gdb) i r     -->查看所有通用寄存器的值
zero               ra               tp               sp
R0   0000000000000000 00000001200006bc 000000fff7ffefe0 000000ffffff32c0
a0               a1               a2               a3
R4   0000000000000001 0000000000000002 000000ffffff3438 000000fff7fb04b0
a4               a5               a6               a7
R8   0000000000000000 000000fff7fe6ea8 000000ffffff3420 0000000000008000
t0               t1               t2               t3
R12  0000000000000002 0000000000000001 0000000000000000 000000fff7fb2eb8
t4               t5               t6               t7
R16  000000fff7fb1d40 000000fff7fb1d40 7f7f7f7f7f7f7f7f 0000000000000000
t8                x               fp               s0
R20  ffff000000000000 0000000000000000 000000ffffff32e0 0000000000000000
s1               s2               s3               s4
R24  00000001200006d8 000000fff7ffb8e8 0000000000000000 0000000120131c50
s5               s6               s7               s8
R28  000000012012f180 000000012011a818 0000000000000000 0000000000000000
pc             0x120000674         0x120000674 <add+36>
badvaddr       0xfff64c4008        0xfff64c4008
(gdb) 
```
这里使用命令 `b add` 将断点设置在 add 函数的起始位置，然后使用命令 `r` 运行程序并停止在函数 add 入口处。从源程序可以看出，函数 add 有两个参数，分别为 int a 和 int b。根据 LoongArch ABI 的函数调用传参规则，调用函数 add 时的参数值1和2分别使用寄存器 a0、a1 传递，因此这里使用命令 `i r a0` 和 `i r a1` 查看寄存器 a0 和 a1 的值，结果分别为1和2。

当然，也可以使用 `i r` 查看 LoongArch 架构中32个通用寄存器的值，以及当前程序寄存器 pc 和 badvaddr 的值。

如果还要查看浮点寄存器或向量寄存器的值，可以使用 `i all-r` 命令。该命令显示的信息较多，这里不做展示。

3.	disassemble命令

使用 disassemble 命令可以查看（也称为反汇编）指定函数或指定地址范围内的汇编指令。其缩写命令为 disass。具体使用方式有如下几种：
``` shell
disass                //查看当前断点所在函数对应的汇编指令
disass func_name      //查看指定函数名为func_name的函数对应汇编指令
disass addr           //查看指定地址addr所在函数对应汇编指令
disass addr1,addr2    //查看指定地址addr1和addr2范围内的汇编指令
```
下面仍以 gdbtest 程序为例演示 disassemble 命令的使用。
``` shell
$ gdb gdbtest -q
Reading symbols from gdbtest...done.
(gdb) b add
Breakpoint 1 at 0x120000674: file gdbtest.c, line 4.
(gdb) r
Starting program: /home/sunguoyun/LABook/gdbtest
Breakpoint 1, add (a=1, b=2) at gdbtest.c:4
4     return a+b;
(gdb) disass
Dump of assembler code for function add:
0x0000000120000650 <+0>: addi.d $r3,$r3,-32(0xfe0)
0x0000000120000654 <+4>: st.d   $r22,$r3,24(0x18)
0x0000000120000658 <+8>: addi.d $r22,$r3,32(0x20)
0x000000012000065c <+12>: move  $r13,$r4
0x0000000120000660 <+16>: move  $r12,$r5
0x0000000120000664 <+20>: slli.w $r13,$r13,0x0
0x0000000120000668 <+24>: st.w   $r13,$r22,-20(0xfec)
0x000000012000066c <+28>: slli.w $r12,$r12,0x0
0x0000000120000670 <+32>: st.w   $r12,$r22,-24(0xfe8)
=> 0x0000000120000674 <+36>: ld.w $r13,$r22,-20(0xfec)
0x0000000120000678 <+40>: ld.w   $r12,$r22,-24(0xfe8)
0x000000012000067c <+44>: add.w  $r12,$r13,$r12
0x0000000120000680 <+48>: move   $r4,$r12
0x0000000120000684 <+52>: ld.d   $r22,$r3,24(0x18)
0x0000000120000688 <+56>: addi.d $r3,$r3,32(0x20)
0x000000012000068c <+60>: jirl   $r0,$r1,0
End of assembler dump.
(gdb) 
```
因为程序运行之前使用命令 `b add` 将断点设置在函数 add 上，所以程序执行到函数 add 处停止。使用 `disass` 命令反汇编得到的是函数 add 对应的全部汇编指令信息。

同时，通过当前程序 pc 所在位置 `=> 0x0000000120000674 <+36>` 可以看出，break 命令设置函数断点时，断点位置在程序栈构建之后，而不是函数入口的第一条指令。
``` shell
(gdb) disass main
Dump of assembler code for function main:
0x0000000120000690 <+0>: addi.d  $r3,$r3,-32(0xfe0)
0x0000000120000694 <+4>: st.d    $r1,$r3,24(0x18)
0x0000000120000698 <+8>: st.d    $r22,$r3,16(0x10)
0x000000012000069c <+12>: addi.d $r22,$r3,32(0x20)
0x00000001200006a0 <+16>: move   $r12,$r4
0x00000001200006a4 <+20>: st.d   $r5,$r22,-32(0xfe0)
0x00000001200006a8 <+24>: slli.w $r12,$r12,0x0
0x00000001200006ac <+28>: st.w   $r12,$r22,-20(0xfec)
0x00000001200006b0 <+32>: addi.w $r5,$r0,2(0x2)
0x00000001200006b4 <+36>: addi.w $r4,$r0,1(0x1)
0x00000001200006b8 <+40>: bl -104(0xfffff98) # 0x120000650 <add>
0x00000001200006bc <+44>: move   $r12,$r0
0x00000001200006c0 <+48>: move   $r4,$r12
0x00000001200006c4 <+52>: ld.d   $r1,$r3,24(0x18)
0x00000001200006c8 <+56>: ld.d   $r22,$r3,16(0x10)
0x00000001200006cc <+60>: addi.d $r3,$r3,32(0x20)
0x00000001200006d0 <+64>: jirl  $r0,$r1,0
End of assembler dump.
(gdb) 
```
若要仅显示当前 $pc 附近的前4条和后4条汇编指令，可以使用如下命令：
``` shell
(gdb) disass $pc-16, $pc+16
Dump of assembler code from 0x120000664 to 0x120000684:
0x0000000120000664 <add+20>: slli.w $r13,$r13,0x0
0x0000000120000668 <add+24>: st.w   $r13,$r22,-20(0xfec)
0x000000012000066c <add+28>: slli.w $r12,$r12,0x0
0x0000000120000670 <add+32>: st.w   $r12,$r22,-24(0xfe8)
=> 0x0000000120000674 <add+36>: ld.w $r13,$r22,-20(0xfec)
0x0000000120000678 <add+40>: ld.w   $r12,$r22,-24(0xfe8)
0x000000012000067c <add+44>: add.w  $r12,$r13,$r12
0x0000000120000680 <add+48>: move   $r4,$r12
End of assembler dump.
(gdb) 
```

4.	x 命令

前面介绍的 display 命令可以查看程序中某个变量或表达式的值，但不能查看指定内存地址中的数据值。GDB 提供了查看内存的命令 x，可用于查看指定内存地址上的数据，并可指定数据格式。x 命令的格式如下：
``` shell
x/FMT 	ADDRESS
```
参数 FMT 由内存单元数量、显示格式和内存单元长度组成。内存单元数量为整数，不指定时默认值为1；显示格式有多种，具体如下所示。

-	x(hex)：按十六进制格式显示变量。

-	d(decimal)：按十进制格式显示变量。

-	u(unsigned decimal)：按十进制格式显示无符号整型。

-	o(octal)：按八进制格式显示变量。

-	t(binary)：按二进制格式显示变量。

-	a(address)：按十六进制格式显示地址。

-	i(instruction)：指令地址格式。

-	c(char)：按字符格式显示变量。

-	f(float)：按浮点数格式显示变量。

-	s(string)：按字符串格式显示。

内存单元长度可由4个字母指定：b 表示单字节，h 表示双字节，w 表示4字节，g 表示8字节；不指定时默认值为 w。

参数 ADDRESS 为一个内存地址，可以是绝对地址（如 0x12000006c），也可以是基于当前 pc 的相对地址（如 $pc-4，表示当前程序暂停位置之前4字节的内存地址）。

以下面的C语言程序为例演示 x 命令的使用。

``` text
/* gdbtest.c */
1 #include <stdio.h>
2 int out = 0;
3
4 int main (int argc, char *argv[]) {
5     out += 3;
6     return 0;
7 } 
```
``` shell
$ gdb -q ./gdbtest
Reading symbols from ./gdbtest...done.
(gdb) b main                   -->在main函数设置断点
Breakpoint 1 at 0x1200006b4: file gdbtest.c, line 5.
(gdb) r                        -->程序运行
Starting program: /home/sunguoyun/c-test/gdbtest
Breakpoint 1, main (argc=1, argv=0xffffff73f8) at gdbtest.c:5
5     out += 3;
(gdb) x/10i $pc                -->查看pc位置开始的10条汇编指令
=> 0x1200006b4 <main+28>: pcaddu12i $r12,8(0x8)
0x1200006b8 <main+32>: addi.d    $r12,$r12,-1640(0x998)
0x1200006bc <main+36>: ldptr.w   $r12,$r12,0
0x1200006c0 <main+40>: addi.w    $r12,$r12,3(0x3)
0x1200006c4 <main+44>: move      $r13,$r12
0x1200006c8 <main+48>: pcaddu12i $r12,8(0x8)
0x1200006cc <main+52>: addi.d    $r12,$r12,-1660(0x984)
0x1200006d0 <main+56>: stptr.w   $r13,$r12,0
0x1200006d4 <main+60>: move      $r12,$r0
0x1200006d8 <main+64>: move      $r4,$r12
(gdb) b *0x1200006d4          -->在地址0x1200006d4处设置断点
Breakpoint 2 at 0x1200006d4: file gdbtest.c, line 5.
(gdb) c                       -->继续程序执行
Continuing.
Breakpoint 2, 0x1200006d4 in main (argc=1, argv=0xffffff73f8) at gdbtest.c:5
(gdb) i r r12 r13             -->查看寄存器r12 和r13的值
r12            0x12000804c         4831871052
r13            0x3                 3
(gdb) x/1d 0x12000804c        -->查看地址0x12000804c一个十进制值
0x12000804c <out>:  3         -->即变量out值
(gdb) 
```

##	查看堆栈信息

1.	backtrace命令

backtrace 命令用于查看当前被调试程序的函数栈信息，以直观显示函数间的调用关系，其缩写命令为 `bt`。具体语法格式如下。
``` shell
backtrace [QUALIFIERS] [COUNT]
```

其中，参数 QUALIFIERS 为可选项，其值可以为 `full` 或 `no-filters`，分别表示输出局部变量的值和禁止执行帧筛选器。参数 COUNT 也为可选项，其值为整数。当值为正整数 n 时，表示输出最里层的 n 个栈帧信息；当值为负整数时，表示输出最外层 n 个栈帧信息；当没有 COUNT 参数时，backtrace 会显示完整的栈帧信息。

以下面C语言程序为例演示 bt 命令的使用。

``` c
/* gdbtest.c */
#include <stdio.h>
int add3( int a, int b) {
return a+b;
}
int add2( int a, int b) {
return add3(a,b);
}
int add1( int a, int b) {
return add2(a,b);
}
int add( int a, int b) {
return add1(a,b);
}
int main (int argc, char *argv[]) {
add(1, 2);
return 0;
}
```
程序运行到函数 add3 时的堆栈信息如下：
``` shell
$ gdb gdbtest -q
Reading symbols from gdbtest...done.
(gdb) b add3
Breakpoint 1 at 0x120000674: file gdbtest.c, line 5.
(gdb) r
Starting program: /home/sunguoyun/LABook/gdbtest
Breakpoint 1, add3 (a=1, b=2) at gdbtest.c:5
5     return a+b;
(gdb) bt
#0  add3 (a=1, b=2) at gdbtest.c:5
#1  0x00000001200006cc in add2 (a=1, b=2) at gdbtest.c:8
#2  0x0000000120000720 in add1 (a=1, b=2) at gdbtest.c:11
#3  0x0000000120000774 in add (a=1, b=2) at gdbtest.c:14
#4  0x00000001200007b8 in main (argc=1, argv=0xffffff3428) at gdbtest.c:18
(gdb) bt 2
#0  add3 (a=1, b=2) at gdbtest.c:5
#1  0x00000001200006cc in add2 (a=1, b=2) at gdbtest.c:8
(More stack frames follow...)
(gdb) bt -2
#3  0x0000000120000774 in add (a=1, b=2) at gdbtest.c:14
#4  0x00000001200007b8 in main (argc=1, argv=0xffffff3428) at gdbtest.c:18
(gdb)
```

2.	frame命令

如果要查看 backtrace 结果中某一层的栈帧信息，可以使用 frame 命令，其缩写为 `f`，完整命令形式如下：
``` shell
frame [frame_num|frame_addr]
```
参数可以是栈帧编号（frame_num）或栈帧地址（frame_addr）。当不指定任何参数时，frame 命令将显示 backtrace 结果中最顶层函数的栈帧。同样以 gdbtest 程序为例，其 frame 信息如下：
``` shell
$ gdb gdbtest -q
Reading symbols from gdbtest...done.
(gdb) b add3
Breakpoint 1 at 0x120000674: file gdbtest.c, line 5.
(gdb) r
Starting program: /home/sunguoyun/LABook/gdbtest
Breakpoint 1, add3 (a=1, b=2) at gdbtest.c:5
5     return a+b;
(gdb) info f
Stack level 0, frame at 0xffffff3280:
pc = 0x120000674 in add3 (gdbtest.c:5); saved pc = 0x1200006cc
called by frame at 0xffffff32a0
source language c.
Arglist at 0xffffff3280, args: a=1, b=2
Locals at 0xffffff3280, Previous frame’s sp is 0xffffff3280
Saved registers:
fp at 0xffffff3278
(gdb) f         -->显示最顶层（即断点处对应函数）的栈信息
#0  add3 (a=1, b=2) at gdbtest.c:5
5     return a+b;
(gdb) f 1       -->显示编号为1的栈信息
#1  0x00000001200006cc in add2 (a=1, b=2) at gdbtest.c:8
8     return add3(a,b);
(gdb) 
```

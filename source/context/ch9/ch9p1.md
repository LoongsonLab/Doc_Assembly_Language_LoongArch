#	GDB 调试器的常用命令

GDB调试器(GNU Debugger)是 Linux 平台下最常用的一款程序调试器，目前可以对C、C++、Go、Objective-C等多种程序设计语言提供调试支持。Linux 平台下的很多带调试功能的 C、C++ 代码的集成开发工具(IDE)，其核心都源自 GDB 调试器。GDB 调试器通常以 gdb 命令的形式在终端(Shell)中使用。gdb命令本身有很多选项（参数），可以帮助我们快速定位到程序异常点，或监控程序执行的每一个细节，例如异常点或断点处的寄存器值、函数调用栈信息、线程调度等。

##	 GDB的启动和退出

GDB既可以以程序二进制名称作为参数随程序一同启动，也可以以进程号为参数动态启动。启动时可以指定程序运行参数或指定配置参数，还可以附带core文件。下面列举几种GDB常用的启动方式。
``` shell
gdb program              //启动gdb并执行程序program
gdb program core         //启动gdb并停止到core文件中的异常位置
gdb -p 1234              //启动并绑定gdb到进程号为1234的程序上
gdb attach -p 1234       //同gdb -p 1234
gdb --args program       //同gdb program，program后面可以带命令行参数
gdb -x gdbinit program   //同gdb program，同时指定gdb配置文件
```
除了上面列举的常用启动参数，可以使用“gdb -h”或“gdb --help”来查看更详细的GDB参数选择。

为了更好地使用gdb调试程序，我们希望被调试程序的二进制文件及其依赖的一些动态库文件中包含符号表信息。而通常情况下，为了让程序占用空间尽量的小，已经发布的产品级的二进制程序文件都是经过瘦身的，即已经剥离了文件中的符号信息和调试信息（当使用Linux命令“file 文件名”查看时会显示“stripped”），这种情况下GDB调试过程将看不到函数名、变量名和行号等直观信息。当使用gcc/g++编译源码时，带上参数-g 选项可以生成带有符号信息和调试信息的二进制文件。

一个简单的带调试信息的程序编译和GDB运行的例子如下:
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

这里在编译gdbtest.c文件时，使用了“-g”参数来通知编译器保留调试信息。GDB启动时默认会输出GDB版本信息、版权说明、帮助提示等，这里通过添加参数“-q”将这部分信息屏蔽掉。GDB启动后，我们就可以在(gdb)后面键入任何GDB支持的命令来控制程序的执行。例如这里键入了两个命令“r”和“q”，分别代表通知程序运行（run命令的缩写形式）和退出（quit命令的缩写形式）GDB。GDB支持的更多命令可以通过在(gdb)后面键入“help”或“help all”来查看。GDB支持的命令很多，本章仅围绕工作中常用的gdb命令的调试功能进行讲解，例如程序断点设置、单步调试、查看堆栈信息、查看寄存器信息等。

##	断点设置

通常我们会在GDB启动后进行断点设置。程序断点的设置可以让GDB通知程序执行到指定位置（如某行、某个函数、某个地址）处暂停下来，等待我们进一步的处理。GDB调试器支持在程序中设置3种类型断点：break断点（又称为程序断点）、watch断点（又称为数据断点）和catch断点（又称为事件断点）。break断点可以让程序执行到指定行或者指定函数位置时暂停下来，也是最常用的断点类型。watch断点用于监视某个数据变量的变化，当指定数据变量或内存地址单元被修改时，程序暂停。catch断点用于捕获程序执行期间产生的指定事件，例如assert、exception、syscall、signal、fork等。下面分别介绍这3种断点的使用方法。

1.	break断点设置

GDB中break断点设置的相关命令如表9-1所示。

```{image} ../../img/ch9/t2p_9_1.png
:alt: break断点设置的相关命令
:class: bg-primary
:scale: 80 %
:align: center
```

在表9-1中，break断点设置的相关命令有5个，其中命令break、tbreak和rbreak被称为软件断点，用于一般程序的断点设置。hbreak和thbreak被称为硬件中断，主要是针对位于EPROM/ROM上的代码调试。设置命令break的缩写命令为“b”。其中，参数LOCATION可以为行号、函数名或者一个具体的内存地址。如果没有指定LOCATION，默认为当前栈帧的PC值。使用选项thread THREADNUM可以设置断点到某一个线程，其中线程号THREADNUM可以通过命令“info threads”查看并获得。选项if CONDITION用于带条件的断点设置，即当条件表达式CONDITION的值为真时，断点才会起效。这对调试某个变量为特定值或者调试循环到指定次数的情况很有用。下面列举几种常用的break设置命令：
``` shell
b a.c:4                //在源C语言文件a.c的第4行设置断点
b main                 //在函数main入口处设置断点
b a.c:add              //在源C语言文件a.c的函数add入口处设置断点
b *0x120000774         //在地址0x120000774处设置断点
b a.c:21 if out == 20  //条件断点，即当变量等于20时，程序在a.c中的21行处暂停
b a.c:21 thread 1      //在文件a.c的21行设置断点，仅对Num为1的线程起效
```
命令tbreak（缩写为tb）和rbreak（缩写为rb）的用法和break类似。区别是tbreak表示临时断点，即此断点只生效一次。rbreak用于对满足匹配规则的所有函数设置断点。使用示例如下：
``` shell
tbreak a.c:21     //在a.c中的21行设置断点，此断点只生效一次
ignore 1 10       //跳过（忽略）1号断点的前10次执行。1为断点号
rbreak .          //对程序中所有函数设置断点
rbreak a.c::.     //仅对a.c文件中的所有函数设置断点
rbreak add*       //对程序中所有以add为前缀的函数设置断点
```
硬件断点hbreak（缩写为hb）和thbreak（缩写为thb）的用法也同break，故不再举例。thbreak也称硬件临时断点，即只生效一次。

断点设置后，我们可以使用命令“info break”或“info b”来查看当前程序已经设置的所有断点信息。下面通过一个具体示例来演示break的使用。具体的C语言程序如下：
``` c
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
对这个程序使用break调试信息如下：
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

clear 命令可以删除指定位置的所有断点，参数location 通常为某一行代码的行号或者某个具体的函数名。当参数 location 为某个函数的函数名时，表示删除位于该函数入口处的所有断点。

delete 命令（缩写形式为 d）可以删除指定编号的断点或全部断点，其参数num为指定断点的编号。当 num没有指定时， delete 命令会删除当前程序中存在的所有断点。

禁用断点可以使用 disable 命令，其参数num1num2…表示一次可以禁用多个断点。例如“disable1”表示禁用编号值为1的断点，“disable 1 2 3”禁用编号值分别为1、2和3的断点；当没有指定编号值时，disable表示禁用当前程序的所有断点。对于禁用的断点，可以使用enable 命令使能，其使用方式同disable。删除和禁用断点的示例如下：
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

在使用 GDB调试程序的过程中，借助watch断点可以监控程序中某个变量或者表达式的值，只要此值发生改变，程序就会停止执行。这对于定位某个变量或内存单元遭到非法篡改的程序时很有帮助。和watch断点设置相关的命令如下：
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
watch断点和break断点使用相同的删除命令clear或者delete。下面通过一个C语言示例演示watch断点的使用。
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
使用watch断点来观测变量tt的值变化的方式如下：
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
在程序运行之前或者运行过程中，都可以进行watch断点的设置。这里是在程序运行之前对变量tt进行 watch断点设置。通过“info watch”可以查看当前程序已经设置的watch断点信息。watch的实现一般需要处理器硬件支持。从上面的信息可以看出，龙芯处理器硬件支持watch断点。

和watch相似的另外两个观察断点命令为rwatch和awatch，区别在于watch用于观察某个变量或内存值的写变化（即其值被修改），rwatch用于观察某个变量或内存值的读变化（即其值被使用但是未被修改），而awatch用于观察某个变量或内存值的读/写变化（即其值被使用或者被修改都会被捕获）。

3.	catch断点设置

catch断点的作用是监控程序中某一事件的发生，例如程序发生某种异常、某一动态库被加载等，一旦目标事件发生，则程序暂停执行。catch断点的设置方式如下：
``` shell
tcache event
```
参数event表示要监控的具体事件。catch常用的event 事件类型如表9-2所示。

```{image} ../../img/ch9/t2p_9_2.png
:alt: Catch常用的event事件类型
:class: bg-primary
:scale: 80 %
:align: center
```

下面列举一个catch断点的设置方式：
``` shell
catch signal SIGBUS   //捕获SIGBUS事件，当此事件发生时程序暂停
tcatch signal SIGBUS  //仅捕获SIGBUS事件一次
catch signal all      //捕获所有信号事件，当此事件发生时程序暂停
catch syscall chroot  //捕获系统调用chroot，当此接口被调用时程序暂停
catch syscall         //捕获所有系统调用
info break            //查看所有的break、watch和catch断点信息
delete 1              //删除Num为1的断点。此断点可以是break、watch或catch断点
```
例如要捕获程序运行时动态库加载的事件，具体示例如下：
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

当程序执行被GDB暂停到某个断点处时，我们可以通过print 命令或 display命令来查看某个变量或表达式的值。其中print 命令可以缩写为“p”。print和display命令常用的格式如下：
``` shell
p variable
p file::variable
print function::variable
display variable
display file::variable
display functon::variable
```
参数variable用来指示要查看或者修改的目标变量。当程序中包含多个作用域不同但名称相同的变量或表达式时，可以在变量前面添加文件名称(file::variable)或者函数名称(functon::variable)。

display 命令也用于调试阶段查看某个变量或表达式的值，它和print命令的区别在于，使用 display命令查看变量或表达式的值，每当程序暂停执行（例如单步执行）时，GDB都会自动输出。

2.	 info register命令

此命令可以在程序暂停在某个断点时，查看一个、多个或所有寄存器的信息。下面列出的命令都是查看寄存器信息的有效方式。
``` shell
info register r4            //查看寄存器r4的值
info register r4  r5        //查看寄存器r4和r5的值
info all-register           //查看所有通用寄存器、浮点寄存器、向量寄存器的值
i r r4                      //查看寄存器r4的值
i r a0                      //查看寄存器a0（即r4）的值
i r r4 r5                   //查看寄存器r4和r5的值
i r f0                      //查看浮点寄存器f0的值
i r                         //查看所有通用寄存器、pc、badvaddr的值
i all-r                     //查看所有通用寄存器、浮点寄存器、向量寄存器的值
```

下面以一个具体示例来介绍查看寄存器信息的方法。使用的C语言程序如下：
``` c
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
调试命令的信息如下：
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
这里使用命令“b add”将断点设置在add函数的起始位置，然后使用命令“r”运行程序并停止在函数add入口处。从源程序可以看出函数add有两个参数，分别为int a、int b。根据LoongArch ABI的函数调用传参规则，调用函数add时的参数值1和2分别使用寄存器a0、a1来传递，故这里使用命令“i r a0”和“i r a1”来查看寄存器a0和a1的值，结果分别为1和2。

当然，我们可以使用“i r”来查看LoongArch架构中32个通用寄存器值，还有当前程序寄存器pc和badvaddr寄存器值。

如果还要查看浮点寄存器或者向量寄存器的值，可以使用信息“i all-r”命令。其命令显示的信息比较多，这里不做展示。

3.	disassemble命令

使用disassemble命令可以查看（也被称为反汇编）指定方法或指定一段地址的汇编指令。其缩写命令为disass。具体使用方式有如下几种：
``` shell
disass                //查看当前断点所在函数对应的汇编指令
disass func_name      //查看指定函数名为func_name的函数对应汇编指令
disass addr           //查看指定地址addr所在函数对应汇编指令
disass addr1,addr2    //查看指定地址addr1和addr2范围内的汇编指令
```
下面还是以gdbtest程序为例来演示disassemble命令的使用。
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
因为程序运行之前使用命令“b add”把断点设置在了函数add上，故程序执行到函数add处停止。使用“disass”命令反汇编出来的指令为函数add对应的全部汇编指令信息。

同时通过当前程序pc所在位置=>0x0000000120000674 <+36>可以看出，break命令在进行函数断点设置时，断点位置在程序栈构建之后位置，而非函数入口的第一条指令。
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
若要仅显示当前$pc开始的前4条和后4条汇编指令，可以为
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

前面介绍的display命令可以查看程序中某个变量或表达式的值，但是不能查看指定内存地址中的数据值。GDB为我们提供了查看内存的命令x，其可查看指定内存地址上的数据，且数据格式还可以指定。x命令的格式如下：
``` shell
x/FMT 	ADDRESS
```
参数FMT由内存单元数量、格式、内存单元长度组成。内存单元数量为整数，不指定时默认值为1；格式有多种，具体如下所示。

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

内存单元长度可由4个字母指定：b表示单字节、h表示双字节、w表示4字节、g表示8字节，且不指定时默认值为w。

参数ADDRESS为一个内存地址，可以是一个绝对地址（如0x12000006c），也可以是基于当前pc的相对地址（如$pc-4，表示当前程序暂停时，地址减4字节的内存位置）。

以下面的C语言程序为例，来演示x命令的使用。

``` c
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

backtrace 命令用于查看当前被调试程序的方法栈信息，以直观显示函数间的调用关系，其缩写命令为“bt”。具体语法格式如下。
``` shell
backtrace [QUALIFIERS] [COUNT]
```

其中，参数QUALIFIERS为可选项，其值可为“full”或者“no-filters”，分别表示输出局部变量的值和限定符禁止执行帧筛选器。参数COUNT也为可选项，其值为一个整数值，当值为正整数n时，表示输出最里层的n个栈帧的信息；当其值为负整数时，那么表示输出最外层n个栈帧的信息；当没有COUNT参数时，backtrace会显示完整的栈帧信息。

以下面C语言程序为例演示bt命令的使用。

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
其运行到方法add3时的堆栈信息如下：
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

如果我们要查看backtrace结果中某一层的方法栈信息，可以使用frame命令，缩写为“f”，其完整的命令形式如下：
``` shell
frame [frame_num|frame_addr]
```
参数可以是栈帧编号(frame_num)或栈帧地址(frame_addr)。当不指定任何参数时，frame命令将显示backtrace结果中最顶层方法的栈帧。同样以gdbtest程序为例，其frame信息如下：
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
(gdb) f         -->显示最顶层（即断点处对应方法）的栈信息
#0  add3 (a=1, b=2) at gdbtest.c:5
5     return a+b;
(gdb) f 1       -->显示编号为1的栈信息
#1  0x00000001200006cc in add2 (a=1, b=2) at gdbtest.c:8
8     return add3(a,b);
(gdb) 
```
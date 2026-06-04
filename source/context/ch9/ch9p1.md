#	GDB 调试器的常用命令

GDB（GNU Debugger）是Linux平台最常用的程序调试器之一，支持C、C++、Go、Objective-C等多种语言。Linux上许多带调试功能的C/C++集成开发环境，其底层调试能力也来自GDB。GDB通常通过终端中的`gdb`命令使用。`gdb`命令提供了大量选项，可帮助用户定位程序异常点，观察断点处的寄存器值、函数调用栈、线程调度等运行细节。

##	GDB 的启动和退出

GDB既可以随程序二进制文件一起启动，也可以通过进程号附加到正在运行的程序。启动时还可以指定程序参数、配置文件或core文件。下面列出几种常见启动方式。
``` shell
gdb program              //启动gdb并执行程序program
gdb program core         //启动gdb并停止到core文件中的异常位置
gdb -p 1234              //启动并绑定gdb到进程号为1234的程序上
gdb attach -p 1234       //同gdb -p 1234
gdb --args program       //同gdb program，program后面可以带命令行参数
gdb -x gdbinit program   //同gdb program，同时指定gdb配置文件
```
除上述常用启动参数外，还可以使用`gdb -h`或`gdb --help`查看更完整的参数说明。

为了更有效地使用GDB调试程序，通常希望被调试程序及其依赖动态库中保留符号表和调试信息。发布版二进制程序为了减小体积，往往会剥离符号和调试信息；使用Linux命令`file 文件名`查看时，通常会显示`stripped`。在这种情况下，GDB调试时无法显示函数名、变量名和源代码行号等直观信息。使用`gcc`或`g++`编译源码时，可以添加`-g`选项生成带符号和调试信息的二进制文件。

下面是一个带调试信息程序的编译和GDB运行示例：
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

这里编译`gdbtest.c`时使用`-g`，表示保留调试信息。GDB启动时默认会输出版本、版权和提示信息，示例中使用`-q`屏蔽这些内容。进入GDB后，可以在`(gdb)`提示符后输入命令控制程序执行。示例中的`r`和`q`分别是`run`和`quit`的缩写，表示运行程序和退出GDB。更多命令可通过`help`或`help all`查看。本章只介绍工作中常用的GDB调试功能，例如断点设置、单步调试、查看堆栈和寄存器等。

##	断点设置

断点通常在GDB启动后设置。断点可以让程序执行到指定位置时暂停，例如某一行、某个函数或某个地址，之后由用户继续检查和控制程序。GDB支持3类常用断点：break断点（程序断点）、watch断点（数据断点）和catch断点（事件断点）。break断点用于让程序在指定行或函数处暂停，是最常用的断点类型；watch断点用于监视变量或内存地址，当其值发生变化时暂停程序；catch断点用于捕获运行期间的特定事件，例如assert、exception、syscall、signal、fork等。

1.	break 断点设置

GDB中break断点相关命令如表9-1所示。

```{image} ../../img/ch9/t2p_9_1.png
:alt: break 断点设置的相关命令
:class: bg-primary
:scale: 80 %
:align: center
```

表9-1列出5个break断点相关命令。其中，`break`、`tbreak`和`rbreak`属于软件断点，常用于普通程序调试；`hbreak`和`thbreak`属于硬件断点，主要用于调试EPROM/ROM上的代码。`break`可缩写为`b`。参数LOCATION可以是行号、函数名或具体内存地址；如果不指定LOCATION，默认使用当前栈帧的PC值。选项`thread THREADNUM`可把断点限定到某个线程，线程号THREADNUM可通过`info threads`查看。选项`if CONDITION`用于设置条件断点，只有条件表达式CONDITION为真时断点才生效，适合调试变量达到特定值或循环执行到特定次数的场景。常用break命令如下：
``` shell
b a.c:4                //在源C语言文件a.c的第4行设置断点
b main                 //在函数main入口处设置断点
b a.c:add              //在源C语言文件a.c的函数add入口处设置断点
b *0x120000774         //在地址0x120000774处设置断点
b a.c:21 if out == 20  //条件断点，即当变量等于20时，程序在a.c中的21行处暂停
b a.c:21 thread 1      //在文件a.c的21行设置断点，仅对Num为1的线程起效
```
`tbreak`（缩写为`tb`）和`rbreak`（缩写为`rb`）用法与`break`类似。区别是：`tbreak`表示临时断点，只生效一次；`rbreak`用于对匹配规则命中的所有函数设置断点。示例如下：
``` shell
tbreak a.c:21     //在a.c中的21行设置断点，此断点只生效一次
ignore 1 10       //跳过（忽略）1号断点的前10次执行。1为断点号
rbreak .          //对程序中所有函数设置断点
rbreak a.c::.     //仅对a.c文件中的所有函数设置断点
rbreak add*       //对程序中所有以add为前缀的函数设置断点
```
硬件断点`hbreak`（缩写为`hb`）和`thbreak`（缩写为`thb`）的用法也与`break`类似，这里不再举例。`thbreak`表示硬件临时断点，也只生效一次。

断点设置完成后，可以使用`info break`或`info b`查看当前程序已经设置的断点。下面通过一个示例演示break的使用。C语言程序如下：
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
使用break调试该程序的信息如下：
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

`clear`命令可删除指定位置的所有断点，参数location通常是某一行代码或某个函数名。当location是函数名时，表示删除该函数入口处的全部断点。

`delete`命令（缩写为`d`）可删除指定编号断点或全部断点，参数num表示断点编号。不指定num时，`delete`会删除当前程序中的所有断点。

禁用断点可使用`disable`命令，其参数`num1 num2 ...`表示一次禁用多个断点。例如，`disable 1`表示禁用1号断点，`disable 1 2 3`表示同时禁用1、2、3号断点；不指定编号时，`disable`表示禁用当前程序所有断点。被禁用的断点可用`enable`重新启用，用法与`disable`相同。示例如下：
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

使用GDB调试程序时，可以借助watch断点监控某个变量或表达式的值。只要该值发生改变，程序就会暂停执行。这对定位变量或内存单元被非法修改的问题很有帮助。watch断点相关命令如下：
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
watch断点和break断点使用相同的删除命令`clear`或`delete`。下面通过C程序演示watch断点的使用。
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
使用watch断点观察变量tt变化的方式如下：
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
watch断点可以在程序运行前设置，也可以在运行过程中设置。这里是在运行前对变量tt设置watch断点。`info watch`可查看当前程序已设置的watch断点。watch通常需要处理器硬件支持；从示例输出看，龙芯处理器支持硬件watch断点。

与watch相似的观察断点还有`rwatch`和`awatch`。区别是：`watch`观察变量或内存值的写变化，`rwatch`观察读变化，`awatch`观察读或写变化。

3.	catch 断点设置

catch断点用于监控程序中的特定事件，例如发生某种异常或加载某个动态库。一旦目标事件发生，程序就会暂停执行。catch断点设置格式如下：
``` shell
tcatch event
```
参数event表示要监控的具体事件。catch常用event类型如表9-2所示。

```{image} ../../img/ch9/t2p_9_2.png
:alt: Catch常用的event事件类型
:class: bg-primary
:scale: 80 %
:align: center
```

常见catch断点设置示例如下：
``` shell
catch signal SIGBUS   //捕获SIGBUS事件，当此事件发生时程序暂停
tcatch signal SIGBUS  //仅捕获SIGBUS事件一次
catch signal all      //捕获所有信号事件，当此事件发生时程序暂停
catch syscall chroot  //捕获系统调用chroot，当此接口被调用时程序暂停
catch syscall         //捕获所有系统调用
info break            //查看所有break、watch和catch断点信息
delete 1              //删除Num为1的断点。此断点可以是break、watch或catch断点
```
例如，捕获程序运行时动态库加载事件，可使用如下命令：
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

当程序在断点处暂停时，可以使用`print`或`display`查看变量或表达式的值。`print`可缩写为`p`。二者常用格式如下：
``` shell
p variable
p file::variable
print function::variable
display variable
display file::variable
display function::variable
```
参数variable表示要查看或修改的变量。当程序中存在多个作用域不同但名称相同的变量或表达式时，可以在变量前加文件名（`file::variable`）或函数名（`function::variable`）限定作用域。

`display`也用于查看变量或表达式的值。它与`print`的区别是：使用`display`设置后，每当程序暂停（例如单步执行后），GDB都会自动输出该值。

2.	info register命令

程序停在断点处时，可以使用该命令查看一个、多个或全部寄存器。下面命令都是查看寄存器信息的有效方式。
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

下面用一个具体示例说明如何查看寄存器。C语言程序如下：
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
这里用`b add`把断点设置在add函数入口，然后用`r`运行程序并停在add函数处。从源程序可知，add有两个参数`int a`和`int b`。根据LoongArch ABI函数调用传参规则，调用add时，参数1和2分别通过寄存器a0、a1传递。因此，使用`i r a0`和`i r a1`查看寄存器值，结果分别为1和2。

也可以使用`i r`查看LoongArch的32个通用寄存器，以及当前程序的pc和badvaddr。如果还要查看浮点寄存器或向量寄存器，可使用`i all-r`。该命令输出较多，这里不展示。

3.	disassemble命令

`disassemble`命令用于查看（反汇编）指定函数或指定地址范围内的汇编指令，缩写为`disass`。常见用法如下：
``` shell
disass                //查看当前断点所在函数对应的汇编指令
disass func_name      //查看指定函数名为func_name的函数对应汇编指令
disass addr           //查看指定地址addr所在函数对应汇编指令
disass addr1,addr2    //查看指定地址addr1和addr2范围内的汇编指令
```
下面仍以gdbtest为例演示`disassemble`命令。
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
因为程序运行前用`b add`把断点设置在add函数，所以程序执行到add处暂停。此时执行`disass`，得到的是add函数对应的全部汇编指令。

从当前pc所在位置`=> 0x0000000120000674 <+36>`还可以看出，使用`break`设置函数断点时，断点位置位于函数栈构建之后，而不是函数入口第一条指令。
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
如果只想显示当前`$pc`附近前4条和后4条汇编指令，可以使用如下命令：
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

前面介绍的`display`可查看程序中的变量或表达式，但不能直接查看指定内存地址中的数据。GDB提供`x`命令查看指定内存地址上的数据，并可指定显示格式。命令格式如下：
``` shell
x/FMT 	ADDRESS
```
参数FMT由内存单元数量、显示格式和内存单元长度组成。内存单元数量为整数，不指定时默认为1；显示格式包括如下类型。

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

内存单元长度可由4个字母指定：b表示单字节，h表示双字节，w表示4字节，g表示8字节；不指定时默认为w。

ADDRESS表示内存地址，可以是绝对地址（如0x12000006c），也可以是基于当前pc的相对地址（如`$pc-4`，表示当前暂停位置之前4字节的内存地址）。

下面用C程序演示`x`命令。

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

`backtrace`命令用于查看当前被调试程序的函数栈信息，直观显示函数调用关系，缩写为`bt`。语法如下：
``` shell
backtrace [QUALIFIERS] [COUNT]
```

QUALIFIERS为可选项，可取`full`或`no-filters`，分别表示输出局部变量值和禁止执行帧筛选器。COUNT也是可选项，取整数。若为正整数n，表示输出最内层n个栈帧；若为负整数，表示输出最外层n个栈帧；不指定COUNT时，显示完整栈帧信息。

下面用C程序演示`bt`命令。

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
程序运行到add3函数时，堆栈信息如下：
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

如果要查看`backtrace`结果中某一层栈帧信息，可以使用`frame`命令，缩写为`f`，完整形式如下：
``` shell
frame [frame_num|frame_addr]
```
参数可以是栈帧编号（frame_num）或栈帧地址（frame_addr）。不指定参数时，`frame`显示`backtrace`结果中最顶层函数的栈帧。同样以gdbtest程序为例，frame信息如下：
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

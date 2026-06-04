#	程序单步调试

程序在断点处暂停后，可以使用`continue`命令（缩写为`c`）恢复执行，也可以使用单步调试命令逐步跟踪程序流程。下面介绍常用单步调试命令。

##	语句单步调试

语句单步调试以源程序（如C语言）的一条语句为单位执行。GDB提供3个常用命令：`next`、`step`和`until`，可分别简写为`n`、`s`和`u`。

`next`是最常用的单步命令。它的特点是：遇到函数调用语句时，会把该函数调用视为一条语句并一次执行完，不会进入被调用函数内部。

`step`执行单步调试时，如果遇到函数调用语句，会进入被调用函数内部继续执行。

`next`和`step`都可以带可选参数count，表示一次执行后续count条语句。例如，`n 2`表示一次执行2条语句。下面以9.1.4小节中的C语言示例演示语句级单步调试。
``` shell
$ gdb -q ./gdbtest
Reading symbols from ./gdbtest...done.
(gdb) b main        -->断点设置在main函数
Breakpoint 1 at 0x1200007d8: file gdbtest.c, line 17.
(gdb) r             -->程序运行
Starting program: /home/sunguoyun/c-test/gdbtest
Breakpoint 1, main (argc=1, argv=0x12014f1d0) at gdbtest.c:17
17 int main (int argc, char *argv[]) {
(gdb) s             -->执行一条语句
18     add(1, 2);
(gdb) s             -->执行一条语句（遇到函数add调用），进入函数内部
add (a=1, b=538254544) at gdbtest.c:14
14 int add( int a, int b) {
(gdb) s             -->执行一条语句
15     return add1(a,b);
(gdb) n             -->执行一条语句（遇到函数add调用），不进入函数内部
16 }
```
`until`命令适合在程序执行到循环体尾部时快速运行完当前循环，并在循环外停止。这里不再单独演示。

##	汇编指令的单步调试

`stepi`（缩写为`si`）和`nexti`（缩写为`ni`）用于按汇编指令单步执行。如果配合`display/i $pc`，还可以在单步跟踪过程中自动显示每条即将执行的汇编指令。`si`和`ni`也都可以带count参数，一次执行连续count条汇编指令。例如：
``` shell
(gdb) x/10i $pc      -->显示PC位置开始的10条汇编指令
=> 0x1200007d8 <main+4>: st.d    $r1,$r3,24(0x18)
0x1200007dc <main+8>: st.d    $r22,$r3,16(0x10)
0x1200007e0 <main+12>: addi.d $r22,$r3,32(0x20)
0x1200007e4 <main+16>: move   $r12,$r4
0x1200007e8 <main+20>: st.d   $r5,$r22,-32(0xfe0)
0x1200007ec <main+24>: slli.w $r12,$r12,0x0
0x1200007f0 <main+28>: st.w   $r12,$r22,-20(0xfec)
0x1200007f4 <main+32>: addi.w $r5,$r0,2(0x2)
0x1200007f8 <main+36>: addi.w $r4,$r0,1(0x1)
0x1200007fc <main+40>: bl -124(0xfffff84) # 0x120000780 <add>
(gdb) ni             -->执行一条汇编指令
0x00000001200007dc 17 int main (int argc, char *argv[]) {
(gdb) ni 8           -->执行8条汇编指令
0x00000001200007fc 18     add(1, 2);
(gdb) ni             -->执行一条汇编指令，遇到函数跳转指令bl并没有进入
0x0000000120000800 18     add(1, 2);
(gdb) 
```
汇编单步命令`ni`和`si`的主要区别在于遇到函数跳转指令时的处理方式：`ni`遇到`bl`等函数调用跳转时不会进入被调用函数内部，而`si`会进入被调用函数继续执行。

##	退出当前函数

在某个函数中调试一段时间后，如果不想继续逐步执行到函数返回处，可以使用`finish`命令直接执行完当前函数。与`finish`类似的还有`return`命令，二者都可以结束当前正在执行的函数。区别在于：`finish`会让函数继续运行直到正常返回；`return`会立即结束当前函数并返回，也就是说，当前函数中尚未执行的代码不会继续执行。同时，`return`命令还可以指定函数返回值。

关于GDB的更多说明和用法，例如如何调试正在运行的程序、如何跟踪多线程程序、如何在调试中屏蔽某个中断信号、如何设置和使用`gdbinit`配置文件等，可以在Linux系统下使用`man gdb`或`gdb --help`查看。表9-3列出了一些本章前面没有介绍但常用的GDB命令。

```{image} ../../img/ch9/t2p_9_3.png
:alt: 基于Linux的操作系统下的GDB常用命令
:class: bg-primary
:scale: 80 %
:align: center
```

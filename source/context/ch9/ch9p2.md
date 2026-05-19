#	程序单步调试

当程序执行到断点位置暂停时，我们可以用continue 命令（缩写命令为“c”）恢复并继续程序的运行，也可以使用单步调试命令一步一步地跟踪程序执行。下面分别介绍单步调试相关命令的使用。

##	语句单步调试

语句单步调试是指以源程序（如C语言）的一条语句为单位，一步一步地执行。GDB提供了3种命令，用于语句单步调试，即使用命令 next、step和 until，分别可以简写为“n”“s”和“u”。

next为最常用的单步调试命令。其最大的特点是当遇到调用函数的语句时，next 命令会将其视为一行语句并一步执行完，不会跳入调用函数内部。

step命令在进行单步调试时，当遇到调用函数的语句时，会进入该调用函数内部继续执行。

next或者step命令都可以选择性地添加count参数，表示一次执行完后面的count条语句。例如“n 2”表示一次执行2条语句。这里以9.1.4小节中使用的C语言示例来演示语句单步调试。
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
util命令可以在程序执行至循环体尾部时，使GDB快速执行完成当前的循环体并运行至循环体外停止。这里暂不做示例演示。

##	汇编指令的单步调试

命令stepi （缩写命令为“si”）和nexti （缩写命令为“ni”）都可以用于单步执行汇编指令。如果辅助命令“display/i $pc”，还可以在单步跟踪过程中输出每一条汇编指令。同时si和ni后面也都可以选择性地使用count参数，一次可执行连续的count条汇编指令。例如：
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
汇编指令的单步调试命令ni和si的区别在于遇到函数跳转指令时的处理，ni遇到函数跳转指令bl时不会进入调用函数内部，而si会进入调用函数内部继续执行。

##	退出当前函数

在某个函数中调试一段时间后，可能不需要再一步一步执行到函数返回处，希望直接执行完当前函数，这时可以使用 finish命令。与finish 命令类似的还有 return 命令，它们都可以结束当前执行的函数。区别在于finish命令会执行函数到正常退出；而 return 命令是立即结束执行当前函数并返回，也就是说，如果当前函数还有剩余的代码未执行完毕，也不会执行了，但是使用return命令可以指定函数的返回值。

关于GDB的更多说明和使用方法，例如如何对一个已运行的程序进行调试、如何跟踪多线程程序的调试、调试过程如何屏蔽某个中断信号、如何设置和使用gdbinit配置文件等，可以在基于Linux的操作系统下使用命令 man gdb或者gdb --help来查看。表9-3中列举了一些GDB中常用但是本章前面没有提到过的命令。

```{image} ../../img/ch9/t2p_9_3.png
:alt: 基于Linux的操作系统下的GDB常用命令
:class: bg-primary
:scale: 80 %
:align: center
```
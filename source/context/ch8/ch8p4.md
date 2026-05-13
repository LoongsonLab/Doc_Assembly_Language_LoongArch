#	脱离libc库的最“小”程序示例

绝大多数情况下，我们编写的程序依赖很多系统库才能运行，其中libc库为必不可少的基础库。例如前文介绍的hello.c，该程序功能虽仅通过调用printf函数实现“Hello World”的输出，编译过程中必不可少的就有crt1.o、 crti.o、crtn.o、crtbegin.o、crtend.o和libc库的参与。如果程序使用了动态链接，那么还需要ld库帮助在程序运行时实现其他动态库的加载。

本章我们将编写一个最“小”程序，同样实现“HelloWorld”的输出，但是因为可以跳过对上述.o文件和libc库的依赖，这个程序最终的目标文件会很小。之前依赖系统库实现的“Hello World”输出功能的程序文件大小近22KB，而本章实例完成同样的功能的程序文件大小将会不到1KB。

##	编写主程序

首先我们要编写带有内嵌汇编的main.c文件，内容如下：
``` c
/* main.c */
#define STR "Hello World \n"
void printf(char* str,int len) {
asm(
"li.w  $r11, 64\n\t"      // sys_write 的系统调用号是 64，放在r11
"li.w  $r4, 1\n\t"        // 参数1：stdout文件描述符是1
"move  $r5, %0\n\t"       // 参数2：字符串地址
"move   $r6, %1\n\t"       // 参数3：字符串长度
"syscall   0   \n\t"       // 系统调用指令
:
:"r"(str),"r"(len)
:"$r11","$r4","$r5","$r6");
}
void exit() {
asm(
"li.w  $r11, 93\n\t"
"li.w  $r4, 0\n\t"        //将进程退出状态码0存入参数1
"syscall  0\n\t"
:::"$r11","$r4");
}
int main() {
printf(STR,13);
exit();
}
```
这里我们自定义了两个与libc库同名的函数printf、exit，函数内部直接对接系统调用接口来完成信息向标准输出设备的写入和进程正常退出功能，这样就跳过了对libc库的接口使用。

##	链接脚本

本书第02章提到了GCC编译的基本流程，其中链接过程实现了把多个目标文件链接成一个可执行文件。链接过程需要一个链接脚本来帮助制定链接规则，例如规定如何把输入文件内的Section放入输出文件内, 并控制输出文件内各部分在程序地址空间内的布局等。这里的输入文件就包括crt1.o、crtbegin.o、libc.so等目标文件。本小节中，我们要实现独立的最小程序，为了避免这个默认链接脚本对一些输入文件的依赖，需要重写一个链接脚本。这里编写了一个名为ld.lds的链接脚本文件，其内容如下：
``` shell
/* ld.lds文件 */
OUTPUT_ARCH(loongarch)
ENTRY(main)
SECTIONS
{
	. = 0x120000000 + SIZEOF_HEAEDERS;
	.text : {
		*(.head.text)
		*(.text*)
	}
	.rodata : {
		*(.rodata*)
		*(.got*)
	}
	.data : {
		*(.data*)
		*(.bss*)
		*(.sbss*)
	}
	/DISCARD/ : {
		*(.comment)
		*(.pdr) /*debug used*/
		*(.options)
		*(.gnu.attributes)
		*(.debug*)
	}
}
```
链接脚本的语法并不复杂。第一行的OUTPUT_ARCH(loongarch)指定了体系架构为LoongArch。ENTRY(main)指定程序入口函数为main。SECTIONS{…}为链接脚本主体，里面包含SECTIONS的变换规则。

.=0x120000000 + SIZEOF_HEAEDERS; 是一条赋值语句，意为将当前程序加载到内存的起始虚拟地址，设置成0x120000000 +SIZEOF_HEAEDERS。“.”代表起始虚拟地址，SIZEOF_HEAEDERS为输出文件的文件头大小。链接脚本里面的语句分为赋值语句和命令语句，OUTPUT_ARCH和ENTRY就属于命令语句，可以用换行代替“;”；赋值语句必须使用“;”结尾。

.text:{*(head.text) *(.text*)} 是段转换规则，意为将所有输入文件中名字为“.head.txt”和“.text.*”的段依次合并到输入文件的.text段。/DISCARD/ : { *(.comment) *(.pdr)... } 意为将所有输入文件中的.comment段、.pdr段、.options段丢弃，不保存在输出文件中。链接文件中需要使用/*…*/作为注释，例如/*debugused*/。

##	程序的运行

前面介绍了内嵌汇编源程序和链接脚本的编写。接下来我们编译并运行这个“最小”程序即可,命令如下：

``` shell
$ gcc -c -fno-builtin main.c
$ ld -T ld.lds main.o -o main
$ ./main
Hello World 
```

上述代码中的相关参数说明如下。

-	-c：编译、汇编到目标代码，不进行链接。

-	-fno-builtin：关闭GCC内置函数功能。

-	-T ld.lds：使用链接脚本ld.lds 。如果不指定-T，那么ld会使用系统默认的链接脚本。

-	-o main：输出可执行文件名为main 。这个程序没有使用其他系统库，通过系统调用就完成了“Hello World”的输出。我们可以查看这个main程序的大小：
``` shell
$ ls -lh main
-rwxrwxr-x 1  985 2月  19 16:24 main
```
可以看到main大小仅为985B。
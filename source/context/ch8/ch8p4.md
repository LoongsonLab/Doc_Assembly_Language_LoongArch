#	脱离 libc 库的最小程序示例

绝大多数情况下，我们编写的程序需要依赖多个系统库才能运行，其中libc库是最基础、最常见的依赖。例如，前文介绍的`hello.c`虽然只是调用`printf`输出“Hello World”，但编译过程中仍需要`crt1.o`、`crti.o`、`crtn.o`、`crtbegin.o`、`crtend.o`和libc库参与。如果程序采用动态链接，运行时还需要ld库帮助加载其他动态库。

本节编写一个最小程序，同样完成“Hello World”的输出。由于可以跳过上述`.o`文件和libc库依赖，最终生成的目标文件会很小。此前依赖系统库实现“Hello World”输出的程序接近22KB，而本节示例完成同样功能的程序不到1KB。

##	编写主程序

首先编写带内嵌汇编的`main.c`文件，内容如下：
``` c
/* main.c */
#define STR "Hello World \n"
void printf(char* str,int len) {
asm(
"li.w  $r11, 64\n\t"      // sys_write 的系统调用号是 64，放在r11
"li.w  $r4, 1\n\t"        // 参数1：stdout文件描述符是1
"move  $r5, %0\n\t"       // 参数2：字符串地址
"move  $r6, %1\n\t"       // 参数3：字符串长度
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
这里自定义了两个与libc库同名的函数`printf`和`exit`。函数内部直接使用系统调用接口，完成向标准输出写入信息和正常退出进程的功能，从而绕过libc库接口。

##	链接脚本

第02章介绍过GCC编译基本流程，其中链接过程负责把多个目标文件链接为一个可执行文件。链接过程需要链接脚本指定规则，例如如何把输入文件中的Section放入输出文件，以及如何控制输出文件各部分在程序地址空间中的布局。这里的输入文件通常包括`crt1.o`、`crtbegin.o`、`libc.so`等目标文件。本小节要实现一个独立的最小程序。为了避免默认链接脚本引入这些输入文件依赖，需要重写链接脚本。下面是名为`ld.lds`的链接脚本：
``` shell
/* ld.lds文件 */
OUTPUT_ARCH(loongarch)
ENTRY(main)
SECTIONS
{
	. = 0x120000000 + SIZEOF_HEADERS;
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
链接脚本语法并不复杂。第一行`OUTPUT_ARCH(loongarch)`指定体系架构为LoongArch。`ENTRY(main)`指定程序入口函数为`main`。`SECTIONS{...}`是链接脚本主体，包含Section转换规则。

`. = 0x120000000 + SIZEOF_HEADERS;`是一条赋值语句，表示把当前程序加载到内存的起始虚拟地址设置为`0x120000000 + SIZEOF_HEADERS`。“.”表示当前位置，也就是起始虚拟地址；`SIZEOF_HEADERS`表示输出文件的文件头大小。链接脚本中的语句分为赋值语句和命令语句，`OUTPUT_ARCH`和`ENTRY`属于命令语句，可以用换行代替“;”；赋值语句必须以“;”结尾。

`.text : { *(.head.text) *(.text*) }`是段转换规则，表示把所有输入文件中名为`.head.text`和`.text*`的段依次合并到输出文件的`.text`段。`/DISCARD/ : { *(.comment) *(.pdr) ... }`表示丢弃所有输入文件中的`.comment`、`.pdr`、`.options`等段，不写入输出文件。链接脚本中的注释需要使用`/*...*/`，例如`/*debug used*/`。

##	程序的运行

前面已经介绍内嵌汇编源程序和链接脚本的编写。接下来编译并运行这个最小程序，命令如下：

``` shell
$ gcc -c -fno-builtin main.c
$ ld -T ld.lds main.o -o main
$ ./main
Hello World 
```

上述命令中的相关参数说明如下。

-	`-c`：只完成编译和汇编，生成目标代码，不执行链接。

-	`-fno-builtin`：关闭GCC内置函数功能。

-	`-T ld.lds`：使用链接脚本`ld.lds`。如果不指定`-T`，`ld`会使用系统默认链接脚本。

-	`-o main`：指定输出可执行文件名为`main`。该程序没有使用其他系统库，只通过系统调用完成“Hello World”输出。可以查看`main`程序大小：
``` shell
$ ls -lh main
-rwxrwxr-x 1  985 2月  19 16:24 main
```
可以看到，`main`大小仅为985B。

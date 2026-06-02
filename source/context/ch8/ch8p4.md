#	脱离 libc 库的最小程序示例

绝大多数情况下，我们编写的程序需要依赖多个系统库才能运行，其中 libc 库是必不可少的基础库。例如，前文介绍的 hello.c 虽然仅通过调用 printf 函数实现“Hello World”的输出，但编译过程中仍需要 crt1.o、crti.o、crtn.o、crtbegin.o、crtend.o 和 libc 库参与。如果程序使用动态链接，还需要 ld 库在程序运行时帮助加载其他动态库。

本节将编写一个最小程序，同样实现“Hello World”的输出。由于可以跳过对上述 .o 文件和 libc 库的依赖，该程序最终生成的目标文件会很小。此前依赖系统库实现“Hello World”输出功能的程序文件大小接近22KB，而本节示例完成同样功能的程序文件大小将不到1KB。

##	编写主程序

首先，编写带有内嵌汇编的 main.c 文件，内容如下：
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
这里自定义了两个与 libc 库同名的函数 printf 和 exit。函数内部直接对接系统调用接口，完成向标准输出设备写入信息和进程正常退出的功能，从而跳过对 libc 库接口的使用。

##	链接脚本

本书第02章介绍了GCC编译的基本流程，其中链接过程用于将多个目标文件链接为一个可执行文件。链接过程需要链接脚本来制定链接规则，例如规定如何将输入文件内的 Section 放入输出文件，并控制输出文件内各部分在程序地址空间中的布局等。这里的输入文件包括 crt1.o、crtbegin.o、libc.so 等目标文件。本小节要实现一个独立的最小程序。为了避免默认链接脚本对这些输入文件的依赖，需要重写一个链接脚本。这里编写一个名为 ld.lds 的链接脚本文件，其内容如下：
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
链接脚本的语法并不复杂。第一行的 OUTPUT_ARCH(loongarch) 指定体系架构为 LoongArch。ENTRY(main) 指定程序入口函数为 main。SECTIONS{...} 为链接脚本主体，其中包含 SECTIONS 的转换规则。

`. = 0x120000000 + SIZEOF_HEADERS;` 是一条赋值语句，表示将当前程序加载到内存的起始虚拟地址设置为 `0x120000000 + SIZEOF_HEADERS`。“.”代表当前位置，即起始虚拟地址；SIZEOF_HEADERS 为输出文件的文件头大小。链接脚本中的语句分为赋值语句和命令语句，OUTPUT_ARCH 和 ENTRY 属于命令语句，可以用换行代替“;”；赋值语句必须使用“;”结尾。

`.text : { *(.head.text) *(.text*) }` 是段转换规则，表示将所有输入文件中名为 `.head.text` 和 `.text*` 的段依次合并到输出文件的 .text 段。`/DISCARD/ : { *(.comment) *(.pdr) ... }` 表示将所有输入文件中的 .comment 段、.pdr 段、.options 段等丢弃，不保存在输出文件中。链接脚本中需要使用 `/*...*/` 作为注释，例如 `/*debug used*/`。

##	程序的运行

前面介绍了内嵌汇编源程序和链接脚本的编写。接下来编译并运行这个最小程序，命令如下：

``` shell
$ gcc -c -fno-builtin main.c
$ ld -T ld.lds main.o -o main
$ ./main
Hello World 
```

上述代码中的相关参数说明如下。

-	-c：编译、汇编到目标代码，但不进行链接。

-	-fno-builtin：关闭GCC内置函数功能。

-	-T ld.lds：使用链接脚本 ld.lds。如果不指定 -T，ld 会使用系统默认的链接脚本。

-	-o main：指定输出可执行文件名为 main。这个程序没有使用其他系统库，仅通过系统调用就完成了“Hello World”的输出。可以查看 main 程序的大小：
``` shell
$ ls -lh main
-rwxrwxr-x 1  985 2月  19 16:24 main
```
可以看到 main 的大小仅为985B。

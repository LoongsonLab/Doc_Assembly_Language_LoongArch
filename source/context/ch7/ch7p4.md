#	汇编源程序实例文件 hello.S

在学习任何程序设计语言之初，入门示例通常都是编写一个输出“Hello World!”的函数。本章前面已经详细介绍了汇编器指令，以及如何在汇编源文件中定义变量和函数等内容。接下来通过编写一个完整的汇编源程序，进一步掌握汇编源程序的编写和编译过程。

汇编源程序实例文件 hello.S 的内容如下：
``` asm
.data
.LC0: 			#	本地标签，指定字符串"Hello World!\0"的地址
.ascii	"Hello World!\0"
.text
.align 	2
.global main
.type 	main,@function
main: 			#	本地标签，指定函数main的开始
	addi.d 		$sp, $sp, -8
	st.d 		$ra, $sp, 0
	la.local 	$r4, .LC0
	bl 			$plt(puts)
	li.w 		$a0, 0
	ld.d 		$ra, $sp, 0
	addi.d 		$sp, $sp, 8
	jr 			$ra
.size 	main, .-main
.section .note.GNU-stack,"",@progbits
```
这段程序基本展示了汇编源程序包含的语法，其中包括汇编器指令和汇编指令。从中也可以看出，汇编器指令通常以字符“.”开头。

示例中命令语句后面的 # 为行注释，不参与汇编器编译，也不占用目标文件中的地址空间。需要注意的是，不同架构对汇编源文件注释的要求可能不同。在LoongArch架构下，行注释符号与 x86 相同，均为 #。汇编源文件中还可以使用块注释，其语法风格与C语言相同，即 `/*...*/`。

main 函数内的汇编指令通过调用 libc 库的 puts 函数接口，完成字符串“Hello World!”的屏幕输出。

写好上述 hello.S 后，可以通过汇编器直接将其编译为可重定位目标文件 hello.o，再通过链接器与依赖的基础 libc 库链接为可执行文件 hello，具体命令如下：
``` shell
$ as hello.S -o hello.o
$ ld hello.o -lc -o hello
```
这两个过程也可以使用 gcc 一次完成，具体命令如下：
``` shell
$ gcc hello.S -o hello
```

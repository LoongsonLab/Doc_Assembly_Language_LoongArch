#	汇编源程序实例文件 hello.S

学习任何程序设计语言时，入门示例通常都是编写一个输出“Hello World!”的程序。本章前面已经介绍了汇编器指令，以及如何在汇编源文件中定义变量和函数。下面通过编写一个完整汇编源程序，进一步了解汇编源程序的编写和编译过程。

汇编源程序示例文件`hello.S`内容如下：
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
这段程序展示了汇编源程序中的基本语法，包括汇编器指令和汇编指令。可以看到，汇编器指令通常以字符“.”开头。

示例中命令语句后的`#`表示行注释，不参与汇编器编译，也不占用目标文件地址空间。需要注意，不同架构对汇编源文件注释的规定可能不同。在LoongArch架构下，行注释符号与x86相同，都是`#`。汇编源文件也可以使用块注释，语法与C语言一致，即`/*...*/`。

`main`函数内的汇编指令调用libc库的`puts`函数接口，完成字符串“Hello World!”的屏幕输出。

写好`hello.S`后，可以先用汇编器编译为可重定位目标文件`hello.o`，再由链接器与依赖的基础libc库链接生成可执行文件`hello`，命令如下：
``` shell
$ as hello.S -o hello.o
$ ld hello.o -lc -o hello
```
这两个步骤也可以通过`gcc`一次完成：
``` shell
$ gcc hello.S -o hello
```

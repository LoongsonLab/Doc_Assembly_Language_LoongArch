#	汇编源程序实例文件hello.S

在学习任何程序设计语言之初，入门阶段基本都是编写一个输出“Hello World !”的函数。本章前面已经详细介绍了汇编器指令，以及在汇编源文件中如何定义一个变量、如何定义一个函数等。接下来就编写一个完整的汇编源程序，来实际掌握汇编源程序的编写和编译过程。

汇编源程序实例文件hello.S的内容如下：
``` asm
.data
.LC0: 			#	本地标签，指定了字符串"Hello World!\0"的地址
.ascii	"Hello World!\0"
.text
.align 	2
.global main
.type 	main,@function
main: 			#	本地标签，指定了函数main的开始
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
这段程序基本展示了汇编源程序包含的语法。里面包括了汇编器指令和汇编指令。从中也可以看出汇编器指令都是以字符“.”开头。

示例中命令语句后面的#为行注释，不参与汇编器的编译，且不占用目标文件中的地址空间。注意汇编源文件中的注释在不同的架构下要求可能不同。在LoongArch架构下，采用和x86相同的行注释符号#。汇编器源文件中还有一种是段注释，采用和C语言语法风格相同的注释符形式，即/*…*/。

main函数内的汇编指令整体功能是通过调用libc库的puts函数接口，完成字符串“Hello World !”的屏幕输出。

上面的hello.S写好后，可以通过汇编器直接将其编译成可重定位目标文件hello.o，再通过链接器和依赖的基础libc库链接成可执行文件hello，具体命令如下：
``` shell
$as hello.S -o hello.o
$ld hello.o -lc -o hello
```
这两个过程也可以使用封装的脚本gcc一次完成，具体命令如下：
``` shell
$gcc hello.S -o hello
```
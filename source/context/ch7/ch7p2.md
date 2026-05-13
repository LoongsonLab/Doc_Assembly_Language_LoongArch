#	汇编源文件中的汇编器指令

汇编指令是机器指令的易读版，所以汇编指令和机器指令一一对应，在GCC编译的汇编阶段，汇编器会将汇编指令翻译成机器指令并存放到目标文件中。汇编器指令和汇编指令完全不同，汇编器指令是为汇编器而生的，是用于指导汇编器如何定义变量和函数、汇编指令在目标文件中如何存放等。即汇编器指令是指导汇编器工作的指令。本节将从符号定义相关、逻辑控制相关两个方面来详细介绍汇编源文件中的汇编器指令。

##	符号定义相关的汇编器指令

第06章介绍目标文件格式ELF时，提到过函数和变量都称为符号，这个概念在这里也适用。编写汇编源程序和使用高级语言一样，需要定义程序需要的常量、变量、函数等符号。汇编器指令中和符号定义相关的指令如下。

1.	定义一个字符串变量

在汇编源程序中定义一个字符串变量需要包括的完整信息有字符串名称、字符串内容、字符串大小、符号类型、对齐方式、变量作用域、变量所在段等。下面是一个使用C语言定义的字符串变量及其对应的汇编器指令。
``` c
//	c language
char str[10] = "hello";
```
``` asm
#	asm
.global str 	#	指定符号str的作用域为全局
.data 			#	指定符号str所在段为.data
.align 3 		#	指定符号str为8字节对齐
.type str,@object 	#	指定符号str的类型为对象
.size str,10 	#	指定符号str大小为10字节
str: 			#	指定符号名称为str
.ascii "hello\000" 	#	指定符号str的内容
```
示例中已经标注了当前定义的字符串相关信息。其中字符串变量内容除了使用汇编器指令.ascii定义外，还可以使用汇编器指令.asciz和.string。三者都可以用于定义一个字符串，且可不带参数或者带多个由逗点分开的字符串，用于把汇编好的每个字符串存入连续的地址。其中.ascii在字符串末尾不自动追加零字节；而.asciz则会在字符串后自动添加结束符\0，其中的“z”代表“zero”。下面3种定义字符串的方式是等价的。

``` asm
.ascii 	"hello\0"
.asciz 	"hello"
.string "hello"
```
这里.string实际上是.string8的缩写。汇编器指令.string是区分字符宽度的，具体宽度有.string8、.string16、.string32、.string64，分别表示一个字符占用8位、16位、32位、64位，且存放时有尾端区分。当使用默认.string（即.string8）时，可以看作和.asciz等价。例如定义字符串.string"Hello World"在龙芯3A5000小尾端机器上的目标文件（.o文件）中存放方式
``` shell
Addr 		Data
0:		6c6c6568 	#	ascii值 lleh
4:		6f77206f 	#	ascii值 ow o
8:		00646c72 	#	ascii值 _dlr
```
而定义成.string16 "Hello World"后，此字符串在汇编后的目标文件中存放方式为
``` shell
Addr 		Data
0:		00650068 	#	ascii值 _e_h
4:		006c006c 	#	ascii值 _l_l
8:		0020006f 	#	ascii值 _ _o
c: 		006f0077 	#	ascii值 _o_w
10: 		006c0072 	#	ascii值 _l_r
14: 		00000064 	#	ascii值 _ _d
```
这里为了更直观地显示，备注说明中使用_代替了ASCII表中的NUL值。

当源文件中要使用的字符串是临时且无名的，可以使用标签.LC0、.LC1等代替。

2.	定义一个整型变量

在汇编源程序中定义一个整型变量和定义一个字符串变量类似，需要的信息包括变量名称、变量值、变量大小、变量类型、对齐方式、作用域和变量所在段等。下面是一个使用C语言定义的整型变量及其对应的汇编器指令。
``` c
static int int_v = 20;
```
``` asm
.data
.align 2
.type int_v,@object
.size int_v,4
int_v:
.word 20
```
汇编器指令中用于定义数据长度的汇编指令有.byte value、.half value、.word value、 .dwordvalue。.byte用于定义一字节（8位）的地址空间，其他命令所定义的空间大小依赖于具体系统，value为变量值。在LoongArch中，.half 为2字节、.word为4字节、.dword为8字节。这里变量int_v为int类型，故使用的数据类型为“.word 20”，数据长度.size为4字节。

3.	定义一个函数

在汇编源程序中定义一个函数，需要的信息包括函数名称、函数汇编指令（可以使用宏指令）、变量大小、变量类型、对齐方式、作用域和变量所在段等。下面是一个使用C语言定义的函数及其对应的汇编器指令。

``` c
int add(int a,int b){
	return a + bl
}
```
``` asm
.text		#	指定符号add数据存放在代码段
.align 2
.global add
.type 	add, @function
add:
add.w 	$a0, $a0, $a1
jr 		$r1
.size 	add,.-add
```
函数一般都会放在代码段，故这里指定符号add所在段为.text。前面列举的字符串变量和整型变量的类型都为@object，而函数add的类型要定义为@function，说明这是一个函数，且通过“.globladd”指定这是一个全局函数。函数符号“add:”后面就可以写汇编指令用于实现相应的函数功能，这里仅包含add.w和jr两条指令。

符号add的大小使用“.size add, .-add”格式定义，其中add表示符号（函数）名称。 .-add表示指令占用的内存大小为当前位置(.)减去函数add的起始位置，当前函数一共2条指令，即8字节大小。这里为何使用“.size add, .-add”而不是“.size add, 8”呢？原因是大部分汇编源文件中存在可能被扩展成多条机器指令的宏指令，例如li.w/li.d，这时函数实际占用内存大小需要汇编器完成机器指令翻译后才能确定，故这里对函数大小的定义使用函数符号的动态计算方式。

4.	符号定义相关的汇编器指令说明

(1)设置符号类型

定义符号类型的汇编器指令为.type，其后面常跟的类型有@function和@object，分别表示当前符号为函数和变量。
``` asm
.type 	add, @function 		#	符号add的类型为函数
.type 	v1,@object 			#	符号v1的类型为变量(对象)
```
(2)设置符号大小

汇编器指令“.size name , expression”用于设置符号（包括变量和函数）的大小，name为符号名称。当设置变量大小时，expression为一个正整数；当设置函数大小时，expression通常为“.-name”表达式。
``` asm
.size 	short_v,2 	#	设置变量short_v的大小为2字节
.size 	main,.-main 	#	设置函数main的大小为当前位置减去main起始位置
```

(3)指定符号对齐方式

汇编器指令“.align expr”用于指定符号的对齐方式，expr为正整数，用于指示接下来的数据在目标文件中存放地址的对齐方式。不同架构下expr代表的意思不同，例如x86中.align 4代表4字节对齐，而LoongArch中为2的4次方即16字节对齐。

如果想避免因不同架构.align对expr定义不同而带来的不可移植性，可以使用其另外两个变种.balign和.p2align。指令“.balign 4”在任何架构都代表4字节对齐。

(4)指定符号的作用域

和C语言一样，在汇编源文件中定义一个变量或函数符号时也要声明其作用域，用于标识当前符号的作用范围。默认情况下不指定当前符号作用域，符号作用域为当前汇编源文件内可见。其他情况需要使用的相关汇编器指令有“.globlsymbol”“.common symbol”“.local symbol”。

“.globl symbol”用于指定符号symbol（通常为一个全局变量或者非静态成员函数）为全局可见，即对链接器(ld)中其他源文件可见。

:::{tip}
出于兼容原因，.globl还有一种写法是.global。
:::

“.common symbol”声明一个通用符号(CommonSymbol)。这里的通用符号可理解为C语言中未初始化的全局变量。在多个汇编源文件中出现的同名通用符号，在编译器的链接阶段可能会被合并，合并的结果是保留占用空间最大的一个。例如在两个汇编源文件a.S 和b.S里都定义了名为v1的全局变量，但是数据类型不同，具体如下：
``` asm
/* a.S */
.comm 	v1,4,4
/* b.S */
.comm 	v1,8,8
```
在文件a.S中的符号v1为4字节，而在文件b.S中的同名符号v1为8字节，那么链接后，这两个源文件会被合并到一个目标文件，目标文件中仅保留一个8字节的符号v1。

“.local symbol”用于声明一个类似C语言中的未初始化的局部静态变量定义。
``` c
//	c语言变量
static int static_v1;
```
``` asm
#	汇编器指令
.local static_v1
```
##	逻辑控制相关的汇编器指令
这里的逻辑控制包括指定符号数据存放段、常量设置和条件编译、本地标签和程序跳转、编译调试、文件引用、循环展开、宏定义等功能。

1.	指定符号数据存放段

在本书第06章中介绍目标文件ELF格式时介绍过，汇编器会把程序中不同的数据放到不同的段，例如将可执行的机器指令放在代码段(.text)、已经初始化的变量放在数据段.data、未初始化的变量放在.bss段等。在汇编源文件中，可以使用“.datasubsection”“.text subsection”等汇编器指令分别指定接下来的语句要存放在目标文件的数据段和代码段。当需要指定更精细的段类型时，可以使用“.section name”。

例如7.2.1小节中定义一个整型变量和函数add分别放在数据段和代码段。
``` asm
.data 	#	指定接下来的数据存放到目标文件的数据段
str:
.ascii 	"hello\000"
.text 	#	指定接下来的数据存放到目标文件的代码段
add:
```
其实如果我们知道本程序中字符串"hello\0"仅仅用于输出到终端显示，那么可以更精细地将其指定到只读数据段。
``` asm
.section .rodata 	#	指定接下来的数据存放到目标文件的.rodata段
str:
.ascii 	"hello\0"
```
2.	常量设置和条件编译

汇编器指令“.set symbol,expression”可用于常量设置，类似C语言中的宏定义，可以配合汇编器指令.if、.else、.endif使用，从而一起完成一些条件编译。例如要实现一个有条件的输出功能，代码示例如下：
``` asm
.set FLAG,0
.LC0:
.ascii 	"Hello World1!\000"
.LC1:
.ascii 	"Hello World2!\000"
main:
	addi.d 	$sp, $sp, -8
	st.d 	$ra, $sp, 0
	.if FLAG == 1
	la.local $r4, .LC0
	.else
	la.local $r4, .LC1
	.endif
	bl 		$plt(puts)
```
指令“.if FLAG == 1”“.else”“.endif”说明当宏值FLAG为1时，输出字符串“Hello World 1 !”，否则输出“Hello World 2 !”。目前源文件中的汇编器指令“.set FLAG,0”将符号FLAG定义为0，故最终结果输出“Hello World 2 !”。

和.set等价的命令还有“.equ symbol, expression”，该命令也是把符号symbol值设置为expression。

汇编器指令中和条件判断.if类似的命令还有几个，.if的变体具体如表7-1所示。

***TODO_TABLE_7_1***

对于条件编译，也可以在汇编源文件中直接使用C语言中的#ifdef、#else、#endif等预处理命令。当使用C语言预处理命令时，汇编源文件不能直接使用汇编器编译，需要提前使用GCC预处理工具cc1进行预处理命令的翻译。

3.	本地标签和程序跳转

为了方便程序的编写，汇编器指令中提供一种本地标签(Local Label)，用于逻辑跳转。本地标签可采用编号（可以为数字、字母、特殊字符或其组合）加冒号“:”的格式，即“N:”，这里的N为正整数。使用时，还可以通过另外两种表示方式（Nf和Nb）来进行同名标签的位置方向索引。其中，Nf中的f代表forward，用于指示紧接着的下一个同编号的标签；Nb中的b代表backward，用于指示紧接着的上一个同编号的标签。GNU汇编手册中对此给了一个很清晰的示例：
```	asm
1: 	branch 	1f 	#	向后跳转到第3条(即1:branch 2f)位置
2:	branch	1b 	#	向前跳转到第1条(即1:branch 1f)位置
1:	branch	2f	#	向后跳转到第4条(即2:branch 1b)位置
2:	branch 	1b	#	向前跳转到第3条(即1:branch 2f)位置
```
这个示例中有两个同名的本地标签1:和2:，其中branch指代任何架构中的跳转指令，在LoongArch中，可以使用b、bl、beq、jirl等指令。整个跳转过程已经使用行注释标出。整个过程如果使用4个不同标签名可以等价实现为
``` asm
label_1:branch label_3
label_2:branch label_1
label_3:branch label_4
label_4:branch label_3
```
4.	编译调试

可用于汇编器编译过程中的信息输出的指令有“.print string”“.fail expression”“.error string”和“.err”。

“.print string”会让汇编器在标准输出上输出一个字符串。

“.fail expression”会生成一个错误(error)或警告(warning)，当expression的值大于或等于500时，汇编器会输出一条警告信息；当expression的值小于500时，汇编器as会输出一条错误信息；expression默认值为0，可直接写成“.fail”。

“.err”可以在汇编过程中输出一条默认的错误信息，如果要自定义错误信息类型可以使用“.errorstring”指令。这些指令在复杂的宏嵌套或条件汇编时会帮助我们定位问题。

具体使用调试指令示例如下：
``` asm
.print "this is a test for print" 	#	输出信息：this is a test for print
.fail 499 		#	输出信息：warning: .fail 500 encountered
.fail 			#	输出信息：error: .fail 0 encountered
.err 			#	输出信息：error: .err encountered
.error "error happen" 	#	输出信息：error: error happen
```

5.	文件引用

在汇编源文件中引用其他文件有两种方式。一种是使用汇编器指令“.include "file"”，默认引用文件路径为当前目录（Linux系统中为符号“.”），当被引用文件的路径不在同目录时，可以通过汇编器的命令行选项参数“-l”来控制搜索路径；另一种是使用C语言预处理命令“#include”，这时要求汇编器文件必须是.S，且要通过GCC工具（具体为工具cc1）进行预处理。

下面是使用汇编器指令“.include "file"”引用文件的示例。
``` asm
#ref.S
.text
add:
	add.d 	$r4, $r5, $r4
	jr 		$r1
```
``` asm
#main.S
.include "ref.S"
```
这里有两个汇编源文件，分别为ref.S和main.S。ref.S中定义了一个名为add的函数。当另一个文件main.S中想要使用此函数接口时，就需要使用指令“.include "ref.S"”。

6.	循环展开

汇编器指令“.rept count”和“.endr”可用于将其内部的语句循环展开count次。例如：
``` asm
.rept 3
	nop
.endr
```
这就相当于通知汇编器在目标文件中生成3条nop指令，这与直接编写3条nop指令是等价的。
``` asm
nop
nop
nop
```
当需要根据实际情况插入不同数量nop指令来实现地址对齐时，使用循环展开指令是很方便的。

还有一种循环展开命令，其格式为“.irpsymbol,values ...”，实现用values替代symbol的语句序列，也以.endr为结尾。指令中使用symbol的格式为“\symbol”。例如要实现将多个寄存器存储到函数栈上，可写为如下形式：
``` asm
.irp n,4,5,6,7,8,9,10,11,12
st.d 	$r\n, $sp, \n*8
.endr
```
这里实现的是将编号r4～r12的寄存器存储到函数栈上。此命令在目标文件中最终展开后的指令如下：
``` asm
st.d 	$r4, $sp, 32(0x20)
st.d 	$r5, $sp, 40(0x28)
st.d 	$r6, $sp, 48(0x30)
st.d 	$r7, $sp, 56(0x38)
st.d 	$r8, $sp, 64(0x40)
st.d 	$r9, $sp, 72(0x48)
st.d 	$r10, $sp, 80(0x50)
st.d 	$r11, $sp, 88(0x58)
st.d 	$r12, $sp, 96(0x60)
```

7.	宏定义

汇编器指令“.macro name args”功能上类似C语言中宏定义功能，其中name 为宏名称，args为参数，以.endm结尾。例如实现一个可以根据不同参数生成不同数量的nop指令的宏定义示例如下：
``` asm
.text
.macro 	INSERT_NOP a
.rept \a
	nop
.endr
.endm
```
这里使用.text来指示接下来的指令存放位置在最终目标文件的代码段。宏名称为INSERT_NOP，参数为a。宏定义体中使用参数时的格式为“\参数”，例如\a。.macro的参数可以为0，也可以为多个参数。当参数为多个时，参数之间可以用逗号或空格分隔。当程序使用时，直接调用此宏即可，参数可变。例如汇编源文件中某位置需要插入3条nop指令或7条nop指令时，可分别写为:
``` asm
INSERT_NOP 3
INSERT_NOP 7
```
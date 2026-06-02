#	汇编源文件中的汇编器指令

汇编指令是机器指令的可读表示，通常与机器指令一一对应。在GCC编译流程的汇编阶段，汇编器会将汇编指令翻译为机器指令，并存放到目标文件中。汇编器指令与汇编指令不同，它面向汇编器本身，用于指导汇编器如何定义变量和函数、如何在目标文件中组织汇编指令等。换言之，汇编器指令是指导汇编器工作的指令。本节将从符号定义和逻辑控制两个方面介绍汇编源文件中的汇编器指令。

##	符号定义相关的汇编器指令

第06章介绍目标文件格式ELF时提到，函数和变量都可以称为符号，这一概念在汇编源程序中同样适用。编写汇编源程序和使用高级语言一样，需要定义程序所需的常量、变量、函数等符号。汇编器指令中与符号定义相关的指令如下。

1.	定义一个字符串变量

在汇编源程序中定义字符串变量时，需要给出字符串名称、字符串内容、字符串大小、符号类型、对齐方式、变量作用域和变量所在段等完整信息。下面是一个使用C语言定义的字符串变量及其对应的汇编器指令。
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
示例中已经标注了当前字符串定义的相关信息。字符串变量的内容除可使用汇编器指令 .ascii 定义外，还可以使用 .asciz 和 .string 定义。三者都可用于定义字符串，可以不带参数，也可以带多个由逗号分隔的字符串，并将汇编后的每个字符串存入连续地址。其中，.ascii 不会在字符串末尾自动追加零字节；.asciz 会在字符串后自动添加结束符 \0，其中的“z”表示“zero”。下面3种定义字符串的方式是等价的。

``` asm
.ascii 	"hello\0"
.asciz 	"hello"
.string "hello"
```
这里 .string 实际上是 .string8 的缩写。汇编器指令 .string 区分字符宽度，具体形式包括 .string8、.string16、.string32 和 .string64，分别表示一个字符占用8位、16位、32位和64位，存放时还受字节序影响。使用默认的 .string（即 .string8）时，可以将其看作与 .asciz 等价。例如，字符串 `.string "Hello World"` 在龙芯3A5000小尾端机器的目标文件（.o文件）中的存放方式如下。
``` shell
Addr 		Data
0:		6c6c6568 	#	ascii值 lleh
4:		6f77206f 	#	ascii值 ow o
8:		00646c72 	#	ascii值 _dlr
```
而定义为 `.string16 "Hello World"` 后，该字符串在汇编后的目标文件中的存放方式如下。
``` shell
Addr 		Data
0:		00650068 	#	ascii值 _e_h
4:		006c006c 	#	ascii值 _l_l
8:		0020006f 	#	ascii值 _ _o
c: 		006f0077 	#	ascii值 _o_w
10: 		006c0072 	#	ascii值 _l_r
14: 		00000064 	#	ascii值 _ _d
```
为便于直观显示，备注说明中使用 `_` 代替ASCII表中的NUL值。

当源文件中使用的字符串是临时且无名的字符串时，可以使用 .LC0、.LC1 等标签表示。

2.	定义一个整型变量

在汇编源程序中定义整型变量与定义字符串变量类似，需要给出变量名称、变量值、变量大小、变量类型、对齐方式、作用域和变量所在段等信息。下面是一个使用C语言定义的整型变量及其对应的汇编器指令。
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
汇编器指令中用于定义数据长度的指令有 `.byte value`、`.half value`、`.word value` 和 `.dword value`。.byte 用于定义一字节（8位）的地址空间，其他命令所定义的空间大小依赖于具体系统，value 为变量值。在LoongArch中，.half 为2字节，.word 为4字节，.dword 为8字节。这里变量 int_v 为 int 类型，因此使用的数据定义为 `.word 20`，数据长度 .size 为4字节。

3.	定义一个函数

在汇编源程序中定义函数时，需要给出函数名称、函数汇编指令（可以使用宏指令）、函数大小、符号类型、对齐方式、作用域和函数所在段等信息。下面是一个使用C语言定义的函数及其对应的汇编器指令。

``` c
int add(int a,int b){
	return a + b;
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
函数通常放在代码段，因此这里指定符号 add 所在段为 .text。前面列举的字符串变量和整型变量的类型均为 @object，而函数 add 的类型应定义为 @function，表示该符号是函数；同时通过 `.global add` 指定它是全局函数。函数符号 `add:` 后即可编写汇编指令以实现相应的函数功能，这里仅包含 add.w 和 jr 两条指令。

符号 add 的大小使用 `.size add, .-add` 格式定义，其中 add 表示符号（函数）名称，`.-add` 表示指令占用的内存大小为当前位置（`.`）减去函数 add 的起始位置。当前函数共有2条指令，即8字节。这里为什么使用 `.size add, .-add`，而不是 `.size add, 8`？原因是许多汇编源文件中存在可能被扩展为多条机器指令的宏指令，例如 li.w/li.d。此时函数实际占用的内存大小需要在汇编器完成机器指令翻译后才能确定，因此这里使用基于函数符号的动态计算方式定义函数大小。

4.	符号定义相关的汇编器指令说明

(1)设置符号类型

定义符号类型的汇编器指令为 .type，其后常见的类型有 @function 和 @object，分别表示当前符号为函数和变量。
``` asm
.type 	add, @function 		#	符号add的类型为函数
.type 	v1,@object 			#	符号v1的类型为变量(对象)
```
(2)设置符号大小

汇编器指令 `.size name, expression` 用于设置符号（包括变量和函数）的大小，name 为符号名称。当设置变量大小时，expression 为一个正整数；当设置函数大小时，expression 通常为 `.-name` 表达式。
``` asm
.size 	short_v,2 	#	设置变量short_v的大小为2字节
.size 	main,.-main 	#	设置函数main的大小为当前位置减去main起始位置
```

(3)指定符号对齐方式

汇编器指令 `.align expr` 用于指定符号的对齐方式，expr 为正整数，用于指示接下来数据在目标文件中的存放地址如何对齐。不同架构下 expr 的含义不同，例如 x86 中 `.align 4` 表示4字节对齐，而LoongArch中表示按2的4次方对齐，即16字节对齐。

如果想避免不同架构对 .align 参数含义不同所带来的不可移植性，可以使用另外两个变体 .balign 和 .p2align。指令 `.balign 4` 在任何架构上都表示4字节对齐。

(4)指定符号的作用域

和C语言一样，在汇编源文件中定义变量或函数符号时，也需要声明其作用域，用于标识当前符号的可见范围。默认情况下，如果不指定当前符号的作用域，该符号仅在当前汇编源文件内可见。其他情况下常用的相关汇编器指令有 `.globl symbol`、`.common symbol` 和 `.local symbol`。

`.globl symbol` 用于指定符号 symbol（通常为全局变量或非静态函数）全局可见，即对链接器（ld）处理的其他源文件可见。

:::{tip}
出于兼容原因，.globl 还可以写作 .global。
:::

`.common symbol` 声明一个通用符号（Common Symbol）。这里的通用符号可理解为C语言中未初始化的全局变量。多个汇编源文件中出现的同名通用符号，在链接阶段可能会被合并，合并结果是保留占用空间最大的一个。例如，两个汇编源文件 a.S 和 b.S 中都定义了名为 v1 的全局变量，但数据类型不同，具体如下：
``` asm
/* a.S */
.comm 	v1,4,4
/* b.S */
.comm 	v1,8,8
```
文件 a.S 中的符号 v1 为4字节，文件 b.S 中的同名符号 v1 为8字节。链接后，这两个源文件会被合并到一个目标文件中，目标文件中仅保留一个8字节的符号 v1。

`.local symbol` 用于声明类似C语言中未初始化的局部静态变量的符号。
``` c
//	c语言变量
static int static_v1;
```
``` asm
#	汇编器指令
.local static_v1
```
##	逻辑控制相关的汇编器指令
这里的逻辑控制包括指定符号数据存放段、常量设置与条件编译、本地标签与程序跳转、编译调试、文件引用、循环展开、宏定义等功能。

1.	指定符号数据存放段

第06章介绍目标文件ELF格式时已经说明，汇编器会把程序中的不同数据放到不同的段中，例如将可执行的机器指令放在代码段（.text），将已初始化的变量放在数据段（.data），将未初始化的变量放在 .bss 段等。在汇编源文件中，可以使用 `.data subsection`、`.text subsection` 等汇编器指令，分别指定接下来的语句存放到目标文件的数据段和代码段。当需要指定更精细的段类型时，可以使用 `.section name`。

例如，7.2.1小节中定义的整型变量和函数 add 分别放在数据段和代码段。
``` asm
.data 	#	指定接下来的数据存放到目标文件的数据段
str:
.ascii 	"hello\000"
.text 	#	指定接下来的数据存放到目标文件的代码段
add:
```
如果已知本程序中的字符串 `"hello\0"` 仅用于输出到终端显示，也可以更精细地将其指定到只读数据段。
``` asm
.section .rodata 	#	指定接下来的数据存放到目标文件的.rodata段
str:
.ascii 	"hello\0"
```
2.	常量设置和条件编译

汇编器指令 `.set symbol, expression` 可用于常量设置，功能类似C语言中的宏定义，也可以配合汇编器指令 .if、.else、.endif 使用，以完成条件编译。例如，要实现一个有条件的输出功能，代码示例如下：
``` text
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
指令 `.if FLAG == 1`、`.else` 和 `.endif` 表示：当宏值 FLAG 为1时，输出字符串“Hello World 1!”；否则输出“Hello World 2!”。当前源文件中的汇编器指令 `.set FLAG, 0` 将符号 FLAG 定义为0，因此最终输出“Hello World 2!”。

与 .set 等价的命令还有 `.equ symbol, expression`，该命令同样用于将符号 symbol 的值设置为 expression。

汇编器指令中还有若干与条件判断 .if 类似的命令，.if 的变体如表7-1所示。

```{image} ../../img/ch7/t2p_7_1.png
:alt: 汇编器指令.if的变体
:class: bg-primary
:scale: 80 %
:align: center
```

对于条件编译，也可以在汇编源文件中直接使用C语言中的 #ifdef、#else、#endif 等预处理命令。使用C语言预处理命令时，汇编源文件不能直接交给汇编器编译，需要先使用GCC预处理工具 cc1 翻译预处理命令。

3.	本地标签和程序跳转

为便于程序编写，汇编器提供了本地标签（Local Label），用于逻辑跳转。本地标签可采用编号（可以为数字、字母、特殊字符或其组合）加冒号“:”的格式，即 `N:`，这里的 N 为正整数。使用时，还可以通过 Nf 和 Nb 两种形式对同名标签进行方向索引。其中，Nf 中的 f 表示 forward，用于指示后方紧接着的下一个同编号标签；Nb 中的 b 表示 backward，用于指示前方紧接着的上一个同编号标签。GNU汇编手册对此给出了一个清晰的示例：
``` text
1: 	branch 	1f 	#	向后跳转到第3条(即1:branch 2f)位置
2:	branch	1b 	#	向前跳转到第1条(即1:branch 1f)位置
1:	branch	2f	#	向后跳转到第4条(即2:branch 1b)位置
2:	branch 	1b	#	向前跳转到第3条(即1:branch 2f)位置
```
该示例中有两组同名的本地标签 `1:` 和 `2:`。其中 branch 指代任意架构中的跳转指令；在LoongArch中，可以使用 b、bl、beq、jirl 等指令。整个跳转过程已经使用行注释标出。如果改用4个不同的标签名，上述过程可以等价实现为：
``` asm
label_1:branch label_3
label_2:branch label_1
label_3:branch label_4
label_4:branch label_3
```
4.	编译调试

可用于汇编器编译过程中信息输出的指令有 `.print string`、`.fail expression`、`.error string` 和 `.err`。

`.print string` 会让汇编器在标准输出上输出一个字符串。

`.fail expression` 会生成一个错误（error）或警告（warning）。当 expression 的值大于或等于500时，汇编器会输出一条警告信息；当 expression 的值小于500时，汇编器 as 会输出一条错误信息。expression 的默认值为0，因此可直接写成 `.fail`。

`.err` 可以在汇编过程中输出一条默认的错误信息；如果要自定义错误信息，可以使用 `.error string` 指令。这些指令在复杂宏嵌套或条件汇编场景中有助于定位问题。

具体使用调试指令示例如下：
``` asm
.print "this is a test for print" 	#	输出信息：this is a test for print
.fail 499 		#	输出信息：error: .fail 499 encountered
.fail 			#	输出信息：error: .fail 0 encountered
.err 			#	输出信息：error: .err encountered
.error "error happen" 	#	输出信息：error: error happen
```

5.	文件引用

在汇编源文件中引用其他文件有两种方式。一种是使用汇编器指令 `.include "file"`，默认引用文件路径为当前目录（Linux系统中为符号“.”）；当被引用文件不在同一目录时，可以通过汇编器命令行选项 `-I` 控制搜索路径。另一种是使用C语言预处理命令 `#include`，这要求汇编源文件必须是 .S 文件，并通过GCC工具（具体为 cc1）进行预处理。

下面是使用汇编器指令 `.include "file"` 引用文件的示例。
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
这里有两个汇编源文件，分别为 ref.S 和 main.S。ref.S 中定义了一个名为 add 的函数。当另一个文件 main.S 需要使用此函数接口时，可以使用指令 `.include "ref.S"`。

6.	循环展开

汇编器指令 `.rept count` 和 `.endr` 可用于将其内部的语句循环展开 count 次。例如：
``` asm
.rept 3
	nop
.endr
```
这相当于通知汇编器在目标文件中生成3条 nop 指令，与直接编写3条 nop 指令等价。
``` asm
nop
nop
nop
```
当需要根据实际情况插入不同数量的 nop 指令以实现地址对齐时，使用循环展开指令较为方便。

还有一种循环展开命令，其格式为 `.irp symbol, values ...`，用于以 values 依次替代 symbol 并展开语句序列，同样以 .endr 结尾。指令中使用 symbol 的格式为 `\symbol`。例如，要将多个寄存器存储到函数栈上，可写为如下形式：
``` text
.irp n,4,5,6,7,8,9,10,11,12
st.d 	$r\n, $sp, \n*8
.endr
```
这里实现的是将编号 r4～r12 的寄存器存储到函数栈上。该命令在目标文件中最终展开后的指令如下：
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

汇编器指令 `.macro name args` 的功能类似C语言中的宏定义，其中 name 为宏名称，args 为参数，宏定义以 .endm 结尾。例如，实现一个可根据不同参数生成不同数量 nop 指令的宏，示例如下：
``` text
.text
.macro 	INSERT_NOP a
.rept \a
	nop
.endr
.endm
```
这里使用 .text 指示接下来的指令存放在最终目标文件的代码段。宏名称为 INSERT_NOP，参数为 a。宏定义体中使用参数时的格式为 `\参数`，例如 `\a`。.macro 的参数可以为0个，也可以为多个；当参数为多个时，参数之间可以用逗号或空格分隔。程序使用时，直接调用此宏并传入参数即可。例如，汇编源文件中某处需要插入3条 nop 指令或7条 nop 指令时，可分别写为：
``` asm
INSERT_NOP 3
INSERT_NOP 7
```

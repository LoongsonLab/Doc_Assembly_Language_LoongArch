#	ELF文件格式解析

ELF是Linux系统中常用的目标文件存储格式。典型目标文件主要分为以下3类。

-	可重定位文件（Relocatable File）：尚未经过链接的目标文件，内部包含编译器生成的汇编代码和数据，可与其他可重定位文件一起链接，形成可执行文件或动态库。常见扩展名为`.o`。

-	可执行文件（Executable File）：经过链接器处理后，可由Linux系统直接执行的目标文件，内部包含可运行的机器指令和数据。此类文件通常没有扩展名。

-	动态库文件（Shared Object File）：动态库用于共享程序代码，内容与可重定位文件类似，包含可参与链接的代码和数据，可视为多个可重定位文件的集合。常见扩展名为`.so`。动态库会参与两个阶段：链接阶段，链接器将其与其他可重定位文件或动态库共同链接生成可执行文件；运行阶段，动态链接器在需要时把动态库加载到内存。

ELF文件保存的是处理器可执行的二进制指令和相关数据。不同系统架构下，ELF内部格式和数据处理方式会有一些差异，但整体组织形式如图6-1所示。

```{image} ../../img/ch6/pic_6_1.png
:alt: ELF文件基本格式
:class: bg-primary
:scale: 80 %
:align: center
```

从图6-1可以看到，不同目标文件类型的基本布局相近，但具体内容有所区别。图6-1(a)中的可重定位文件由ELF文件头（ELF Header）、节（Section）和节头表（Section Header Table）组成。图6-1(b)中的可执行文件由ELF文件头、段（Segment）和程序头表（Program Header Table）组成。可重定位文件中的节和可执行文件中的段都用于保存程序代码、数据等内容。区别在于，可执行文件中的一个段通常由多个可重定位文件中属性相关的节合并而来，并且代码已经是重定位后的最终机器指令，如图6-2所示。

```{image} ../../img/ch6/pic_6_2.png
:alt: 多个可重定向文件的相同节映射到一个可执行文件的段域
:class: bg-primary
:scale: 80 %
:align: center
```

日常开发中，许多程序员并不会严格区分Section和Segment，通常都称为“段”，不少教材也不会深入区分。因此，本章后续统一称为“段”，只有在确实需要区分时再补充英文说明。

##	ELF文件头

ELF文件头描述目标文件的整体组织方式，是目标文件基本信息的汇总，包括字长、字节序、ELF头大小、目标文件类型、机器类型、节头表/程序头表大小和数量、程序入口点等。ELF文件头必须位于目标文件最开始的位置。可以使用`readelf`查看可重定位文件`hello.o`的ELF头信息。
``` shell
$ readelf -h hello.o
ELF 头：
Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
Class:                           ELF64
Data:                            2补码，小端序(little endian)
Version:                         1(current)
OS/ABI:                          UNIX - System V
ABI Version:                     0
Type:                            REL（可重定向文件）
Machine:                         LoongArch
Version:                         0x1
入口点地址：                       0x0
程序头起点：                       0(bytes into file)
Start of section headers:        856 (bytes into file)
标志：                            0x3, LP64
本头的大小：       64（字节）
程序头大小：       0（字节）
Number of program headers:         0
节头大小：         64（字节）
节头数量：         13
字符串表索引节头： 10
```
这里，`readelf`参数`-h`表示header，用于查看目标文件头信息。部分字段含义如下。

-	魔数（Magic）：用于识别文件格式和类型。前4字节`7f 45 4c 46`表示这是ELF格式文件。第5字节`02`表示该文件面向64位体系架构（`01`表示32位）。第6字节`01`表示小端序（`02`表示大端序），也可从后面的Data字段看出。第7字节`01`表示ELF版本，后续9字节未定义。

-	Type：表示当前文件类型，可为可重定位文件、可执行文件或动态库文件。示例中类型为可重定位文件。可重定位文件通常没有入口点，因此入口点地址为0x0；同时也没有程序头，所以程序头起点、程序头大小和Number of program headers均为0。

-	Machine：表示处理器架构。当前结果为LoongArch，表示龙芯架构，其数值为258。

-	Start of section headers：表示节头表数据在当前文件中的起始位置。

-	标志：用于表示处理器相关的ABI类型。对龙芯处理器而言，ABI类型可以是LP64s、LP64f、LP64d、ILP32s、ILP32f、ILP32d等。

##	可重定位文件中的段和段头表

可重定位文件中的节头表用于描述ELF各个节（Section）的信息，包括节名称、长度、文件偏移、读写权限、地址等。可以使用`readelf -S`查看可重定位文件`hello.o`中节头表的详细内容。
``` shell
# readelf -S hello.o
There are 11 section headers, starting at offset 0x358:
节头：
[号]名称           类型      地址             偏移量    大小  旗标 链接 信息 对齐
[0]                NULL     0000000000000000 00000000 000000    0  0   0
[1] .text          PROGBITS 0000000000000000 00000040 000034 AX 0  0   4
[2] .rela.text     RELA     0000000000000000 000001b0 000150  I 8  1   8
[3] .data          PROGBITS 0000000000000000 00000074 000000 WA 0  0   1
[4] .bss           NOBITS   0000000000000000 00000074 000000 WA 0  0   1
[5] .rodata       PROGBITS  0000000000000000 00000078 00000e  A 0  0   8
[6] .note.GNU-stack PROGBITS 0000000000000000  00000086  000000    0  0   1
[7] .comment       PROGBITS 0000000000000000 00000086 00028  MS 0  0  1
[8] .symtab        SYMTAB   0000000000000000 000000b0 0000f0    9  8  8
[9] .strtab        STRTAB   0000000000000000 000001a0 000010    0  0  1
[10] .shstrtab     STRTAB   0000000000000000 00000300 000052    0  0  1
Key to Flags:
W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
L (link order), O (extra OS processing required), G (group), T (TLS),
C (compressed), x (unknown), o (OS specific), E (exclude),
p (processor specific)
```
从节头表可以看出，当前`hello.o`中包含多个节。每个节的信息包括名称、类型、地址、偏移量、大小、旗标、链接、信息和对齐。

1.	段名

输出中的第一列为段名，在表头中显示为“名称”。段名通常以`.`开头，常见名称包括`.text`、`.data`、`.bss`等。通过段名可以直观看出该段的大致用途。例如，`.text`用于保存代码（机器指令），也称代码段；`.data`用于保存数据，也称数据段；`.rodata`用于保存只读数据，也称只读数据段。常见段名及功能如表6-1所示。

```{image} ../../img/ch6/t2p_6_1.png
:alt: 常见的段名及其功能描述
:class: bg-primary
:scale: 80 %
:align: center
```

一个简单C程序被编译为目标文件后，各类内容在目标文件中的存放位置如图6-3所示。

```{image} ../../img/ch6/pic_6_3.png
:alt: c语言程序在目标文件中存放的位置
:class: bg-primary
:scale: 80 %
:align: center
```

通常，C程序编译生成的机器指令存放在代码段（`.text`）；已初始化的全局变量和局部静态变量保存在数据段（`.data`）；未初始化的全局变量和局部静态变量保存在`.bss`段；字符串常量、使用`const`修饰的不可修改变量等保存在只读数据段（`.rodata`）。这种分段存储有多方面好处：运行时，不同段会被映射到具有不同权限的内存区域，如只读、可写、可执行等，从而提升安全性，避免代码段被错误修改；多个进程运行同一程序时，可共享同一份指令段，节省内存；同时，分段也有助于提高缓存命中率，改善性能。

2.	段类型

对编译器来说，段名并不是决定段属性的关键，真正决定段属性的是段类型（输出中的“类型”）和段标志（输出中的“旗标”）。段类型可分为程序段、重定位表段、符号表段等。常见段类型及含义如表6-2所示。

```{image} ../../img/ch6/t2p_6_2.png
:alt: 常见的段类型及其含义
:class: bg-primary
:scale: 80 %
:align: center
```

程序中的段类型通常以`SHT_`开头，如`SHT_NULL`、`SHT_SYMTAB`等，但`readelf`显示时会省略`SHT_`前缀。

3.	段标志

段标志（Flag）在输出中显示为“旗标”，用于表示该段在进程虚拟空间中的访问属性。常见属性包括可写（Write）、可执行（Execute）和可分配（Alloc），所有段默认可读。例如，示例中`.text`段的标志AX表示Alloc+Execute，即该段在内存中可分配且可执行，但不可写。如果程序运行时错误写入该段，就会触发SIGSEGV异常。`.data`段和`.bss`段的标志WA表示Write+Alloc，即可写并可分配空间；`.symtab`、`.strtab`段标志为空，表示没有可写和可执行权限。

4.	段地址、偏移量和大小

段地址（输出中的“地址”）记录该段加载到内存后的虚拟起始地址。由于当前`hello.o`尚未重定位，它在进程中的最终位置还不确定，所以所有段地址都显示为0000000000000000。如果查看最终可执行文件`hello`，段地址会显示为类似下面的非零有效地址。

``` shell
 [10] .text  PROGBITS  00000001200008f0  000008f0 000270   AX   0   0   16
```
这里的虚拟地址就是`.text`段最终加载到内存后的起始虚拟地址。虽然进程理论上拥有一定范围的虚拟地址空间，但一般不能使用全部空间，系统通常会预留一部分虚拟地址用于自身配置。

偏移量（Offset）表示该段在ELF文件中的位置。例如，前面`hello.o`段表中`.text`段的Offset为0x40（64字节），而`readelf -h hello.o`显示“本头的大小：64（字节）”，说明ELF头之后紧接着就是`.text`段。

大小（Size）表示该段长度。例如，`.text`段大小为000040，即十六进制0x40，换算为十进制后表示代码段占64字节。`.data`段和`.bss`段大小均为000000，表示这些段当前没有内容，不占文件空间。

5.	段地址对齐

如果某个段有地址对齐要求，段地址对齐字段（输出中的“对齐”）会给出对齐方式。例如，`.text`段对齐值为4，表示该段在内存中的起始地址必须按4字节对齐，也就是起始地址能被4整除。对齐值为0或1时，可认为该段没有额外对齐要求。示例中的`.bss`和`.data`段对齐值均为1，表示存放到内存时没有额外对齐限制。

##	可执行文件中的段和程序头表

可重定位文件中描述Section属性的结构称为节头表（Section Header Table），而可执行文件和动态库文件中描述Segment属性的结构称为程序头表（Program Header Table）。程序头表指导系统如何把多个段（Segment）加载到内存。前文提到，Segment可看作多个可重定位文件（`.o`）中属性相似Section的合并，即一个Segment包含一个或多个属性相近的Section。这里的属性相似主要指权限（由段标志Flag指定）。链接器会把多个`.o`文件中具有可执行权限的`.text`、`.init`等段放入最终可执行文件的同一个Segment，以节省内存。因为ELF文件加载时以系统页为单位，如果一个ELF文件有10个段且每个段都小于一个内存页，按每段占一个页计算，进程就需要10个页；若链接器把权限相同的段合并映射，所需内存页通常会少于10个，从而提高内存页利用率并减少碎片。例如，`.data`和`.bss`都具有WA权限，就可以放在同一个页或多个连续页中。程序头表记录的正是这些合并后的Segment信息。

可以使用`readelf -S`或`readelf -l`查看可执行文件`hello`中的段信息和程序头表信息。
``` shell
# readelf -l hello
Elf 文件类型为 EXEC（可执行文件）
Entry point 0x120000580
There are 9 program headers, starting at offset 64
程序头：
Type             Offset      VirtAddr    FileSiz     MemSiz   Flags Align
PHDR          0x000000040  0x120000040  0x0001f8  0x0000001f8   R   0x8
INTERP        0x000000238  0x120000238  0x00000f  0x00000000f   R   0x1
LOAD          0x000000000  0x120000000  0x0007f8  0x0000007f8  RE  0x4000
LOAD          0x000003e30  0x120007e30  0x000230  0x000000238  RW  0x4000
DYNAMIC       0x000003e40  0x120007e40  0x0001c0  0x0000001c0  RW  0x8
NOTE          0x000000248  0x120000248  0x000044  0x000000044   R  0x4
GNU_EH_FRAME  0x0000007a8  0x1200007a8  0x000014  0x000000014   R  0x4
GNU_STACK     0x000000000  0x000000000  0x000000  0x000000000  RW  0x10
GNU_RELRO     0x000003e30  0x120007e30  0x0001d0  0x0000001d0   R  0x1
Section to Segment mapping:
段节...
00
01  .interp
02  .interp .note.ABI-uag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .plt .text .rodata .eh_frame_hdr .eh_frame
03  .init_array .fini_array .dynamic .got.plt .got .sdata .bss
04  .dynamic
05  .note.ABI-tag .note.gnu.build-id
06  .eh_frame_hdr
07
08  .init_array .fini_array .dynamic 
```
程序头信息显示当前文件共有9个Segment，编号从00到08。从输出看，Segment不再需要段名，但保留了Section与Segment之间的映射关系，即`Section to Segment mapping:`部分。访问属性相同的Section会被归入同一个Segment。例如，具有可读可执行权限的`.text`、`.rodata`等段被放入编号02的Segment，对应程序头信息中的第3行。
``` shell
Type        Offset      VirtAddr    FileSiz       MemSiz     Flags  Align
LOAD     0x000000000  0x120000000   0x0007f8    0x0000007f8    RE   0x4000
```
其中，LOAD表示程序运行时该段需要加载到内存。VirtAddr表示该段加载到进程内存后的起始虚拟地址；FileSiz表示该段在ELF文件中占用的空间长度；MemSiz表示该Segment在进程内存中占用的长度。对于代码段，MemSiz通常等于FileSiz；对于数据段，MemSiz可能大于FileSiz。Flags表示权限属性，包括可读（R）、可写（W）和可执行（E）。当前02段为代码段（`.text`）所在段，因此权限为RE，不可写。Align表示该Segment加载到内存时的对齐方式。

##	符号和符号表

目标文件中的函数和变量统称为符号（Symbol）。这里的变量指全局变量或静态局部变量，不包括函数栈中的普通局部变量。函数名和变量名统称为符号名（Symbol Name）。每个可重定位目标文件都包含符号表（Symbol Table），用于记录该文件中使用到的所有符号，以及符号名、符号类型、符号大小等信息。依靠符号和符号表，编译器才能在链接阶段解析多个目标文件之间变量和函数的关系，并配合重定位表完成重定位，最终把多个目标文件合并为可执行文件或动态库。

符号按定义方式可分为局部符号、全局符号、外部符号和段符号4类。

-	局部符号：对应C语言函数内部定义的静态局部变量和静态函数，例如下面的变量a和函数`fun()`。
``` c
static int a;
static int fun(){
}
```
这里有两个局部符号，符号名分别为a和fun。它们只在当前编译单元内部，也就是当前目标文件内部可见。

-	全局符号：定义在当前目标文件中，并且可以被其他文件引用的变量或函数。例如下面C程序中，`global_var`和`main`都是全局符号。
``` c
int global_var;
int main(int arg,char* arg[]){
}
```

-	外部符号（External Symbol）：当前目标文件中引用、但定义在其他目标文件或库中的全局符号。例如常用的`printf`函数（定义在libc中），或通过`extern`声明的外部变量。

-	段符号：由编译器生成的`.text`、`.data`等段名也属于符号。

目标文件中，符号表所在段为`.symtab`。如果是可执行目标文件，还会有`.dynsym`段存放动态符号表。可以使用`readelf -s 文件名`查看符号表信息。例如下面的C语言代码：
``` c
#include <stdio.h>
char* str="HELLO";
int global_var;
int main(){
	int a = 0,b = 0;
	static int static_a = 0;
	printf("%s %d\n",str,a+b+static_a);
	return 0;
}
```
编译生成可重定位文件`hello.o`后，其符号表信息如下：
``` shell
$ readelf -s hello.o
Symbol table '.symtab' contains 13 entries:
Num:   Value          Size  Type    Bind     Vis    Ndx    Name
0: 0000000000000000    0   NOTYPE   LOCAL  DEFAULT  UND
1: 0000000000000000    0   SECTION  LOCAL  DEFAULT     1
2: 0000000000000000    0   SECTION  LOCAL  DEFAULT     3
3: 0000000000000000    0   SECTION  LOCAL  DEFAULT     5
4: 0000000000000000    0   SECTION  LOCAL  DEFAULT     6
5: 0000000000000000    4   OBJECT    LOCAL  DEFAULT     5    static_a
6: 0000000000000000    0   SECTION  LOCAL  DEFAULT     7
7: 0000000000000000    0   NOTYPE   LOCAL  DEFAULT     6    .LC0
8: 0000000000000000    0   SECTION  LOCAL  DEFAULT     8
9: 0000000000000000    8   OBJECT    GLOBAL DEFAULT    3     str
10: 0000000000000000    4   OBJECT    GLOBAL DEFAULT    COM    global_var
11: 0000000000000000   60  FUNC      GLOBAL DEFAULT     1     main
12: 0000000000000000    0   NOTYPE   GLOBAL DEFAULT    UND    puts
```

可以看到，`hello.o`中包含多个符号，编号显示在Num列。每个符号包含以下属性。

-	符号值（Value）：每个符号都有对应值。如果符号是函数或变量，该值通常表示函数或变量的虚拟地址。由于`hello.o`尚未重定位，符号值大多为0。在可执行文件中，符号值可能是符号虚拟地址或符号在函数中的偏移。对于COM类型符号，Value列表示其对齐方式。

-	符号大小（Size）：变量的符号大小就是其数据类型大小，单位为字节。例如示例中的`static_a`是`int`类型，大小为4字节；字符串指针`str`在LA64架构上为8字节。函数的符号大小是该函数编译后所有机器指令占用的字节数。例如`main`大小为60字节，龙芯指令每条4字节，因此可推算`main`编译后共有15条机器指令。

-	符号类型(Type)分为如下种类。

	-	NOTYPE：未知符号类型，包括目标文件中用于条件跳转的标签、外部定义符号等。

	-	OBJECT：数据对象，如C语言变量、字符串、数组等。

	-	FUNC：函数或其他可执行代码。

	-	SECTION：一个段。

	-	FILE：文件名。
-	绑定信息（Bind）分为如下种类。

	-	LOCAL：局部符号。例如上面定义的静态局部变量`static_a`。

	-	GLOBAL：全局符号，包括本文件中定义的全局变量`global_var`、`str`、`main`，以及外部函数`printf`。

	-	WEAK：弱引用符号。示例中没有体现。对于C/C++，编译器默认把函数和已初始化的全局变量视为强符号，把未初始化的全局变量和使用`__attribute__((weak))`定义的变量视为弱符号。

-	Vis：可扩展符号功能，目前未定义具体功能，可忽略。

-	符号所在段（Ndx）：如果符号定义在当前目标文件中，该字段表示符号所在段在段表中的下标。例如静态变量`static_a`所在段索引为4。Ndx还有如下3种特殊值。

	-	UND：未定义，通常表示外部符号，不在当前目标文件中定义。例如libc中的`printf`函数，或用`extern`声明的外部变量。

	-	ABS：表示该符号包含一个绝对值，比如符号hello.c。

	-	COM：表示该符号是未初始化的全局符号，例如变量`global_var`。

-	符号名（Name）：符号表最后一列，如`static_a`、`.LC0`、`str`、`global_var`、`main`和`puts`。有些符号没有名字，只能通过编号识别。无名符号通常是段（可从Type列的SECTION看出）或未知类型符号。

##	重定位和重定位表

重定位分为链接时重定位和加载时重定位。链接时重定位是指编译器在链接阶段把多个可重定位目标文件合并为一个可执行目标文件时，确定文件中程序数据和函数调用指令地址的过程。动态库中的数据加载和函数调用定位不属于链接时重定位，而需要在加载时重定位完成。加载时重定位主要面向动态库，即程序运行过程中加载动态库时，确定动态库中函数调用和数据引用的绝对地址。

同一文件内部的函数调用通常不需要重定位，因为函数之间的相对位置是固定的，链接时同一文件内函数通常连续存放。重定位主要处理不同文件或不同模块之间的函数调用和数据引用，这里的模块可以是动态库或可执行目标文件。下面给出一个简单的链接时重定位示例：在两个C文件中，`a.c`调用`b.c`中的`temp`函数。
``` c
/* a.c */
extern void temp();
int main(){
	temp();
}
```
``` c
/* b.c */
void temp(){
	// do nothing
}
```
在编译器尚未执行重定位前，目标文件`a.o`和`b.o`中的指令信息如下：
``` shell
$objdump -d a.o
Disassembly of section .text:
0000000000000000 <main>:
0:			02ffc063 	addi.d 		$r3, $r3, -16(0xff0)
4:			29c02061 	st.d 		$r1, $r3, 8(0x8)
8: 			27000076 	stptr.d 	$r22, $r3, 0
c:			02c04076 	addi.d 		$r22, $r3, 16(0x10)
10: 		54000000 	bl 			0 		# 10 <main + 0x10>
...
$objdump -d b.o
Disassembly of section .text:
0000000000000000 <temp>:
0:			02ffc063 	addi.d 		$r3, $r3, -16(0xff0)
4:			29c02076 	st.d 		$r22, $r3, 8(0x8)
...
```
前文已经说明，链接前的可重定位目标文件中，所有段起始地址都是0。当前`a.o`的代码段（`.text`）中只有`main`，因此`main`起始地址为0000000000000000；`b.o`的代码段中只有`temp`，因此`temp`起始地址也为0000000000000000。相对跳转指令`bl 0`用于调用`temp`，但此时跳转目标地址为0，说明编译器还不知道`temp`的实际位置。链接时确定`temp`地址后，需要重新修正这条指令。

再看链接后生成的可执行目标文件`a.out`中相关函数地址和指令：
``` shell
$objdump -d a.out
Disassembly of section .text:
000000000000066c <main>:
66c:		02ffc063 	addi.d 		$r3, $r3, -16(0xff0)
670:		29c02061 	st.d 		$r1, $r3, 8(0x8)
674: 		27000076 	stptr.d 	$r22, $r3, 0
678:		02c04076 	addi.d 		$r22, $r3, 16(0x10)
67c: 		54001c00 	bl 			28(0x1c) 		
...
0000000000000698 <temp>:
698:		02ffc063 	addi.d 		$r3, $r3, -16(0xff0)
69c:		29c02076 	st.d 		$r22, $r3, 8(0x8)
...
```
链接后的`a.out`中，`main`和`temp`的起始地址已经确定，分别为0x00000066c和0x000000698。调用`temp`的相对跳转指令`bl`也已经被修正，由`bl 0`变为`bl 28`。当前PC为0x00000067c，加上偏移值28（0x1c）后正好得到0x000000698，也就是`temp`的起始地址。

链接过程中最重要的两项工作是地址分配和重定位。地址分配会处理所有输入文件（此处为`a.o`和`b.o`），读取符号信息、段长度、属性等内容，并据此合并相同属性的段、确定符号地址，例如确定`main`和`temp`的起始地址为0x00000066c和0x000000698。重定位则在地址分配基础上，确定数据加载指令或函数调用指令的地址并修改对应机器指令，例如把`bl 0`改为`bl 28`。

并非所有数据加载指令或函数调用指令都需要修改。那么哪些指令需要修正、如何修正？这些信息由目标文件中的重定位表记录。在可重定位目标文件中，每个包含待修正指令的段，都会对应一个重定位段。例如，`a.o`的`.text`段中有需要修正的`bl`指令，因此`a.o`中会有一个`.rela.text`段，用来记录待修正指令的位置、修正方式、目标符号名称等信息。如果`.data`段中也有需要修正的内容，也会有相应重定位段。重定位表就是这些信息的集合。

使用`objdump -r`可以查看`a.o`中的重定位信息：
``` shell
$ objdump -r a.o
a.o：   文件格式 elf64-loongarch
RELOCATION RECORDS FOR [.text]:
OFFSET    TYPE      VALUE
0000000000000010 R_LARCH_SOP_PUSH_PLT_PCREL  temp
```
这表示`a.o`代码段中存在需要地址修正的指令，其在当前目标文件中的偏移地址为0x10，即前面看到的指令：
``` shell
10: 	54000000 	bl 	0 	#	10<main + 0x10>
```
符号名`temp`说明修正后的目标是函数`temp`。地址修正类型，也称重定位类型，为`R_LARCH_SOP_PUSH_PLT_PCREL`，它说明如何进行地址修正。每种体系架构都有自己的重定位类型集合，这属于架构ABI范畴。查阅龙芯架构参考手册ABI部分可知，`R_LARCH_SOP_PUSH_PLT_PCREL`表示使用跳转目标地址与当前PC之间的相对寻址进行修正。这里跳转目标为`temp`，链接地址分配后其地址为0x000000698，当前PC为`bl 0`所在地址0x00000067c。因此，相对偏移为0x1c（0x000000698-0x00000067c），修正后的`bl`机器指令从54000000变为54001c00。

LoongArch ABI支持60余种重定位类型，完整列表可参阅龙芯架构参考手册ABI部分。表6-3列出了部分LoongArch重定位类型。

```{image} ../../img/ch6/t2p_6_3.png
:alt: 部分LoongArch支持的重定位类型
:class: bg-primary
:scale: 80 %
:align: center
```

表6-3列举了4种重定位类型。对于与某个重定位相关联的符号，计算方式中RtAddr表示该符号的运行时地址，A表示额外加数，B表示该重定位所在段所属模块被加载到内存时的装载地址。

#	ELF文件格式解析

ELF文件是用在Linux系统下的一种目标文件存储格式。典型的目标文件有以下3类。

-	可重定向文件(Relocatable File)：还未经过链接的目标文件。其内容包含经过编译器编译的汇编代码和数据，用于和其他可重定向文件一起链接形成一个可执行文件或者动态库。通常文件扩展名为.o。

-	可执行文件(Executable File)：经过链接器链接，可被Linux系统直接执行的目标文件。其内容包含可以运行的机器指令和数据。通常此文件无扩展名。

-	动态库文件(Shared Object File)：动态库文件是共享程序代码的一种方式，其内容和可重定向文件类似，包含可用于链接的代码和程序，可看作多个可重定向文件的集合。通常文件扩展名为.so。动态库用于两个过程，首先链接器把它和其他可重定向文件、动态库一起链接形成一个可执行文件。程序运行时，动态链接器负责在需要的时候动态加载动态库文件到内存。

ELF文件中存放的是可以在处理器上执行的二进制指令和数据。不同的系统架构，ELF里面的格式和数据处理方式会略有不同，但基本格式如图6-1所示。

***TODO_PIC_6_1***

从图6-1可以看出，不同目标文件类型，格式基本类似，内容略有不同。在图6-1(a)中，可重定向文件的格式由ELF文件头(ELF Header)、节(Section)和段头表(Section Header Table)3部分组成。在图6-1(b)中，可执行文件的格式由ELF文件头、段(Segment)和程序头表(Program Header Table)3部分组成。可重定向文件中的节和可执行文件中的段都存储了程序的代码部分、数据部分等，区别是可执行文件中的某个段就是结合了多个可重定向文件中的相关节，且代码部分是经过重定向的最终机器指令，如图6-2所示。

***TODO_PIC_6_2***

平时在工作中，很多程序开发人员并不会过多区分Section和Segment，基本都将之称作段，甚至很多教材中也不会过多区分。所以，本章介绍中统一描述为“段”，在需区分处补充了英文以示不同。

##	ELF文件头

ELF文件头描述了一个目标文件的组织，是对目标文件基本信息的描述，包括字的大小和字节序列（尾端）、ELF文件头的大小、目标文件类型、机器类型、节头表/段头表的大小和数量、程序入口点等。ELF文件头信息必须位于目标文件的最开始部分。我们可以使用工具readelf来查看一个可重定向文件hello.o的ELF头信息。
``` shell
$readelf -h hello.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           LoongArch
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          1000 (bytes into file)
  Flags:                             0x43, LP64, DOUBLE-FLOAT
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         14
  Section header string table index: 13
```
这里readelf的参数-h代表header，表示查看目标文件的头信息。头信息中的部分域结果说明如下。

-	魔数(Magic)：用于确定文件的格式和类型。这里魔数的前4字节“7f 45 4c 46”标识了这是一个ELF格式的文件。第5字节“02”代表文件运行在64位体系架构（“01”代表32位）。第6字节“01”表示小尾端（02代表大尾端）字节序列，通过后面的Data信息也可获知。第7字节“01”代表ELF版本。后面的9字节未定义。

-	Type：标识当前文件类型，具体可为可重定向文件、可执行文件、动态库文件中的一种。当前头信息中显示当前目标文件类型是可重定向文件。可重定向文件的入口点地址都是0x0，而且可重定向文件没有程序头，故程序头起点、程序头大小和Number of program headers都显示为0字节。

-	Machine：标识处理器架构。当前结果LoongArch表示龙芯架构，其值为258。

-	Start of section headers：标识段头表数据在当前文件中的起始位置。这里值为1000字节。

-	标志：用于特定于处理器的ABI类型。对于龙芯处理器，ABI类型可以是LP64s、LP64f、LP64d、ILP32s、ILP32f、ILP32d中的一种。

##	可重定向文件中的段和段头表

一个可重定向文件中的段头表描述了ELF的各个段(Section)的信息，比如每个段的名称、长度、在文件中的偏移、读写权限、地址等。我们可以使用工具readelf 带参数“-S”来查看可重定向文件hello.o中段头表的详细信息。
``` shell
$readelf -S hello.o
There are 14 section headers, starting at offset 0x3e8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000000  0000000000000000  AX       0     0     1
  [ 2] .data             PROGBITS         0000000000000000  00000040
       0000000000000004  0000000000000000  WA       0     0     4
  [ 3] .bss              NOBITS           0000000000000000  00000044
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .rodata.str1.8    PROGBITS         0000000000000000  00000048
       0000000000000004  0000000000000001 AMS       0     0     8
  [ 5] .text.startup     PROGBITS         0000000000000000  0000004c
       0000000000000030  0000000000000000  AX       0     0     4
  [ 6] .rela.text.s[...] RELA             0000000000000000  00000280
       0000000000000078  0000000000000018   I      11     5     8
  [ 7] .comment          PROGBITS         0000000000000000  0000007c
       0000000000000013  0000000000000001  MS       0     0     1
  [ 8] .note.GNU-stack   PROGBITS         0000000000000000  0000008f
       0000000000000000  0000000000000000           0     0     1
  [ 9] .eh_frame         PROGBITS         0000000000000000  00000090
       0000000000000030  0000000000000000   A       0     0     8
  [10] .rela.eh_frame    RELA             0000000000000000  000002f8
       0000000000000078  0000000000000018   I      11     9     8
  [11] .symtab           SYMTAB           0000000000000000  000000c0
       0000000000000198  0000000000000018          12    14     8
  [12] .strtab           STRTAB           0000000000000000  00000258
       0000000000000028  0000000000000000           0     0     1
  [13] .shstrtab         STRTAB           0000000000000000  00000370
       0000000000000076  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), p (processor specific)
```
从段头表信息可以看出当前hello.o文件中共有14个段，段号从0至13。每个段的段信息包括名称、类型、地址、偏移量、大小、旗标、链接、信息、对齐。

1.	段名

上面显示的第一列为段名，在上面信息中显示为名称。段名都以.开头，常见的段名有.text、.data、.bss等。从段名上可以直观了解此段的基本功能，例如.text段用于存放代码（即机器指令），也称为代码段；.data段用于存放数据，也称为数据段；.rodata段用于存放只读数据，也称只读数据段。常见的段名及其功能描述如表6-1所示。

***TODO_TABLE_6_1***

一个简单的C语言程序被编译成目标文件后，在目标文件中存放的位置如图6-3所示。

***TODO_PIC_6_3***

一般来说，C语言程序编译成的机器指令都被存放在代码段(.text)，已经被初始化的全局变量和局部静态变量都保存在数据段(.data)，未被初始化的全局变量和局部静态变量都保存在.bss段。而一些字符串常量（使用宏定义#define声明）、不可改变的变量（使用const修饰）都存放在只读数据段(.rodata)。这样分段存储有很多好处。当程序运行时，不同段的内容被映射到内存中具有不同管理权限（例如只读、可写、可执行等）的区域，首先可以保证安全性（防止指令段数据被修改）；其次可以节省内存（当系统运行多个该程序时只需要各保存一份指令即可），同时也利于性能提升（提升缓存命中率）。

2.	段类型

对编译器来说，段名没有实际意义，决定段属性的是段类型（在上面段信息中显示为类型）和段标志（在上面段信息中显示为旗标）。段类型可分为程序段、重定位表段、符号表段等。常见的段类型及其含义如表6-2所示。

***TODO_TABLE_6_2***

程序中段类型以SHT_开头，如SHT_NULL、SHT_SYMTAB等，但是readelf显示时省略了SHT_。

3.	段标志

段标志(Flag)在上面段信息中显示为旗标，用于表示该段在进程虚拟空间中的访问属性。常见访问属性包括可写(Write)、可执行(Execute)、可分配(Alloc)，而所有段都是可读的。例如上面段信息中的.text段的段标志AX代表Alloc+Execute，表示该段在内存中的访问属性为可执行并且可以申请空间，但不可写。如果在程序执行过程中错误地往此段写数据，那么程序会触发一个SIGSEGV异常。上面段信息中的.data段、.bss段的标志WA代表Write+Alloc，表示该段可写并可分配空间；.symtab段、.strtab段的标志为空，即该段为只读，没有可写权限和可执行权限。

4.	段地址、偏移量和大小

段地址（在上面段信息中显示为地址）记录了当前段被加载到内存后的虚拟起始地址值。因为当前hello.o文件是还未做重定向的目标文件，在进程中的位置还不确定，所以当前所有段的地址都显示为0000000000000000 。如果我们读取最终的可执行文件hello，那么段地址信息都将是类似如下显示的非零有效地址值。

``` shell
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000000  0000000000000000  AX       0     0     1
```
这里虚拟地址0x0000000040就是.text段最终被加载到内存后的虚拟地址。尽管理论上进程可以使用40位的全部虚拟地址空间，但是一般情况下进程并不能使用全部的虚拟地址空间，系统通常预留一部分虚拟地址空间用于自身配置。在龙芯平台下，一个进程大概可用的地址空间范围在0x00000000～0x80000000。

偏移量(Offset)用来表示该段在ELF文件中的偏移。例如上面读取hello.o符号表信息中.text段的Offset值为0x40（64字节），而“readelf -h hello.o”显示的“ 本头的大小： 64（字节）”，说明hello.o的ELF文件中，头信息之后紧接着就是.text段。

大小(Size)用于表示该段的大小。例如上面信息中.text的大小为000040（即十六进制0x40），转换为十进制后表示代码段占用60字节；而.data段、.bss段显示的大小都为000000，表示此段没有内容，不占据内存空间。

5.	段地址对齐

如果某段有地址对齐要求，那么段地址对齐（在上面的段信息中显示为对齐）就指定了地址对齐方式。例如上面的段信息中的.text段的对齐值为4，即表示该段在内存中的起始地址必须以4字节对齐，即该段数据在内存存放的起始地址必须可以被4整除。当对齐值为0和1时，可看作此段没有对齐要求。上面段表信息中的.bss段、.data段的对齐值都为1，说明此段在内存中存放时没有对齐要求。

##	可执行文件中的段和程序头表

可重定向文件中描述Section属性结构叫段头表(Section Header Table)，而可执行文件和动态库文件中描述Segment属性结构的叫程序头表(ProgramHeader Table)，它指导系统如何把多个段(Segment)加载到内存空间。前面说过Segment可以看作多个可重定向文件（.o文件）中的相同节(Section)的合并，即一个Segment包含一个或多个属性相似的Section。这里的属性相似更多是指权限（在段标志Flag中指定），链接器会把多个.o文件中的都具有可执行的.text和.init段都放在最终可执行文件中的一个Segment段内，这样的好处就是可以更多地节省内存空间。因为ELF文件被加载时以系统页为单位，如果一个ELF文件中有10个段且每个段的大小都小于一个内存页，那么按一个段占据一个内存页，当前进程就需要10个内存页。而如果链接器对具有相同权限的段合并到一起去映射，当前进程所需要的内存页肯定就会小于10，从而可以充分利用内存页，减少内存碎片。例如上面段表信息中的.data段和.bss段具有相同的权限WA，就可以被存放在同一个页或多个连续页上。程序头表中记录了那些具有相同权限的段被合并起来的信息。

我们可以使用readelf -S或readelf -l来查看一个可执行文件hello中的段信息和程序头表信息。
``` shell
$readlef -l hello
Elf file type is DYN (Position-Independent Executable file)
Entry point 0x600
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R      0x8
  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x0000000000000025 0x0000000000000025  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-loongarch-lp64d.so.1]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000828 0x0000000000000828  R E    0x4000
  LOAD           0x0000000000003e20 0x0000000000007e20 0x0000000000007e20
                 0x0000000000000240 0x0000000000000248  RW     0x4000
  DYNAMIC        0x0000000000003e30 0x0000000000007e30 0x0000000000007e30
                 0x00000000000001d0 0x00000000000001d0  RW     0x8
  NOTE           0x0000000000000260 0x0000000000000260 0x0000000000000260
                 0x0000000000000020 0x0000000000000020  R      0x4
  GNU_EH_FRAME   0x00000000000007b8 0x00000000000007b8 0x00000000000007b8
                 0x000000000000001c 0x000000000000001c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000003e20 0x0000000000007e20 0x0000000000007e20
                 0x00000000000001e0 0x00000000000001e0  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .plt .text .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .dynamic .got.plt .got .sdata .bss 
   04     .dynamic 
   05     .note.ABI-tag 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .dynamic
```
这里程序头信息中显示当前文件中共有9个段(Segment)，编号从00至08。从输出信息来看，Segment中已经不再需要段名信息，但是对Section和Segment的对应关系做了保留，即后面的“Section to Segment mapping:”部分信息。所有具有相同访问属性的Section被归类到一个Segment中，例如都具有可读可执行权限的.test、.rodata段都被统一安排到编号为02的Segment中，02对应程序头信息的第3行。
``` shell
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000828 0x0000000000000828  R E    0x4000
```
其中，类型LOAD是指当程序运行时，本段是需要被加载到内存的。虚拟地址(VirtAddr)0x000000000是指当该段被进程加载到内存时存放的起始地址；FileSiz表示此段在ELF文件中所占空间的长度；MemSiz表示此Segment在进程内存中所占的长度，对于代码段，此值和FileSiz相等，但是对于数据段，此值可能大于FileSiz；权限属性(Flags)包括可读(R)、可写(W)和可执行(E)，当前02段为代码段(.text)所在段，故权限为RE，没有可写权限；对齐属性Align表示此Segment在内存加载时的对齐方式，其值为2的Align次方，比如上面的Align值为4，那么对齐要求就是16。

##	符号和符号表

在目标文件中，将函数和变量统称为符号(Symbol)。这里的变量是指不占用函数栈空间的全局变量或静态局部变量，不包括函数内的局部变量。函数名和变量名统称为符号名(SymbolName)。每一个可重定向的目标文件中都会有一个符号表(Symbol Table)，用于记录目标文件中所用到的所有符号及其符号名、符号类型、符号大小等信息。有了符号和符号表的存在，编译器在链接阶段才能正确地解析多个目标文件中的变量、函数之间的关系，配合重定位表来正确地完成重定位，最终正确地将多个目标文件合并在一起形成可执行文件或动态库文件。

符号按定义的种类可以分为局部符号、全局符号、外部符号和段符号这4类。

-	局部符号：对应C语言函数内部定义的静态局部变量和静态函数，例如下面的变量a和函数fun()。
``` c
static int a;
static int fun(){
}
```
这里有两个整型局部符号，符号名分别为a和fun。这类符号只在编译单元内部（当前目标文件内部）可见。

-	全局符号：定义在当前目标文件，但可以被其他文件引用的变量和函数。例如下面C语言程序中定义的两个符号名分别为global_var 和main的全局符号。
``` c
int global_var;
int main(int arg,char* arg[]){
}
```

-	外部符号(External Symbol)：在当前目标文件中引用的全局符号。比如我们经常使用的printf函数（定义在模块libc内的符号）或者使用extern声明的变量。

-	段符号：由编译器产生的.text、.data等段名也都称为符号。

一个目标文件中符号表所在的段为.symtab。如果是可执行目标文件，还会有一个段.dynsym用于存放动态符号表。我们可以使用“readelf -s 文件名”来查看一个目标文件中的符号表信息。例如下面的C语言代码：
``` c
#include <stdio.h>
char* str="HELLO";
int global_var;
int main(){
	int a = 0,b = 0;
	static int static_a = 0;
	printf("%s %d\n",str,a+b+static);
	return 0;
}
```
其编译后生成的可重定向文件hello.o中符号表信息如下：
``` shell
$readelf -s hello.o
Symbol table '.symtab' contains 67 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000238     0 SECTION LOCAL  DEFAULT    1 .interp
     2: 0000000000000260     0 SECTION LOCAL  DEFAULT    2 .note.ABI-tag
     3: 0000000000000280     0 SECTION LOCAL  DEFAULT    3 .hash
     4: 00000000000002c8     0 SECTION LOCAL  DEFAULT    4 .gnu.hash
     5: 00000000000002e8     0 SECTION LOCAL  DEFAULT    5 .dynsym
     6: 0000000000000420     0 SECTION LOCAL  DEFAULT    6 .dynstr
     7: 000000000000049a     0 SECTION LOCAL  DEFAULT    7 .gnu.version
     8: 00000000000004b8     0 SECTION LOCAL  DEFAULT    8 .gnu.version_r
     9: 00000000000004d8     0 SECTION LOCAL  DEFAULT    9 .rela.dyn
    10: 00000000000005c8     0 SECTION LOCAL  DEFAULT   10 .rela.plt
    11: 0000000000000600     0 SECTION LOCAL  DEFAULT   11 .plt
    12: 0000000000000640     0 SECTION LOCAL  DEFAULT   12 .text
    13: 0000000000000818     0 SECTION LOCAL  DEFAULT   13 .rodata
    14: 0000000000000830     0 SECTION LOCAL  DEFAULT   14 .eh_frame_hdr
    15: 0000000000000850     0 SECTION LOCAL  DEFAULT   15 .eh_frame
    16: 0000000000007e20     0 SECTION LOCAL  DEFAULT   16 .init_array
    17: 0000000000007e28     0 SECTION LOCAL  DEFAULT   17 .fini_array
    18: 0000000000007e30     0 SECTION LOCAL  DEFAULT   18 .dynamic
    19: 0000000000008000     0 SECTION LOCAL  DEFAULT   19 .data
    20: 0000000000008008     0 SECTION LOCAL  DEFAULT   20 .got.plt
    21: 0000000000008028     0 SECTION LOCAL  DEFAULT   21 .got
    22: 0000000000008060     0 SECTION LOCAL  DEFAULT   22 .sdata
    23: 0000000000008068     0 SECTION LOCAL  DEFAULT   23 .bss
    24: 0000000000000000     0 SECTION LOCAL  DEFAULT   24 .comment
    25: 0000000000000000     0 SECTION LOCAL  DEFAULT   25 .debug_aranges
    26: 0000000000000000     0 SECTION LOCAL  DEFAULT   26 .debug_info
    27: 0000000000000000     0 SECTION LOCAL  DEFAULT   27 .debug_abbrev
    28: 0000000000000000     0 SECTION LOCAL  DEFAULT   28 .debug_line
    29: 0000000000000000     0 SECTION LOCAL  DEFAULT   29 .debug_str
    30: 0000000000000000     0 SECTION LOCAL  DEFAULT   30 .debug_line_str
    31: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS abi-note.c
    32: 0000000000000260    32 OBJECT  LOCAL  DEFAULT    2 __abi_tag
    33: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS init.c
    34: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    35: 00000000000006a0     0 FUNC    LOCAL  DEFAULT   12 deregister_tm_clones
    36: 00000000000006e4     0 FUNC    LOCAL  DEFAULT   12 register_tm_clones
    37: 000000000000073c     0 FUNC    LOCAL  DEFAULT   12 __do_global_dtors_aux
    38: 0000000000008068     1 OBJECT  LOCAL  DEFAULT   23 completed.0
    39: 0000000000007e28     0 OBJECT  LOCAL  DEFAULT   17 __do_global_dtor[...]
    40: 0000000000000794     0 FUNC    LOCAL  DEFAULT   12 frame_dummy
    41: 0000000000007e20     0 OBJECT  LOCAL  DEFAULT   16 __frame_dummy_in[...]
    42: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
    43: 0000000000008070     4 OBJECT  LOCAL  DEFAULT   23 static_a.0
    44: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    45: 00000000000008a0     0 OBJECT  LOCAL  DEFAULT   15 __FRAME_END__
    46: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    47: 0000000000000600     0 OBJECT  LOCAL  DEFAULT  ABS _PROCEDURE_LINKA[...]
    48: 0000000000008060     0 OBJECT  LOCAL  DEFAULT   22 __dso_handle
    49: 0000000000007e30     0 OBJECT  LOCAL  DEFAULT  ABS _DYNAMIC
    50: 0000000000000830     0 NOTYPE  LOCAL  DEFAULT   14 __GNU_EH_FRAME_HDR
    51: 0000000000008028     0 OBJECT  LOCAL  DEFAULT   21 __TMC_END__
    52: 0000000000008028     0 OBJECT  LOCAL  DEFAULT  ABS _GLOBAL_OFFSET_TABLE_
    53: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
    54: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_mai[...]
    55: 0000000000008068     0 NOTYPE  GLOBAL DEFAULT   22 _edata
    56: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND abort@GLIBC_2.36
    57: 000000000000806c     4 OBJECT  GLOBAL DEFAULT   23 global_var
    58: 0000000000000818     4 OBJECT  GLOBAL DEFAULT   13 _IO_stdin_used
    59: 0000000000008078     0 NOTYPE  GLOBAL DEFAULT   23 _end
    60: 0000000000000640    96 FUNC    GLOBAL DEFAULT   12 _start
    61: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.36
    62: 0000000000008000     8 OBJECT  GLOBAL DEFAULT   19 str
    63: 0000000000008068     0 NOTYPE  GLOBAL DEFAULT   23 __bss_start
    64: 00000000000007ac   108 FUNC    GLOBAL DEFAULT   12 main
    65: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
    66: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@G[...]
```

从上面的信息可以看出，hello.o文件中共有67个符号，编号(Num)从0到66。每个符号都有如下属性。

-	符号值(Value)：每个符号都有一个对应的值，如果此符号是一个函数或变量，其符号值就是函数或变量的虚拟地址。上面hello.o是未做重定向的ELF文件，所以符号值都为0。可执行文件中，符号值可能是符号的虚拟地址、符号所在函数偏移等。但对于OBJECT类型的符号，Value列表示的是其对齐方式。

-	符号大小(Size)：对于变量，符号大小就是数据类型的大小，单位是字节。例如上述C语言示例中的变量static_a的数据类型为int，所以其大小为4字节。字符串str的数据类型为指针，在LA64架构上为8字节。对于函数，符号大小就是该函数被编译器编译后的所有机器指令占用的字节数，例如符号main的大小为60字节，龙芯指令集中每条指令是4字节，可以推算出main函数被编译后共有15条机器指令。

-	符号类型(Type)分为如下种类。

	-	NOTYPE：未知符号类型。包括目标文件中用于条件跳转的标签、在外部定义的符号等。

	-	OBJECT：数据对象，比如C语言变量、字符串、数组等。

	-	FUNC：函数或其他可执行代码。

	-	SECTION：一个段。

	-	FILE：文件名。
-	 绑定信息(Bind)分为如下种类。

	-	LOCAL：局部符号。例如上面定义的局部变量static_a。

	-	GLOBAL：全局符号。包括本文件内定义的全局变量global_var、str、main和外部函数printf。

	-	WEAK：弱引用符号。在这里没有体现。对于C/C++语言，编译器默认函数和已经初始化的全局变量为强符号，而未初始化的全局变量和使用__attribute__((weak))定义的变量为弱符号。

-	Vis：可扩展符号功能，暂未定义其具体功能，可忽略。

-	符号所在段(Ndx)：如果符号定义在本目标文件中，那么这个成员表示符号所在的段在段表中的下标。比如静态变量static_a所在的段索引为4。Ndx还有如下3种特殊值。

	-	UND：未定义。通常表示这是个外部符号，故不在本目标文件中定义。例如定义在libc库的printf函数或者使用extern声明的外部变量。

	-	ABS：表示该符号包含一个绝对值，比如符号hello.c。

	-	COM：表示该符号是个未初始化的全局符号，例如变量global_var。

-	符号名(Name)：符号表的最后一列。如static_a、.LC0、str、global_var、main和puts。还有的符号是没有名字的，只能通过编号来识别。没有名字的符号是段（从Type列的SECTION可以看出）或未知符号类型。

##	重定位和重定位表

重定位包括链接时重定位和加载时重定位。链接时重定位指的是在编译器链接阶段将多个可重定位目标文件合并成一个可执行目标文件时，对文件中所有的程序数据和函数调用指令进行地址确定的过程。链接时重定位不包括对动态库中数据加载和函数调用指令的定位，这个过程要在加载时重定位。加载时重定位就是针对动态库而言的，在程序运行过程中需要加载动态库时，对所有动态库中函数调用的绝对地址引用进行地址确定的过程。

对于同一个文件内的函数调用，由于函数之间的相对位置是固定的（在链接时同文件内的函数是连续存放的），所以不存在需要重定位的情况。故重定位指的是多个文件之间或多个模块之间（这里模块指动态库或可执行目标文件）存在函数调用和数据引用的处理。这里列举一个简单的链接时重定位的情况，在如下两个C语言文件中，a.c中调用了b.c文件中的temp函数，具体内容如下：
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
在编译器没有进行重定位之前的目标文件a.o和b.o中的指令信息为
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
前面介绍过，链接前的可重定位目标文件中的所有段的起始地址都是0，当前a.o文件中的代码段(.text)中只有函数main，故函数main的起始地址就为0000000000000000 。b.o文件中的代码段中只有函数temp，故函数temp的起始地址也为0000000000000000 。而相对跳转指令“bl 0”代表要进行函数temp的调用，但是这里跳转的目标地址为0，证明现在还不清楚函数temp所在位置。要待链接时函数temp地址确定后，重新修正这条指令。

我们再看编译器链接后生成的可执行目标文件a.out中的相关函数地址和指令情况，具体如下。
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
可以看到，在链接后的目标文件a.out中，函数main和temp的起始地址都已经确定，分别为0x00000066c和0x000000698。用于调用函数temp的相对跳转指令bl中的地址也已经修正，由“bl 0”变为“bl 28”。当前PC(0x00000067c)加上偏移值28(0x1c)后的地址恰好为0x000000698，即函数temp的起始地址。

编译器的链接过程最主要的两件事是地址分配和重定位。地址分配的过程就是处理所有输入文件（这里指的是a.o和b.o），获取所有的符号信息、段长度、属性等信息，并以此为依据将相同属性的段合并和确定符号的地址，例如确定函数main和temp的起始地址为0x00000066c和0x000000698。重定位过程在地址分配的基础上对数据加载指令或函数调用指令做地址确定并修改，例如将指令“bl 0”修改为“bl 28”。

但是，并不是所有的数据加载指令或函数调用指令都需要修改。那么哪些指令需要修改，如何修改呢？这就需要目标文件中的重定位表。在可重定位的目标文件中，每一个需要地址修正指令所在的段，都会对应一个重定位段。例如目标文件a.o中的代码段.text里面需要修正的指令bl，那么a.o中就会有一个“.rel.text”的段，段内记录了需要进行地址修正的指令所在位置、修正方法、修正后的符号名称等信息。如果代码段.data中也有需要地址修正的指令，还会有一个.data.text段与之对应。而重定位表对所有这些信息进行了记录。

这里使用命令“objdump -r ”查看a.o中的重定向信息：
``` shell
$objdump -r a.o
a.o 	:
***TODO***
```
这说明在a.o中的代码段中有需要地址修正的指令，其所在当前目标文件中的偏移地址为0x10，即上面a.o中的指令：
``` shell
10: 	54000000 	bl 	0 	#	10<main + 0x10>
```
符号名temp指明指令中地址修正后指向的目标是函数temp。地址修正类型（也叫重定位类型）R_LARCH_SOP_PUSH_PLT_PCREL指明如何做地址修正。每一个体系架构都有一套独立的地址修正类型，这属于架构ABI范畴。通过查看龙芯架构参考手册的ABI部分可知，这里R_LARCH_SOP_PUSH_PLT_PCREL代表修正方式是利用跳转目标地址与当前PC的相对寻址修正。这里跳转目标为函数temp，其在链接过程地址分配之后确定的地址为0x000000698，当前PC地址为指令“bl 0”所在地址0x00000067c。那么相对寻址修正后的值为0x1c(0x000000698-0x00000067c)，修正后的跳转指令bl机器指令由之前的54000000修改为54001c00 。

LoongArch ABI支持的重定位类型多达60种，全面的重定位类型可参看龙芯架构参考手册的ABI部分，表6-3列举了部分LoongArch支持的重定位类型。

***TODO_TABLE_6_3***

表6-3中列举了4种重定位类型。对于和一个重定位相关联的符号，计算方式中RtAddr代表这个符号的运行时地址，A代表一个额外的加数，B代表是该重定位的段所在模块被加载进内存的装载地址。
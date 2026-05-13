#	龙芯汇编语言程序编写示例

和C语言类似，汇编程序也是以函数（也称为方法）为单位编写，函数可以有输入参数或者输出参数。汇编程序所在文件称为汇编源文件（扩展名为.S）。编写后的汇编源文件使用GCC编译器进行编译链接，和其他C语言文件形成最终可执行的二进制文件（即内部已是机器指令）。龙芯汇编源文件的编写、编译、执行全过程的示例如下：
``` asm
#	文件 add.S
#	接口定义 int add_f(int a, int b,int c,int d)
#	功能定义 return (a + b + c + d)
.text
.align	2
.global add_f
.type	add_f,@function
add_f:
	add.w	$a0, $a0, $a1
	add.w	$a0, $a0, $a2
	add.w	$a0, $a0, $a3
	jr		$ra
.size	add_f,.-add_f
```
add.S源文件里实现了一个add_f函数，其功能为实现4个32位整型数据（分别为寄存器a0、a1、a2和a3）的加法操作，并将结果返回（使用寄存器a0作为返回值）。汇编指令“jr $r1”意为函数返回。

接下来C语言文件test.c对这个汇编源文件中的汇编程序进行调用。
``` c
#include <stdio.h>
extern int add_f(int a, int b, int c, intd);

int main(){
	int ret = add_f(1,2,3,4);
	printf("ret = %d\n",ret);
	return 0;
}
```
C语言文件test.c对汇编程序的调用与对其他C语言外部函数的调用方式是一致的，使用前通过关键字extern声明即可。

下面通过GCC编译器将汇编源文件add.S和C语言文件test.c编译成最终可执行文件test_add。
``` shell
$ gcc test.c add.S -o test_add
```
最后我们可以运行这个可执行文件test_add并查看结果，结果显示为10(1+2+3+4)，说明汇编源程序编写和执行正确。
``` shell
$./a.out
ret = 10
```
可见，编写汇编语言程序还是还挺简单的。龙芯汇编源程序更详细的语法和编写方式会在后续章节中详细介绍。
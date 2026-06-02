#	龙芯汇编语言程序编写示例

与C语言类似，汇编程序通常也以函数（也称为方法）为单位编写，函数可以具有输入参数和输出结果。汇编程序所在文件称为汇编源文件，扩展名通常为`.S`。编写完成后，可使用GCC编译器对汇编源文件进行编译和链接，并与其他C语言文件共同生成最终可执行的二进制文件（其内部已是机器指令）。下面给出龙芯汇编源文件从编写、编译到执行的完整示例：
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
`add.S`源文件实现了一个`add_f`函数，其功能是对4个32位整型数据（分别位于寄存器a0、a1、a2和a3）进行加法操作，并将结果返回（使用寄存器a0作为返回值）。汇编指令`jr $ra`表示函数返回。

接下来，C语言文件`test.c`调用该汇编源文件中的汇编程序。
``` c
#include <stdio.h>
extern int add_f(int a, int b, int c, int d);

int main(){
	int ret = add_f(1,2,3,4);
	printf("ret = %d\n",ret);
	return 0;
}
```
C语言文件`test.c`调用汇编程序的方式，与调用其他C语言外部函数的方式一致，在使用前通过关键字`extern`声明即可。

下面通过GCC编译器将汇编源文件`add.S`和C语言文件`test.c`编译为最终可执行文件`test_add`。
``` shell
$ gcc test.c add.S -o test_add
```
最后运行可执行文件`test_add`并查看结果。结果显示为10（1+2+3+4），说明汇编源程序编写和执行正确。
``` shell
$ ./test_add
ret = 10
```
由此可见，简单汇编语言程序的编写、编译和调用过程并不复杂。龙芯汇编源程序更详细的语法和编写方式将在后续章节中介绍。

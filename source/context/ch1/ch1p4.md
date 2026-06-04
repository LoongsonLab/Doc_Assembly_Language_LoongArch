#	龙芯汇编语言程序编写示例

与C语言类似，汇编程序通常也以函数（或方法）为基本单位组织。函数可以接收输入参数，也可以返回结果。保存汇编程序的文件称为汇编源文件，扩展名通常为`.S`。编写完成后，可以使用GCC编译器对汇编源文件进行编译和链接，并与其他C语言文件一起生成最终可执行的二进制文件；该文件内部已经是机器指令。下面给出一个从编写、编译到运行的龙芯汇编程序示例：
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
`add.S`源文件定义了函数`add_f`，用于把4个32位整型参数相加。这4个参数分别存放在寄存器a0、a1、a2和a3中，计算结果通过寄存器a0返回。汇编指令`jr $ra`表示从当前函数返回。

接着，在C语言文件`test.c`中调用这个汇编函数。
``` c
#include <stdio.h>
extern int add_f(int a, int b, int c, int d);

int main(){
	int ret = add_f(1,2,3,4);
	printf("ret = %d\n",ret);
	return 0;
}
```
在`test.c`中调用汇编函数的方式，与调用其他C语言外部函数基本相同。使用前通过`extern`声明函数接口即可。

然后使用GCC把汇编源文件`add.S`和C语言文件`test.c`一起编译，生成可执行文件`test_add`。
``` shell
$ gcc test.c add.S -o test_add
```
最后运行`test_add`并查看输出。结果为10，即1+2+3+4，说明汇编源程序能够正确编译和执行。
``` shell
$ ./test_add
ret = 10
```
由此可以看出，简单汇编程序的编写、编译和调用流程并不复杂。关于龙芯汇编源程序更完整的语法和写法，后续章节将继续介绍。

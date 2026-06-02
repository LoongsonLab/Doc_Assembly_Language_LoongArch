#	习题

1. 编写一段程序，验证当前系统使用的字节序。

2. 什么是函数栈？一个函数是否必须有函数栈？请举例说明。

3. 使用`objdump`工具查看如下C语言函数的指令生成情况，并描述其函数栈结构。
``` c
long test(int a,int b,int c,int d,int e,int f,int g,float h,int i){
	return 0;
}
```

4. 编写一段系统调用汇编指令，用于实现获取当前线程TID的功能。

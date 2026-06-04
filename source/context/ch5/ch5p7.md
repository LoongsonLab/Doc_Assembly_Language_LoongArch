#	习题

1. 编写程序，验证当前系统采用的字节序。

2. 什么是函数栈？函数是否一定需要函数栈？请举例说明。

3. 使用`objdump`工具查看下面C语言函数生成的指令，并描述其函数栈结构。
``` c
long test(int a,int b,int c,int d,int e,int f,int g,float h,int i){
	return 0;
}
```

4. 编写一段系统调用汇编指令，实现获取当前线程TID的功能。

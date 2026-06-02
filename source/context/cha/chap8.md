#	习题

1.汇编程序中常用的优化手段有哪些？请逐一列举。

2.编写一个示例程序来验证地址对齐访问与非对齐访问的性能差距。

3.函数内联是C/C++语言程序中常用的一种优化手段。请具体说明什么是函数内联，以及使用函数内联的优缺点。

4.查看龙芯架构参考手册的向量指令集部分，对如下C语言程序使用向量指令进行优化，并使用 perf 工具查看优化前后的性能。
``` c
uint8_t *yuv444 = (uint8_t *) malloc(sizeof(uint8_t) * width * height * 3);
for (x = 0, y = 0; x < width * height * 2 && y < width * height * 3; x += 4, y += 6) {
	yuv444[y + 0] = yuv422[x + 0];
	yuv444[y + 1] = yuv422[x + 1];
	yuv444[y + 2] = yuv422[x + 3];
	yuv444[y + 3] = yuv422[x + 2];
	yuv444[y + 4] = yuv422[x + 1];
	yuv444[y + 5] = yuv422[x + 3];
}
```

5.对如下C语言程序，分别使用参数 -O0 和 -O3 编译。使用 objdump 工具查看编译后生成的指令信息，并说明差别。
``` c
void test(float *farray, int *iarray, int length) {
	for (int i = 0; i < length; i++) {
		farray[i] += 2.0;
		iarray[i] += 2;
	}
}
```

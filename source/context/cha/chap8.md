#	习题

1.汇编程序中常见的优化方法有哪些？请分别列出。

2.编写示例程序，验证地址对齐访问和非对齐访问之间的性能差异。

3.函数内联是 C/C++ 程序中常见的优化方式。请说明函数内联的含义，并分析它的优点和缺点。

4.查阅龙芯架构参考手册中的向量指令集部分，对下面的 C 语言程序进行向量化优化，并使用 perf 工具比较优化前后的性能。
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

5.对下面的 C 语言程序分别使用 `-O0` 和 `-O3` 参数编译。使用 objdump 查看生成的指令信息，并说明两种编译结果的差别。
``` c
void test(float *farray, int *iarray, int length) {
	for (int i = 0; i < length; i++) {
		farray[i] += 2.0;
		iarray[i] += 2;
	}
}
```

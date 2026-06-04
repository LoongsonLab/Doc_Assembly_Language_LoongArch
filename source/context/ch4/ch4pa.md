#	习题

1. C语言双精度浮点数变量`3.14`对应的二进制数值是什么？

2. 执行下面指令后，条件标志寄存器fcc0的值是多少？
``` asm
li.w 		$r4, 3
movgr2fr.w 	$f0, $r0
movgr2fr.w 	$f1, $r4
fcmp.slt.s 	fcc0, $f0, $f1
```

3. 将双精度浮点数`1.00`加载到浮点寄存器f0，应使用哪些LoongArch汇编指令？

4. 编写下面C语言函数对应的汇编语句。
``` c
double max(double fva, double fvb){
	return (fva > fvb)? fva:fvb;
}
```

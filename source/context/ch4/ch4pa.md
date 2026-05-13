#	习题

1.C语言变量双精度浮点数3.14对应的二进制数值是多少？

2.执行下列指令后，条件标志寄存器fcc0的值是多少？
``` asm
li.w 		$r4, 3
movgr2f.w 	$f0, $r0
movgr2f.w 	$f1, $r4
fcmp.slt.s 	fcc0, $f0, $f1
```

3.加载一个双精度浮点数1.00到浮点寄存器f0的LoongArch汇编指令是什么？

4.编写如下C语言函数对应的汇编语句。
``` c
double max(double fva, double fvb){
	return (fva > fvb)? fva:fvb;
}
```
#	系统调用约定

从用户程序角度看，内核通常是相对透明的系统层。用户程序大多通过libc库运行，而不会直接调用内核接口。内核是操作系统的核心，负责管理进程、内存、设备驱动、文件系统和网络系统等资源。它位于计算机硬件之上，是第一层软件扩展，并向上提供操作系统应用程序接口（Application Program Interface，API），这些API也称为系统调用。通常，libc会对系统调用接口进行封装，可视为用户程序和内核之间的中间层。例如，`printf`函数的调用过程如图5-16所示。

由图5-16可以看出，真正完成数据输出显示的是内核，libc库只是对该功能做了接口封装。有了这层封装，用户程序不必关注太多系统底层细节，也能获得更好的兼容性和移植性。内核提供的每个系统调用都有一个系统调用号，用于唯一标识该接口。用户空间程序执行系统调用时，需要通过系统调用号说明要执行哪个系统调用。

```{image} ../../img/ch5/pic_5_16.png
:alt: 函数printf的调用过程
:class: bg-primary
:scale: 80 %
:align: center
```

了解系统调用约定后，在必要时可以编写汇编程序直接调用内核接口。LoongArch ABI规定，寄存器a7用于传递系统调用号，a0~a6用于传递系统调用参数，同时a0也用于保存返回值。与普通函数调用约定不同，系统调用返回后，a0~a6中的值可能已被破坏。

内核提供的接口函数名称及系统调用号，可以在内核源代码文件`include/uapi/asm-generic/unistd.h`或系统文件`<asm/unistd.h>`中查询。下面列出LoongArch下部分内核I/O接口、文件读写接口的函数名称和系统调用号信息。

``` c
#define __NR_io_setup 0
__SC_COMP(__NR_io_setup, sys_io_setup, compat_sys_io_setup)
#define __NR_io_destroy 1
__SYSCALL(__NR_io_destroy, sys_io_destroy)
#define __NR_io_submit 2
__SC_COMP(__NR_io_submit, sys_io_submit, compat_sys_io_submit)
#define __NR_io_cancel 3
__SYSCALL(__NR_io_cancel, sys_io_cancel)

#define __NR_exit 93
__SYSCALL(__NR_exit, sys_exit)
#define __NR_exit_group 94
__SYSCALL(__NR_exit_group, sys_exit_group)
#define __NR_waitid 95
__SC_COMP(__NR_waitid, sys_waitid, compat_sys_waitid)

#define __NR3264_lseek 62
__SC_3264(__NR3264_lseek, sys_llseek, sys_lseek)
#define __NR_read 63
__SYSCALL(__NR_read, sys_read)
#define __NR_write 64
__SYSCALL(__NR_write, sys_write)
#define __NR_readv 65
__SC_COMP(__NR_readv, sys_readv, sys_readv)
#define __NR_writev 66
__SC_COMP(__NR_writev, sys_writev, sys_writev)
#define __NR_pread64 67
__SC_COMP(__NR_pread64, sys_pread64, compat_sys_pread64)
#define __NR_pwrite64 68
__SC_COMP(__NR_pwrite64, sys_pwrite64, compat_sys_pwrite64)
#define __NR_preadv 69
__SC_COMP(__NR_preadv, sys_preadv, compat_sys_preadv)
#define __NR_pwritev 70
__SC_COMP(__NR_pwritev, sys_pwritev, compat_sys_pwritev)
```

这些函数的接口声明可以在`include/linux/syscalls.h`中找到。例如，内核I/O接口和文件读写接口声明如下：

``` c
asmlinkage long sys_io_setup(unsigned nr_reqs, aio_context_t __user *ctx);
asmlinkage long sys_io_destroy(aio_context_t ctx);
asmlinkage long sys_io_submit(aio_context_t, long,
			struct iocb __user * __user *);
asmlinkage long sys_io_cancel(aio_context_t ctx_id, struct iocb __user *iocb,
			      struct io_event __user *result);
asmlinkage long sys_io_getevents(aio_context_t ctx_id,
				long min_nr,
				long nr,
				struct io_event __user *events,
				struct __kernel_timespec __user *timeout);
asmlinkage long sys_io_pgetevents(aio_context_t ctx_id,
				long min_nr,
				long nr,
				struct io_event __user *events,
				struct __kernel_timespec __user *timeout,
				const struct __aio_sigset __user *sig);
asmlinkage long sys_exit(int error_code);
asmlinkage long sys_exit_group(int error_code);
asmlinkage long sys_waitid(int which, pid_t pid,
			   struct siginfo __user *infop,
			   int options, struct rusage __user *ru);
asmlinkage long sys_llseek(unsigned int fd, unsigned long offset_high,
			unsigned long offset_low, loff_t __user *result,
			unsigned int whence);
asmlinkage long sys_lseek(unsigned int fd, off_t offset,
			  unsigned int whence);
asmlinkage long sys_read(unsigned int fd, char __user *buf, size_t count);
asmlinkage long sys_write(unsigned int fd, const char __user *buf,
			  size_t count);
asmlinkage long sys_readv(unsigned long fd,
			  const struct iovec __user *vec,
			  unsigned long vlen);
asmlinkage long sys_writev(unsigned long fd,
			   const struct iovec __user *vec,
			   unsigned long vlen);
asmlinkage long sys_pread64(unsigned int fd, char __user *buf,
			    size_t count, loff_t pos);
asmlinkage long sys_pwrite64(unsigned int fd, const char __user *buf,
			     size_t count, loff_t pos);
asmlinkage long sys_preadv(unsigned long fd, const struct iovec __user *vec,
			   unsigned long vlen, unsigned long pos_l, unsigned long pos_h);
asmlinkage long sys_pwritev(unsigned long fd, const struct iovec __user *vec,
			    unsigned long vlen, unsigned long pos_l, unsigned long pos_h);
```

掌握这些信息后，就可以使用系统调用指令`syscall`调用内核接口函数。

【例5.12】 使用指令syscall实现字符串“helloworld”的屏幕输出

要把字符串输出到屏幕，需要使用内核接口函数`sys_write`。其系统调用号为64，函数接口形式为：
``` c
long sys_write(unsigned int fd, const char __user*buf, size_t count);
```
该函数有3个参数，分别为文件描述符、待输出字符串地址和字符串长度；返回值用于接收接口执行结果。屏幕对应标准输出设备`/dev/stdout`，文件描述符为1。字符串`hello world`长度为11，地址由编译器决定。具体汇编实现如下：
``` asm
li.d 	$a7, 64 	#	将sys_write系统调用号64写到寄存器a7
li.d 	$a0, 1 		#	将/dev/stdout文件描述符写到第一个参数寄存器a0
la.local	$a1, .LC0 	#	将字符串地址写到第二个参数寄存器a1
li.d 	$a2, 11 	#	将字符串长度11写到第三个参数寄存器a2
syscall 0 			#	系统调用
.section 	.rodata
.LC0:
.ascii 	"hello world"
```

这个示例没有处理`sys_write`的返回值。实际上，返回值会保存在寄存器a0中。上述示例最后3条不是LoongArch真实汇编指令，而是GCC编译器支持的汇编器指令，用于通知汇编器把字符串`hello world`放入当前进程的只读数据区，并用`.LC0`标记其位置。使用时，再通过伪指令`la.local`把该地址加载到指定寄存器。相关详细语法将在后续章节介绍。

#	系统调用约定

从用户程序角度看，内核是一个相对透明的系统层，因为用户程序通常通过libc库运行，而不会直接调用内核接口。内核是操作系统的核心，负责管理系统的进程、内存、设备驱动程序、文件和网络系统等。它是计算机硬件之上的第一层软件扩展，并向上提供操作系统的应用程序接口（Application Program Interface，API），这些API也称为系统调用。通常，libc库会对系统调用接口进行封装，可视为用户程序与内核之间的中间层。例如，函数`printf`的调用过程如图5-16所示。

从图5-16可以看出，真正实现数据输出显示的是内核，而libc库只是对该功能进行接口封装。有了这层封装，用户程序就不需要关心过多系统底层细节，也能获得更好的兼容性和移植性。内核提供的每个系统调用都会被赋予一个系统调用号，用于唯一标识系统调用接口。当用户空间程序执行系统调用时，就会使用该系统调用号指明要执行哪个系统调用。

```{image} ../../img/ch5/pic_5_16.png
:alt: 函数printf的调用过程
:class: bg-primary
:scale: 80 %
:align: center
```

了解系统调用约定后，在必要时就可以编写汇编程序直接调用内核接口。LoongArch ABI规定，寄存器a7用于传递系统调用号，寄存器a0~a6用于传递参数，同时寄存器a0也用于传递返回值。不同于普通函数调用约定，系统调用返回后，寄存器a0~a6的值可能被破坏。

内核提供的所有接口函数名称及其系统调用号，可以在内核源代码文件`include/uapi/asm-generic/unistd.h`或系统文件`<asm/unistd.h>`中查到。下面列举LoongArch下部分内核I/O接口函数、文件读写接口函数对应的函数名称及系统调用号信息。

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

这些函数对应的接口声明可在`include/linux/syscalls.h`中查到。例如，内核提供的I/O接口函数、文件读写接口函数对应声明如下：

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

有了这些信息，就可以使用系统调用指令`syscall`实现对内核接口函数的调用。

【例5.12】 使用指令syscall实现字符串“helloworld”的屏幕输出

要实现字符串的屏幕输出功能，需要使用的内核接口函数为`sys_write`。其对应系统调用号为64，函数接口形式为
``` c
long sys_write(unsigned int fd, const char __user*buf, size_t count);
```
该函数有3个参数，分别传递文件描述符、待输出字符串地址和字符串长度；返回值用于接收此接口函数的执行结果。屏幕使用标准输出设备`/dev/stdout`的文件描述符为1，字符串`hello world`长度为11，地址由编译器决定。具体实现汇编指令如下：
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

这个示例没有对内核接口函数`sys_write`的返回值做接收处理，实际上返回值存放在寄存器a0中。上述示例中的后3条不是LoongArch汇编指令，而是GCC编译器的汇编器指令，用于通知汇编器将字符串`hello world`存放在当前进程的只读数据区，具体位置通过`.LC0`标注，使用时通过伪指令`la.local`将其地址加载到指定寄存器。这部分详细语法将在后续章节介绍。

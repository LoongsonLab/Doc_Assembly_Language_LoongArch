#	系统调用约定

从用户程序的角度看，内核是一个透明的系统层，因为用户程序都通过libc库运行，而不会直接调用内核接口。内核是一个操作系统的核心，负责管理系统的进程、内存、设备驱动程序、文件和网络系统等，它是计算机硬件的第一层软件扩充，对上提供操作系统的应用程序接口(Application Program Interface，API)，这些API也叫系统调用。通常libc库对这些系统调用的接口做了封装，被看作用户程序和内核的中间层，例如函数printf的调用过程如图5-16所示。

从图5-16可以看出，内核才是真正实现数据的输出显示，而libc库就是对此功能的接口封装。有了这层封装，用户程序就不用去关心过多的系统底层细节，也确保了用户程序具有更好的兼容性和移植性。内核提供的每个系统调用被赋予一个系统调用号，它就像是人的身份证号，用于唯一地标识一个系统调用接口。当用户空间的程序执行一个系统调用时，就会用到这个系统调用号，还指明要执行哪个系统调用。

***TODO_PIC_5_16***

了解系统调用约定，在必要的时候我们就可以编写汇编程序直接实现对内核接口的调用。LoongArch ABI规定寄存器a7 用于传递系统调用号，寄存器a0～a6用于传递参数，同时寄存器a0也用来传递返回值。不同于普通函数调用约定，系统调用回来以后，寄存器a0～a6的值可能会被破坏掉。

内核提供的所有接口函数的名称及其系统调用号可以在内核源代码文件include\/uapi\/asm-generic\/unistd.h或者系统文件\<asm\/unistd.h\>中查到。下面列举了LoongArch部分内核提供的I\/O接口函数、文件读写接口函数对应的函数名称及其系统调用号信息。

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

这些函数对应的接口声明在include/linux/syscalls.h中可以查到，例如内核提供的I/O接口函数、文件读写接口函数对应的声明如下：

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

有了这些信息，我们就可以轻松使用系统调用指令syscall来实现一个内核接口函数的调用。

【例5.12】 使用指令syscall实现字符串“helloworld”的屏幕输出

要实现一个字符串的屏幕输出功能，需要使用的内核接口函数为sys_write。其对应的系统调用号为64，函数接口形式为
``` c
long sys_write(unsigned int fd, const char __user*buf, size_t count);
```
即有3个参数，分别传递文件描述符、待输出的字符串地址、字符串长度。1个返回值用于接收此接口函数的执行返回值。屏幕使用标准输出设备/dev/stdout的文件描述符为1，字符串“helloworld”长度为11，地址由编译器来决定。具体实现汇编指令如下：
``` asm
li.d 	$a7, 64 	#	将sys_write系统调用号64写到寄存器a7
li.d 	$a0, 1 		#	将/dev/stdout文件秒舒服写到第一个参数寄存器a0
la.local	$a1, .LC) 	#	将字符串地址写到第二个参数寄存器a1
li.d 	$a2, 11 	#	将字符串长度11写到第三个参数寄存器a2
syscall 0 			#	系统调用
.section 	.rodata
.LC0:
.ascii 	"hello world"
```

这个示例没有对内核接口函数sys_write的返回值做接收处理，实际上返回值存在寄存器a0上。上述示例中的后3条不是LoongArch汇编指令，而是GCC编译器的汇编器指令，用于通知汇编器工作时将字符串“hello world”存放在当前进程的只读数据区，具体位置通过.LC0标注，使用时用伪指令la.local将其加载到指定的寄存器。这部分的详细语法在后面章节会介绍。


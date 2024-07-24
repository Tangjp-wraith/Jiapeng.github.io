# Operating System Organization

## 1 User mode and supervisor mode

为了实现进程隔离，RISC-V CPU在硬件上提供3种执行命令的模式：*machine mode*, *supervisor mode*, *user mode*。

1. machine mode的权限最高，CPU以machine mode启动，machine mode的主要目的是为了配置电脑，之后立即切换到supervisor mode。
2. supervisor mode运行CPU执行privileged instructions，比如中断管理、对存储页表地址的寄存器进行读写操作、执行system call。运行在supervisor mode也称为在kernel space中运行。
3. 应用程序只能执行user mode指令，比如改变变量、执行util function。运行在user mode也称为在user space中运行。要想让CPU从user mode切换到supervisor mode，RISC-V提供了一个特殊的`ecall`指令，要想从supervisor mode切换到user mode，调用`sret`指令

## 2 Kernel organization

monolithic kernel：整个操作系统在kernel中，所有system call都在supervisor mode下运行。xv6是一个monolithic kernel

micro kernel：将需要运行在supervisor mode下的操作系统代码压到最小，保证kernel内系统的安全性，将大部分的操作系统代码执行在user mode下

![alt text](image/QQ_1721730688212.png)

如2.1所示，文件系统是一个user-level的进程，为其他进程提供服务，因此也叫做server

xv6 kernel source file如下所示

| 文件 | 描述 |
| --- | --- |
| bio.c | 文件系统的磁盘块缓存 |
| console.c | 连接到用户的键盘和屏幕 |
| entry.S | 首次启动指令 |
| exec.c | exec()系统调用 |
| file.c | 文件描述符支持 |
| fs.c | 文件系统 |
| kalloc.c | 物理页面分配器 |
| kernelvec.S | 处理来自内核的陷入指令以及计时器中断 |
| log.c | 文件系统日志记录以及崩溃修复 |
| main.c | 在启动过程中控制其他模块初始化 |
| pipe.c | 管道 |
| plic.c | RISC-V中断控制器 |
| printf.c | 格式化输出到控制台 |
| proc.c | 进程和调度 |
| sleeplock.c | Locks that yield the CPU |
| spinlock.c | Locks that don’t yield the CPU. |
| start.c | 早期机器模式启动代码 |
| string.c | 字符串和字节数组库 |
| swtch.c | 线程切换 |
| syscall.c | Dispatch system calls to handling function. |
| sysfile.c | 文件相关的系统调用 |
| sysproc.c | 进程相关的系统调用 |
| trampoline.S | 用于在用户和内核之间切换的汇编代码 |
| trap.c | 对陷入指令和中断进行处理并返回的C代码 |
| uart.c | 串口控制台设备驱动程序 |
| virtio_disk.c | 磁盘设备驱动程序 |
| vm.c | 管理页表和地址空间 |

## 3 Process

隔离的单元叫做进程，一个进程不能够破坏或者监听另外一个进程的内存、CPU、文件描述符，也不能破坏kernel本身。

为了实现进程隔离，xv6提供了一种机制让程序认为自己拥有一个独立的机器。一个进程为一个程序提供了一个私有的内存系统，或address space，其他的进程不能够读/写这个内存。xv6使用page table(页表)来给每个进程分配自己的address space，页表再将这些address space，也就是进程自己认为的虚拟地址(*virtual address*)映射到RISC-V实际操作的物理地址(*physical address*)

<center>![alt text](image/QQ_1721730716690.png){width="400"}</center>

虚拟地址从0开始，往上依次是指令、全局变量、栈、堆。RISC-V上的指针是64位的，xv6使用低38位，因此最大的地址是$2^{38}-1$=0x3fffffffff=MAXVA

进程最重要的内核状态：1. 页表 `p->pagetable` 2. 内核堆栈`p->kstack` 3. 运行状态`p->state`，显示进程是否已经被分配、准备运行/正在运行/等待IO或退出

每个进程中都有线程(*thread*)，是执行进程命令的最小单元，可以被暂停和继续

每个进程有两个堆栈：用户堆栈(*user stack*)和内核堆栈(*kernel stack*)。当进程在user space中进行时只使用用户堆栈，当进程进入了内核(比如进行了system call)使用内核堆栈

## 4 Starting the first process

RISC-V启动时，先运行一个存储于ROM中的bootloader程序`kernel.ld`来加载xv6 kernel到内存中，然后在machine模式下从`_entry`开始运行xv6。bootloader将xv6 kernel加载到0x80000000的物理地址中，因为前面的地址中有I/O设备

在`_entry`中设置了一个初始stack，`stack0`来让xv6执行`kernel/start.c`。在`start`函数先在machine模式下做一些配置，然后通过RISC-V提供的`mret`指令切换到supervisor mode，使program counter切换到`kernel/main.c`

`main`先对一些设备和子系统进行初始化，然后调用`kernel/proc.c`中定义的`userinit`来创建第一个用户进程。这个进程执行了一个`initcode.S`的汇编程序，这个汇编程序调用了`exec`这个system call来执行`/init`，重新进入kernel。`exec`将当前进程的内存和寄存器替换为一个新的程序(`/init`)，当kernel执行完毕`exec`指定的程序后，回到`/init`进程。`/init`(`user/init.c`)创建了一个新的console device以文件描述符0,1,2打开，然后在console device中开启了一个shell进程，至此整个系统启动了
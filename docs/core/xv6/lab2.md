# System calls

## System call tracing
这个作业中，我们需要添加一个系统调用追踪功能
看一下给的示例1:
```sh
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
```
这个示例中，trace调用grep，仅仅跟踪了read系统调用，32是1<<SYS_read，SYS_read是系统调用编号，在kernel/syscall.h中给出了声明：
```c
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_pipe    4
#define SYS_read    5
...
```
根据比特位如果是1就跟踪指定的系统调用，例如上一个例子中32=(100000)，只跟踪read系统调用，如果trace参数为(1111111)，那么上面给出的系统调用将会被全部追踪

我们根据给的Hints一步步来看：
>首先在Makefile中的UPROGS中添加`$U/_trace`
>将系统调用的原型添加到user/user.h
```c
// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
...
int trace(int);
```

>存根添加到user/usys.pl
```perl
entry("fork");
entry("exit");
entry("wait");
entry("pipe");
entry("read");
...
entry("trace");
```

>将系统调用编号添加到kernel/syscall.h
```c
// System call numbers
#define SYS_trace  22
```

完成这些之后，我们执行make qemu，编译正确，接下来我们需要实现trace这个系统调用

>在kernel/sysproc.c中添加一个`sys_trace()`函数，它通过将参数保存到`proc`结构体（请参见kernel/proc.h）里的一个新变量中来实现新的系统调用，从用户空间检索系统调用参数的函数在kernel/syscall.c中，您可以在kernel/sysproc.c中看到它们的使用示例

>修改kernel/syscall.c中的`syscall()`函数以打印跟踪输出。您将需要添加一个系统调用名称数组以建立索引

通过这两点，我们可以知道，我们最后需要实现的打印输出在syscall函数中，梳理一下流程，我们在sysproc.c中实现的sys_trace函数接收一个mask掩码，保存到proc这个进程结构体中，syscall函数从proc结构体中拿到掩码，再打印输出

首先在proc结构体中添加变量：
```c
uint64 trace_mask;
```

实现sys_trace函数，将mask保存到proc
```c
uint64 
sys_trace(void){
  int mask;
  if(argint(0, &mask) < 0)
    return -1;
  struct proc *p=myproc();
  p->trace_mask=mask;
  return 0;
}
```

最后，实现syscall函数，添加一个系统调用名称数组以建立索引，注意num从1开始，我们添加的数组也要从1开始
```c
static char* syscall_nums[]={
  "null","fork","exit","wait","pipe",
  "read","kill","exec","fstat","chdir",
  "dup","getpid","sbrk","sleep","uptime",
  "open","write","mknod","unlink","link",
  "mkdir","close","trace"
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    int trace_mask=p->trace_mask;
    if((trace_mask>>num)&1){
      printf("%d: syscall %s -> %d\n",p->pid,syscall_nums[num],p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

现在我们执行make qemu，执行示例成功
```sh
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
```

但是，我们发现不调用trace也会打印输出，但是我们发现这个输出和上面示例输出一致，意味着在这个命令中，mask值可能是上一次遗留的
```sh
$ grep hello README
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
```

原因是，我们需要在proc.c中的freeproc函数中，添加上`p->trace_mask=0;`，在每次释放freeproc时候，将mask值清空

>修改`fork()`（请参阅kernel/proc.c）将跟踪掩码从父进程复制到子进程。

```c
// copy trace_mask in the child
np->trace_mask=p->trace_mask;
```

至此，运行测试用例，PASS

## Sysinfo
本作业中，我们需要添加一个系统调用sysinfo，它收集有关正在运行的系统的信息，看一下头文件中的结构体，freemem记录空闲内存的字节数，nproc记录state字段不为UNUSED的进程数
```c
struct sysinfo {
  uint64 freemem;   // amount of free memory (bytes)
  uint64 nproc;     // number of process
};
```

还是根据Hints一步步来，仿照上一个作业，修复编译问题，不做赘述

>`sysinfo`需要将一个`struct sysinfo`复制回用户空间；请参阅`sys_fstat()`(kernel/sysfile.c)和`filestat()`(kernel/file.c)以获取如何使用`copyout()`执行此操作的示例

参照sysfile.c中sys_fstat函数中下面这段代码
```c
uint64 st; // user pointer to struct stat
if(argfd(0, 0, &f) < 0 || argaddr(1, &st) < 0)
    return -1;
```

参照file.c中filestat函数中下面这段代码
```c
struct proc *p = myproc();
struct stat st;
if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
      return -1;
```

从这两段代码中，我们大致知道我们需要做类似的事情，就是把sysinfo中的信息，复制给一个addr指针，传回用户空间，可以得到下面这段代码
```c
uint64 
sys_sysinfo(void){
  struct sysinfo info;
  uint64 addr;
  struct proc *p=myproc();
  if(argaddr(0,&addr)<0){
    return -1;
  }
  info.nproc=get_nproc();
  info.freemem=get_freemem();
  if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
      return -1;
  return 0;
}
```

>要获取空闲内存量，请在kernel/kalloc.c中添加一个函数
>要获取进程数，请在kernel/proc.c中添加一个函数

这里，我们要实现上面那段代码中的get_nproc和get_freemem函数，分别是获取进程数和空闲内存量

根据对应提示，仿照对应文件中的其他函数，可以容易写出来
```c
uint64 
get_nproc(){
  struct proc *p;
  uint64 cnt=0;
  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state != UNUSED) {
      cnt++;
    } 
    release(&p->lock);
  }
  return cnt;
}

uint64 
get_freemem(){
  struct run *r;
  uint64 cnt=0;
  acquire(&kmem.lock);
  r=kmem.freelist;
  while(r){
    r=r->next;
    cnt++;
  }
  release(&kmem.lock);
  return cnt*PGSIZE;
}
```

至此，执行make qemu，运行sysinfotest
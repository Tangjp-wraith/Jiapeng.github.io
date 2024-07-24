# Page tables

## Print a page table

本作业中，要求完成一个打印页表内容的函数
根据给的Hints，我们看一下/kernel/vm.c中的freewalk函数

```c
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```

这个函数中，我们需要了解的是PTE_V是用来判断页表项是否有效，而(pte & (PTE_R|PTE_W|PTE_X)) == 0则是用来判断是否不在最后一层。有了这两个判断条件，我们就可以完成vmprint函数了,下面是示例输出：

```sh
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

根据示例输出，我们发现我们需要完成一个递归函数，..的缩进表明在树中的深度，因此，我们需要引入一个辅助参数depth，这样利用dfs的思想，我们很容易完成vmprint函数

```c
char *prefix[]={
  "..",".. ..",".. .. .."
};

void 
vmprint(pagetable_t pagetable,uint64 depth){
  if(depth==0){
    printf("page table %p\n",pagetable);
  }
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte&PTE_V){
      printf("%s%d: ",prefix[depth],i);
      uint64 child = PTE2PA(pte);
      printf("pte %p pa %p\n",pte,child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        vmprint((pagetable_t)child, depth + 1);
      }
    }
  }
}
```

## A kernel page table per process

本作业中，要求修改内核来让每一个进程在内核中执行时使用它自己的内核页表的副本

根据给的Hints，一步步来看：

> 在 `struct proc`中为进程的内核页表增加一个字段

第一步比较容易，我们只需要在proc结构体中添加一个kernel pgtable

```c
pagetable_t kpagetable;      // Kernel page table
```

> 为一个新进程生成一个内核页表的合理方案是实现一个修改版的 `kvminit`，这个版本中应当创造一个新的页表而不是修改 `kernel_pagetable`。你将会考虑在 `allocproc`中调用这个函数

第二步，初始化内核态页表，根据提示我们要仿照kvminit函数，实现如下函数，需要在 kernel/defs.h 添加函数声明

```c
pagetable_t     kkvminit(void);
```

参考kvminit函数中为kernel_pagetable添加映射的kvmmap函数，重新实现一个映射函数kkvmmap

```c
void 
kkvmmap(pagetable_t kpagetable,uint64 va,uint64 pa,uint64 sz,int perm){
  if(mappages(kpagetable,va,sz,pa,perm)!=0){
    panic("kkvmmap");
  }
}
```

然后仿照kvminit函数的方式初始化

```c
pagetable_t
kkvminit(){
  pagetable_t kpagetable = (pagetable_t) kalloc();
  memset(kpagetable,0,PGSIZE);
  kkvmmap(kpagetable,UART0, UART0, PGSIZE, PTE_R | PTE_W);
  kkvmmap(kpagetable,VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  kkvmmap(kpagetable,CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  kkvmmap(kpagetable,PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  kkvmmap(kpagetable,KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  kkvmmap(kpagetable,(uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  kkvmmap(kpagetable,TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  return kpagetable;
}
```

最后在kernel/proc.c 中的 allocproc 函数里添加调用函数的代码：

```c
// An empty kernel page table
  p->kpagetable=kkvminit();
  if(p->kpagetable==0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
```

> 确保每一个进程的内核页表都关于该进程的内核栈有一个映射。在未修改的XV6中，所有的内核栈都在 `procinit`中设置。你将要把这个功能部分或全部的迁移到 `allocproc`中

第三步，初始化内核栈，根据提示原本所有的内核栈都在 `procinit`中设置

```c
void
procinit(void)
{
  struct proc *p;
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
      char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;
  }
  kvminithart();
}
```

我们需要把这个功能部分或全部的迁移到 `allocproc`中，紧接上一步初始化内核态页表的代码下面接着添加初始化内核栈的代码：

```c
char *pa=kalloc();
if(pa==0){
  panic("kalloc");
}
uint64 va=KSTACK((int) (p-proc));
kkvmmap(p->kpagetable,va,(uint64)pa,PGSIZE,PTE_R|PTE_W);
p->kstack=va;
```

> 修改 `scheduler()`来加载进程的内核页表到核心的 `satp`寄存器(参阅 `kvminithart`来获取启发)。不要忘记在调用完 `w_satp()`后调用 `sfence_vma()`
> 没有进程运行时scheduler()应当使用kernel_pagetable

第四步，进程调度时，切换内核页。将当前进程的kernel page存入stap寄存器中

```c
w_satp(MAKE_SATP(p->kpagetable));
sfence_vma();
```

没有进程在运行则使用内核原来的kernel pagtable

```c
kvminithart();
```

> 在 `freeproc`中释放一个进程的内核页表
> 你需要一种方法来释放页表，而不必释放叶子物理内存页面

第五步，释放内核页表
释放页表的第一步是先释放页表内的内核栈，因为页表内存储的内核栈地址本身就是一个虚拟地址，需要先将这个地址指向的物理地址进行释放：

```c
if(p->kstack){
  pte_t* pte=walk(p->kpagetable,p->kstack,0);
  if(pte==0){
    panic("freeproc: walk");
  }
  kfree((void*)PTE2PA(*pte));
}
p->kstack=0;
```

这里的地址释放，需要手动释放内核栈的物理地址。因此需要单独对内核栈进行处理。

然后是释放页表，直接遍历所有的页表，释放所有有效的页表项即可。仿照 freewalk 函数。由于 freewalk 函数将对应的物理地址也直接释放了，我们这里释放的内核页表仅仅只是用户进程的一个备份，释放时仅释放页表的映射关系即可，不能将真实的物理地址也释放了。因此不能直接调用freewalk 函数，而是需要进行更改，自己实现一个proc_freewalk函数：

```c
static void
proc_freewalk(pagetable_t pagetable){
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      pagetable[i] = 0;
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        uint64 child = PTE2PA(pte);
        proc_freewalk((pagetable_t)child);
      } 
    }
  }
  kfree((void*)pagetable);
}
```

再在 freeproc 中进行调用：

```c
if(p->kpagetable){
  proc_freewalk(p->kpagetable);
}
p->kpagetable=0;
```

最后，切换进程内核页表，更改kvmpa函数

```c
pte = walk(myproc()->kpagetable, va, 0);
```

## Simplify copyin/copyinstr

本作业衔接上一个作业，主要目的是将用户进程页表的所有内容都复制到内核页表中，这样的话，就完成了内核态直接转换虚拟地址的方法。

首先写一个复制页表项的方法（需要注意的是，这里的 FLAG 标记位，PTE_U需要设置为0，以便内核访问。）：

```c
void u2kvmcopy(pagetable_t upagetable, pagetable_t kpagetable, uint64 oldsz, uint64 newsz) {
  oldsz = PGROUNDUP(oldsz);
  for (uint64 i = oldsz; i < newsz; i += PGSIZE) {
    pte_t* pte_from = walk(upagetable, i, 0);
    pte_t* pte_to = walk(kpagetable, i, 1);
    if(pte_from == 0) panic("u2kvmcopy: src pte do not exist");
    if(pte_to == 0) panic("u2kvmcopy: dest pte walk fail");
    uint64 pa = PTE2PA(*pte_from);
    uint flag = (PTE_FLAGS(*pte_from)) & (~PTE_U);
    *pte_to = PA2PTE(pa) | flag;
  }
}
```

分别找 fork()，sbrk()，exec() 函数，在其中添加代码。

```c
np->cwd = idup(p->cwd);
// 添加
u2kvmcopy(np->pagetable,np->kpagetable,0,np->sz);

safestrcpy(np->name, p->name, sizeof(p->name));
```

sbrk() 需要到 `sysproc.c` 找，可以发现，调用的是 `growproc` 函数，在其中添加防止溢出的函数：

```c
if(n > 0){
  if (PGROUNDUP(sz + n) >= PLIC)
    return -1;
  if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
    return -1;
  }
  u2kvmcopy(p->pagetable, p->kpagetable, sz-n, sz);
} else if(n < 0){
  sz = uvmdealloc(p->pagetable, sz, sz + n);
}
```

exec() 在 `kernel/exec.c` 中，在执行新的程序前，初始化之后，进行页表拷贝：

```c
u2kvmcopy(pagetable,p->kpagetable,0,sz);
// Push argument strings, prepare rest of stack in ustack.
```

userinit() 函数内也需要调用复制页表的函数。

```c
uvminit(p->pagetable, initcode, sizeof(initcode));
p->sz = PGSIZE;
u2kvmcopy(p->pagetable,p->kpagetable,0,p->sz);
```

将copyin和copyinstr函数内部全部注释掉，改为调用copyin_new和copyinstr_new函数即可。

```c
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable,dst,srcva,len);
}
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable,dst,srcva,max);
}
```

至此，添加answers-pgtbl.txt，time.txt，make grade，pass

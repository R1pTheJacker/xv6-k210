## 1. XV6中的进程
### 1.1 XV6中的进程调度
#### 1.1.1 进程调度时机
在xv6中，进程调度由时间中断 (***timer interrupt***) 控制，时间中断发生后，内核在 ***usertrap*** 函数中捕获该中断，然后跳转到 ***yield()*** 函数，***yield()*** 的函数主要作用是把进程的状态从 *RUNNING* 设置到 *RUNNABLE* ，代表进程让出此cpu,并且跳跃到 ***sched()*** 函数，***sched()*** 函数是实际进行上下文切换的函数。

```C
//usertrap
if(which_dev == 2) yield(); 
//如果是时间中断，yield

//proc.c
void yield(void)
{
    //...
    p->state = RUNNABLE;
    sched();
    //...
}
```

#### 1.1.2 上下文切换
- 正如从用户态进入到内核态需要保护所有用户态的寄存器，从内核态恢复到用户态需要恢复所有的寄存器，进程切换也是一样的。从A进程切换到B进程需要保护A进程的现场，从其他进程切换回A进程的时候需要恢复A进程的寄存器，并且从切换走的地方继续执行。

- 在xv6中，保护进程切换寄存器的位置位于内核的进程结构体 ***proc*** 中，有一个专门的 ***proc->context*** 结构。

- 在xv6中，内核初始化的时候会给每一个核心绑定一个程序scheduler，该程序的内容很简单，死循环所有的进程列表(***Round Robin***)，找到 *RUNNABLE* 的进程，切换过去。 由此xv6的进程切换可以实现成如下：

  - 假设只有A，B两个进程，单个核心，当前A进程在运行

  - 时间中断发生，A进程的 ***state*** 被设置成 *RUNNABLE*
  
  - 保存A进程的所有寄存器，恢复scheduler循环的所有寄存器，寻找到B进程的状态为 *RUNNABLE*

  - 保存scheduler的所有寄存器，恢复B进程的所有寄存器，然后把B进程 的状态设置成 *RUNNING*，完成进程切换
  
  - 时间中断发生，B进程让出CPU，切换到scheduler的状态中
  
  - 由于之前scheduler保存了状态，所以B进程已经被循环过了，此时循环到了结束，重头开始，回到了A进程
  
  - 恢复A进程的寄存器，A进程上下文恢复，继续执行，中间被切换掉的过程对A进程是无感知的

### 1.2 XV6中的*PCB*

- xv6 使用结构体 ***struct proc*** 来维护一个进程的众多状态。一个进程最为重要的状态是进程的页表，内核栈，当前运行状态。我们接下来会用 ***p->xxx*** 来指代 ***proc*** 结构中的元素。

- 每个进程都有一个运行线程（或简称为线程）来执行进程的指令。线程可以被暂时挂起，稍后再恢复运行。系统在进程之间切换实际上就是挂起当前运行的线程，恢复另一个进程的线程。线程的大多数状态（局部变量和函数调用的返回地址）都保存在线程的栈上。

- 每个进程都有用户栈和内核栈（***p->kstack***）。当进程运行用户指令时，只有其用户栈被使用，其内核栈则是空的。然而当进程（通过系统调用或中断）进入内核时，内核代码就在进程的内核栈中执行；进程处于内核中时，其用户栈仍然保存着数据，只是暂时处于不活跃状态。进程的线程交替地使用着用户栈和内核栈。要注意内核栈是用户代码无法使用的，这样即使一个进程破坏了自己的用户栈，内核也能保持运行。

- 当进程使用系统调用时，处理器转入内核栈中，提升硬件的特权级，然后运行系统调用对应的内核代码。当系统调用完成时，又从内核空间回到用户空间：降低硬件特权级，转入用户栈，恢复执行系统调用指令后面的那条用户指令。线程可以在内核中“阻塞”，等待 I/O, 在 I/O 结束后再恢复运行。

- ***p->state*** 指示了进程的状态：新建、准备运行、运行、等待 I/O 或退出状态中。

- ***p->pgdir*** 以 ***RISC-V*** 硬件要求的格式保存了进程的页表。xv6 让分页硬件在进程运行时使用 ***p->pgdir***。进程的页表还记录了保存进程内存的物理页的地址。
```C
struct proc {
    struct spinlock lock;

    // p->lock must be held when using these:
    enum procstate state;        // Process state
    struct proc *parent;         // Parent process
    void *chan;                  // If non-zero, sleeping on chan
    int killed;                  // If non-zero, have been killed
    int xstate;                  // Exit status to be returned to parent's wait
    int pid;                     // Process ID

    // these are private to the process, so p->lock need not be held.
    uint64 kstack;               // Virtual address of kernel stack
    uint64 sz;                   // Size of process memory (bytes)
    pagetable_t pagetable;       // User page table
    pagetable_t kpagetable;      // Kernel page table
    struct trapframe *trapframe; // data page for trampoline.S
    struct context context;      // swtch() here to run process
    struct file *ofile[NOFILE];  // Open files
    struct dirent *cwd;          // Current directory
    char name[16];               // Process name (debugging)
    int tmask;                    // trace mask
};
```
### 1.3 XV6中的*fork()*
```C
int fork(void) {
    int i, pid;
    struct proc *np;
    struct proc *p = myproc();

    // Allocate process.
    if((np = allocproc()) == NULL) {
        return -1;
    }

    // Copy user memory from parent to child.
    if(uvmcopy(p->pagetable, np->pagetable, np->kpagetable, p->sz) < 0){
        freeproc(np);
        release(&np->lock);
        return -1;
    }
    np->sz = p->sz;

    np->parent = p;

    // copy tracing mask from parent.
    np->tmask = p->tmask;

    // copy saved user registers.
    *(np->trapframe) = *(p->trapframe);

    // Cause fork to return 0 in the child.
    np->trapframe->a0 = 0;

    // increment reference counts on open file descriptors.
    for(i = 0; i < NOFILE; i++)
        if(p->ofile[i])
            np->ofile[i] = filedup(p->ofile[i]);
    np->cwd = edup(p->cwd);

    safestrcpy(np->name, p->name, sizeof(p->name));

    pid = np->pid;

    np->state = RUNNABLE;

    release(&np->lock);

    return pid;
}
```
- ***fork()*** 函数首先调用 ***allocproc()*** 函数获得并初始化一个*PCB* ***struct proc***。

- 此外，在 ***allocproc()*** 函数中还会对进程的内核栈进行初始化，在内核栈里设置一个 ***Trap Frame***，把 ***Trap Frame*** 的上下文部分都置为0。
- 然后，***fork()*** 函数使用 ***copyuvm()*** 函数复制原进程的虚拟内存结构。
- 为了能让子进程返回时处在和父进程一模一样的状态，***Trap Frame*** 也会被拷贝。
- 为了让子进程系统调用的返回值为0，子进程的 ***a0*** 寄存器会被置为0。
- 然后，父进程打开的文件描述符会被全部拷贝给子进程，还有父进程所处于的目录。这些操作都会增加文件描述符和目录的被引用数。
- 最后，***fork()*** 函数拷贝了父进程的名字，设置了子进程的状态为*RUNNABLE*，然后返回子进程 ***pid*** 给父进程。
- 子进程被创建后，在某个时刻调度子进程运行时，***fork()*** 函数会第二次返回给子进程，此时返回值为0。


##Reference
[1] xv6 fork的实现，https://www.blurredcode.com/2020/11/xv6fork%E7%9A%84%E5%AE%9E%E7%8E%B0/

[2] xv6: a simple, Unix-like teaching operating system, https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf
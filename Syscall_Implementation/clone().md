## clone() 调用的实现

***clone()*** 调用创建了一个新的内核线程，并且此线程与其进程共享地址空间。
- 若创建成功
  - 将线程 **ID**（**Tid**）返回给父进程
  - 将 **0** 返回给此线程
- 若创建失败
  - 将 **-1** 返回给父进程

### 修改 kernel/sysproc.c

在 **sysproc.c** 中添加系统调用的包裹函数，防止用户传递错误的参数。
```C
int sys_clone(void) {
    void *fcn, *arg, *stack;
    if (argptr(0, (void*)&fcn, sizeof(*fcn)) < 0)
        return -1;
    if (argptr(1, (void*)&arg, sizeof(*arg)) < 0)
        return -1;
    if (argptr(2, (void*)&stack, sizeof(*stack)) < 0)
        return -1;

    return clone(fcn, arg, stack);
}
```

### 修改 kernel/proc.c
完整的 ***clone()*** 实现
```C
int clone(void (*fcn)(void*), void *arg, void *stack) {
    int i;
    struct proc *thread;
    struct proc *curproc = myproc();

    if ((curproc->sz - (uint) stack) < PGSIZE || ((uint) stack % PGSIZE) != 0) {
        return -1;
    }
    if ((thread = allocproc()) == 0) {
        return -1;
    }

    thread->pgdir = curproc->pgdir;
    thread->sz = curproc->sz;
    thread->parent = curproc;
    *thread->tf = *curproc->tf;

    uint user_stack[2];
    user_stack[0] = 0xffffffff; 
    user_stack[1] = (uint) arg;
    uint stack_top = (uint) stack + PGSIZE;
    stack_top -= 8;
    if (copyout(thread->pgdir, stack_top, user_stack, 8) < 0) {
        return -1;
    }

    thread->tf->sp = (uint) stack_top;
    thread->tf->a0 = 0;

    for (i = 0; i < NOFILE; i++)
        if (curproc->ofile[i])
            thread->ofile[i] = filedup(curproc->ofile[i]);

    thread->cwd = idup(curproc->cwd);
    safestrcpy(thread->name, curproc->name, sizeof(curproc->name));

    acquire(&ptable.lock);
    thread->state = RUNNABLE;
    release(&ptable.lock);

    return thread->pid;
}
```

### 添加系统调用流程
- 在xv6-user/user.h中，添加系统调用的声明
```C
int clone(void(*fcn)(void*), void *arg, void *stack);
```
- 在xv6-user/usys.pl末尾，添加入口
```pl
entry("clone");
```

- 在kernel/include/syscall.h中，添加新的系统调用号宏定义
```C
#define SYS_clone xxx
```

- 在kernel/include/syscall.c中
  - 添加系统调用声明
   ```C
    extern uint64 sys_clone(void);
   ```
  - 修改系统调用表 
   ```C
    static uint64 (*syscalls[])(void) = {
        ......
        [SYS_clone]    sys_clone,
    };
   ```
- 在kernel/defs/h中添加函数原型
```C
int clone(void(*fcn)(void*), void *arg, void *stack);
```


##Reference
[1] xv6 fork的实现，https://www.blurredcode.com/2020/11/xv6fork%E7%9A%84%E5%AE%9E%E7%8E%B0/

[2] xv6: a simple, Unix-like teaching operating system, https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf
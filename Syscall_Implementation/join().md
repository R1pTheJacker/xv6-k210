## join () 调用的实现

***join()*** 系统调用会遍历所有运行中的进程来找到当前进程所拥有的线程。若找到了该线程，就将他杀死，并清理进程表。
- 如果没有找到子线程就返回 **-1**
- 如果子线程仍在运行，就调用 ***sleep()***

### 修改 kernel/sysproc.c

在 **sysproc.c** 中添加系统调用的包裹函数。
```C
int sys_join(void)
{
	return join();
}
```

### 修改 kernel/proc.c
完整的 ***join()*** 实现
```C
int join(void) {
    struct proc *t;
    int haveThreads, pid;
    struct proc *curproc = myproc();
	
    acquire(&ptable.lock);
    for(;;) {
	haveThreads = 0;
	for(t = ptable.proc; t < &ptable.proc[NPROC]; t++) {
            if((t->parent != curproc) || (t->parent->pgdir != curproc->pgdir) )
		continue;
	        haveThreads = 1;
	        if(t->state == ZOMBIE) {	
		    pid = t->pid;
		    kfree(t->kstack);
		    t->kstack = 0;
		    t->pid = 0;
		    t->parent = 0;
		    t->name[0] = 0;
		    t->killed = 0;
		    t->state = UNUSED;
		    release(&ptable.lock);

		    return pid;
            }
        }

        if(!haveThreads || curproc->killed) {
            release(&ptable.lock);
            return -1;
        }

        sleep(curproc, &ptable.lock);
    }
}

```

### 添加系统调用流程
- 在xv6-user/user.h中，添加系统调用包裹函数的声明
```C
int join(void);
```
- 在xv6-user/usys.pl末尾，添加入口
```pl
entry("join");
```

- 在kernel/include/syscall.h中，添加新的系统调用号宏定义
```C
#define SYS_join xxx
```

- 在kernel/include/syscall.c中
  - 添加系统调用声明
   ```C
    extern uint64 sys_join(void);
   ```
  - 修改系统调用表 
   ```C
    static uint64 (*syscalls[])(void) = {
        ......
        [SYS_join]    sys_join,
    };
   ```
- 在kernel/defs/h中添加函数原型
```C
int join(void);
```


##Reference
[1] xv6 fork的实现，https://www.blurredcode.com/2020/11/xv6fork%E7%9A%84%E5%AE%9E%E7%8E%B0/

[2] xv6: a simple, Unix-like teaching operating system, https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf
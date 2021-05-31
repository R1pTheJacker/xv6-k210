## 3. XV6中的trap
### 3.1 Trapframe
***trapframe*** 是一个保存了内核和用户空间的寄存器的数据结构。

```C
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process’s kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
};
```

### 3.2 Trampoline
#### 3.2.1 什么是 ***trampoline***
***trampoline*** 是指保存在 **trampoline.s** 的汇编代码，它负责在用户空间与内核空间的切换。在用户空间和内核空间中，这段代码被映射到相同的虚拟地址，这保证了切换页表后，程序仍能正常运行。

#### 3.2.2 ***trampoline*** 在地址空间的位置

- ***RISC-V*** 的硬件在发生 *trap* 时不会切换页表，因此用户页表需要包含一个 ***stvec*** 寄存器（***stvec***：保存 ***trap handler*** 的地址，当发生 *trap* 时，将他的值设为 ***pc***）指向的中断向量映射表。

- 中断向量需要将 ***satp*** 寄存器的内容切换为内核根页表。
- 为了避免崩溃，无论在内核态还是用户态，存储在 ***trampoline*** 中的中断向量指令必须映射在页表中的同一虚拟地址。


### 3.3 内核空间的 Trap
内核空间下会可能会发生两种 ***trap***
- 异常（***Exceptions***）
- 设备中断（***Device Interrupt***）

#### 3.3.1 Trap 发生后的处理步骤

- 保存所有寄存器
- 调用 ***kerneltrap()*** 函数
- 恢复所有寄存器
- 执行 ***sret*** 指令
  - 它将 ***sepc*** 复制到 ***Program Counter*** 然后继续执行中断之前的内核程序
  - 当 ***trap*** 发生时，***sepc*** 寄存器中负责保存当前 ***Program Counter***

#### 3.3.2 部分实现细节
##### 3.3.2.1 Kernevec

内核空间中，***stvec*** 寄存器保存 kernelvec汇编代码的地址。kernelvec保存了所有在同一内核线程中发生中断时的寄存器。

```C
kernelvec:
        // make room to save registers.
        addi sp, sp, -256

        // save the registers.
        sd ra, 0(sp)
        sd sp, 8(sp)
        sd gp, 16(sp)
        sd tp, 24(sp)
        sd t0, 32(sp)
        sd t1, 40(sp)
        sd t2, 48(sp)
        sd s0, 56(sp)
        sd s1, 64(sp)
        sd a0, 72(sp)
        sd a1, 80(sp)
        sd a2, 88(sp)
        sd a3, 96(sp)
        sd a4, 104(sp)
        sd a5, 112(sp)
        sd a6, 120(sp)
        sd a7, 128(sp)
        sd s2, 136(sp)
        sd s3, 144(sp)
        sd s4, 152(sp)
        sd s5, 160(sp)
        sd s6, 168(sp)
        sd s7, 176(sp)
        sd s8, 184(sp)
        sd s9, 192(sp)
        sd s10, 200(sp)
        sd s11, 208(sp)
        sd t3, 216(sp)
        sd t4, 224(sp)
        sd t5, 232(sp)
        sd t6, 240(sp)

    // call the C trap handler in trap.c
        call kerneltrap

        // restore registers.
        ld ra, 0(sp)
        ld sp, 8(sp)
        ld gp, 16(sp)
        // not this, in case we moved CPUs: ld tp, 24(sp)
        ld t0, 32(sp)
        ld t1, 40(sp)
        ld t2, 48(sp)
        ld s0, 56(sp)
        ld s1, 64(sp)
        ld a0, 72(sp)
        ld a1, 80(sp)
        ld a2, 88(sp)
        ld a3, 96(sp)
        ld a4, 104(sp)
        ld a5, 112(sp)
        ld a6, 120(sp)
        ld a7, 128(sp)
        ld s2, 136(sp)
        ld s3, 144(sp)
        ld s4, 152(sp)
        ld s5, 160(sp)
        ld s6, 168(sp)
        ld s7, 176(sp)
        ld s8, 184(sp)
        ld s9, 192(sp)
        ld s10, 200(sp)
        ld s11, 208(sp)
        ld t3, 216(sp)
        ld t4, 224(sp)
        ld t5, 232(sp)
        ld t6, 240(sp)

        addi sp, sp, 256

        // return to whatever we were doing in the kernel.
        sret
```

##### 3.3.2.2 内核中断处理程序

```C
// interrupts and exceptions from kernel code go here via kernelvec,
// on whatever the current kernel stack is.
// must be 4-byte aligned to fit in stvec.
void 
kerneltrap()
{
    int which_dev = 0;
    uint64 sepc = r_sepc();
    uint64 sstatus = r_sstatus();
    uint64 scause = r_scause();

    if((sstatus & SSTATUS_SPP) == 0)
        panic("kerneltrap: not from supervisor mode");
    if(intr_get() != 0)
        panic("kerneltrap: interrupts enabled");

    // check if it’s an external interrupt or software interrupt,
    // and handle it.
    // returns 2 if timer interrupt,
    // 1 if other device,
    // 0 if not recognized.
    if((which_dev = devintr()) == 0){
        printf("scause %p\n", scause);
        printf("sepc=%p stval=%p\n", r_sepc(), r_stval());
        panic("kerneltrap");
    }

    // give up the CPU if this is a timer interrupt.
    if(which_dev == 2 && myproc() != 0 && myproc()->state == RUNNING)
        yield();

    // the yield() may have caused some traps to occur,
    // so restore trap registers for use by kernelvec.S's sepc instruction.
    w_sepc(sepc);
    w_sstatus(sstatus);
}
```

##### 3.3.2.3 屏蔽中断

***trap*** 触发后，***stvec*** 寄存器还没有来得及设置，因此存在一段空窗期。在次其间需要确保没有中断发生。***RISC-V*** 在决定处理一个 ***trap*** 后屏蔽中断，并且在stvec被设置前不会开启中断。

### 3.4 用户空间的 Trap
在用户空间中，相比内核空间，要多一种trap
- 异常（***Exceptions***）
- 设备中断（***Device Interrupt***）
- 系统调用（***System Call***）

***user trap*** 的处理顺序
- ***trampoline.s***
  - **uservec**
- ***trap.c***
  - **usertrap**
  - **usertrapret**
- ***trampoline.s***
  - **userret**

#### 3.4.1 System Call Trap 处理流程
一个执行 ***write()*** 系统调用的例子
```
  write()
      trampoline / uservec
          usertrap() in trap.c
              syscall() in syscall.c
                  sys_write() in sysfile.c
          usertrapret()
      trampoline / userret
  write()
```

##### 第一阶段
- 当执行用户代码时，***stvec*** 寄存器被设置为 ***uservec***
- 用户空间的 ***trap*** 从uservec开始执行
- 寄存器 ***sscratch*** 指向一个 ***trapframe***


##### 第二阶段
- 用户程序执行了一个系统调用，调用了 ***ecall***
- 跳转到 ***stvec*** ( i.e. 将 ***pc*** 设置为  ***stvec*** )
- 将 ***pc*** 保存到 ***sepc***
- 将 ***user mode*** 切换为 ***supervisor mode*** (抽象概念，不出现在代码中).
- 从 ***uservec*** 处开始运行代码
- 将用户寄存器保存在 ***trapframe*** 中
- 从 ***trapframe*** 中恢复内核栈指针
- 保持当前的 ***hartid***
- 加载 ***usertrap()*** 代码，内核空间中的代码将处理这个用户 ***trap***
- 从 ***trapframe*** 中恢复内核页表
- 跳转到 ***usertrap()***

##### 第三阶段
***usertrap()*** 函数开始处理中断，异常或者来自用户态的系统调用。

##### 第四阶段
调用 ***usertrapret*** (kernel/trap.c)。这个函数设置了如 ***stvec***, ***sepc*** 等寄存器，为将来返回用户空间做准备。

##### 第五阶段
```C
  // tell trampoline.S the user page table to switch to.*
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which*
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
```
It jumps to userret, and switch to user page table. satp is passed in as a param.
这个阶段跳转到 ***userret***，并且从内核页表切换为用户页表，***satp***（用于保存根页表） 在此过程中被当作参数传递。

##### 第六阶段
- 切换为用户页表
- 恢复寄存器
- 返回用户模式和用户 ***Program Counter***

### 3.5 定时中断
- 定时中断是 ***RISC-V*** 时钟硬件定时发出的，它被用来维护 ***XV6*** 的时钟，以及用来驱动切换进程

- ***RISC-V*** 要求定时中断在 ***machine mode*** 中处理，而不是 ***supervisor mode***。而 ***machine mode*** 没有启用分页，并且有一套自己的控制寄存器，一般不把内核代码运行在 ***machine mode*** 中。所以 ***XV6*** 处理定时中断的方式和上面提到的 ***trap*** 是不同的。

- 在 ***machine mode*** 下执行的代码在 ***start.c*** 里，在 ***main*** 之前执行，设置接受定时中断：一方面是对 ***CLINT***（core-local interrupt) 进行编程，使其在固定事件后发出中断信号，另一方面是设置 ***scratch*** 区，类似 ***trapframe***，以便 ***timer interrupt handler*** 能够保存寄存器，并且找到 ***CLINT*** 寄存器的地址。最后，***start*** 设置 ***mtvec*** 为 ***timervec***，启用定时中断

- 定时中断可能发生在 *用户态* 和 *内核态*，内核不能再关键操作时禁用中断，因此定时中断处理程序在完成自己任务时必须保证不能打乱被中断的内核代码。基本策略是让定时中断请求 ***RISC-V*** 发出一个 ***software interrupt***，然后马上返回。 ***RISC-V*** 使用普通的 ***trap*** 机制将 ***software interrupt*** 传给内核，允许内核禁用他们。处理定时中断产生的软中断的代码在 ***devintr*** 里

- ***machine mode*** 定时中断向量是 ***timervec***，保存一些寄存器到 ***scratch*** 里，告诉 ***CLINT*** 何时产生下一个中断，让 ***RISC-V*** 发出一个软中断，然后恢复寄存器并返回



## Reference
[1] xv6: a simple, Unix-like teaching operating system, https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf
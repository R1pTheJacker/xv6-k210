## 2. XV6中的内存管理
### 2.1 XV6中的*PTE*
```C
typedef uint64 pte_t;
typedef uint64 *pagetable_t;// 512 PTEs
```

- ***pagetable_t*** 是指向512个 *PTE* 的指针
- 页表通过 ***kalloc()*** 被初始化
- ***pte*** 存储物理页面

### 2.2 XV6的分页系统
#### 2.2.1 如何翻译虚拟地址
- ***RISC-V*** 页表在逻辑上是由 **2^27^** （134,217,728）个页表项（***PTE***）构成的数组。 

- 分页硬件通过使用 **39** 位中的前 **27** 位索引到页表中以查找 ***PTE*** ，并生成56位物理地址（其前 **44** 位来自 ***PTE*** 中的 ***PPN*** 且其最低位）来转换虚拟地址。 末位 **12** 位直接从虚拟地址复制过来。 
- 页表以 **4096**（2^12^）字节的对齐块的粒度为操作系统提供了对虚拟到物理地址转换的控制。这样的字节块称为页面。

#### 2.2.2 申请地址空间


- ***walk()*** 函数负责为当前虚拟地址找到合适的物理页面并返回它的地址。
  - ```C
    static pte_t * walk(pagetable_t pagetable, uint64 va, int alloc)
    ```

  - 从二级页表遍历至零级页表，找到当前 ***PTE*** 对应的页表等级和虚拟地址
  - 找到 ***PTE*** 对应的页面
  - 若此 ***PTE*** 存在或者 ***valid bit*** 为 0，分配一页新的页面
  - 结束循环遍历并返回此 ***PTE***
  
- ***mappages()*** 为找到的 ***PTE*** 建立新的映射。具体流程如下。
  - 找到给定虚拟地址对应的 ***PTE*** 
  - 找到物理地址对应的 ***PTE*** 并设置它的 ***permission bit***
  - 返回是否成功 


- ***kvminit()*** 中， 首先为内核地址空间分配一页物理内存来保存根页表。然后运行一系列内核需要的程序，包括：指令，数据，物理内存上限和其他设备的内存上限。


#### 2.2.3 分页如何运作

一个分页系统运作的简单示例：
> 1. 进程P1收到时钟中断
> 2. 页表从P1的页表切换为内核页表
> 3. 调度器找到了另一个可运行的进程P2
> 4. 页表从内核页表切换为P2的页表

- 内核和每个进程都有自己的页表
  - 用户空间切换到内核空间需要把页表从此用户进程页表切换为内核页表 

- 在用户空间和内核空间切换需要设置 ***satp*** 寄存器为根页表的地址
  - ***satp*** 寄存器每次只能存储一个根页表

- ***MMU*** 负责将虚拟地址翻译为物理地址




#### 2.2.4 代码具体实现

##### 2.2.4.1 分配 *PTE*：***walk()***
- ***walk()*** 函数返回了对应给定虚拟地址下可用的 ***PTE***。若 ***alloc*** != **0**，则分配任何所需的页面。

- ***RISC-V64*** 的 *sv39* 标准中分页系统有三级页表，每一个页表有 **512** 个 **64** 位的 ***PTE***。
  
- 64位的虚拟地址被分为如下五个区域
  - ***39...63 -- 均为 0***
  - ***30...38 -- 9位一级页表索引***
  - ***21...29 -- 9位二级页表索引***
  - ***12...20 -- 9位三级页表索引***
  - ***00...12 -- 12位页面偏移量***


```C
static pte_t *
walk(pagetable_t *pagetable*, uint64 *va*, int *alloc*)
{
  if(va >= MAXVA)
    panic(“walk”);

  for(int level = 2; level > 0; level—) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

##### 2.2.4.2 建立页面映射：***mappages()***
Create PTEs for virtual addresses starting at va that refer to physical addresses starting at pa. va and size might not be page-aligned.

***mappages()*** 在从虚拟地址 ***va*** 对应的 *PTE* 和物理地址 ***pa*** 对应的 *PTE* 并建立映射，***va*** 和 ***size*** 不保证字节对齐

- 成功时返回 **0** ，若 ***walk()*** 分配页面失败则返回 **-1**

```C
int
mappages(pagetable_t *pagetable*, uint64 *va*, uint64 *size*, uint64 *pa*, int *perm*)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

## Reference
[1] xv6: a simple, Unix-like teaching operating system, https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf
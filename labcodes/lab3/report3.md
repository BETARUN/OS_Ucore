# 实验四：虚拟内存管理

17343025 冯浚轩 软件工程1班

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 了解虚拟内存的Page Fault异常处理实现
- 了解页替换算法在操作系统中的实现

## 实验要求

- 本次实验是在实验二的基础上，借助于页表机制和实验一中涉及的中断异常处理机制，完成Page Fault异常处理和FIFO页替换算法的实现，结合磁盘提供的缓存空间，从而能够支持虚存管理，提供一个比实际物理内存空间“更大”的虚拟内存空间给系统使用。这个实验与实际操作系统中的实现比较起来要简单，不过需要了解实验一和实验二的具体实现。实际操作系统系统中的虚拟内存管理设计与实现是相当复杂的，涉及到与进程管理系统、文件系统等的交叉访问
- 可以尝试完成扩展练习，实现clock页替换算法（选做）

## 实验方案

填写已有实验，完成本实验中的相应练习

## 实验过程

### 新增数据结构的理解

在前面的实验中我们建立了对物理内存的分页管理，为了支持虚拟内存管理，我们还需要建立描述虚拟内存空间的数据结构，在该系统中使用两个结构体进行管理

```c
struct mm_struct {
    list_entry_t mmap_list;        // linear list link which sorted by start addr of vma
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    void *sm_priv;                   // the private data for swap manager
};

struct vma_struct {
    struct mm_struct *vm_mm; // the set of vma using the same PDT 
    uintptr_t vm_start;      // start addr of vma      
    uintptr_t vm_end;        // end addr of vma, not include the vm_end itself
    uint32_t vm_flags;       // flags of vma
    list_entry_t list_link;  // linear list link which sorted by start addr of vma
};
```

管理方式类似于空闲页管理，其中一个`mm_struct`结构体对应一个一级页表，在`*pgdir`变量中保存一级页表首地址，可以将该结构体其视为双向链表的占位头节点

而`vma_struct`结构体是双向链表的实际数据节点，每个结构体实例描述了一段连续的虚拟地址，用`vm_start`和`vm_end`表示起始，每个实例内还保存了该段虚拟内存的状态位，在vmm.h文件中有状态位的定义如下，分别指示是否可读，可写和可执行

```c
#define VM_READ                 0x00000001
#define VM_WRITE                0x00000002
#define VM_EXEC                 0x00000004
```

在实验手册中给出了这些内存管理结构的映射关系

![struct map](vmm_struct.png)


### Page Fault异常处理

当CPU的访存操作无法正常访问到物理内存时就会产生Page Fault异常，经过异常中断后CPU控制权会转移到`do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr)`函数进行异常处理，对于传入的参数

- `*mm`为产生异常时使用的一级页表对应的`mm_struct`结构体实例的地址
- `error_code`为异常产生时生成的错误码，错误码包括一下几位
  - 第0位存在位，为0表示对应物理页不存在
  - 第1位读写位，为1表示写异常，写了只读页
  - 第2位权限位，为1表示权限异常，用户态程序访问了内核空间
- `addr`为产生异常时CPU加载到CR2寄存器的线性地址，通过这个地址可以找到对应的页表

函数开始时首先要找到产生异常对应的`vma_struct`结构体实例

```c
struct vma_struct *vma = find_vma(mm, addr);
```

然后再根据`error_code`和`vma`中的状态位进行权限检查，在该处理函数中若权限检查不通过就直接退出，若能通过检查才进行接下来涉及页处理的语句

```c
//If the addr is in the range of a mm's vma?
if (vma == NULL || vma->vm_start > addr) {
    cprintf("not valid addr %x, and  can not find it in vma\n", addr);
    goto failed;
}
//check the error_code
switch (error_code & 3) {
default:
        /* error code flag : default is 3 ( W/R=1, P=1): write, present */
case 2: /* error code flag : (W/R=1, P=0): write, not present */
    if (!(vma->vm_flags & VM_WRITE)) {
        cprintf("do_pgfault failed: error code flag = write AND not present, but the addr's vma cannot write\n");
        goto failed;
    }
    break;
case 1: /* error code flag : (W/R=0, P=1): read, present */
    cprintf("do_pgfault failed: error code flag = read AND present\n");
    goto failed;
case 0: /* error code flag : (W/R=0, P=0): read, not present */
    if (!(vma->vm_flags & (VM_READ | VM_EXEC))) {
        cprintf("do_pgfault failed: error code flag = read AND not present, but the addr's vma cannot read or exec\n");
        goto failed;
    }
}
/* IF (write an existed addr ) OR
    *    (write an non_existed addr && addr is writable) OR
    *    (read  an non_existed addr && addr is readable)
    * THEN
    *    continue process
    */
```

进入页处理流程，首先要设置`perm`变量用于接下来的权限表示，然后再将`addr`线性地址对齐到页大小以便获取二级页表项

```c
uint32_t perm = PTE_U;
if (vma->vm_flags & VM_WRITE) {
    perm |= PTE_W;
}
addr = ROUNDDOWN(addr, PGSIZE);
```

然后调用`get_pte()`函数获取`addr`线性地址对应的二级页表项，若页表项全为0表示该线性地址与物理地址尚未建立映射或者已经撤销，需要调用`pgdir_alloc_page()`函数申请一块物理页并建立其与页表的映射

```c
ptep = get_pte(mm->pgdir, addr, 1);
if (*ptep == 0) {
    pgdir_alloc_page(mm->pgdir, addr, perm);
}
```

若页表项不全为0表示相应的物理页在硬盘上，首先检查硬盘换页机制是否能工作，若不能工作则处理失败，否则继续处理，调用`swap_in()`函数将硬盘上的物理页调入内存，函数结束后`page`变量获得调入内存页的页描述符地址，再建立该物理页与页表的映射关系，最后设置该页为可交换页

```c
else {
    if (swap_init_ok) {
        struct Page *page = NULL;
        swap_in(mm, addr, &page);
        page_insert(mm->pgdir, page, addr, perm);
        swap_map_swappable(mm, addr, page, 1);
    }
    else {
        cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
        goto failed;
    }
}
```

## 实验总结和对比

### 对比答案说明差异

### 本实验中重要的知识点

### OS原理中很重要但实验中没有对应的知识点

### 总结

## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)
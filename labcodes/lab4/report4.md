# 实验五：内核线程管理

17343025 冯浚轩 软件工程1班

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 了解内核线程创建/执行的管理过程
- 了解内核线程的切换和基本调度过程

## 实验要求

实验2/3完成了物理和虚拟内存管理，这给创建内核线程（内核线程是一种特殊的进程）打下了提供内存管理的基础。当一个程序加载到内存中运行时，首先通过ucore OS的内存管理子系统分配合适的空间，然后就需要考虑如何分时使用CPU来“并发”执行多个程序，让每个运行的程序（这里用线程或进程表示）“感到”它们各自拥有“自己”的CPU。

本次实验将首先接触的是内核线程的管理。内核线程是一种特殊的进程，内核线程与用户进程的区别有两个：

- 内核线程只运行在内核态，而用户进程会在在用户态和内核态交替运行
- 所有内核线程共用ucore内核内存空间，不需为每个内核线程维护单独的内存空间，而用户进程需要维护各自的用户内存空间

## 实验方案

接着填写已有实验，并完成练习

## 实验过程

### 新增数据结构的理解

在proc.h文件中定义了进程控制块，这是控制进程和线程的关键数据结构，如下：

```c
struct proc_struct {
    enum proc_state state;
    int pid;
    int runs;
    uintptr_t kstack;
    volatile bool need_resched;
    struct proc_struct *parent;
    struct mm_struct *mm;
    struct context context;
    struct trapframe *tf;
    uintptr_t cr3;
    uint32_t flags;
    char name[PROC_NAME_LEN + 1];
    list_entry_t list_link;
    list_entry_t hash_link;
};
```

解释一下其中的一些变量：

- kstack: 线程对应的内核栈物理地址，对于内核线程该栈就是线程运行栈，对于用户线程该栈保存特权级改变时的中断信息
- mm: 指向内存管理块，具体功能在前两个实验有描述
- context: 进程的上下文，在进程切换时需要保存/恢复上下文
- tf: 指向内核栈中的中断帧的指针，用于特权级改变时记录中断信息
- cr3: 页表的物理地址，在进程切换时直接使用该值切换页表
- list_link: 将进程控制块组织成双向链表以进行管理
- hash_link: 将进程控制块组织成哈希表用于通过pid快速查找进程

---

在proc.h文件中另有两个更进一步的数据结构proc_state和context

```c
enum proc_state {
    PROC_UNINIT = 0,  // uninitialized
    PROC_SLEEPING,    // sleeping
    PROC_RUNNABLE,    // runnable(maybe running)
    PROC_ZOMBIE,      // almost dead, and wait parent proc to reclaim his resource
};
```

该结构列出了进程的4种状态用于进程控制

```c
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

该结构保存进程的上下文，里面保存一些寄存器的值用于上下文切换

### 练习1：分配并初始化一个进程控制块

该练习需要完成`alloc_proc()`函数，该函数用于分配并初始化一个进程控制块，下面是实现

```c
// alloc_proc - alloc a proc_struct and init all fields of proc_struct
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    //LAB4:EXERCISE1 17343025
    /*
     * below fields in proc_struct need to be initialized
     *       enum proc_state state;                      // Process state
     *       int pid;                                    // Process ID
     *       int runs;                                   // the running times of Proces
     *       uintptr_t kstack;                           // Process kernel stack
     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
     *       struct proc_struct *parent;                 // the parent process
     *       struct mm_struct *mm;                       // Process's memory management field
     *       struct context context;                     // Switch here to run process
     *       struct trapframe *tf;                       // Trap frame for current interrupt
     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
     *       uint32_t flags;                             // Process flag
     *       char name[PROC_NAME_LEN + 1];               // Process name
     */
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&proc->context, 0, sizeof(struct context));
        proc->tf = NULL;
        proc->cr3 = boot_cr3;
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN + 1);
    }
    return proc;
}
```

首先需要申请一块内存空间作为进程控制块，如果空间申请失败自然进程无法创建，若空间能够申请成功，需要对其成员进行初始化，其中进程状态为`PROC_UNINIT`表示未初始化，pid为-1表示也表示未初始化状态，cr3赋值为内核页目录的地址，其它的成员变量全部赋值为0，完成初始化后返回该进程控制块的指针

---

回答问题：**请说明proc_struct中struct context context和struct trapframe \*tf成员变量含义和在本实验中的作用是啥？**

两者都是在用于保存和恢复中断现场的，其中context是通用寄存器的状态，而tf指向的中断帧会记录该进程使用的其它寄存器比如段寄存器的状态，两者在进程切换和特权级切换过程中是不可或缺的

### 练习2：为新创建的内核线程分配资源

### 练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的

## 实验总结和对比

### 对比答案说明差异


### 本实验中重要的知识点


### OS原理中很重要但实验中没有对应的知识点


### 总结


## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)
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

首先需要申请一块内存空间作为进程控制块，如果空间申请失败自然进程无法创建，返回NULL，若空间能够申请成功，需要对其成员进行初始化，其中进程状态为`PROC_UNINIT`表示未初始化，pid为-1表示也表示未初始化状态，cr3赋值为内核页目录的地址，其它的成员变量全部赋值为0，完成初始化后返回该进程控制块的指针

---

回答问题：**请说明proc_struct中struct context context和struct trapframe \*tf成员变量含义和在本实验中的作用是啥？**

两者都是在用于保存和恢复中断现场的，其中context是通用寄存器的状态，而tf指向的中断帧会记录该进程使用的其它寄存器比如段寄存器的状态，两者在进程切换和特权级切换过程中是不可或缺的

### 练习2：为新创建的内核线程分配资源

`kernel_thread()`函数用于创建内核线程，该函数主要为新创建的线程建立中断栈，具体如下：

```c
int
kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    tf.tf_cs = KERNEL_CS;
    tf.tf_ds = tf.tf_es = tf.tf_ss = KERNEL_DS;
    tf.tf_regs.reg_ebx = (uint32_t)fn;
    tf.tf_regs.reg_edx = (uint32_t)arg;
    tf.tf_eip = (uint32_t)kernel_thread_entry;
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
```

首先是段寄存器的设置，由于新建的是内核线程这些寄存器的设置是统一的常量，接下来设置线程的入口函数和参数，然后将eip寄存器设置为统一的`kernel_thread_entry`入口处，最后调用`do_fork()`函数完成具体的线程创建工作

再看`kernel_thread_entry`入口，这里定义在entry.S文件中，里面只有很简单的内容，将函数参数指针压栈后调用线程入口函数，函数退出后调用`do_exit()`函数结束线程

```x86asm
kernel_thread_entry:        # void kernel_thread(void)

    pushl %edx              # push arg
    call *%ebx              # call fn

    pushl %eax              # save the return value of fn(arg)
    call do_exit            # call do_exit to terminate current thread
```

再看回重点的`do_fork()`函数，传入参数为控制`flags`、父线程的用户栈地址`stack`和`kernel_thread()`函数中设置的中断帧的指针，如果`stack`为0表示创建的是内核线程

```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
    // ...
    proc= alloc_proc();
    if (proc == NULL)
        goto fork_out;
    ret = setup_kstack(proc);
    if (ret != 0)
        goto bad_fork_cleanup_proc;
    ret = copy_mm(clone_flags, proc);
    if (ret != 0)
        goto bad_fork_cleanup_kstack;
    copy_thread(proc, stack, tf);
    proc->pid = get_pid();
    hash_proc(proc);
    list_add(&proc_list, &proc->list_link);
    nr_process++;
    wakeup_proc(proc);
    ret = proc->pid;

fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

该函数首先检查现有线程是否大于或等于系统上限，若是则创建失败，返回`-E_NO_FREE_PROC`错误码，否则进行更进一步的线程创建过程

接着调用`alloc_proc()`函数申请并初始化一块进程控制块，若返回值为NULL说明内存空间不足，返回`-E_NO_MEM`错误码，否则继续流程

然后调用`setup_kstack()`函数为新创建的线程设置内核线程栈，查看该函数定义如下：

```c
// setup_kstack - alloc pages with size KSTACKPAGE as process kernel stack
static int
setup_kstack(struct proc_struct *proc) {
    struct Page *page = alloc_pages(KSTACKPAGE);
    if (page != NULL) {
        proc->kstack = (uintptr_t)page2kva(page);
        return 0;
    }
    return -E_NO_MEM;
}
```

可以看到该函数申请一块内存空间作为内核栈，若成功则返回0，否则返回`-E_NO_MEM`错误码，再回到`do_fork()`函数，在调用`setup_kstack()`函数后我们检查其返回值，若不是0说明出错，跳转到`bad_fork_cleanup_proc`处释放进程控制块后退出函数，若是0则继续下面的流程

然后再调用`copy_mm()`函数，其作用是根据`clone_flags`的值决定是复制还是共享父进程的内存管理块`mm`，同样是成功时返回0，检查返回值，若不是0说明出错，由于上面分配内核栈已成功，这次需要跳转到`bad_fork_cleanup_kstack`处进行更进一步的资源释放然后再退出函数，否则继续下面的流程

这一步之后相关资源分配完成，调用`copy_thread()`函数设置新线程的中断帧、入口地址和栈，再调用`get_pid()`函数获取一个唯一的pid赋值到进程控制块的`pid`变量中，将该进程控制块加入链表和哈希表中，再把现存进程计数器加1，最后调用`wakeup_proc()`函数将新建的线程设置为`PROC_RUNNABLE`状态，该线程的创建工作完成可以运行了，`do_fork()`函数返回新建线程的pid即可

---

回答问题：**请说明ucore是否做到给每个新fork的线程一个唯一的id？**

这个问题涉及到`get_pid()`函数，仔细了解其运行原理可知，在分配pid时会遍历进程控制块列表来得到已被分配的pid，找到一个未被分配的pid返回

### 练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的

下面是`proc_run()`函数的实现：

```c
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```

该函数的作用是进行进程的切换，首先要判断将要运行的进程是否是当前进程，若是则不用切换，若不是则进行进程切换，首先调用`local_intr_save()`函数保存当前的中断状态标志，然后将当前进程控制块指针`current`赋值为将要运行的进程对应的进程控制块，接着调用`load_esp0()`函数加载新进程的内核栈地址到任务状态段中，然后调用`lcre()`函数将新进程的页目录基址加载到cr3页目录基址寄存器完成页表的切换，最后调用`switch_to()`函数切换进程的上下文，这些工作完成后再调用`local_intr_restore()`函数恢复中断状态标志

---

回答问题：

- **在本实验的执行过程中，创建且运行了几个内核线程？**
- **语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?**



## 实验总结和对比

### 对比答案说明差异


### 本实验中重要的知识点


### OS原理中很重要但实验中没有对应的知识点


### 总结


## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

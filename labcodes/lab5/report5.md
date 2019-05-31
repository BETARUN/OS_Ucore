# 实验六：用户进程管理

17343025 冯浚轩 软件工程1班

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 了解第一个用户进程创建过程
- 了解系统调用框架的实现机制
- 了解ucore如何实现系统调用sys_fork/sys_exec/sys_exit/sys_wait来进行进程管理

## 实验要求

实验4完成了内核线程，但到目前为止，所有的运行都在内核态执行。实验5将创建用户进程，让用户进程在用户态执行，且在需要ucore支持时，可通过系统调用来让ucore提供服务。为此需要构造出第一个用户进程，并通过系统调用sys_fork/sys_exec/sys_exit/sys_wait来支持运行不同的应用程序，完成对用户进程的执行过程的基本管理

## 实验方案

接着填写已有实验，并完成练习，注意对前面的实验可能需要进行修改来让内核正常工作

## 实验过程

### 练习1：加载应用程序并执行

`do_execve()`函数用于加载应用程序，该函数调用`exit_mmap()`函数和`put_pgdir()`函数回收当前进程的内存资源，然后调用`load_icode()`来加载并解析一个处于内存中的ELF执行文件格式的应用程序，改函数的主要工作是分配应用程序需要的内存空间

1. 分配内存管理块`mm_struct`
2. 分配页目录和页表
3. 解析ELF文件得到各个段的信息，为各个段分配内存，拷贝ELF文件中各个段的内容到新分配的内存中
4. 建立用户栈
5. 将进程控制块中关于内存管理块`mm_struct`和页目录基址`pgdir`的项设定为新进程的值，完成内存管理的切换
6. 设置中断帧为新进程的相关内容，当前进程加载成为新进程后需要从特权模式切换回用户模式，在这里需要正确设置中断帧让中断恢复正常进行

最后将进程控制块中的进程名设为新进程的名称，当前进程就完全被替换为新进程了，可以等待调度切换

其中我需要完成对中断帧的设置，具体如下：

```c
tf->tf_cs = USER_CS;
tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
tf->tf_esp = USTACKTOP;
tf->tf_eip = elf->e_entry;
tf->tf_eflags = FL_IF;
```

在中断帧中设定各个段寄存器用于段，设置栈寄存器`esp`为用户栈栈顶，设置指令寄存器`eip`为新加载应用程序的入口地址，设置状态寄存器`eflags`让CPU在用户态下可以接收中断

---

回答问题：**当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的**

查看`do_execve()`函数的调用链条如下：`SYS_exec`系统调用 --> `sys_exer()`函数 --> `do_execve()`函数

`do_execve()`函数结束后，新的应用程序已经被加载，需要从系统调用返回，由于我们设置好了中断帧，在从系统调用返回后会从内核态切换到用户态，并在新的应用程序的入口处开始执行

### 练习2：父进程复制自己的内存空间给子进程

`do_fork()`函数调用`copy_mm()` --> `dup_mmap()` --> `copy_range()`函数拷贝父进程的有效内存到子进程空间，在该练习中需要完成`copy_range()`函数，该函数如下：

```c
int
copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
    assert(start % PGSIZE == 0 && end % PGSIZE == 0);
    assert(USER_ACCESS(start, end));
    // copy content by page unit.
    do {
        //call get_pte to find process A's pte according to the addr start
        pte_t *ptep = get_pte(from, start, 0), *nptep;
        if (ptep == NULL) {
            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
            continue ;
        }
        //call get_pte to find process B's pte according to the addr start. If pte is NULL, just alloc a PT
        if (*ptep & PTE_P) {
            if ((nptep = get_pte(to, start, 1)) == NULL) {
                return -E_NO_MEM;
            }
        uint32_t perm = (*ptep & PTE_USER);
        //get page from ptep
        struct Page *page = pte2page(*ptep);
        // alloc a page for process B
        struct Page *npage=alloc_page();
        assert(page!=NULL);
        assert(npage!=NULL);
        int ret=0;

        void* src_kvaddr = page2kva(page);
        void* dst_kvaddr = page2kva(npage);
        memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
        ret = page_insert(to, npage, start, perm);
        assert(ret == 0);
        }
        start += PGSIZE;
    } while (start != 0 && start < end);
    return 0;
}
```

该函数传入两个进程的页目录基址、需要复制的一块内存的线性地址的起止点和状态码，以页大小为间隔遍历线性地址范围

在每一步中，首先根据线性地址调用`get_pte()`函数取到父进程的页表项，这里不允许分配新空间，那么当返回`NULL`时表明当前页目录项没有对应页表，调整`start`到下一个页目录项对应的线性地址；再调用`get_pte()`函数找到子进程的页表项，此处再根据页表项找到两者对应的页管理块，在这里允许分配新空间让子进程的页表空间得以建立

再由这两个页管理块调用`page2kva()`函数得到两个页起始的虚拟地址，调用`memcpy()`函数复制内容，最后再调用`page_insert()`函数将新分配的页插入到子进程的页表中

---

回答问题：**如何设计实现“Copy on Write"机制”**

首先操作系统需要得知某个进程要修改共享空间，这需要通过缺页中断实现，进一步，我们需要在fork新进程时把父进程和子进程的页表项全部置为只读，这样在任何一方试图修改时都能通知操作系统

但是这样就不能区分原本内存空间的读写态了，不过在虚拟内存控制块中仍然保留了原始的状态位，使用线性地址找到相应的虚拟内存页控制块，对比其跟页表项的状态位就能判断修改请求是否合法，若该请求合法就申请一块新的内存空间将原先内容拷贝过去，修改要写该空间的进程的页表指向新的内存空间，这样就完成了写时复制，而对进程本身这是不可见的

这里还涉及到共享分页对多个线程的数据一致性问题，“Copy on Write”机制本质上是进程共享，需要有足够的数据结构支持多个进程共享内存页，比如说在共享页被换入换出时，所有的关联进程的页表都需要修改，这是虚拟内存管理实现的完整性问题

### 练习3：阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现

#### 系统调用的实现

通过软中断指令陷入到0x80号中断实现系统调用，在unistd.h文件中定义了不同系统调用功能对应的系统调用号，先前在`idt_init()`函数初始化中断时定义了系统调用的入口

```c
SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
```

这让用户态的程序得以通过中断进入内核态，是系统调用的前提

再查找中断解析函数得到系统调用的处理方法

```c
case T_SYSCALL:
    syscall();
    break;
```

在syscall.c文件中找到上面的函数对应的内容

```c
static int (*syscalls[])(uint32_t arg[]) = {
    [SYS_exit]              sys_exit,
    [SYS_fork]              sys_fork,
    [SYS_wait]              sys_wait,
    [SYS_exec]              sys_exec,
    [SYS_yield]             sys_yield,
    [SYS_kill]              sys_kill,
    [SYS_getpid]            sys_getpid,
    [SYS_putc]              sys_putc,
    [SYS_pgdir]             sys_pgdir,
};

#define NUM_SYSCALLS        ((sizeof(syscalls)) / (sizeof(syscalls[0])))

void
syscall(void) {
    struct trapframe *tf = current->tf;
    uint32_t arg[5];
    int num = tf->tf_regs.reg_eax;
    if (num >= 0 && num < NUM_SYSCALLS) {
        if (syscalls[num] != NULL) {
            arg[0] = tf->tf_regs.reg_edx;
            arg[1] = tf->tf_regs.reg_ecx;
            arg[2] = tf->tf_regs.reg_ebx;
            arg[3] = tf->tf_regs.reg_edi;
            arg[4] = tf->tf_regs.reg_esi;
            tf->tf_regs.reg_eax = syscalls[num](arg);
            return ;
        }
    }
    print_trapframe(tf);
    panic("undefined syscall %d, pid = %d, name = %s.\n",
            num, current->pid, current->name);
}
```

`syscall()`函数读取中断帧中保存的寄存器值得到应用程序想要实现的调用，其中`eax`寄存器指定系统调用号，另外五个通用寄存器传递系统调用的参数，上面还维护了一个系统调用功能函数指针表，通过系统调用号索引该表，将相应系统调用功能的实现交由对应函数，函数的返回值保存在中断帧的`eax`寄存器随中断恢复返回给应用程序，至此系统调用实现完成

#### fork的实现

`syscall()`函数调用`sys_fork()`函数进行进程的复制

```c
static int
sys_fork(uint32_t arg[]) {
    struct trapframe *tf = current->tf;
    uintptr_t stack = tf->tf_esp;
    return do_fork(0, stack, tf);
}
```

该函数主要是调用`do_fork()`函数完成进程复制，在上一个实验中已经详细讨论过这个函数，但是在这里会有一点修改

首先是在申请进程控制块之后，此处需要让新进程的父进程指针指向其父进程，并且确保父进程的等待态为空

```c
proc->parent = current;
if(current->wait_state != 0)
    goto bad_fork_cleanup_proc;
```

然后将之前的把新进程控制块加入链表的函数替换为`set_links()`函数，该函数在之前的加入链表功能的基础上加入了设置各进程控制块之间关系指针的功能，以维护各进程的相互关系

此外，上述修改都涉及到进程控制块中新添加的变量，需要修改`alloc_proc()`函数完成对它们的初始化

```c
proc->wait_state = 0;
proc->cptr = proc->yptr = proc->optr = NULL;
```

至此`do_fork()`函数的实现完成，调用该函数后进程得到复制

#### exec的实现

`syscall()`函数调用`sys_exec()`函数进行加载新应用程序

```c
static int
sys_exec(uint32_t arg[]) {
    const char *name = (const char *)arg[0];
    size_t len = (size_t)arg[1];
    unsigned char *binary = (unsigned char *)arg[2];
    size_t size = (size_t)arg[3];
    return do_execve(name, len, binary, size);
}
```

其功能是从传入参数获取进程名和ELF文件的内存地址传参调用`do_execve()`函数，接下来该函数按照上面练习1中的介绍加载应用程序

#### wait的实现

`syscall()`函数调用`sys_wait()`函数使父进程等待子进程退出，并回收该子进程的进程控制块资源，主要是获取`pid`和子进程退出码的地址，传参调用`do_wait()`函数

```c
static int
sys_wait(uint32_t arg[]) {
    int pid = (int)arg[0];
    int *store = (int *)arg[1];
    return do_wait(pid, store);
}
```

下面是`do_wait()`函数的具体实现

```c
int
do_wait(int pid, int *code_store) {
    struct mm_struct *mm = current->mm;
    if (code_store != NULL) {
        if (!user_mem_check(mm, (uintptr_t)code_store, sizeof(int), 1)) {
            return -E_INVAL;
        }
    }

    struct proc_struct *proc;
    bool intr_flag, haskid;
repeat:
    haskid = 0;
    if (pid != 0) {
        proc = find_proc(pid);
        if (proc != NULL && proc->parent == current) {
            haskid = 1;
            if (proc->state == PROC_ZOMBIE) {
                goto found;
            }
        }
    }
    else {
        proc = current->cptr;
        for (; proc != NULL; proc = proc->optr) {
            haskid = 1;
            if (proc->state == PROC_ZOMBIE) {
                goto found;
            }
        }
    }
    if (haskid) {
        current->state = PROC_SLEEPING;
        current->wait_state = WT_CHILD;
        schedule();
        if (current->flags & PF_EXITING) {
            do_exit(-E_KILLED);
        }
        goto repeat;
    }
    return -E_BAD_PROC;

found:
    if (proc == idleproc || proc == initproc) {
        panic("wait idleproc or initproc.\n");
    }
    if (code_store != NULL) {
        *code_store = proc->exit_code;
    }
    local_intr_save(intr_flag);
    {
        unhash_proc(proc);
        remove_links(proc);
    }
    local_intr_restore(intr_flag);
    put_kstack(proc);
    kfree(proc);
    return 0;
}
```

首先检查传入的子进程返回地址`code_store`是否合法，然后使用传入的`pid`找到要等待的子进程的进程控制块，若传入`pid`为0则选取父进程的子进程列表中的第一个，若能取到子进程，将父进程设置为`PROC_SLEEPING`和`WT_CHILD`状态表示其等待子进程，然后调用`schedule()`函数执行进程调度让出父进程CPU

当子进程退出后唤醒父进程，重新取到子进程的控制块，确认子进程为`PROC_ZOMBIE`状态后，首先将子进程返回码复制到传入地址中，然后回收子进程的进程控制块和内核栈资源，子进程完全结束，之后返回到系统调用点

#### exit的实现

`syscall()`函数调用`sys_exit()`函数使当前进程退出，其功能是从参数中取到错误码传参调用`do_exit()`函数

```c
static int
sys_exit(uint32_t arg[]) {
    int error_code = (int)arg[0];
    return do_exit(error_code);
}
```

下面是`do_exit()`函数的实现

```c
int
do_exit(int error_code) {
    if (current == idleproc) {
        panic("idleproc exit.\n");
    }
    if (current == initproc) {
        panic("initproc exit.\n");
    }
    
    struct mm_struct *mm = current->mm;
    if (mm != NULL) {
        lcr3(boot_cr3);
        if (mm_count_dec(mm) == 0) {
            exit_mmap(mm);
            put_pgdir(mm);
            mm_destroy(mm);
        }
        current->mm = NULL;
    }
    current->state = PROC_ZOMBIE;
    current->exit_code = error_code;
    
    bool intr_flag;
    struct proc_struct *proc;
    local_intr_save(intr_flag);
    {
        proc = current->parent;
        if (proc->wait_state == WT_CHILD) {
            wakeup_proc(proc);
        }
        while (current->cptr != NULL) {
            proc = current->cptr;
            current->cptr = proc->optr;
    
            proc->yptr = NULL;
            if ((proc->optr = initproc->cptr) != NULL) {
                initproc->cptr->yptr = proc;
            }
            proc->parent = initproc;
            initproc->cptr = proc;
            if (proc->state == PROC_ZOMBIE) {
                if (initproc->wait_state == WT_CHILD) {
                    wakeup_proc(initproc);
                }
            }
        }
    }
    local_intr_restore(intr_flag);
    
    schedule();
    panic("do_exit will not return!! %d.\n", current->pid);
}
```

首先获取当前进程的内存控制块，只有用户进程才能取到内存控制块，将cr3寄存器加载为内核页目录，然后回收当前进程的内存空间、页表和内存控制块，然后设置当前进程的状态为`PROC_ZOMBIE`并设置返回码，子进程退出等待父进程回收进程控制块

再找到父进程控制块，若其处于`WT_CHILD`状态，那么调用`wakeup_proc()`函数唤醒父进程使其等待被调度后回收子进程的剩余资源

若子进程还存在子进程，需要将其父进程设置为`initproc`进程，若该进程也处于退出状态，唤醒`initproc`进程让其完成资源回收

到此进程退出设置完成，最后调用`schedule()`函数调度进程

---

回答问题：

- **请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？**
- **请给出ucore中一个用户态进程的执行状态生命周期图**

fork和exec都会创建进程，进程经由初始态进入可运行状态；wait将阻塞父进程，父进程被设置为等待状态；exit回收进程的大部分资源，进程转入退出态，并且唤醒父进程使父进程转回可运行态，等待父进程完成最后的资源回收，进程结束

下面是用户态进程的状态周期图：

```text
PROC_UNINIT --> PROC_RUNNABLE --> PROC_ZOMBIE --> exit
                  ^      |
                  |      V
                PROC_SLEEPING
```

### 实验结果

在当前目录下运行**make grade**可得到各用户程序的测试结果，该实验能够通过测试

## 实验总结和对比

### 对比答案说明差异


### 本实验中重要的知识点


### OS原理中很重要但实验中没有对应的知识点

### 总结


## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

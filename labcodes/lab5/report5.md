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



---

回答问题：

- **请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？**
- **请给出ucore中一个用户态进程的执行状态生命周期图**



### 实验结果


## 实验总结和对比

### 对比答案说明差异


### 本实验中重要的知识点


### OS原理中很重要但实验中没有对应的知识点

### 总结


## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

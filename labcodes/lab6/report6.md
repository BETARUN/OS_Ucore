# 实验七：调度器

17343025 冯浚轩 软件工程1班

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 理解操作系统的调度管理机制
- 熟悉 ucore 的系统调度器框架，以及缺省的Round-Robin 调度算法
- 基于调度器框架实现一个(Stride Scheduling)调度算法来替换缺省的调度算法

## 实验要求

实验六完成了用户进程的管理，可在用户态运行多个进程。但到目前为止，采用的调度策略是很简单的FIFO调度策略。本次实验，主要是熟悉ucore的系统调度器框架，以及基于此框架的Round-Robin（RR） 调度算法。然后参考RR调度算法的实现，完成Stride Scheduling调度算法。

## 实验方案

接着填写已有实验，并完成练习

## 实验过程

### 练习1：使用 Round Robin 调度算法

在内核初始化时调用`sched_init()`函数进行进程调度初始化

```c
void
sched_init(void) {
    list_init(&timer_list);

    sched_class = &default_sched_class;

    rq = &__rq;
    rq->max_time_slice = MAX_TIME_SLICE;
    sched_class->init(rq);

    cprintf("sched class: %s\n", sched_class->name);
}
```

该函数主要是初始化调度器框架`sched_class`和进程队列`rq`，此时RR调度器被引入框架中，并设置进程队列的最大时间片长度

为了实现进程队列和时间片控制，需要修改`proc_struct`结构体，在其中新增以下成员

```c
struct run_queue *rq;                       // running queue contains Process
list_entry_t run_link;                      // the entry linked in run queue
int time_slice;                             // time slice for occupying the CPU
```

其中`rq`指向进程队列，`run_link`用于将各个进程控制块连接成双向链表，`time_slice`表示当前进程剩余的时间片大小

在创建进程的时候，需要对上述成员变量进行初始化

```c
proc->rq = NULL;
list_init(&proc->run_link);
proc->time_slice = 0;
```

在每个时钟中断时调用`sched_class_proc_tick()`函数，在RR算法下相关函数定义如下：

```c
void
sched_class_proc_tick(struct proc_struct *proc) {
    if (proc != idleproc) {
        sched_class->proc_tick(rq, proc);
    }
    else {
        proc->need_resched = 1;
    }
}

static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```

其中`sched_class_proc_tick()`函数会判断当前进程是否是`idleproc`，若不是则调用`RR_proc_tick()`函数，该函数的作用是将当前进程的时间片计数减1，若该计数已为零则将`need_resched`设置为1表示该进程需要被调度

在中断结束时会检查当前进程的`need_resched`变量是否为1，若是则调用`schedule()`函数进行进程调度

```c
if (!in_kernel) {
    if (current->flags & PF_EXITING) {
        do_exit(-E_KILLED);
    }
    if (current->need_resched) {
        schedule();
    }
}
```

接下来看`schedule()`函数，该函数在之前版本的基础上进行扩充，

```c
void
schedule(void) {
    bool intr_flag;
    struct proc_struct *next;
    local_intr_save(intr_flag);
    {
        current->need_resched = 0;
        if (current->state == PROC_RUNNABLE) {
            sched_class_enqueue(current);
        }
        if ((next = sched_class_pick_next()) != NULL) {
            sched_class_dequeue(next);
        }
        if (next == NULL) {
            next = idleproc;
        }
        next->runs ++;
        if (next != current) {
            proc_run(next);
        }
    }
    local_intr_restore(intr_flag);
}
```

首先需要重置当前进程的`need_resched`变量为0，然后判断当前进程是否是运行态，若是则将其放入进程队列末尾，然后取进程队列的队头，若队头有效则将其从队列中删去，该进程成为将要运行进程，调用`proc_run()`函数进行进程切换使其运行，若没有可运行进程时最终会运行`idleproc`进程等待新进程被创建和调度

由此看出RR调度算法通过一个队列维护各可运行进程，通过取队头——放入队尾的循环让各进程共同使用CPU时间

---

#### 请理解并分析sched_class中各个函数指针的用法，并结合Round Robin 调度算法描述ucore的调度执行过程

查看`sched_class`的定义如下：

```c
struct sched_class {
    // the name of sched_class
    const char *name;
    // Init the run queue
    void (*init)(struct run_queue *rq);
    // put the proc into runqueue, and this function must be called with rq_lock
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // get the proc out runqueue, and this function must be called with rq_lock
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // choose the next runnable task
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    // dealer of the time-tick
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```

各个可运行进程通过进程队列组织起来，在上面的调度器框架中定义了对这个队列的插入、删除和取节点操作，对于不同的调度器来说对该队列的操作也有不同，因此这些函数通过函数指针定义成为接口供各个不同调度器实现，最后在该框架内还有一个`proc_tick()`函数用于处理时钟中断，以实现进程抢占式调度

在这里`RR_enqueue()`函数需要特别注意

```c
static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
}
```

在这里会看到对于剩余时间片已为0或大于队列上限的进程，需要重置该进程的剩余时间片为队列上限，这样的操作可以让时间片用完而被抢占的进程在入队后重新获取时间片，而由于I/O或主动让出CPU的进程在入队时是不会改变剩余时间片的数值的

#### 简要说明如何设计实现“多级反馈队列调度算法”

在上面的RR调度器算法中是按照完全的FIFO队列进行处理的，各个进程的时间片也是相同的，在这里可以通过给不同进程设置不同的时间片长度并调整各个进程在队列中的位置来实现多级反馈队列调度算法

### 练习2：实现 Stride Scheduling 调度算法

为了实现这个调度算法，需要再在`proc_struct`结构体中增加成员变量

```c
skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process 
uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
```

其中`lab6_run_pool`用于连接各进程控制块形成堆以实现高效的优先队列，`lab6_stride`变量记录了进程的调度权值，在每次进行调度时选择该值最小的进程，`lab6_priority`变量表示该进程的调度优先级，值越大的进程分配的时间越多

这些变量同样需要在进程创建时初始化

```c
skew_heap_init(&proc->lab6_run_pool);
proc->lab6_stride = 0;
proc->lab6_priority = 0;
```

在每次调度时，进行`lab6_stride += BIG_STRIDE / lab6_priority`运算，这样对于优先级越大的进程其权值的增量就越小，对各个进程的权值作差得到大小关系形成优先队列，取权值最小的作为将要运行的进程

这里会出现`lab6_stride`变量的溢出问题，在这里权值和优先级都是无符号整数，而两者作差时当作有符号整数处理，这样就能通过判断差与0的关系得到大小关系，要使溢出后结果正负号不变，权值的增量必须有一个上限，由于优先级大于或等于1，该上限即为`BIG_STRIDE`，经过计算该上限是权值和优先级的当前位数所能表示的最大整数，在这里是32位的整数，故`BIG_STRIDE`可取2147483647

接下来需要按照新的调度算法实现调度器框架中的各个函数

首先是`stride_proc_tick()`函数，与RR算法的`proc_tick()`函数内容完全一致，都是在响应时钟中断时减少当前进程时间片直到为0触发调度

然后是`stride_init()`函数，该函数初始化进程队列，只需要赋零值即可

```c
static void
stride_init(struct run_queue *rq) {
     list_init(&rq->run_list);
     rq->lab6_run_pool = NULL;
     rq->proc_num = 0;
}
```

接着是`stride_enqueue()`函数，这个函数将进程添加到进程优先队列中，对于时间片归零而被抢占调度的进程来说需要重置时间片大小

```c
static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
     rq->lab6_run_pool = 
          skew_heap_insert(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
     proc->rq = rq;
     rq->proc_num++;
}
```

然后是`stride_dequeue()`函数，这个函数将移除队头元素，用于将离开可运行状态的进程移出等待队列

```c
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     rq->lab6_run_pool = 
          skew_heap_remove(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
     rq->proc_num--;
}
```

最后是`stride_pick_next()`函数，用于取队头的进程查看，若当前队列中没有进程需要返回`NULL`，否则取到该进程并增加该进程的权值，注意在刚初始化完毕时该进程的优先级被设置为0，需要单独处理以避免除零错误

```c
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     skew_heap_entry_t* he = rq->lab6_run_pool;
     if (he == NULL)
          return NULL;
     struct proc_struct* proc = le2proc(he, lab6_run_pool);
     uint32_t step;
     if (proc->lab6_priority == 0)
          step = BIG_STRIDE;
     else
          step = BIG_STRIDE / proc->lab6_priority;
     proc->lab6_stride += step;
     return proc;
}
```

至此该调度算法实现完成，使用`make grade`命令检查也能通过测试

### 练习3：结合中断处理和调度程序，再次理解进程控制块中的trapframe和context在进程切换时作用

对于用户态进程来说，发生进程调度时会产生内核态与用户态之间的切换，这需要通过`trapframe`中断帧保存中断现场信息，在产生中断时寄存器值保存到中断帧中，在切换到将要运行的进程从中断恢复时将寄存器值恢复回该进程被中断前的状态，保证进程切换不影响进程的执行过程

对于内核态进程之间的切换，由于内核态进程共享内核空间，进行进程切换时不存在中断现场的保存和恢复过程，因此在进程控制块中设置`context`结构体保存进程的上下文，使得在内核态中也能完成上下文的切换

在进程切换时调用`proc_run()`函数切换到新进程，对于内核进程来说在`switch_to()`函数中完成上下文切换，而对于用户进程来说这一步是多余的，在退出中断时寄存器上下文会被中断帧中保存的内容覆盖，确保返回时进程的执行是正确的

## 实验总结和对比

### 对比答案说明差异

对比答案无明显差异

### 本实验中重要的知识点

- Round Robin 调度算法的实现
- Stride Scheduling 优先级调度算法的实现
- 进程切换点
- 进程切换的过程

### OS原理中很重要但实验中没有对应的知识点

大概没有吧

### 总结

这个实验讨论了调度器的实现细节，需要通过它给出的 Round Robin 调度算法了解调度器的运作原理，其中进程被强制剥夺CPU使用权的时机是值得注意的，这种强制夺取CPU是操作系统得以介入并调度进程的基础

为了实现调度器，需要有相适应的数据结构，在这里定义了进程的等待队列，在进程控制块中也要新增成员变量表示该进程的时间片，在进行调度时主要通过等待队列和时间片之间的管理进行进程切换

接着该实验要求实现 Stride Scheduling 调度算法，这个算法可以实现按照进程优先级对各个进程分配不同的时间，在这里一个很重要的问题是进程权值累加溢出的问题，需要仔细思考`BIG_STRIDE`的正确取值

## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

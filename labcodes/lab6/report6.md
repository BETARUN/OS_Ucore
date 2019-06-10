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


### 练习3：结合中断处理和调度程序，再次理解进程控制块中的trapframe和context在进程切换时作用


### 实验结果


## 实验总结和对比

### 对比答案说明差异


### 本实验中重要的知识点

### OS原理中很重要但实验中没有对应的知识点


### 总结


## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

# 实验八：同步互斥

17343025 冯浚轩 软件工程1班

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 理解操作系统的同步互斥的设计实现；
- 理解底层支撑技术：禁用中断、定时器、等待队列；
- 在ucore中理解信号量（semaphore）机制的具体实现；
- 了解经典进程同步问题，并能使用同步机制解决进程同步问题

## 实验要求

实验7（lab6）完成了用户进程的调度框架和具体的调度算法，可调度运行多个进程。如果多个进程需要协同操作或访问共享资源，则存在如何同步和有序竞争的问题。本次实验，主要是熟悉ucore的进程同步机制—信号量（semaphore）机制，以及基于信号量的哲学家就餐问题解决方案。在本次实验中，在kern/sync/check_sync.c中提供了一个基于信号量的哲学家就餐问题解法

哲学家就餐问题描述如下：有五个哲学家，他们的生活方式是交替地进行思考和进餐。哲学家们公用一张圆桌，周围放有五把椅子，每人坐一把。在圆桌上有五个碗和五根筷子，当一个哲学家思考时，他不与其他人交谈，饥饿时便试图取用其左、右最靠近他的筷子，但他可能一根都拿不到。只有在他拿到两根筷子时，方能进餐，进餐完后，放下筷子又继续思考

## 实验方案

接着填写已有实验，并完成练习

## 实验过程

### 练习1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

#### 内核信号量的实现

在sem.h文件中可以看到信号量的相关定义

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;

void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);
```

可见信号量是一个结构体，其中包含一个整数变量和一个等待队列，在该头文件中还声明了信号量的初始化、P操作和V操作的相关函数

首先来看初始化函数

```c
void
sem_init(semaphore_t *sem, int value) {
    sem->value = value;
    wait_queue_init(&(sem->wait_queue));
}
```

该函数将信号量中的整数变量设定为输入值，并初始化等待队列，可以说是非常简单

---

现在来看P操作的相关函数，这里分为不强制进入临界区的`try_down()`函数以及等待进入临界区的`down()`函数

首先来看`try_down()`函数

```c
bool
try_down(semaphore_t *sem) {
    bool intr_flag, ret = 0;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --, ret = 1;
    }
    local_intr_restore(intr_flag);
    return ret;
}
```

该函数关中断后判断信号量中的`value`是否大于零，若是则表示资源可以使用，减少`value`值表示使用了资源，设置返回值为1，否则返回值为0，然后开中断返回，使用这个函数时需要使用者根据返回值自行决定是否可以进入临界区

接下来看`down()`函数

```c
void
down(semaphore_t *sem) {
    uint32_t flags = __down(sem, WT_KSEM);
    assert(flags == 0);
}

static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}

void
wait_current_set(wait_queue_t *queue, wait_t *wait, uint32_t wait_state) {
    assert(current != NULL);
    wait_init(wait, current);
    current->state = PROC_SLEEPING;
    current->wait_state = wait_state;
    wait_queue_add(queue, wait);
}

```

调用函数后首先关中断，判断`value`是否大于零，若是表示资源可用，开中断返回，否则在当前进程下创建一个`wait_t`结构，将其加入到信号量的等待队列中，设置当前进程的运行状态为阻塞状态，然后开中断并调用`schedule()`函数进行进程调度，在进程被阻塞期间`wait_t`局部变量一直有效，进而使信号量的队列能正常工作，当该进程被唤醒后在关中断状态下从队列中删除当前进程，然后按情况返回相应的值

---

现在看进行V操作的`up()`函数

```c
void
up(semaphore_t *sem) {
    __up(sem, WT_KSEM);
}

static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```

同样需要关中断保证原子性，使用`wait_queue_fist()`函数获取队头的`wait_t`，若不存在则增加`value`值，否则表明有进程在等待，使用`waitup_wait()`函数唤醒与该`wait_t`结构关联的进程，然后开中断并返回

---

至此信号量的初始化、P操作和V操作都实现完成，按照一般的同步互斥问题中信号量的使用方法来使用即可

#### 基于信号量的哲学家就餐问题

在check_sync.c文件中有通过信号量实现的哲学家就餐问题

对每一位哲学家有

```c
int philosopher_using_semaphore(void * arg) /* i：哲学家号码，从0到N-1 */
{
    int i, iter=0;
    i=(int)arg;
    cprintf("I am No.%d philosopher_sema\n",i);
    while(iter++<TIMES)
    { /* 无限循环 */
        cprintf("Iter %d, No.%d philosopher_sema is thinking\n",iter,i); /* 哲学家正在思考 */
        do_sleep(SLEEP_TIME);
        phi_take_forks_sema(i); 
        /* 需要两只叉子，或者阻塞 */
        cprintf("Iter %d, No.%d philosopher_sema is eating\n",iter,i); /* 进餐 */
        do_sleep(SLEEP_TIME);
        phi_put_forks_sema(i); 
        /* 把两把叉子同时放回桌子 */
    }
    cprintf("No.%d philosopher_sema quit\n",i);
    return 0;    
}
```

在无限循环中，每一位哲学家在每一轮循环都思考一段时间，尝试获取两只叉子，阻塞在获取叉子状态直到得到两只叉子，然后用一段时间进食，最后放回两只叉子，此处用`do_sleep()`函数主动睡眠线程

拿叉子的过程涉及到`phi_take_forks_sema()`函数

```c
void phi_take_forks_sema(int i) /* i：哲学家号码从0到N-1 */
{ 
        down(&mutex); /* 进入临界区 */
        state_sema[i]=HUNGRY; /* 记录下哲学家i饥饿的事实 */
        phi_test_sema(i); /* 试图得到两只叉子 */
        up(&mutex); /* 离开临界区 */
        down(&s[i]); /* 如果得不到叉子就阻塞 */
}
```

通过互斥信号量修改哲学家i的状态为饥饿状态，然后使用`phi_test_sema()`函数试图得到两只叉子，该函数如下

```c
void phi_test_sema(i) /* i：哲学家号码从0到N-1 */
{ 
    if(state_sema[i]==HUNGRY&&state_sema[LEFT]!=EATING
            &&state_sema[RIGHT]!=EATING)
    {
        state_sema[i]=EATING;
        up(&s[i]);
    }
}
```

只有在当前哲学家饥饿且左右哲学家都不在吃东西时，当前哲学家可用拿到叉子吃东西，修改状态为进餐状态，对当前哲学家的信号量执行V操作

回到`phi_take_forks_sema()`函数，再对当前哲学家的信号量执行P操作，如果之前已经拿到叉子，此处会继续执行，否则当前哲学家就会被阻塞

---

哲学家就餐完毕后调用`phi_put_forks_sema()`函数放下叉子

```c
void phi_put_forks_sema(int i) /* i：哲学家号码从0到N-1 */
{ 
        down(&mutex); /* 进入临界区 */
        state_sema[i]=THINKING; /* 哲学家进餐结束 */
        phi_test_sema(LEFT); /* 看一下左邻居现在是否能进餐 */
        phi_test_sema(RIGHT); /* 看一下右邻居现在是否能进餐 */
        up(&mutex); /* 离开临界区 */
}
```

首先通过互斥信号量修改当前哲学家的状态为正在思考，然后调用`phi_test_sema()`函数检查左右的哲学家的状态，结合该函数的内容可知，若相邻哲学家被阻塞，此时可能会满足可以获得两只叉子的情况，于是会修改对应哲学家的状态为就餐状态，并使用V过程唤醒该哲学家

#### 用户态信号量的实现方案

在内核信号量的实现中使用了关中断和一些内核函数，这些操作在用户态程序中是无法实现的，对于用户态程序，操作系统可以将上述的几个信号量函数封装成为系统调用供用户态程序使用

## 实验总结和对比

### 本实验中重要的知识点

- 内核信号量的实现
- 使用信号量解决哲学家就餐问题

### OS原理中很重要但实验中没有对应的知识点

在老师的实验要求里面是没有管程的，这个在OS原理中很重要

### 总结

这次实验主要探讨了同步互斥问题，关于信号量，在操作系统原理中也提到过其结构，可以推测出实现方法，在实验中则是给出了一个具体的实现方法，其中重点在于信号量的P操作和V操作的实现，其中涉及到等待队列和线程调度的综合应用，了解其实现内容能让我对操作系统的原理有更加深刻的了解

本次实验的另一个内容是理解根据信号量实现的哲学家就餐问题，这个问题是同步互斥问题中的一个经典问题，在之前的学习中只是提到过而没有更加深入了解，在本次实验中可以了解其实现细节，让我对同步互斥问题有更多认识

## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

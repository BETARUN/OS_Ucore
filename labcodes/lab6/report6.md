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

---

回答问题：**请理解并分析sched_class中各个函数指针的用法，并结合Round Robin 调度算法描述ucore的调度执行过程**


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

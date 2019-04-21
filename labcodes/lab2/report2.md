# 实验三：物理内存管理

17343025 冯浚轩 软件工程

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 理解基于段页式内存地址的转换机制
- 理解页表的建立和使用方法
- 理解物理内存的管理方法

## 实验要求

本次实验包含三个部分

1. 首先了解如何发现系统中的物理内存
2. 然后了解如何建立对物理内存的初步管理，即了解连续物理内存管理
3. 最后了解页表相关的操作，即如何建立页表来实现虚拟内存到物理内存之间的映射，对段页式内存管理机制有一个比较全面的了解

## 实验方案

将LAB1中完成的代码填入LAB2中的对应部分，然后完成相应练习

## 实验过程

### 练习 1：实现 first-fit 连续物理内存分配算法

#### 分析内存管理的初始化过程

首先需要认识一些与内存管理有关的重要数据结构，在memlayout.h文件中都有定义

首先是用于探测物理内存分布和大小的`e820map`

```c
struct e820map {
    int nr_map;
    struct {
        uint64_t addr;
        uint64_t size;
        uint32_t type;
    } __attribute__((packed)) map[E820MAX];
};
```

其中`nr_map`变量表示内存块数，在其之下用一个结构体表示每个内存块，每个内存块有首地址、大小和类型三个属性，其中类型`type`为1时表示内存块可用，为2时表示内存块被保留不可用，在bootloader启动时通过BIOS中断完成物理内存的探测完成上面的物理内存分布表

---

然后是页描述符`Page`，每个`Page`结构体描述一个物理页

```c
struct Page {
    int ref;
    uint32_t flags;
    unsigned int property;
    list_entry_t page_link;
};
```

其中的`ref`变量记录该页被引用的次数，`flags`记录该页的状态信息，后两个属性用于空闲页块的起始元素，其中`property`记录该空闲页块的大小，`page_link`是双向链表的一个节点，用于连接各空闲页块

对于`flags`变量有如下具体定义

```c
/* Flags describing the status of a page frame */
#define PG_reserved                 0
#define PG_property                 1
```

使用了`flags`变量中的两个二进制位，其中`PG_reserved`位指示该页是否由操作系统保留，`PG_property`位表示该页的分配状态，对于页块头部元素，该位为1表示页块空闲，为0表示页块已分配，对于非头部元素该位设定为0



在调用内核启动函数`kern_init()`后，调用`pmm_init()`函数完成内存管理的初始化，进一步查看该函数发现在开始部分有两个主要函数：`init_pmm_manager()`和`page_init()`，前者初始化内存管理器，后者初始化分页管理

查看`init_pmm_manager()`函数有

```c
//init_pmm_manager - initialize a pmm_manager instance
static void
init_pmm_manager(void) {
    pmm_manager = &default_pmm_manager;
    cprintf("memory management: %s\n", pmm_manager->name);
    pmm_manager->init();
}
```

该函数将内存管理器设定为默认内存管理器，在pmm.h文件中定义了内存管理器的结构

```c
struct pmm_manager {
    const char *name;
    void (*init)(void);
    void (*init_memmap)(struct Page *base, size_t n);
    struct Page *(*alloc_pages)(size_t n);
    void (*free_pages)(struct Page *base, size_t n);
    size_t (*nr_free_pages)(void);
    void (*check)(void);
};
```

可见其由名字跟一系列指向相关处理函数的函数指针组成，再在default_pmm.c文件末尾可以找到默认内存管理器的定义

```c
const struct pmm_manager default_pmm_manager = {
    .name = "default_pmm_manager",
    .init = default_init,
    .init_memmap = default_init_memmap,
    .alloc_pages = default_alloc_pages,
    .free_pages = default_free_pages,
    .nr_free_pages = default_nr_free_pages,
    .check = default_check,
};
```

其中的处理函数在该项目中都有实现，通过内存管理器的接口也可以实现自定义的内存管理器

接着`init_pmm_manager()`函数调用内存管理器的`init()`函数，定义如下

```c
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```

在该函数内初始化空闲块链表，开始时没有内存块被管理，空闲空间为0

### 练习2：实现寻找虚拟地址对应的页表项

### 练习3：释放某虚地址所在的页并取消对应二级页表项的映射

## 实验总结

看实验报告要求

## 参考文献

- [实验指导手册](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2/lab2_3_2_2_phymemlab_files.html)
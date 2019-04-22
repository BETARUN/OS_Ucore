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

#### 分析分页内存管理所需的数据结构

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

这里也定义了对`flags`进行操作的宏函数和通过链表元素转换为页描述符的宏函数

最后是用于管理空闲页块的数据结构，由双向链表的占位空表头和空闲块计数组成

```c
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

#### 分析分页内存管理的初始化过程，实现相关函数

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

在该函数内初始化空闲块链表，开始时没有内存块被管理，总空闲块计数为0

接着`pmm_init()`函数再调用`page_init()`函数初始化分页管理，该函数首先解析`e820map`表得到物理内存大小以及需要的分页数，先设置所有页描述符`flags`的保留位

```c
extern char end[];
npage = maxpa / PGSIZE;
pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);

for (i = 0; i < npage; i ++) {
    SetPageReserved(pages + i);
}
```

然后对于内核结束地址`end`所在的页，其下一页开始的非保留空间可以作为空闲空间，需要清空这些页描述符`flgas`的保留位，接下来需要找到这些空闲块调用`init_memmap()`函数

```c
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```

可以看到该函数清空一个块内的的页描述符的各个属性值，再将页块头的`property`变量赋值为块大小，设置`PG_property`位表示当前块空闲可分配，然后将总空闲块计数增加n，最后将该页块头添加到链表中

对所有空闲块调用上述函数后分页管理初始化完毕，接下来调用内存管理器的`check()`函数检查内存分配算法是否正确，这涉及`default_alloc_pages()`函数和`default_free_pages()`函数

---

使用`default_alloc_pages()`函数分配大小为n的空闲页，由于使用first-fit算法，先搜索空闲块链表找到第一个大于n的空闲块

```c
struct Page *page = NULL;
list_entry_t *le = &free_list;
while ((le = list_next(le)) != &free_list) {
    struct Page *p = le2page(le, page_link);
    if (p->property >= n) {
        page = p;
        break;
    }
}
```

若能找到需要的空闲块则`page`被赋值为该空闲块的首地址，接下来将该块从空闲块链表中删去并清空`PG_property`位，若空闲块大于n还需要把剩下的空闲块添加回空闲块链表并设置`PG_property`位，更新空闲块大小，最后将总空闲块计数减去n

```c
if (page != NULL) {
    list_entry_t *prev = list_prev(&(page->page_link));
    list_del(&(page->page_link));
    ClearPageProperty(page);
    if (page->property > n) {
        struct Page *p = page + n;
        p->property = page->property - n;
        list_add(prev, &(p->page_link));
        SetPageProperty(p);
    }
    nr_free -= n;
}
```

最后返回`page`变量，即返回申请到的页的首地址

---

使用`default_free_pages()`函数释放已分配的大小为n的块，对链表中邻接的空闲块还要进行合并

首先要对将要释放的块设置相关属性表示已释放，然后将总空闲块计数加上n

```c
struct Page *p = base;
for (; p != base + n; p ++) {
    assert(!PageReserved(p) && !PageProperty(p));
    p->flags = 0;
    set_page_ref(p, 0);
}
base->property = n;
SetPageProperty(base);
nr_free += n;
```

接下来将该空闲块插入空闲块链表中，需要区分以下情况

1. 若空闲块链表为空，直接将该块插入到占位空表头后方即可

    ```c
    list_entry_t *le1 = list_next(&free_list);
    list_entry_t *le2 = list_prev(&free_list);
    if (le1 == &free_list) {
        list_add(&free_list, &(base->page_link));
    }
    ```

2. 若链表不为空，需要找到该空闲块对应的位置，这通过对比将要插入的块的首地址和链表中的空闲块的首地址实现

    ```c
    struct Page *p1 = le2page(le1, page_link);
    struct Page *p2 = le2page(le2, page_link);
    // check smallest
    if(p1 > base) {
        // ...
    }
    // check biggest
    else if (p2 < base) {
        // ...
    }
    // check middle page
    else {
        // ...
    }
    ```

3. 对于要插入链表中间的情况，首先需要定位到对应位置

    ```c
    p = p1;
    while (p < base) {
        le1 = list_next(le1);
        p = le2page(le1, page_link);
    }
    // p at base next now
    p1 = le2page(list_prev(&p->page_link), page_link);
    p2 = p;
    ```

    然后对链表中邻接的空闲块还要进行合并，分为左右合并、左合并、右合并和不合并4种情况

    ```c
    // merge both
    if (p1 + p1->property == base && base + base->property == p2) {
        p1->property += (base->property + p2->property);
        ClearPageProperty(base);
        ClearPageProperty(p2);
        list_del(&(p2->page_link));
    }
    // merge left
    else if (p1 + p1->property == base) {
        p1->property += base->property;
        ClearPageProperty(base);
    }
    // merge right
    else if (base + base->property == p2) {
        base->property += p2->property;
        ClearPageProperty(p2);
        list_del(&(p2->page_link));
        list_add(&(p1->page_link), &(base->page_link));
    }
    // no merge
    else {
        list_add(&(p1->page_link), &(base->page_link));
    }
    ```

    对于插入到链表头部或尾部的情况，在上述完整过程的基础上简化即可得到

---

编译运行当前的操作系统，截取部分输出结果

```text
memory management: default_pmm_manager
e820map:
  memory: 0009fc00, [00000000, 0009fbff], type = 1.
  memory: 00000400, [0009fc00, 0009ffff], type = 2.
  memory: 00010000, [000f0000, 000fffff], type = 2.
  memory: 07ee0000, [00100000, 07fdffff], type = 1.
  memory: 00020000, [07fe0000, 07ffffff], type = 2.
  memory: 00040000, [fffc0000, ffffffff], type = 2.
check_alloc_page() succeeded!
```

从输出可看出加载了默认内存管理器，解析了`e820map`得到内存布局的相关信息，然后检查分页内存管理，表明该功能成功实现

### 练习2：实现寻找虚拟地址对应的页表项

### 练习3：释放某虚地址所在的页并取消对应二级页表项的映射

## 实验总结

看实验报告要求

## 参考文献

- [实验指导手册](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2/lab2_3_2_2_phymemlab_files.html)
# UCORE综合实验

17343025 冯浚轩 软件工程1班

同步到[GitHub](https://github.com/sky-5462/OS_Ucore)

## 实验目的

- 考察对操作系统的文件系统的设计实现了解；
- 考察操作系统内存管理的虚存技术的掌握；
- 考察操作系统进程调度算法的实现。

## 实验要求

- 在前面ucore实验lab1-lab7的基础上，完成ucore文件系统(参见ucore_os_docs.pdf中的lab8及相关视频)；
- 在上述实验的基础上，修改ucore调度器为采用多级反馈队列调度算法的，队列共设6个优先级（6个队列），最高级的时间片为q，并且每降低1级，其时间片为上一级时间片乘2（参见理论课）；
- 在上述实验的基础上，修改虚拟存储中的页面置换算法为某种工作集页面置换算法，具体如下：
  - 对每一新建进程分配3帧物理页面作为；
  - 当需要页面置换时，选择缺页次数最少的进程中的页面置换到外存；
  - 对进程中的页面置换算法用改进的clock页替换算法。

## 实验过程

### 内容1：完成ucore文件系统

由于本次实验内容较多，这部分就只大概陈述一下要完成编码的部分

#### 练习1：完成读文件操作的实现

这个练习中要求补全`sfs_io_nolock()`函数，函数签名如下：

```c
/*  
 * sfs_io_nolock - Rd/Wr a file contentfrom offset position to offset+ length  disk blocks<-->buffer (in memroy)
 * @sfs:      sfs file system
 * @sin:      sfs inode in memory
 * @buf:      the buffer Rd/Wr
 * @offset:   the offset of file
 * @alenp:    the length need to read (is a pointer). and will RETURN the really Rd/Wr lenght
 * @write:    BOOL, 0 read, 1 write
 */
static int
sfs_io_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, void *buf, off_t offset, size_t *alenp, bool write)
```

其作用是从指定文件中的`offset`位置处开始最多读取或写入`*alenp`个字节，并返回实际读写的字节数保存在`*alenp`中，读取到的或需要写入的数据通过`buf`传递

在函数开始首先是对文件类型和读写位置的合法性进行检查，读写结束点`endpos`可能会根据文件系统最大文件大小限制以及文件本身大小现在做出截尾操作

```c
struct sfs_disk_inode *din = sin->din;
assert(din->type != SFS_TYPE_DIR);
off_t endpos = offset + *alenp, blkoff;
*alenp = 0;
// calculate the Rd/Wr end position
if (offset < 0 || offset >= SFS_MAX_FILE_SIZE || offset > endpos) {
    return -E_INVAL;
}
if (offset == endpos) {
    return 0;
}
if (endpos > SFS_MAX_FILE_SIZE) {
    endpos = SFS_MAX_FILE_SIZE;
}
if (!write) {
    if (offset >= din->size) {
        return 0;
    }
    if (endpos > din->size) {
        endpos = din->size;
    }
}
```

之后根据是读还是写将`sfs_buf_op`和`sfs_block_op`函数指针复制到对应的读或写函数，这样在下面部分就能保持比较好的一致性

```c
int (*sfs_buf_op)(struct sfs_fs *sfs, void *buf, size_t len, uint32_t blkno, off_t offset);
int (*sfs_block_op)(struct sfs_fs *sfs, void *buf, uint32_t blkno, uint32_t nblks);
if (write) {
    sfs_buf_op = sfs_wbuf, sfs_block_op = sfs_wblock;
}
else {
    sfs_buf_op = sfs_rbuf, sfs_block_op = sfs_rblock;
}
```

接下来是一些用到的局部变量的定义，其中`size`为对硬盘块一次读写的大小，`alen`是总的累计读写大小，`ino`是要读写的硬盘块物理块号，`blkno`是读写起始点的逻辑块号，`count`是需要读写的总块数，`blkoff`是每次读写时的块内偏移

```c
int ret = 0;
size_t size, alen = 0;
uint32_t ino;
uint32_t blkno = offset / SFS_BLKSIZE;          // The NO. of Rd/Wr begin block
uint32_t nblks = endpos / SFS_BLKSIZE - blkno;  // The size of Rd/Wr blocks
uint32_t count = nblks + 1;
blkoff = offset % SFS_BLKSIZE;
```

在这里，我将一次读写请求分为下面三个步骤：

1. 首先获取读写开始点所在块，若开始位置没有对齐到块起始点，对开始位置到块结尾位置（文件包含多个快）或结束位置（文件只在一个块内）进行读写操作
2. 文件所在的第二块到倒数第二块必定都属于文件内容，通过`sfs_block_op()`函数进行整体读写
3. 再将最后一块从开头到`endpos`的内容进行读写

首先来看第一步：

```c
if (blkoff != 0) {
    sfs_bmap_load_nolock(sfs, sin, blkno, &ino);
    size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
    sfs_buf_op(sfs, buf, size, ino, blkoff);
    alen += size;
    count--;
    blkno++;
}
```

对于起始点不对齐块起始点的情况，首先使用`sfs_bmap_load_nolock()`函数找到文件的第`blkno`个逻辑块对应的硬盘物理块保存在`ino`中，然后计算需要读写的字节数后使用`sfs_buf_op()`函数进行读写，接着对`alen`进行累加，`blkno`增加为下一个逻辑块，剩余块数`count`减1

接下来看第二、三步：

```c
if (count > 0) {
    sfs_bmap_load_nolock(sfs, sin, blkno, &ino);
    count--;
    if (count > 0) {
        sfs_block_op(sfs, (void*)((size_t)buf + alen), ino, count);
        alen += count * SFS_BLKSIZE;
    }

    blkno = endpos / SFS_BLKSIZE;
    sfs_bmap_load_nolock(sfs, sin, blkno, &ino);
    size = endpos % SFS_BLKSIZE;
    sfs_buf_op(sfs, (void*)((size_t)buf + alen), size, ino, 0);
    alen += size;
}
```

此时`blkno`对应着第二个逻辑块，拿到其对应的物理块`ino`后，对`count`自减，若`count > 0`表示存在中间的完整块，使用`sfs_block_op()`函数整体读出`count`个块，注意缓冲区指针需要根据累计读写字节数`alen`进行偏移，最后更新`alen`

然后是第三步，根据`endpos`得到逻辑块序号进而得到物理块序号，以及得到读写字节数，使用`sfs_buf_op()`函数完成读写，再更新`alen`即可

在函数的末尾将最终成功读写字节数`alen`赋值到返回值`alenp`处，如果结束点位置大于文件大小说明写入了新内容，更新文件大小并设置脏位

```c
out:
    *alenp = alen;
    if (offset + alen > sin->din->size) {
        sin->din->size = offset + alen;
        sin->dirty = 1;
    }
    return ret;
```

#### 练习2：完成基于文件系统的执行程序机制的实现

需要改写proc.c中的load_icode函数和其他相关函数，实现基于文件系统的执行程序机制

首先是`alloc_proc()`函数，现在创建进程控制块需要创建与之管理的文件系统管理模块，即

```c
proc->filesp = files_create();
```

接下来是`do_fork()`函数，现在复制进程需要根据`clone_flags`进行文件系统管理模块的相应复制，在开中断前加入下面语句

```c
copy_files(clone_flags, proc);
```

最后来到重点的`load_icode()`函数，这个函数的大部分内容可以参考lab5中的，区别在于之前实验中用户程序是跟随操作系统内核加载到内存中，可以之间在内存中加载用户程序，而在本实验中需要通过文件系统读入硬盘中的文件，在这里只给出不同的部分

函数中需要使用`load_icode_read()`函数读入文件，其函数签名如下

```c
//load_icode_read is used by load_icode in LAB8
static int
load_icode_read(int fd, void *buf, size_t len, off_t offset)
```

其中`fd`为打开文件的文件标识，`offset`为读取位置偏移量，读取`len`个字节存放与`buf`指向的内存中

首先读入elf文件的头部，检查其是否为合法的elf文件

```c
struct Page *page;
struct elfhdr elf;
ret = load_icode_read(fd, (void*)&elf, sizeof(elf), 0);
if (ret != 0)
    goto bad_elf_cleanup_pgdir;
if (elf.e_magic != ELF_MAGIC) {
    ret = -E_INVAL_ELF;
    goto bad_elf_cleanup_pgdir;
}
```

再根据elf文件中的段数目`elf.e_phnum`设定循环次数，每次循环读出文件中的一段，对于每一段，首先读出段头部

```c
struct proghdr ph;
ret = load_icode_read(fd, (void*)&ph, sizeof(ph), elf.e_phoff + i * sizeof(ph));
```

接着以块为单位读出一个段内的内容，复制到新加载程序的相应内存空间中

```c
char seg[4096];
if (load_icode_read(fd, seg, size, from) != 0)
    goto bad_cleanup_mmap;
memcpy(page2kva(page) + off, seg, size);
start += size, from += size;
```

接下来需要设置用户栈，这里涉及的操作较多，首先看开始的几行，其用意是将栈空间`USTACKTOP-USTACKSIZE`添加到虚拟内存空间中，然后分配4个物理页

```c
vm_flags = VM_READ | VM_WRITE | VM_STACK;
if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
    goto bad_cleanup_mmap;
}
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
```

接着我们需要将程序加载参数放入栈中，在`do_execve()`函数中将加载参数复制到了内核空间中`kargv`数组中，在这里我们需要再将其复制到用户空间中

首先设置指针数组`uargv`用来记录用户虚拟地址中各参数的首地址，用变量`esp`记录栈顶位置，对每个参数进行类似压栈的动作，需要注意的是这些操作是在内核空间中运行的，需要将对应的用户虚拟地址转换为内核虚拟地址才能使用`memcpy()`函数完成复制

```c
char *uargv[EXEC_MAX_ARG_NUM];
memset(uargv, 0, sizeof(uargv));
uint32_t esp = USTACKTOP;
for (int i = 0; i < argc; ++i) {
    int len = strlen(kargv[i]) + 1;
    esp -= len;
    page = get_page(mm->pgdir, esp, NULL);
    memcpy(page2kva(page) + (esp - ROUNDDOWN(esp, PGSIZE)), kargv[i], len);
    uargv[i] = esp;
}
```

参数压栈完毕后再对参数指针数组`uargv`以及参数个数`argc`进行压栈

```c
esp -= sizeof(uargv);
page = get_page(mm->pgdir, esp, NULL);
memcpy(page2kva(page) + (esp - ROUNDDOWN(esp, PGSIZE)), uargv, sizeof(uargv));
esp -= sizeof(argc);
page = get_page(mm->pgdir, esp, NULL);
memcpy(page2kva(page) + (esp - ROUNDDOWN(esp, PGSIZE)), &argc, sizeof(argc));
```

接着切换到新加载程序的页表，以及设置好中断帧来让中断恢复时该程序能正确开始运行，在这里中断帧的栈寄存器需要设置为`esp`变量的值

```c
tf->tf_esp = esp;
tf->tf_eip = elf.e_entry;
```

至此从文件加成程序部分完成，在终端中使用`make qemu-nox`命令运行内核后，可以输入程序名加载程序运行，例如下面是输入`ls`命令的结果

```text
@ is  [directory] 2(hlinks) 24(blocks) 6144(bytes) : @'.'
   [d]   2(h)       24(b)     6144(s)   .
   [d]   2(h)       24(b)     6144(s)   ..
   [-]   1(h)       10(b)    40200(s)   badarg
   [-]   1(h)       10(b)    40204(s)   badsegment
   [-]   1(h)       10(b)    40220(s)   divzero
   [-]   1(h)       10(b)    40224(s)   exit
   [-]   1(h)       10(b)    40204(s)   faultread
   [-]   1(h)       10(b)    40208(s)   faultreadkernel
   [-]   1(h)       10(b)    40228(s)   forktest
   [-]   1(h)       10(b)    40252(s)   forktree
   [-]   1(h)       10(b)    40200(s)   hello
   [-]   1(h)       10(b)    40360(s)   ls
   [-]   1(h)       10(b)    40304(s)   matrix
   [-]   1(h)       10(b)    40192(s)   pgdir
   [-]   1(h)       10(b)    40292(s)   priority
   [-]   1(h)       10(b)    40344(s)   sfs_filetest1
   [-]   1(h)       12(b)    48604(s)   sh
   [-]   1(h)       10(b)    40220(s)   sleep
   [-]   1(h)       10(b)    40204(s)   sleepkill
   [-]   1(h)       10(b)    40200(s)   softint
   [-]   1(h)       10(b)    40196(s)   spin
   [-]   1(h)       10(b)    40224(s)   testbss
   [-]   1(h)       10(b)    40332(s)   waitkill
   [-]   1(h)       10(b)    40200(s)   yield
lsdir: step 4
```

### 内容2：修改ucore调度器为多级反馈队列调度算法

## 实验总结和对比

### 本实验中重要的知识点


### OS原理中很重要但实验中没有对应的知识点


### 总结


## 参考文献

- [实验指导手册](https://github.com/chyyuu/ucore_os_docs/blob/master/SUMMARY.md)

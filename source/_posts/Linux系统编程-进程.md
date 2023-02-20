---
title: Linux系统编程-进程
top_img: transparent
date: 2022-04-25 18:23:12
updated: 2022-11-29 15:13:47
tags:
  - Linux
  - 进程
categories: Linux
keywords:
description: Linux 中的进程创建、回收。
---

> 下面的内容如果想要深入、全面地了解，还是建议看书。书上解释得更全面，也可能会有更多例子方便理解，此博客只是为了方便我自己理解以及后续回顾。

## 进程

**进程管理**、内存管理和文件管理是操作系统的三大核心功能。那么什么是进程呢？顾名思义，**进程就是进展中的程序，或者说进程是执行中的程序**。那么什么是程序？程序就是一段机器指令，它规定了计算机要做哪些事。

> 计算机程序的运行实际上是一件十分复杂的事情，牵扯到方方面面。
>
> 首先，当然得进行编程，而编程需要计算机程序设计语言作为基础。对于绝大多数编写程序的人来说，使用的编程语言称为**高级程序设计语言**，如C、C++、Java等。但由于计算机并不认识高级语言编写的程序，编好的程序需要进行编译变成计算机能够识别的机器语言程序，而这**需要编译器和汇编器的帮助**。其次，机器语言程序需要**加载到内存，形成一个运动中的程序，即进程**，而这需要操作系统的帮助。**进程需要在计算机芯片CPU上执行才算是真正在执行**，而将进程调度到CPU上运行也由操作系统完成。最后，在CPU上执行的机器语言指令需要变成能够在一个个时钟脉冲里执行的基本操作，这**需要指令集结构和计算机硬件的支持**，而整个程序的执行过程还需要操作系统提供的服务和程序语言提供的执行环境（runtime environment）。这样，一个从程序到微指令执行的过程就完成了。

简单而言，一个进程的运行过程为：**程序**（即硬盘上的一个可执行文件）、**运行程序**（将程序中存储的机器语言加载到内存，形成进程，计算机上的多个进程实际上共享了物理内存，容易造成混乱，因此使用了**虚拟内存**对进程的内存地址空间进行隔离）、**在CPU上执行**（对单核的CPU而言，任意时刻只能执行一条指令，因此CPU上任意时刻只能执行一个进程。多个进程通过“时分复用”的方式共享CPU，也就是**进程调度**）。

> 发明进程的根本动机是什么？
>
> 答：是为了在计算机上同时运行多个程序，提高CPU的利用率。

### 程序

程序是包含了一系列信息的文件，这些信息描述了如何在运行时创建一个进程，所包括的内容如下所示：

- 二进制格式标识：每个程序文件都包含用于描述可执行文件格式的元信息，现在，大多数 UNIX 实现（包括 Linux）采用可执行连接格式（ELF）。
- 机器语言指令：对程序算法进行编码。
- **程序入口地址**：标识程序开始执行时的起始指令位置。
- 数据：程序文件包含的**变量初始值**和程序使用的**字面常量**`literal constant`值，比如字符串。
- **符号表**及重定位表：描述程序中函数和变量的位置及名称。这些表格有多种用途，其中包括**调试和运行时的符号解析**（动态链接）。
- **共享库和动态链接信息**：程序文件所包含的一些字段，列出了程序运行时需要使用的共享库，以及加载共享库的动态链接器的路径名。
- 其他信息：程序文件还包含许多其他信息，用以描述如何创建进程。

可以用一个程序来创建许多进程，或者反过来说，许多进程运行的可以是同一程序。

### 虚拟内存

在没有虚拟内存的情况下，各个程序（进程）都直接对物理内存地址进行操作，这样可能会出现多个进程修改同一个内存地址的情况，导致数据混乱，程序崩溃。而解决这个问题的办法就是让进程使用虚拟内存地址，然后再通过**页表**将虚拟内存转换为物理内存地址，在这个过程中，操作系统的内存管理模块就保证了两个进程的虚拟内存地址不会映射到同一个物理内存地址上。

关于虚拟内存的具体信息可以看：[虚拟内存](https://xiaolincoding.com/os/3_memory/vmem.html#%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)。

**页式存储管理**是一种常见而高效的方式，操作系统将内存空间分为若干个固定大小的页，并维护虚拟页地址和物理页地址的映射关系（即页表）。页大小涉及页分配的粒度和页表所占空间，目前的操作系统常用`4KB`的页。此时，**虚拟内存地址可表示为虚拟页地址和页内偏移两部分**，在进行地址转换时**通过查表的方式**将虚拟页地址替换为物理页地址就可得到对应的物理内存地址。

任一时刻，每个程序仅有部分页需要驻留在物理内存页帧中。这些页构成了所谓驻留集。程序未使用的拷贝保存在交换区内——这是磁盘空间中的保留区域，作为计算机 RAM 的补充，仅在需要时才会载入物理内存。若进程欲访问的页面目前并未驻留在物理内存中，将会发生页面错误，内核即刻挂起该进程的执行，同时从磁盘中将该页面载入内存。

> 大多数程序都展现了两种类型的局部性。
>
> - **空间局部性**：是指程序倾向于访问在最近访问过的内存地址附近的内存（由于指令是顺序执行的，且有时会按顺序处理数据结构）。
> - **时间局部性**：是指程序倾向于在不久的将来再次访问最近刚访问过的内存地址（由于循环）。

在 32 位系统中，采用`4KB`页时，单个完整页表需要`1M`项，对每个进程维护页表需要相当可观的空间代价，而`CPU Cache`一般只有数十`M`，因此页表只能放在内存中。若每次进行地址转换时都需要先查询内存，则会对性能产生明显的影响（内存的访问速度远低于`Cache`）。为了提高页表访问的速度，现代处理器中通常包含一个**转换后援缓冲器**（Translation Lookaside Buffer，简称 `TLB`）来**实现快速的虚实地址转换**。`TLB`也称**页表缓存或快表**，借由**局部性原理**，存储当前处理器中最经常访问页的页表。一般 `TLB` 访问与 `Cache` 访问同时进行，而 `TLB` 也可以被视为页表的 `Cache` 。

> 即使是从内存中查找页表，第一次找到后不就会将部分页表信息读入`Cache`了吗？后续的访问应该也很快，为什么还要专门的`TLB`模块？
>
> 猜测：`Cache` 中还存储了许多进程的程序指令、数据等信息，因此 `Cache` 会频繁地刷新数据。而页表信息会非常高频地进行访问，如果只用 `Cache` 缓存页表，会导致大量的`Cache miss`，从而带来很大的性能损耗。而 `TLB` 只存储页表信息，相对来说刷新频率没有 `Cache` 那么高，也就是命中率更高，因此相对来说更好、更快。
>
> ------
>
> 假设变量 a 在虚拟内存中的地址为 0x1000，现在需要对它 +1，CPU的流程是在TLB中查询0x1000对应的物理内存地址，找到了0x20000，再查找L1 cache 是否有这个地址的数据，如果有就直接修改，然后写回物理内存？
>
> chatgpt: 是的，在进程访问虚拟内存时，CPU 的流程通常是这样的：
>
> 1. CPU 会先在 TLB 中查找虚拟地址 0x1000 对应的物理地址。如果 TLB 中有这个地址的信息，就会直接使用 TLB 中的信息。
> 2. 如果 TLB 中没有这个地址的信息，就会从内存中读取页表信息，并将读取的信息存储到 TLB 中，供以后使用。
> 3. 然后 CPU 会检查 L1 cache 是否有物理地址 0x20000 的数据。如果有，就直接使用 L1 cache 中的数据。
> 4. 如果 L1 cache 中没有这个地址的数据，就会继续检查 L2 cache 和 L3 cache 是否有这个地址的数据。如果有，就直接使用缓存中的数据。
> 5. 如果 L2 和 L3 cache 中也没有这个地址的数据，就会从物理内存中读取数据，并将读取的数据存储到 L1 cache 中，供以后使用。
> 6. 最后，CPU 会将修改后的数据写回物理内存。
>
> 总之，在进程访问虚拟内存时，CPU 会优先查询 TLB，如果 TLB 中不存在，再查询 cache。如果 cache 中也不存在，最后才会到内存中读取页表信息，这个过程会将页表信息加载到 cache 中。
>
> ------
>
> 虚拟内存带来了哪些优点？
>
> 答：虚拟内存管理使进程的虚拟地址空间与物理内存地址空间隔离开来，优点有：
>
> - 进程与进程、进程与内核**相互隔离**，所以一个进程不能读取或修改另一进程或内核的内存。
> - 适当情况下，两个或者更多进程**能够共享内存**。这是由于内核可以使不同进程的页表条目指向相同的 RAM 页。
> - **便于实现内存保护机制**；也就是说，可以对页表条目进行标记，以表示相关页面内容是可读、可写、可执行亦或是这些保护措施的组合。多个进程共享 RAM 页面时，允许每个进程对内存采取不同的保护措施。例如，一个进程可能以只读方式访问某页面，而另一进程则以读写方式访问同一页面。
> - 程序员和编译器、链接器之类的工具无需关注程序在 RAM 中的物理布局。
> - 因为需要驻留在内存中的仅是程序的一部分，所以**程序的加载和运行都很快**。而且，一个进程所占用的内存（即虚拟内存大小）**能够超出 RAM 容量**。
> - 由于每个进程使用的 RAM 减少了，RAM 中同时可以**容纳的进程数量就增多**了。进而**提高了CPU的利用率**。

### 内存布局

每个进程所分配的**虚拟内存**由很多部分组成，通常称之为“段（segment）”。

> `size`命令可显示二进制可执行文件的文本段、初始化数据段、非初始化数据段(`bss`)的段大小。

- **文本段**：包含了进程运行的程序机器语言指令。

- **初始化数据段**：包含显式初始化的**全局变量和静态变量**。当程序加载到内存时，从可执行文件中读取这些变量的值。

- **未初始化数据段`BSS`**：包含了**未进行显式初始化**的全局变量和静态变量。程序启动之前，系统将本段内所有内存初始化为0。

  > 1. BSS 段的内存会被初始化为0，也就意味着未进行显式初始化的**全局变量和静态变量**的初始值为0。但需要注意，未进行显式初始化的**局部变量**，并不在 BSS 段中，也不会被初始化为0！
  > 2. 将经过初始化的与未经初始化全局变量和静态变量分开存放，其主要原因在于程序在磁盘上存储时，没有必要为未经初始化的变量分配存储空间。相反，可执行文件只需记录未初始化数据段的位置及所需大小，直到运行时再由程序加载器来分配这一空间。

- 栈`stack`：是一个**动态增长和收缩**的段，由栈帧（`stack frames`）组成。系统会**为每个当前调用的函数分配一个栈帧**。栈帧中存储了函数的局部变量（所谓自动变量）、实参和返回值。此外，**也可使用`alloca()`在栈上动态分配空间**。

- 堆`heap`：是可在运行时（为变量）动态`malloc()`进行内存分配的一块区域。堆顶端称作 `program break`。

![image-20220526222503936](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B/image-20220526222503936.png)

### 内核态与用户态

就像世界上的人并不平等一样，并不是所有的程序都是平等的。世界上有的人占有资源多，有的人占有资源少，有的人来了，别人得让出资源，有的人则专门为别人让出资源。程序也是这样的，有的程序可以访问计算机的任何资源，有的程序则只能访问少量受限资源。操作系统作为计算机的管理者，自然不能和被管理者享受一样的待遇。它应该享有更多的方便或权限。**为了区分不同的程序的不同权限**，人们发明了内核态和用户态的概念。

那么什么是内核态，什么又是用户态呢？只要想一想现实生活中处于社会核心的人与处于社会边缘的人有什么区别，就能明白处于核心的人拥有的资源多！因此，**内核态就是拥有资源多的状态，或者说访问资源多的状态，称为特权态**。相对来说，**用户态就是非特权态，在此种状态下访问的资源将受到限制**。例如，如果要访问操作系统的内核数据结构，如进程表，则需要在特权态下才能办到。如果要访问用户程序里的数据，则在用户态就可以了。

运行在内核态的程序可以访问的资源多，但**可靠性、安全性要求高，维护管理都较复杂**；

> 那么计算机是如何知道现在正在运转的程序是内核态程序呢？或者说内核态的本质是？
>
> 而正确做出内核态或用户态的判断对系统的正确运行至关重要。显然做出这种判断需要某种标志。**这个标志就是处理器的一个状态位**。这个状态位是CPU状态字里面的一个字位。**也就是说，所谓的用户态、内核态实际上是处理器的一种状态，而不是程序的状态**。我们通过设置该状态字，可以将CPU设置为内核态、用户态或者其他的子态（有的CPU有更多种子态）。**一个程序运行时，CPU是什么态，这个程序就运行在什么态。**
>
> ------
>
> 前面说过，内核态是特权态，而用户态是普通态。特权态下运行的程序可以访问任何资源，而用户态下的访问则受到限制。那么这种限制是如何实现的呢？
>
> 显然，要限制一个程序对资源的访问，需要对程序执行的每一条指令进行检查才能完成。而这种检查就是地址翻译。程序发出的每一条指令都要经过这个地址翻译过程。而**通过对翻译的控制，就可以限制程序对资源的访问**。（地址翻译就是将进程的虚拟内存地址翻译为物理内存地址，页表查询的过程。）
>
> 为了赋予内核态程序访问所有资源的权限，**当系统处于内核态时，内核程序可以绕过内存地址翻译而直接执行特权指令**，如停机指令。这种绕过翻译的做法突破了系统对资源的控制。

在 Linux 操作系统中，虚拟地址空间的内部又被分为**内核空间和用户空间**两部分，不同位数的系统，地址空间的范围也不同。比如最常见的 32 位和 64 位系统，如下所示：

![3a6cb4e3f27241d3b09b4766bb0b1124](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B/3a6cb4e3f27241d3b09b4766bb0b1124.png)

> 为什么内存地址空间被分为内核空间和用户空间？内核空间为什么是共享的？
>
> todo....

### 进程状态

进程的三种典型状态：

1. 就绪态：已经具备运行条件，位于进程的就绪队列中，等待进程调度被分配CPU资源。
2. 运行态：当前占据CPU资源的进程的状态。
3. 阻塞态：处于运行态的进程因发出某种资源请求，操作系统将其CPU资源剥夺，等待请求的事件满足，再变成就绪态，加入就绪队列。

这里阐述的进程的3种典型状态并不是唯一的分类方式，事实上，许多商业操作系统的进程状态不止三个。

- 创建态：进程正在被创建，操作系统需要为其分配资源、初始化PCB。
- 终止态：进程正在从系统中撤销，等待操作系统会回收进程拥有的资源、撤销PCB。

![七种状态变迁](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B/10-%E8%BF%9B%E7%A8%8B%E4%B8%83%E4%B8%AD%E7%8A%B6%E6%80%81.jpg)

> 那么进程挂起有哪些原因呢？
>
> 1. 首先是一个进程在运行过程中**执行了某种阻塞操作**，如读写磁盘。由于阻塞操作需要等待结果后才能继续执行，因此操作系统将把这个进程挂起，让其他进程运转。
> 2. 另外一种情况是一个**进程执行的时间太长了**，为了公平，操作系统将其挂起，让其他进程也有机会执行。

## 进程常用 API

从内核角度看，进程由用户内存空间（user-space memory）和一系列内核数据结构组成，其中用户内存空间包含了程序代码及代码所使用的变量，而内核数据结构则用于维护进程状态信息。记录在内核数据结构中的信息包括许多与进程相关的标识号（IDs）、虚拟内存表、打开文件的描述符表、信号传递及处理的有关信息、进程资源使用及限制、当前工作目录和大量的其他信息。

对进程的相关操作实际上就是操作内核中记录进程相关信息的结构体。

### 进程控制块

操作系统中，进程是**资源分配的基本单位**，操作系统对进程的管理通过**进程表**来实现，进程表里存放的是关于系统上所有进程的一切信息。实际上，进程表就是一个`PCB`链表，**每个进程的一切信息都记录在一个进程控制块**`PCB`中，包括进程的状态，进程的资源，进程的输入输出等。在 `linux` 中，`PCB` 的实现是 `task_struct` 结构体。

具体信息看源码：[linux v2.6.39 task_struct](https://elixir.bootlin.com/linux/v2.6.39/source/include/linux/sched.h#L1193)，下面是省略了大部分成员后的结构体，[结构体说明](https://blog.csdn.net/qq_44836294/article/details/108637404)：

```c
struct task_struct {
    /* 进程状态相关 */
    volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
    void *stack;            /* 指向内核栈 */
    int exit_code;          /* 进程退出码 */

    /* 进程亲缘关系 */
    struct list_head tasks; /* 用于加入进程表 */
    pid_t pid;  /* 进程号 */
    pid_t tgid; /* 线程组ID */
    struct task_struct *real_parent;    /* 指向创建其的父进程，如果其父进程不存在，则指向init进程 */
    struct task_struct *parent;         /* 指向当前的父进程，通常与real_parent一致，接收SIGCHLD信号 */
    struct list_head children;          /* 子进程链表 */
    struct list_head sibling;           /* 兄弟进程链表 */
    struct task_struct *group_leader;   /* threadgroup leader */
    struct pid_link pids[PIDTYPE_MAX];  /* PID/PID hash table linkage. */

    /* 进程下的线程 */
    struct list_head thread_group;  /* 线程链表 */

    /* 进程打开的文件信息 */
    int link_count, total_link_count;
    struct fs_struct *fs;       /* 目录信息 */
    struct files_struct *files; /* 进程所有打开的文件都会在这里面的一个指针数组里，文件描述符就是该数组的下标 */

    /* 信号相关信息 */
    struct signal_struct *signal;
    struct sighand_struct *sighand;
    sigset_t blocked, real_blocked;
    sigset_t saved_sigmask; /* restored if set_restore_sigmask() was used */
    struct sigpending pending;

    /* 进程调度相关，优先级 */
    int prio, static_prio, normal_prio;
    unsigned int rt_priority;
    const struct sched_class *sched_class;
    struct sched_entity se;
    struct sched_rt_entity rt;
    unsigned int policy;            /* 进程的调度策略 SCHED_FIFO, SCHED_RR, SCHED_OTHER, etc */
    ...
    /* and more */
};
```

进程控制块的主要内容有：

- 身份标识（进程号），进程树（父子进程关系）；
- 进程的状态信息：就绪、运行、阻塞等；
- 进程的调度策略，优先级等；
- 进程中存在的线程信息；
- 进程打开的文件信息（文件描述符）；
- 信号处理相关信息；
- 进程的资源：内存、CPU、IO、磁盘等；
- 等等。

### fork

基于当前进程创建一个子进程。在操作系统中，进程是资源分配的基本单位，所以子进程也会重新分配进程控制块`PCB`，也就是说，父子进程间很多东西都是独享的；不过子进程是当前进程的一个“分身”，或者说**复制**品，因此子进程的`PCB`与父进程的`PCB`有很多**相同之处**，比如：环境变量、全局变量、宿主目录位置、进程工作目录位置、文件列表、信号处理方式、文本段、堆、栈。

不过父子进程仍然存在进程间的内存隔离，因此这里说的相同仅仅是值相同，而且仅仅是刚创建这一时刻，因为后续子进程可以修改这些信息。**或者说父子进程没有内容是“共享的”**更恰当。

对于一些进程身份信息相关的内容，显然**父子进程是不同的**，包括：进程号、父进程号、进程创建时间等。

不够由于进程控制块`PCB`相当大，如果在`fork`过程中，完全进行内存拷贝会消耗很多时间，因此某些内容采用了**读时共享、写时复制**的策略，如全局变量。而某些内容则直接采用**共享**的方式：文件描述符、`mmap`映射区等。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);

// example 1
pid_t pid = fork();
if (pid == 0) {
    // child process
} else if (pid > 0) {
    // parent process
} else {
    // error
}

// example 2: 循环创建子进程
int i;
for (i = 0; i < 5; i++) {
    if (fork() == 0) { // child process
        break;
    }
}
if (i == 5)
    printf("this is parent process\n");
else
    printf("this is %dth child process\n", i);
```

- 返回值：
  - 成功，父进程中返回子进程的 `PID`，子进程返回 `0`。
  - 失败，返回 `-1`，并设置 `errno` 。

> [系统调用 fork 的源码](https://elixir.bootlin.com/linux/v2.6.39/source/arch/x86/kernel/process.c#L235)

### getpid/getppid

> `PID`是一个数字，用于标识一个进程，就像学生的学号一样，每个进程都有一个唯一的编号，保存在进程控制块的 `pid` 字段中。一般的，在系统运行期间，`PID` 都是被顺序编号，比如进程A的`PID`为`10`，那下个创建的进程的`PID`则为`11`。不过`PID`的值有一个上限，当内核使用的PID达到这个上限后就会循环开始找已闲置的小`PID`号。

获取当前进程的 `PID` 和父进程的 `PID`。使用`pstree`命令可查看到这一“家族树”。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
```

- 返回值：
  - 成功，返回 `PID`。
  - 失败，这些函数总是成功的！

[getpid 的源码](https://elixir.bootlin.com/linux/v2.6.39/source/kernel/timer.c#L1356)：

```c
/* kernel/timer.c:1353 */
SYSCALL_DEFINE0(getpid)
{
    return task_tgid_vnr(current);
}
/* include/linux/sched.h:1600 */
static inline pid_t task_tgid_vnr(struct task_struct *tsk)
{
    return pid_vnr(task_tgid(tsk));
}
```

> `getpid()`函数返回的值其实是当前进程的`tgid`而不是`pid`的值，而由于线程组中领头线程`tgid`和`pid`相同，因而`getpid()`对这类进程所起到的作用和一般进程是一样的。

### wait/waitpid

**回收子进程**！

> 为什么要回收进程？
>
> 正如前面所述，每个进程被分配了一个很大的结构体——进程控制块，该结构体会占用不少的内存空间，如果不回收（释放该结构体），那这些内存就被白白浪费了。此外，每个进程都有一个进程号，如果不回收进程，那它所占用的进程号就不能被再次利用。长时间运行后，必然会有新进程由于无法分配到进程号，而创建失败。
> 如果不回收子进程，这种已经结束但没有被回收的子进程称为**僵尸进程**。
>
> ------
> 那操作系统为什么不自动回收进程呢？
>
> 有些时候，父进程希望知道子进程是如何结束的（由于异常错误？还是正常退出），而进程的退出状态就记录在进程控制块的`exit_code`中，父进程通过子进程的进程号就可以访问到子进程的`PCB`，进而获知其结束的原因。

- `wait`：**阻塞等待**任意一个子进程结束，并回收。参数为传出变量，写入被回收的子进程的结束码。可设置为`NULL`，表示不接收。
- `waitpid`：回收指定`pid`的子进程，可以设置是否阻塞。

这两种系统调用每次只能回收一个子进程，如果父进程需要回收多个子进程，可以多次调用 `wait` 函数。

> **注意：进程回收只能由父进程调用该函数，兄弟进程之间无法互相回收，子进程也无法被父进程的父进程回收**！

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);

/* waitpid 回收多个子进程 */
int main() {
    int i;
    for (i = 0; i < 5; i++) {
        if (fork() == 0) { // child process
            break;
        }
    }
    pid_t wpid;
    if (i == 5) { /* 父进程执行 */
        // 当前进程不存在子进程时会返回 -1
        while((wpid = waitpid(-1, NULL, WNOHANG)) != -1) {
            if (wpid > 0)
                printf("wait child %d\n", wpid);
            else
                sleep(1);
        }
        printf("this is parent process\n");
    } else { /* 子进程执行 */
        sleep(i);
        printf("this is %dth child process\n", i);
    }
    return 0;
}
```

- `wstatus` 用来接收子进程的终止状态。可设置为NULL，表示不接收。
- `pid` 参数：
  - 如果 `pid = 0`，则等待任意同组的子进程终止；
  - 如果 `pid = -1`，则等待任意子进程终止；
  - 如果 `pid > 0`，则等待指定的子进程终止；
  - 如果 `pid < -1`，则等待任意进程组 id 为 abs(pid) 的子进程终止。
- `options` 参数：
  - `0`：阻塞等待；
  - `WNOHANG`：如果没有子进程终止，则立即返回，不阻塞父进程。
- 返回值：
  - `>0`，成功，返回子进程的 `pid`；
  - `0`，调用时指定了 `WNOHANG`，并且没有子进程终止；
  - `-1`，失败，并设置 `errno`。**当前进程不存在子进程**或**回收的进程P不是当前进程的子进程**时会返回 `-1`。

> **孤儿进程**：父进程先于子进终止，子进程沦为“孤儿进程”，会被 `init` 进程领养。
>
> **僵尸进程**：子进程终止，父进程尚未对子进程进行回收，在此期间，子进程为“僵尸进程”。 `kill`**对其无效**。**每个进程结束后都必然会经历僵尸态**，时间长短的差别而已。回收僵尸进程，得 `kill` 它的父进程，让孤儿院去回收它。
>
> **杀死父进程并不会导致子进程死亡**，只是子进程的父进程会变为 `init`。

### exec 函数族

`fork` 创建子进程后执行的是和父进程相同的程序，但有可能执行不同的代码分支，不过这种情况并不多见，因为用线程可以更好更快地达到这个目的。子进程往往要调用一种 `exec` 函数以执行另一个程序。

当进程调用一种 `exec` 函数时，该进程的用户空间**代码和数据完全被新程序替换**，从新程序的启动例程开始执行。**调用 `exec` 并不创建新进程，所以调用 `exec` 前后该进程的 `id` 并未改变**。

`exec` 函数一旦调用成功，即执行新的程序，**不返回**。只有失败才返回，错误值`-1`，所以通常我们直接在 `exec` 函数调用后直接调用 `perror()`，和 `exit()`，无需 `if` 判断。

```c
#include <unistd.h>

extern char **environ;

int execl(const char *pathname, const char *arg, ...
                /* (char  *) NULL */);
int execlp(const char *file, const char *arg, ...
                /* (char  *) NULL */);
int execle(const char *pathname, const char *arg, ...
                /*, (char *) NULL, char *const envp[] */);
int execv(const char *pathname, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[],
                char *const envp[]);
```

- `l` 命令行参数列表。
- `v` 使用命令行参数数组。
- `p` 表示要借助环境变量`PATH`来加载可执行文件。
- `e` 使用环境变量数组，不用进程原有的环境变量，设置新加载程序运行的环境变量。

```c
execl("/usr/bin/ls", "ls", "-l", "/tmp", NULL); // 指定绝对路径，参数列表以 NULL 结尾。
execlp("ls", "ls", "-l", "/tmp", NULL); // 指定程序名，参数列表以 NULL 结尾。

char *argv[] = {"ls", "-l", "/tmp", NULL};
execv("/usr/bin/ls", argv); // 指定绝对路径，参数列表以 NULL 结尾。
execvp("ls", argv); // 指定程序名，参数列表以 NULL 结尾。
```

这些函数都是库函数，都是通过系统调用`execve`实现的。

### vfork

在早期的 BSD 实现中，`fork()`会对父进程的数据段、堆和栈施行严格的复制。如前所述，这是一种浪费，尤其是在调用 `fork()`后立即执行 `exec()`的情况下。出于这一原因，BSD 的后期版本引入了 `vfork()`系统调用，尽管其运作含义稍微有些不同（实则有些怪异），但效率要远高于 BSD `fork()`。现代 UNIX 采用写时复制技术来实现 `fork()`，其效率较之于早期的 `fork()`实现要高出许多，进而将对 `vfork()`的需求剔除殆尽。虽然如此，Linux（如同许多其他的 UNIX 实现一样）还是提供了具有 BSD 语义的 `vfork()`系统调用，以期为程序提供尽可能快的 `fork` 功能。不过，鉴于 `vfork()`的怪异语义可能会导致一些难以察觉的程序缺陷（bug），除非能给性能带来重大提升（这种情况发生的概率极小），否则**应当尽量避免使用这一调用**。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t vfork(void);
```

类似于 `fork()`，`vfork()`可以为调用进程创建一个新的子进程。然而，`vfork()`是为子进程立即执行 `exec()`的程序而专门设计的。

`vfork()`因为如下两个特性而更具效率，这也是其与 `fork()`的区别所在。

- 无需为子进程复制虚拟内存页或页表。相反，**子进程共享父进程的内存**，直至其成功执行了 `exec()`或是调用`_exit()`退出。
- 在子进程调用 `exec()`或`_exit()`之前，**将暂停执行父进程。**

## 进程组与会话

进程组和会话在进程之间形成了一种两级层次关系：**进程组是一组相关进程的集合，会话是一组相关进程组的集合**。

### 进程组

**进程组由一个或多个共享同一进程组标识符（`PGID`）的进程组成**。进程组 ID 是一个数字，其类型与进程 ID 一样（pid_t）。一个进程组拥有一个进程组首进程，该进程是创建该组的进程，其进程 ID 为该进程组的 ID，**新进程会继承其父进程所属的进程组 ID**。

**进程组**拥有一个生命周期，其开始时间为首进程创建组的时刻，结束时间为最后一个成员进程退出组的时刻。一个进程可能会因为终止而退出进程组，也可能会因为加入了另外一个进程组而退出进程组。进程组首进程无需是最后一个离开进程组的成员（**当一个进程组的组长死亡时，只要有其他进程存在，则该进程组存在。并且组ID仍是已故的组长进程ID**）。**会话也类似**。

> 进程组有什么作用呢？
>
> 一种用处为：可以通过组 ID 同时向该进程组中的所有进程发送信号。
>
> ```bash
> kill -9 -{{group_id}} # 组ID前面要加负号`-`，就可以给该进程组中的所有进程发送信号
> ```

#### getpgrp/getpgid

获取一个进程的进程组ID。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpgid(pid_t pid);   /* 很少会检索调用者以外的进程的 PGID */
pid_t getpgrp(void);        /* POSIX.1 version 首选方法 */
```

`getpgrp`就是返回当前调用进程所属的进程组ID，**首选方法**。

`getpgid`可以返回指定进程所属的进程组ID，如果`pid`为`0`，就是返回当前调用进程所属的进程组ID。

#### setpgrp/setpgid

`setpgid()`系统调用将进程 ID 为 `pid` 的进程的进程组 ID 修改为 `pgid`。

```c
#include <sys/types.h>
#include <unistd.h>

int setpgid(pid_t pid, pid_t pgid);  /* 首选方法 */
int setpgrp(void);                   /* System V version, 相当于 setpgid(0, 0) */
```

如果 `pid` 和 `pgid` 参数指定了同一个进程（`pgid`为`0`或`pgid == pid`），就会**创建一个新进程组**，并且指定的进程会成为这个新组的首进程。

如果两个参数的值不同，则会将指定进程从一个进程组中移到另一个进程组中。

在调用`setpgid()`时存在以下限制：

- `pid` 参数可以仅指定调用进程或其中一个子进程；
- 在组之间移动进程时，调用进程、由 `pid` 指定的进程以及目标进程组必须要属于同一个会话；
- `pid` 参数所指定的进程不能是会话首进程；
- 一个进程在其子进程已经执行 `exec()` 后就无法修改该子进程的进程组 ID 了。

### 会话

**会话是一组进程组的集合**。进程的会话成员关系是由其会话标识符（`SID`）确定的，会话标识符与进程组 ID 一样，是一个类型为 `pid_t` 的数字。会话首进程是创建该新会话的进程，其进程 ID 会成为会话 ID。**新进程会继承其父进程的会话 ID**。

**一个会话中的所有进程共享单个控制终端**。控制终端会在会话首进程首次打开一个终端设备时被建立。一个终端最多可能会成为一个会话的控制终端。

> **前台和后台进程组**
>
> 在终端中运行程序时（如`ls`），该程序产生的输出会显示在终端上，如果需要，也会从终端读入用户键盘输入的信息。
> 有时希望将一个程序A启动后，在当前终端下继续运行程序B，这时一般会通过`A &`让程序后台运行，此时就还可以继续运行B。
>
> 总的来说，前台和后台的区别在于是否能中终端读入用户输入，或者说运行时是否使用了`&`。需要注意，后台进程组的输出仍然可以输出到终端，比如`sleep 2 && ls &`.

在任一时刻，会话中的其中一个进程组会成为终端的**前台进程组**，其他进程组会成为**后台进程组**。**只有前台进程组中的进程才能从控制终端中读取输入**。当用户在控制终端中输入其中一个信号生成终端字符之后，该信号会被发送到前台进程组中的**所有成员**。

> 那么会话有什么用呢？
>
> 同样，也可以通过会话`SID`给会话中的所有进程发送信号！（当然，会话的主要用途不是这个...）
>
> ```bash
> # 在一个终端中启动多个进程
> $ cat | cat | cat | cat  # 启动多个进程
>
> # 在另一个终端中运行
> $ ps aj
>    PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
>    5115   54152   54152   54152 pts/3      62248 Ss    1000   0:00 /bin/bash
>   54152   62248   62248   54152 pts/3      62248 S+    1000   0:00 cat
>   54152   62249   62248   54152 pts/3      62248 S+    1000   0:00 cat
>   54152   62250   62248   54152 pts/3      62248 S+    1000   0:00 cat
>   54152   62251   62248   54152 pts/3      62248 S+    1000   0:00 cat
>
> # 终端中运行的程序其会话 SID 与该终端的进程ID相同，因此下面的命令会将终端也关闭！
> $ kill -9 -54152
> ```

#### getsid

获取指定进程所属的会话 `SID` 。`pid`为0表示获取当前调用进程的会话 `SID` 。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getsid(pid_t pid);
```

- 返回值：成功返回调用进程的会话 ID，失败返回`-1`，设置 `errno`。

#### setsid

创建一个新会话，并以自己的 `PID` 设置为进程组 `PGID`，同时也是新会话的 `SID` ！

```c
#include <sys/types.h>
#include <unistd.h>

pid_t setsid(void);
```

- 返回值：成功返回调用进程的会话 `SID`，失败返回 `-1`，设置 `errno` 。

`setsid()`系统调用会按照下列步骤创建一个新会话：

- 调用进程成为新会话的首进程和该会话中新进程组的首进程。调用进程的进程组 `PGID` 和会话 `SID` 会被设置成该进程的进程 ID。
- **调用进程没有控制终端**。所有之前到控制终端的连接都会被断开。

在调用`setsid()`时存在一个限制：**调用进程不能是一个进程组的首进程**。避免这个错误发生的最简单的方式是执行一个 `fork()`并让父进程终止以及让子进程调用 `setsid()`。进程组中，组长死亡后，不会再自动产生新组长。

> 为什么不能是进程组的首进程？
>
> 因为如果没有这个约束的话，进程组组长就能够将其自身迁移至另一个（新的）会话中了，而该进程组的其他成员则仍然位于原来的会话中。这会破坏会话和进程组之间严格的两级层次，因为一个进程组的所有成员必须属于同一个会话。

### 终端

在UNIX系统中，用户通过终端登录系统后得到一个Shell进程，这个终端成为Shell进程的控制终端 (Controlling Terminal)。

控制终端是保存在`PCB`中的信息，而我们知道`fork`会复制`PCB`中的信息，因此由Shell进程启动的其它进程的控制终端也是这个终端。

默认情况下（没有重定向），每个进程的标准输入、标准输出和标准错误输出都指向控制终端，进程从标准输入读也就是读用户的键盘输入，进程往标准输出或标准错误输出写也就是输出到显示器上。

每个进程都可以通过一个特殊的设备文件`/dev/tty`访问它的控制终端。`ttyname`函数可以由文件描述符查出对应的文件名，该文件描述符必须指向一个终端设备而不能是任意文件。

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    printf("stdin  tty: %s\n", ttyname(STDIN_FILENO));
    printf("stdout tty: %s\n", ttyname(STDOUT_FILENO));
    printf("stderr tty: %s\n", ttyname(STDERR_FILENO));
}

// stdin  tty: /dev/pts/2
// stdout tty: /dev/pts/2
// stderr tty: /dev/pts/2
```

### 守护进程

守护进程`daemon`指的是具有特殊用途的进程，通常采用以`d`结尾的命名方式，系统创建和处理此类进程的方式与其他进程相同。

但以下特征是其所独有的：

- 长生不老，守护进程通常在系统引导时启动，直至系统关闭前，会一直“健在”，**不受用户登录注销影响**。
- 守护进程在后台运行，且无控制终端供其读取或写入数据。周期性的等待某个事件发生或周期性执行某一动作。

> 很多标准的 `daemon` 会作为特权进程运行，即有效用户 ID 为 0。

**创建守护进程的步骤**：

1. 执行一个 `fork()`，之后父进程退出，**子进程**继续执行（子进程被确保不会成为一个进程组首进程）。
2. 子进程调用 `setsid()` 开启一个**新会话**并释放它与控制终端之间的所有关联关系。
3. 清除进程的 `umask` 以确保当 `daemon` 创建文件和目录时拥有所需的**权限**`022 -- 755`。
4. 修改进程的**当前工作目录**，通常会改为根目录`/`，防止工作目录被卸载，导致进程死亡。
5. 关闭 `daemon` 从其父进程继承而来的所有打开着的文件描述符，主要是针对 `0, 1, 2`，一般是重定向到 `/dev/null`，防止了后面使用描述符 1 或 2 打开一个文件的情况，因为库函数会将这些描述符当做标准输出和标准错误来写入数据。
6. 守护进程**业务逻辑**`while()`。

```c
int become_daemon()
{
    int maxfd, fd;
    switch (fork()) {
        case -1: return -1;
        case 0: break;      /* 1. 子进程继续，也就是守护进程 */
        default: exit(EXIT_SUCESS); /* 父进程退出 */
    }

    if (setsid() == -1)     /* 2. 创建新会话 */
        return -1;

    if (umask(0022) == -1)  /* 3. 重设文件权限掩码 */
        return -1;

    if (chdir("/") == -1)   /* 4. 修改进程的当前工作目录 */
        return -1;

    close(STDIN_FILENO);    /* 5. 通常根据需要，关闭/重定向 文件描述符 */
    fd = open("/dev/null", O_RDWR); /* fd --> 0 */
    if (fd == -1)
        printf("open error");

    dup2(fd, STDOUT_FILENO); /* 重定向 stdout 和 stderr */
    dup2(fd, STDERR_FILENO);

    return 0;
}
```

## 进程优先级

操作系统上总是会有很多运行中的进程，而CPU却很有限，那么哪个进程能获得CPU的使用权呢？Linux 与大多数其他 UNIX 实现一样，调度进程使用 CPU 的**默认模型是循环时间共享**。在这种模型中，每个进程轮流使用 CPU 一段时间，这段时间被称为**时间片或量子**。

但进程并不是完全平等的，某些重要进程应该被分配更多的CPU使用权。那么怎么描述进程的重要性呢？进程特性 `nice` 值允许进程间接地影响内核的调度算法。每个进程都拥有一个 `nice` 值，其取值范围为（高优先级）`−20～19`（低优先级），默认值为 0。非特权进程只能降低自己的优先级（不过这个说法已经不正确了），即赋一个大于默认值 0 的 `nice` 值。这样做之后它们就对其他进程“友好（nice）”了，这个特性的名称也由此而来。

进程的调度不是严格按照 `nice` 值的层次进行的，相反，`nice` 值是一个权重因素，它导致内核调度器倾向于调度拥有高优先级的进程。给一个进程赋一个低优先级（即高 `nice` 值）并不会导致它完全无法用到 CPU，但会导致它使用 CPU 的时间变少。

进程控制块`PCB`中就记录了进程的调度策略和优先级等值。

```c
struct task_struct {
    /* 进程调度相关，优先级 */
    int prio, static_prio, normal_prio;
    unsigned int rt_priority;
    const struct sched_class *sched_class;
    struct sched_entity se;
    struct sched_rt_entity rt;
    unsigned int policy;            /* 进程的调度策略 SCHED_FIFO, SCHED_RR, SCHED_OTHER, etc */
    ...
}
```

### getpriority/set

`getpriority()`和 `setpriority()`系统调用允许一个进程获取和修改自身或其他**进程的 `nice` 值**。

```c
#include <sys/time.h>
#include <sys/resource.h>

int getpriority(int which, id_t who);
int setpriority(int which, id_t who, int prio);
```

`which`**用于确定`who`如何被解释**：

- `PRIO_PROCESS`：操作进程 `PID` 为 who 的进程。如果 who 为 0，那么使用调用者的进程 ID。
- `PRIO_PGRP`：操作进程组 `PGID` 为 who 的进程组中的所有成员。如果 who 为 0，那么使用调用者的进程组。
- `PRIO_USER`：操作所有真实用户 `USRID` 为 who 的进程。如果 who 为 0，那么使用调用者的真实用户 ID。

调用`getpriority`时，如果有多个进程符合指定的标准，那么将会返回**优先级最高**的进程的 `nice` 值（即最小的数值）。

> 由于`getpriority`可能会在成功时返回`−1`，因此在调用这个函数之前必须要将 `errno` 设置为 0，接着在调用之后额外检查`errno`确认是否发生了错误。

```c
#include <stdio.h>
#include <errno.h>
#include <unistd.h>
#include <sys/resource.h>

int main() {
    int nice;
    errno = 0;
    nice = getpriority(PRIO_PROCESS, 0);
    if (nice == -1 && errno != 0) {
        /* 说明调用时发生了错误 */
        perror("getpriority failed!");
    }
    printf("current nice: %d\n", nice);
}
```

调用`setpriority`时，试图将 `nice` 值设置为一个超出允许范围的值（`-20～+19`）时会直接将 `nice` 值设置为边界值（还存在其他限制）。

> 需要说明一下，系统调用只能通过`syscall`直接发起，我们使用的`getpriority`等其实是`c`库函数对系统调用进行封装后的函数，库函数的返回结果并不一定是系统调用的直接返回值（虽然这种情况很少见，不过`getpriority`就是一个特例）。
>
> `getpriority()`系统调用服务例程不会返回实际的 nice 值，相反，它会返回一个范围在 1（低优先级）～40（高优先级）之间的数字，这个数字是通过公式 `unice=20-knice` 计算得来的。这样做是为了避免让系统调用服务例程返回一个负值，因为负值一般都表示错误。应用程序是不清楚系统调用服务例程对返回值所做的处理的，因为 C 库函数 `getpriority()`做了相反的计算操作，它将 `20-unice` 值返回给了调用程序。

下面简单看一下这两个系统调用的实现：[linux-2.6.39 getpriority](https://elixir.bootlin.com/linux/v2.6.39/source/kernel/sys.c#L236)，[glibc 库函数 getpriority](https://elixir.bootlin.com/glibc/glibc-2.36/source/sysdeps/unix/sysv/linux/getpriority.c#L35)

```c
#define TASK_NICE(p)        PRIO_TO_NICE((p)->static_prio)
int task_nice(const struct task_struct *p)
{
    return TASK_NICE(p);
}
SYSCALL_DEFINE2(getpriority, int, which, int, who)
{
    switch (which) {
        case PRIO_PROCESS:
            if (who)
                p = find_task_by_vpid(who);
            else
                p = current;
            if (p) {
                niceval = 20 - task_nice(p); /* 将返回值设置为正数 */
                if (niceval > retval)
                    retval = niceval;
            }
        break;
        ...
    }
}

SYSCALL_DEFINE3(setpriority, int, which, int, who, int, niceval)
{
    if (niceval < -20) /* 上下界判断 */
        niceval = -20;
    if (niceval > 19)
        niceval = 19;
    switch (which) {
        case PRIO_PROCESS:
            if (who)
                p = find_task_by_vpid(who);
            else
                p = current;
            if (p)
                error = set_one_prio(p, niceval, error);
            break;
    }
}

/* 库函数 getpriority */
#define PZERO 20
int __getpriority (enum __priority_which which, id_t who)
{
    int res;

    res = INLINE_SYSCALL (getpriority, 2, (int) which, who);
    if (res >= 0)
        res = PZERO - res; /* 再次转换 */
    return res;
}
```

通过`nice`命令可以以指定的`nice`值运行程序。如`nice -3 ./test`，指定nice值为3，如果想指定为负数需要再加一个负号`nice --3 ./test`（需要特权）。

`top` 中显示的 `NI` 一栏就是进程的 `nice` 值！`ps -l` 也可以查看。

``` bash
$ top
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
275133 yogurt    20   0 8589112   4.5g   4.3g S  45.8  14.3 109:53.85 VirtualBoxVM
  3894 yogurt    20   0 7470576 331140 125048 S   4.3   1.0  13:52.23 gnome-shell
     3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
     4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par_gp
```

从版本号为 `2.6.12` 的内核开始，Linux 提供了 `RLIMIT_NICE` 资源限制，即允许非特权进程提升 `nice` 值。非特权进程能够将自己的 `nice` 值最高提高到公式 `20−rlim_cur` 指定的值，其中 `rlim_cur` 是当前的 `RLIMIT_NICE` 软资源限制。如假设一个进程的 `RLIMIT_NICE` 软限制是 25，那么其 `nice`值可以被提高到−5。根据这个公式以及 `nice` 值的取值范围为（低）`+19～−20`（高）的事实可以得出 `RLIMIT_NICE` 的有效范围为（低）`1～40`（高）的结论。如何修改该值就涉及到下面的进程资源了。

## 进程资源

每个进程都用一组资源限值，它们可以用来限制进程能够消耗的各种系统资源。如在执行任意一个程序之前如果不想让它消耗太多资源，则可以设置该进程的资源限制。

### getrlimit/set

```c
#include <sys/time.h>
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);

struct rlimit {
    rlim_t rlim_cur;  /* Soft limit */
    rlim_t rlim_max;  /* Hard limit (ceiling for rlim_cur) */
};
```

`resource` 指定资源，这里有很多选项，查看`man page`等资料即可，下面只列几项内容：

| resource 取值   | 说明                           |
| --------------- | ------------------------------ |
| `RLIMIT_NICE`   | 能够为进程设置的最大 `nice` 值 |
| `RLIMIT_RTPRIO` | 能够为进程设置的最高实时优先级 |
| `RLIMIT_STACK`  | 进程栈的最大字节数             |

**软限制**规定了进程能够消耗的资源数量。一个进程可以将软限制调整为从 0 到硬限制之间的值。对于大多数资源来讲，**硬限制的唯一作用是为软限制设定了上限**。特权（`CAP_SYS_RESOURCE`）进程能够增大和缩小硬限制（只要其值仍然大于软限制），但非特权进程则只能缩小硬限制（这个行为是不可逆的）。取值为 `RLIM_INFINITY` 表示没有限制。`prlimit`可以打印出当前的资源限制情况。

```c
$ prlimit
RESOURCE   DESCRIPTION                              SOFT       HARD UNITS
AS         address space limit                 unlimited  unlimited bytes
CORE       max core file size                          0  unlimited bytes
CPU        CPU time                            unlimited  unlimited seconds
DATA       max data size                       unlimited  unlimited bytes
FSIZE      max file size                       unlimited  unlimited bytes
LOCKS      max number of file locks held       unlimited  unlimited locks
MEMLOCK    max locked-in-memory address space 4173918208 4173918208 bytes
MSGQUEUE   max bytes in POSIX mqueues             819200     819200 bytes
NICE       max nice prio allowed to raise              0          0
NOFILE     max number of open files                 1024    1048576 files
NPROC      max number of processes                127101     127101 processes
RSS        max resident set size               unlimited  unlimited bytes
RTPRIO     max real-time priority                      0          0
RTTIME     timeout for real-time tasks         unlimited  unlimited microsecs
SIGPENDING max number of pending signals          127101     127101 signals
STACK      max stack size                        8388608  unlimited bytes
```

## 实时进程调度

在一个系统上一般会同时运行交互式进程和后台进程，标准的内核调度算法一般能够为这些进程提供足够的性能和响应度。但实时应用对调度器有更加严格的要求：

- 实时应用必须要为外部输入提供担保最大响应时间。为了满足这种要求，内核必须要提供工具让高优先级进程能快速地取得 CPU 的控制权，**抢占当前运行的所有进程**。
- 高优先级进程应该能够保持互斥地访问 CPU **直至它完成**或自动释放 CPU。
- 实时应用应该能够精确地控制其组件进程的调度顺序。

SUSv3 规定的实时进程调度 API 提供了两个实时调度策略：`SCHED_RR` 和 `SCHED_FIFO`。使用这两种策略中任意一种策略进行调度的进程的优先级要高于默认的标准循环时间分享`SCHED_OTHER`策略来调度的进程。

每个实时策略允许一个优先级范围。在每个调度策略中，拥有高优先级的可运行进程在尝试访问 CPU 时总是优先于优先级较低的进程。Linux 提供了 99 个实时优先级，**其数值从 1（最低）～99（最高）**，并且这个取值范围同时适用于两个实时调度策略。

> 对于多处理器 Linux 系统（包括超线程系统）来讲，高优先级的可运行进程总是优先于优先级较低的进程的规则并不适用。在多处理器系统中，各个 CPU 拥有独立的运行队列（这种方式比使用一个系统层面的运行队列的性能要好），并且每个 CPU 的运行队列中的进程的优先级都局限于该队列。如假设一个双处理器系统中运行着三个进程，进程 A 的实时优先级为 20，并且它位于 CPU 0 的等待队列中，而该 CPU 当前正在运行优先级为 30 的进程 B，即使 CPU 1 正在运行优先级为 10 的进程 C，进程 A 还是需要等待 CPU 0。

### CPU 亲和性

设置进程的CPU亲和性用于限制进程在指定的CPU上运行，内核尝试了给进程保证**软 CPU 亲和力** — **在条件允许的情况下**进程重新被调度到原来的CPU 上运行。但这并不是强制性的，条件不允许仍然会将进程调度到其他CPU上运行。有时候需要为进程设置**硬 CPU 亲和力**，这样就能显式地将其限制在可用 CPU 中的一个或一组 CPU 上运行，原因如下：

- 可以避免由使高速缓冲器中的数据失效所带来的性能影响。
- 如果多个线程（或进程）访问同样的数据，那么当将它们限制在同样的 CPU 上的话可能会带来性能提升，因为它们无需竞争数据并且也不存在由此而产生的高速缓冲器未命中。
- 对于时间关键的应用程序来讲，可能需要为此应用程序预留一个或更多 CPU，而将系统中大多数进程限制在其他 CPU 上。

> 使用 isolcpus 内核启动参数能够将一个或更多 CPU 分离出常规的内核调度算法。将一个进程移到或移出被分离出来的 CPU 的唯一方式是使用本节介绍的 CPU 亲和力系统调用。isolcpus启动参数是实现上面列出的最后一种场景的首选方式，具体可参考内核源文件 Documentation/kernel-parameters.txt。

#### sched_setaffinity/get

```c
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sched.h>

int sched_setaffinity(pid_t pid, size_t cpusetsize,
                        const cpu_set_t *mask);
int sched_getaffinity(pid_t pid, size_t cpusetsize,
                        cpu_set_t *mask);
```

- `pid`：要设置的进程号，也可简单的用`0`来表示调用进程，也可用`gettid()`传入线程号
- `cpusetsize`：应该指定 `mask` 参数的字节数，通常设定为`sizeof(cpu_set_t)`
- `mask`：核的掩码。
- 返回值：成功返回0，失败返回`-1`，并设置`errno`
  - 如果`mask`中指定的 CPU 与系统中的所有 CPU 都不匹配，返回`EINVAL`错误

`taskset -p PID` 可查看当前进程的`mask`，可通过 `taskset -pc $pid` 来获取某线程与CPU核心的亲和性。

虽然 `cpu_set_t` 数据类型实现为一个位掩码，但应该将其看成是一个不透明的结构。

所有对这个结构的操作都应该使用宏来完成，下面是部分常用的：

```c
/* man CPU_SET */
#include <sched.h>

void CPU_ZERO(cpu_set_t *set);          /* 将 set 初始化为空 */
void CPU_SET(int cpu, cpu_set_t *set);  /* 将 CPU cpu 添加到 set 中 */
void CPU_CLR(int cpu, cpu_set_t *set);  /* 从 set 中删除 CPU cpu */
int  CPU_ISSET(int cpu, cpu_set_t *set);/* 在 CPU cpu 是 set 的一个成员时返回 true */
```

注意上面宏参数`cpu`编号是从0开始。

## 环境列表

每一个进程都有与其相关的称之为环境列表（environment list）的字符串数组，或简称为环境（environment）。其中每个字符串都以名称=值（name=value）形式定义。因此，环境是“名称-值”的成对集合，可存储任何信息。常将列表中的名称称为环境变量。

**新进程在创建之时，会继承其父进程的环境副本！**可通过命令`printenv`查看当前shell环境变量。

在 C 语言程序中，可以使用全局变量 `char **environ` 访问环境列表。（C 运行时启动代码定义了该变量并以环境列表位置为其赋值。）`environ` 与 `argv` 参数类似，指向一个以 `NULL` 结尾的指针列表，每个指针又指向一个以空字节终止的字符串。

```c
#include <stdio.h>

extern char **environ;

int main() {
    char **ep;
    for (ep = environ; *ep; ep++) {
        printf("%s\n", *ep);
    }
    return 0;
}
```

### getenv

`getenv()`函数能够从进程环境中检索单个值。

```c
#include <stdlib.h>

char *getenv(const char *name);
```

- name：指向要检索的环境变量名称的字符串。
- 返回值：如果找到该环境变量，则返回该环境变量的值，否则返回 NULL。

### putenv

有时，对进程来说，修改其环境很有用处。原因之一是这一修改对该进程后续创建的所有子进程均可见。另一个可能的原因在于设定某一变量，以求对于将要载入进程内存的新程序（“execed”）可见。从这个意义上讲，环境不仅是一种进程间通信的形式，还是程序间通信的方法。

```c
#include <stdlib.h>

int putenv(char *string);
```

- string：参数 string 是一指针，指向 name=value 形式的字符串。
- 返回值：如果成功，则返回 0，否则返回 非0值，并设置 errno。

调用 `putenv()` 函数后，该字符串就成为环境的一部分，换言之，`putenv` 函数将设定 `environ` 变量中某一元素的指向与 `string` 参数的指向位置相同，而非 `string` 参数所指向字符串的复制副本。

## proc 文件系统

对于系统中每个进程，内核都提供了相应的目录，命名为`/proc/PID`，其中 `PID` 是进程的ID。在此目录中的各种文件和子目录包含了进程的相关信息。

**可通过`man 5 proc`查看具体介绍**！描述的所有文件的作用以及其内容的含义。

| 文件    | 描述                                                         |
| ------- | ------------------------------------------------------------ |
| cmdline | 以\0 分隔的命令行参数                                        |
| cwd     | 指向当前工作目录的符号链接                                   |
| Environ | NAME=value 键值对环境列表，以\0 分隔                         |
| exe     | 指向正在执行文件的符号链接                                   |
| fd      | 文件目录，包含了指向由进程打开文件的符号链接                 |
| maps    | **内存映射**                                                 |
| mem     | 进程虚拟内存（在 I/O 操作之前必须调用 `lseek()`移至有效偏移量） |
| mounts  | 进程的安装点                                                 |
| root    | 指向根目录的符号链接                                         |
| status  | 各种信息（比如，进程 ID、凭证、内存使用量、信号）            |
| task    | 为进程中的**每个线程均包含一个子目录**（始自 Linux 2.6）     |

### maps

| 名称            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `address`       | 部分显示的是该段映射的虚拟地址。                             |
| `perms`         | 内存块权限 `READ|WRITE|EXEC`，可见性：`p(Private)`，`s(Share)` |
| `offset`        | 该段映射在文件/其它设备上的偏移量                            |
| `dev`           | 设备号                                                       |
| `inode`         | 文件或者设备的`inode`节点，0表示没有`inode`与该段内存关联    |
| `pathname`      | 路径名是与该段内存关联的文件路径，若为空，可能是通过`mmap`创建的匿名映射 |
| `[stack]`       | 进程（主线程）的栈空间                                       |
| `[stack:<tid>]` | 对应线程`tid`的栈空间                                        |
| `[heap]`        | 进程的堆空间                                                 |
| `[vdso]`        | virtual dynamic shared object，`man 7 vdso`查看介绍          |

```bash
$ ldd debug
linux-vdso.so.1 (0x00007ffc60be6000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f4b7c418000)
/lib64/ld-linux-x86-64.so.2 (0x00007f4b7c662000)

$ cat /proc/19898/maps
# The format of the file is:
# address                 perms offset  dev    inode        pathname
562bc9b9e000-562bc9b9f000 r--p 00000000 103:07 3422919      /tmp/debug
562bc9b9f000-562bc9ba0000 r-xp 00001000 103:07 3422919      /tmp/debug
562bc9ba0000-562bc9ba1000 r--p 00002000 103:07 3422919      /tmp/debug
562bc9ba1000-562bc9ba2000 r--p 00002000 103:07 3422919      /tmp/debug
562bc9ba2000-562bc9ba3000 rw-p 00003000 103:07 3422919      /tmp/debug
562bc9cb2000-562bc9cd3000 rw-p 00000000 00:00  0             [heap]
7f94721de000-7f94721e1000 rw-p 00000000 00:00  0
7f94721e1000-7f9472209000 r--p 00000000 103:07 12847599     /usr/lib/x86_64-linux-gnu/libc.so.6
7f9472209000-7f947239e000 r-xp 00028000 103:07 12847599     /usr/lib/x86_64-linux-gnu/libc.so.6
7f947239e000-7f94723f6000 r--p 001bd000 103:07 12847599     /usr/lib/x86_64-linux-gnu/libc.so.6
7f94723f6000-7f94723fa000 r--p 00214000 103:07 12847599     /usr/lib/x86_64-linux-gnu/libc.so.6
7f94723fa000-7f94723fc000 rw-p 00218000 103:07 12847599     /usr/lib/x86_64-linux-gnu/libc.so.6
7f94723fc000-7f9472409000 rw-p 00000000 00:00  0
7f9472424000-7f9472426000 rw-p 00000000 00:00  0
7f9472426000-7f9472428000 r--p 00000000 103:07 12847231     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7f9472428000-7f9472452000 r-xp 00002000 103:07 12847231     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7f9472452000-7f947245d000 r--p 0002c000 103:07 12847231     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
# 自己添加的文件映射
7f947245d000-7f947245e000 rw-p 00000000 103:07 12066390     /tmp/mmap_file1.c

7f947245e000-7f9472460000 r--p 00037000 103:07 12847231     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7f9472460000-7f9472462000 rw-p 00039000 103:07 12847231     /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2

7fff5f37d000-7fff5f39e000 rw-p 00000000 00:00  0             [stack]
7fff5f3df000-7fff5f3e3000 r--p 00000000 00:00  0             [vvar]
7fff5f3e3000-7fff5f3e5000 r-xp 00000000 00:00  0             [vdso]

ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0     [vsyscall]
```

## 参考资料

- 《Linux/Unix系统编程手册》
- 《操作系统之哲学原理 第二版》
- 《计算机体系结构基础》
- [PCB 进程控制块](https://www.cnblogs.com/yungyu16/p/13024626.html)
- [PCB(task_struct)源码](https://elixir.bootlin.com/linux/v5.4.190/source/include/linux/sched.h#L624)
- [操作系统--进程](https://github.com/szza/LearningNote/blob/master/4.%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/3_%E8%BF%9B%E7%A8%8B.md)
- [物理内存和虚拟内存的映射关系](https://www.bilibili.com/video/BV1KE411q7ee?p=77)
- [小林coding--为什么要有虚拟内存？](https://xiaolincoding.com/os/3_memory/vmem.html#linux-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)
- [小林coding--进程管理](https://xiaolincoding.com/os/4_process/process_base.html)
- [linux源码分析 - 进程](https://www.cnblogs.com/tolimit/p/4530370.html)
- [搞懂进程组、会话、控制终端关系，才能明白守护进程如何创建](https://zhuanlan.zhihu.com/p/266720121)
- [进程间的关系以及终端的概念](https://blog.csdn.net/qq_31828515/article/details/73849568)
- 《Linux系统编程——6.4虚拟内存管理》
- [/proc/pid/maps文件格式](https://blog.csdn.net/sunao2002002/article/details/84132505)

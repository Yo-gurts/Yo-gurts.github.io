---
title: Linux系统编程-线程
top_img: transparent
date: 2022-05-04 21:19:30
updated: 2022-05-04 21:19:30
tags:
  - Linux
  - 线程
  - 进程
categories: Linux
keywords:
description: Linux 中线程的相关知识
---

## 线程概念

Linux 下，线程又称为`LWP: light weight process`，轻量级进程。

| 进程                     | 线程                                               |
| ------------------------ | -------------------------------------------------- |
| **有独立的进程地址空间** | 没有独立的地址空间，多个线程共享                   |
| 有独立的 PCB             | 有独立的 PCB （但PCB中指向内存资源的三级页表相同） |
| 分配资源的最小单位       | CPU 执行的最小单位                                 |
| 查看 `ps aux / ps ajx`   | 查看线程号 `ps -Lf 进程id`                         |

**传统意义上的 UNIX 进程只是多线程程序的一个特例，该进程只包含一个线程。**

进程中创建线程后，原进程也降为线程了！进程相当于独居，创建线程后就变成合租（共享地址空间）。

![image-20220504212829735](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%BA%BF%E7%A8%8B/image-20220504212829735.png)

线程是 CPU 执行的最小单位，以下图为例，A分配的CPU时间为`3/5`，而 B, C 各只有 `1/5`。但也不是说线程越多越好，如下图右为的曲线。

![image-20220504213809444](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%BA%BF%E7%A8%8B/image-20220504213809444.png)

### 线程共享资源

1. 文件描述符表
2. 每种信号的处理方式
3. 当前工作目录
4. 用户 ID 和组 ID
5. 内存地址空间 (text/data/bss/heap/共享库 **全局变量**)，不包括栈。

### 线程非共享资源

1. 线程 ID
2. 处理器现场和栈指针（内核栈）
3. 独立的栈空间（用户空间栈）
4. errno 变量
5. 信号屏蔽字
6. 调度优先级

### 线程的优缺点

优点：

1. 提高程序并发性
2. 开销小
3. 数据通信、共享数据方便。只需将数据复制到共享（全局或堆）变量中即可。

缺点：

1. 库函数，不稳定
2. 调试、编写困难、gdb 不支持
3. 对信号支持不好。

优点相对突出，缺点均不是硬伤。Linux 下由于实现方法导致进程、线程差别不是很大。

## 常用 API

线程相关的函数的`man page`可能需要额外下载，`sudo apt install manpages-posix manpages-posix-dev`，也可通过`man -k pthread`查看相关的函数。

> 除了上面下载`manpage`，也可以[在线查看manpage](https://man7.org/linux/man-pages/index.html)
>
> `Pthread`相关的源码也可[在线查看](https://sourceware.org/git/?p=glibc.git;a=tree;f=nptl;h=d0ce23d37e9b77d6ff82ffdee0f7a1f8f137aa41;hb=HEAD)
>
> [glibc source code](https://elixir.bootlin.com/glibc/glibc-2.36/source)

### pthread_self

获取线程 id，类似与进程中的 `getpid()`！

线程 id 是在**进程地址空间内部**，用来标识线程身份的 id。通过 `ps -LF 进程id` 得到的是线程号`LWP`，是操作系统用来区分以分配CPU资源标识，与进程ID的功能类似。

```c
#include <pthread.h>

pthread_t pthread_self(void);
```

- 返回值：线程 id

### pthread_create

创建线程，编译和链接时加 `-lpthread`！

```c
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                    void *(*start_routine) (void *), void *arg);
```

- thread：线程 id，传出参数
- attr: 线程属性，默认为 NULL
- start_routine：线程入口函数，函数原型要与 `void *(*start_routine) (void *arg)` 一致
- arg：上一个参数“传入线程入口函数”的参数
- 返回值：0 成功，非 0 失败，返回的是 errno

循环创建多个线程：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <time.h>
#include <sys/types.h>
#include <unistd.h>
#include <pthread.h>

void *thread_func(void *arg)
{
    printf("this is %dth thread, pid = %d, tid = %ld\n", (int)arg, getpid(), pthread_self());
    return NULL;
}

int main() {
    long long ret, i;
    pthread_t tid;

    for (i = 0; i < 5; i++) {
        ret = pthread_create(&tid, NULL, thread_func, (void *)i);
        if (ret != 0) {
            printf("pthread_create error: %s\n", strerror(ret));
            exit(1);
        }
    }
    sleep(3);
    printf("main: pid = %d, tid = %ld\n", getpid(), pthread_self());
    return 0;
}
```

注意上面的代码中，线程入口函数的参数是一个整型变量，而不是指针！

通过值传递，避免与主线程中的变量冲突。若传入的参数是一个指针，因为线程创建需要一定的时间，而这段时间内，主线程可能会改变这个指针的值。导致结果与预期不一致。

检查线程返回错误号，不能使用 `perror()`，只能用 `strerror()`！

```c
    fprintf(stderr, "pthread_create error: %s\n", strerror(ret));
```

### pthread_exit

在线程函数函数内部调用，直接结束当前线程，可设置线程退出值。

```c
#include <pthread.h>

void pthread_exit(void *retval);
```

- retval：退出值，无则设 NULL

```c
void *thread_func(void *arg)
{
    int i = (int)arg;
    if (i == 2) {
        // exit(0);         // 退出进程（会把所有线程都退出！）
        // return NULL;     // 退出到调用者
        pthread_exit(NULL); // 退出当前线程
        pthread_exit((void*)0); // 设定线程退出值
    }
    printf("this is %dth thread, pid = %d, tid = %ld\n", i, getpid(), pthread_self());
    return NULL;
}

```

### pthread_join

阻塞回收线程，类似于进程中的 `waitpid()`！**注意，回收线程不一定是由父线程完成，兄弟线程之间可互相回收！**

```c
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);
```

- retval：线程的返回值，传出参数
- 返回值：0 成功，非 0 失败，返回的是 errno

### pthread_cancel

杀死线程，类似于进程中的 `kill()`！

被杀死的线程会调用 `pthread_exit()`，并且会返回 `PTHREAD_CANCELED`！

```c
#include <pthread.h>

int pthread_cancel(pthread_t thread);
```

- thread：要杀死的线程 id
- 返回值：0 成功，非 0 失败，返回的是 errno

```c
void *thread_func(void *arg)
{
    while(1) {
        // pthread_testcancel()
    }
    return NULL;
}
```

在上面的线程函数中，由于没有进入系统调用，无法用 `pthread_cancel()` 来杀死线程！

`pthread_cancel` 只有线程进入系统调用后，才能被杀死！如果子线程逻辑上没有调用系统调用，可以在程序中手动添加取消点 `pthread_testcancel()`。

### pthread_detach

设置线程分离，这样线程结束时，线程资源 PCB 会被自动释放，而不需要等待主线程回收！

```c
#include <pthread.h>

int pthread_detach(pthread_t thread);
```

- thread：要分离的线程 id
- 返回值：0 成功，非 0 失败，返回的是 errno

分离后，再次调用 `pthread_join()` 时，会报错 `Invalid argument`！

### 进程线程对比

| 进程                | 线程               |
| ------------------ | ------------------ |
| `fork()`           | `pthread_create()` |
| `getpid()`         | `pthread_self()`   |
| `exit()`           | `pthread_exit()`   |
| `wait()/waitpid()` | `pthread_join()`   |
| `kill()`           | `pthread_cancel()` |
|                    | `pthread_detach()` |

## 线程属性

> [pthread_attr](https://elixir.bootlin.com/glibc/glibc-2.36/source/sysdeps/nptl/internaltypes.h#L26)

在线程创建时，就可以设置线程的属性，主要有：

```c
struct pthread_attr {
    struct sched_param  schedparam;     /* 线程的调度参数：优先级 */
    int                 schedpolicy;    /* 线程调度策略 */
    int                 flags;          /* 线程的分离状态、作用域属性 */
    size_t              guardsize;      /* 线程栈末尾的警戒缓冲区大小 */
    void *              stackaddr;      /* 线程栈的位置（最低地址） */
    size_t s            tacksize;       /* 线程栈的位置（最低地址） */

    /* Allocated via a call to __pthread_attr_extension once needed.  */
    struct pthread_attr_extension *extension;
    void *unused;
};

struct sched_param {
    int sched_priority;
};

struct pthread_attr_extension {
    /* Affinity map.  */
    cpu_set_t *cpuset;
    size_t cpusetsize;

    sigset_t sigmask;
    bool sigmask_set;
};
```

一般不直接对线程属性实例进行修改，而是通过提供的函数来设置！**下面介绍的函数都是对线程属性实例进行了修改，也就是说，执行执行函数后也还没有任何一个线程受到这些属性的影响。只有用该实例去创建新线程时才生效**。

如果想修改当前运行中的线程的属性，往往有对应的不带`attr`的函数。

`man pthread_attr_init`中有获取线程属性并打印输出的例子，可以查看线程的默认属性。

### pthread_attr_init/destroy

对线程属性实例初始化和销毁的函数。

```c
#include <pthread.h>

pthread_attr_t attr;

int pthread_attr_init(pthread_attr_t *attr);    // 初始化线程属性
int pthread_attr_destroy(pthread_attr_t *attr); // 销毁线程属性
```

- attr：线程属性结构体指针
- 返回值：0 成功，非 0 失败，返回的是 errno

`init` 与 `destroy` 函数要配套使用，类似于 `malloc()` 与 `free()`！

可以看到上面`pthread_attr`的结构体中有指针成员，就会涉及到动态内存分配`malloc`和内存释放`free`，因此每次用完`attr`后需要调用`destroy`释放内存，避免内存泄露。

### pthread_attr_setdetachstate/get

设置线程分离，这样线程结束时，线程资源 PCB 会被自动释放，而不需要等待主线程回收！

```c
#include <pthread.h>

int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);
```

- `attr`：线程属性结构体指针
- `detachstate`：线程分离状态，可以是以下值：
  - `PTHREAD_CREATE_JOINABLE`：线程分离状态为非分离（默认选项）
  - `PTHREAD_CREATE_DETACHED`：线程分离状态为分离
- 返回值：0 成功，非 0 失败，返回的是 `errno`

```c
void *thread_func(void *arg)
{
    while(1) {
        sleep(1);
        printf("thread func. \n");
    }
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    // 设置线程属性为 分离
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

    int ret = pthread_create(&tid, &attr, thread_func, NULL);
    if (ret != 0) {
        printf("pthread_create error: %s\n", strerror(ret));
        exit(1);
    }

    pthread_attr_destroy(&attr);
    ret = pthread_join(tid, NULL);
    if (ret != 0) // 报错说明设置线程分离状态成功
        printf("pthread join error: %s\n", strerror(ret));
    // 主线程结束，子线程也被 结束！
    return 0;
}
```

### pthread_attr_setschedpolicy/get

设置线程调度的策略，支持的有：`SCHED_FIFO`, `SCHED_RR`, `SCHED_OTHER`，关于这几种策略的描述见：[man7 sched](https://man7.org/linux/man-pages/man7/sched.7.html#DESCRIPTION)

```c
#include <pthread.h>

int pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy);
int pthread_attr_getschedpolicy(const pthread_attr_t *attr, int *policy);
```

返回值：成功返回，失败返回错误号。

- `SCHED_FIFO`：设置了该策略的线程会一直运行，直到它被IO阻塞或被更高优先级的线程抢占，或者它调用`sched_yield`。
- `SCHED_RR`：基于`SCHED_FIFO`，但设置了最大执行时间`quantum`，执行这么长时间后就会中止，并放入该优先级的调度队列末尾。
- `SCHED_OTHER`：`Linux`的默认策略，是一种相对*公平*的调度策略，类似与时间片轮转，但高优先级分配的时间会更多。

### pthread_attr_setschedparam/get

`SCHED_FIFO`是基于优先级抢占的，该函数用于设置线程进行调度时的优先级。

```c
#include <pthread.h>

int pthread_attr_setschedparam(pthread_attr_t *attr,
                               const struct sched_param *param);
int pthread_attr_getschedparam(const pthread_attr_t *attr,
                               struct sched_param *param)

struct sched_param {
    int sched_priority;     /* Scheduling priority */
};
```

### pthread_attr_setinheritsched/get

设置线程的继承属性，其实只有**调度属性**可以继承。

```c
#include <pthread.h>

int pthread_attr_setinheritsched(pthread_attr_t *attr,
                                 int inheritsched);
int pthread_attr_getinheritsched(const pthread_attr_t *attr,
                                 int *inheritsched);
```

`inheritsched`只有两种取值：

- `PTHREAD_INHERIT_SCHED`：新线程将继承调用`pthread_create`的线程的调度策略。
- `PTHREAD_EXPLICIT_SCHED`：新线程的调度策略以线程属性中指定的为准。

**也就是说，在使用`pthread_attr_setschedpolicy`时，必须也要设置`PTHREAD_EXPLICIT_SCHED`，否则调度策略不会生效**。

### pthread_attr_setscope/get

线程作用域属性描述特定线程将与哪些线程竞争资源。

```c
#include <pthread.h>

int pthread_attr_setscope(pthread_attr_t *attr, int scope);
int pthread_attr_getscope(const pthread_attr_t *attr, int *scope);
```

线程可以在两种竞争域内竞争资源，也是`scope`的两种取值：

- `PTHREAD_SCOPE_SYSTEM`：系统域，与系统中的所有线程。一个具有系统域的线程将与整个系统中所有具有系统域的线程按照优先级竞争处理器资源，进行调度。
- `PTHREAD_SCOPE_PROCESS`：进程域，与同一进程内的其他线程竞争。

### pthread_attr_setguardsize/get

设置线程栈保护区的大小，**默认保护大小与系统页面大小相同**。

在线程栈的末尾分配之一至少`guardsize`字节的区域作为堆栈保护区，如果一个线程溢出它的堆栈到保护区，在大多数硬架构上，会产生`SIGSEGV`信号，从而通知它溢出。

```c
#include <pthread.h>

int pthread_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);
int pthread_attr_getguardsize(const pthread_attr_t *attr, size_t *guardsize);
```

### pthread_attr_setstackaddr/get

当进程栈地址空间不够用时，指定新建线程使用由`malloc`分配的空间作为自己的栈空间。

[Do not use these functions!](https://man7.org/linux/man-pages/man3/pthread_attr_setstackaddr.3.html#NOTES)

### pthread_attr_setstacksize/get

设置线程栈的大小，默认线程栈的大小为`8M`。

```c
#include <pthread.h>

int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
int pthread_attr_getstacksize(const pthread_attr_t *attr, size_t *stacksize);
```

当进程中有很多线程时，可能需要减小每个线程栈的默认大小，防止进程的地址空间不够用。

当线程调用的函数会分配很大的局部变量或者函数调用层次很深时，可能需要增大线程栈的默认大小。

### pthread_attr_setaffinity_np/get

设置线程的CPU亲和性，让线程在指定的某一个核或一组核上运行。

```c
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <pthread.h>

int pthread_attr_setaffinity_np(pthread_attr_t *attr,
                                size_t cpusetsize, const cpu_set_t *cpuset);
int pthread_attr_getaffinity_np(const pthread_attr_t *attr,
                                size_t cpusetsize, cpu_set_t *cpuset);
```

- `cpusetsize`：应该指定 `cpuset` 参数的字节数，通常设定为`sizeof(cpu_set_t)`。
- `cpuset`：核的掩码。

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

### pthread_getattr_np

获取当前线程的属性，写入到`attr`中。

```c
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <pthread.h>

int pthread_getattr_np(pthread_t thread, pthread_attr_t *attr);
```

### 运行时调整线程属性

除了上面在线程创建时设置属性，部分属性也支持运行时进行调整。

| 静态设置                      | 运行时                   |
| ----------------------------- | ------------------------ |
| `pthread_attr_setschedparam`  | `pthread_setschedparam`  |
| `pthread_attr_setaffinity_np` | `pthread_setaffinity_np` |

### CPU亲和性

设置进程在某一个核或一组核上运行，在某些情况下可以提升性能。如果该进程有多个线程，它们都只能在指定的一组核上面运行。也可单独为某一个线程设置亲和性。

```c
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

## 线程注意事项

1. 主线程退出其他线程不退出，主线程应调用 pthread_exit
2. 避免僵尸线程：
    - pthread_join
    - pthread_detach
    - pthread_create 指定分离属性
    - 被 join 线程可能在 join 函数返回前就释放完自己的所有内存资源，所以不应当返回被回收线程栈中的值;
3. malloc 和 mmap 申请的内存可以被其他线程释放
4. 应避免在多线程模型中调用 fork 除非，马上 exec，子进程中只有调用 fork 的线程存在，其他线程在子进程中均 pthread_exit
5. 信号的复杂语义很难和多线程共存，应避免在多线程引入信号机制 （多线程中，信号由哪个线程处理不确定！每个线程各有信号屏蔽字mask，共享未决信号集，如果想指定某个线程处理特定信号，可通过设置其他线程的信号屏蔽字）

## 一次性初始化

> Linux-Unix系统编程手册——31.2节

多线程程序有时有这样的需求：不管创建了多少线程，有些初始化动作只能发生一次。如果由主线程来创建新线程，那么这一点易如反掌，可以在创建依赖于该初始化的线程之前进行初始化。不过，对于库函数而言，这样处理就不可行，因为调用者在初次调用库函数之前可能已经创建了这些线程。故而需要这样的库函数：无论首次为任何线程所调用，都会执行初始化动作。

### pthread_once 函数

保证无论多少线程、无论调用多少次`pthread_once`，都只会执行一次`init_routine`初始化函数。

```c
#include <pthread.h>

int pthread_once(pthread_once_t *once_control,
                 void (*init_routine)(void));
pthread_once_t once_control = PTHREAD_ONCE_INIT;
```

- `once_control`：必须是一指针，指向初始化为 `PTHREAD_ONCE_INIT` 的静态变量。
- `init_routine`：需要执行的函数，该函数没有任何参数。
- 成功返回`0`。

## 相关资料

- [线程原理--三级页表](https://www.bilibili.com/video/BV1KE411q7ee?p=148&spm_id_from=pageDriver)
- [循环创建子线程](https://www.bilibili.com/video/BV1KE411q7ee?p=153&spm_id_from=pageDriver)
- [在线查看manpage](https://man7.org/linux/man-pages/index.html)
- [glibc source code](https://elixir.bootlin.com/glibc/glibc-2.36/source)
- [线程属性](https://www.cnblogs.com/FREMONT/p/9480376.html)
- [线程/进程和核绑定（CPU亲和性）](https://blog.csdn.net/qq_38232598/article/details/114263105)

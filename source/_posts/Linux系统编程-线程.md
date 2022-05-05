---
title: Linux系统编程-线程
top_img: transparent
date: 2022-05-04 21:19:30
updated: 2022-05-04 21:19:30
tags:
  - Linux
  - 线程
categories: Linux
keywords:
description: Linux 中线程的相关知识
---

## 线程

Linux 下，线程又称为`LWP: light weight process`，轻量级进程。

| 进程                     | 线程                                               |
| ------------------------ | -------------------------------------------------- |
| **有独立的进程地址空间** | 没有独立的地址空间，多个线程共享                   |
| 有独立的 PCB             | 有独立的 PCB （但PCB中指向内存资源的三级页表相同） |
| 分配资源的最小单位       | CPU 执行的最小单位                                 |
| 查看 `ps aux / ps ajx`   | 查看线程号 `ps -Lf 进程id`                         |

进程中创建线程后，原进程也降为进程了！进程相当于独居，创建线程后就变成合租（共享地址空间）。

![image-20220504212829735](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%BA%BF%E7%A8%8B/image-20220504212829735.png)

线程是 CPU 执行的最小单位，以下图为例，A分配的CPU时间为`3/5`，而 B, C 各只有 `1/5`。但也不是说线程越多越好，如下图右为的曲线。

![image-20220504213809444](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E7%BA%BF%E7%A8%8B/image-20220504213809444.png)

### 线程共享资源

1. 文件描述符表
2. 每种信号的处理方式
3. 当前工作目录
4. 用户 ID 和组 ID
5. 内存地址空间 (text/data/bss/heap/共享库 **全局变量**)

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
3. 数据通信、共享数据方便

缺点：
1. 库函数，不稳定
2. 调试、编写困难、gdb 不支持
3. 对信号支持不好。

优点相对突出，缺点均不是硬伤。Linux 下由于实现方法导致进程、线程差别不是很大。

## pthread_self 函数

获取线程 id，类似与进程中的 `getpid()`！

线程 id 是在**进程地址空间内部**，用来标识线程身份的 id。通过 `ps -LF 进程id` 得到的是线程号`LWP`，是操作系统用来区分以分配CPU资源标识，与进程ID的功能类似。

```c
#include <pthread.h>

pthread_t pthread_self(void);
```

- 返回值：线程 id

## pthread_create 函数

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


## pthread_exit 函数

退出调用该函数的线程。

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

## pthread_join 函数

阻塞回收线程，类似于进程中的 `waitpid()`！

```c
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);
```

- retval：线程的返回值，传出参数
- 返回值：0 成功，非 0 失败，返回的是 errno

## pthread_cancel 函数

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

## pthread_detach 函数

设置线程分离，这样线程退出时，线程资源 PCB 会被自动释放，而不需要等待主线程回收！

```c
#include <pthread.h>

int pthread_detach(pthread_t thread);
```

- thread：要分离的线程 id
- 返回值：0 成功，非 0 失败，返回的是 errno

分离后，再次调用 `pthread_join()` 时，会报错 `Invalid argument`！

## 进程线程对比

| 进程                | 线程               |
| ------------------ | ------------------ |
| `fork()`           | `pthread_create()` |
| `getpid()`         | `pthread_self()`   |
| `exit()`           | `pthread_exit()`   |
| `wait()/waitpid()` | `pthread_join()`   |
| `kill()`           | `pthread_cancel()` |
|                    | `pthread_detach()` |

## 线程属性

在线程创建时，就可以设置线程的属性，比如分离线程，设置线程的优先级，设置线程的栈大小等！一般不直接对线程属性结构体进行设置，而是通过提供的函数来设置！

```c
#include <pthread.h>

pthread_attr_t *attr;

int pthread_attr_init(pthread_attr_t *attr);    // 初始化线程属性
int pthread_attr_destroy(pthread_attr_t *attr); // 销毁线程属性
```

- attr：线程属性结构体指针
- 返回值：0 成功，非 0 失败，返回的是 errno

`init` 与 `destroy` 函数要配套使用，类似于 `malloc()` 与 `free()`！

### 线程分离状态

```c
#include <pthread.h>

int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);
```

- attr：线程属性结构体指针
- detachstate：线程分离状态，可以是以下值：
  - PTHREAD_CREATE_JOINABLE：线程分离状态为非分离（默认选项）
  - PTHREAD_CREATE_DETACHED：线程分离状态为分离
- 返回值：0 成功，非 0 失败，返回的是 errno

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

## 相关资料

- [线程原理--三级页表](https://www.bilibili.com/video/BV1KE411q7ee?p=148&spm_id_from=pageDriver)
- [循环创建子线程](https://www.bilibili.com/video/BV1KE411q7ee?p=153&spm_id_from=pageDriver)

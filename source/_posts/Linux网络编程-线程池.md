---
title: Linux网络编程-线程池
top_img: transparent
date: 2022-05-16 13:42:10
updated: 2022-05-16 13:42:10
tags:
  - 线程
  - 线程池
  - Linux
categories: Linux
keywords:
description: Linux 下线程池的实现
---

在多任务并发处理的场景下，如果每来一个任务，就新建一个线程来处理，虽然功能上没问题，但由于线程的创建和销毁会带来很大的开销。线程池就是通过预先创建一定数量的线程，当有任务来时，就将任务分配给一个线程去处理！

## 线程池模型

下面是线程池的结构！

```c
struct threadpool_t {
    pthread_mutex_t lock;           /* 用于锁住本结构体 */
    pthread_mutex_t thread_counter; /* 记录忙状态线程个数的锁 */

    pthread_cond_t queue_not_full;  /* 当任务队列满时，添加任务任务的线程阻塞，等待此条件变量 */
    pthread_cond_t queue_not_empty; /* 任务队列不为空时，通知等待任务的线程 */

    pthread_t *threads;             /* 存放线程池中每个线程的 tid 数组 */
    pthread_t adjust_tid;           /* 存管理线程tid */
    threadpool_task_t *task_queue;  /* 任务队列，数组首地址 */

    int min_thr_num;                /* 线程池最小线程数 */
    int max_thr_num;                /* 线程池最大线程数 */
    int live_thr_num;               /* 当前存活线程个数 */
    int busy_thr_num;               /* 忙状态线程个数 */
    int wait_exit_thr_num;          /* 要销毁的线程个数 */

    int queue_front;                /* task_queue 队头下标 */
    int queue_rear;                 /* task_queue 队尾下标 */
    int queue_size;                 /* task_queue 队中实际任务数 */
    int queue_max_size;             /* task_queue 队中可容纳的任务数上限 */

    bool shutdown;                  /* 标志位，线程池使用状态，true或false */
};
```

### 线程数组和任务队列

线程池的核心也就是线程数组和任务队列！

![image](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-%E7%BA%BF%E7%A8%8B%E6%B1%A0/threadpool1.png)

**1. 线程数组：threads。**

注意此处的实现中，线程的数量不是固定的，可在最小线程数和最大线程数之间动态调整。当突发任务数量大于最小线程数时，线程池会创建新线程以处理突发任务，任务处理完之后，若没有其他任务需要处理，**管理线程**会定时回收空闲的线程，但保证线程池中线程数不会小于最小线程数。

当突发任务数量大于最大线程数时，线程池创建的线程数会被限制在最大线程数。

基于上述原理，线程池结构中，以下5个成员变量用于辅助管理线程数组：

```c
pthread_t *threads;     /* 存放线程池中每个线程的 tid 数组 */

int min_thr_num;        /* 线程池最小线程数 */
int max_thr_num;        /* 线程池最大线程数 */
int live_thr_num;       /* 当前存活线程个数 */
int busy_thr_num;       /* 忙状态线程个数 */
int wait_exit_thr_num;  /* 要销毁的线程个数 */
```

**2. 任务队列：task_queue。**

与线程数组不同，任务队列是一个固定大小的**循环队列**，用于存放待处理的任务，线程池需要处理的**任务类型**和**参数**不一，无法预先定义在线程中，只能通过任务传入。也就是说，任务队列中存储的内容要包括**任务类型`（void *(*func)(void*arg)）`**和**参数`（void *arg）`**。

使用线程池时，只需调用函数将任务添加到任务队列尾部即可，线程池会自动为任务分配线程处理。但当任务队列满时，线程池会拒绝添加新任务。

```c
typedef struct {
    void *(*function)(void *);  /* 函数指针，回调函数 */
    void *arg;                  /* 上面函数的参数 */
} threadpool_task_t;            /* 各子线程任务结构体 */

threadpool_task_t *task_queue;  /* 任务队列，数组首地址 */

int queue_front;                /* task_queue 队头下标 */
int queue_rear;                 /* task_queue 队尾下标 */
int queue_size;                 /* task_queue 队中实际任务数 */
int queue_max_size;             /* task_queue 队中可容纳的任务数上限 */
```

### 线程池管理线程

线程池中除了专门用于处理任务的线程，还需要有一个**管理线程**，用于管理线程池中的任务线程。线程池的动态扩容和销毁都是通过管理线程来完成的，管理线程**定期**根据当前线程数组和任务数组情况，决定是否需要扩容或销毁线程。

```c
pthread_t adjust_tid;           /* 存管理线程tid */
```

### 条件变量与互斥锁

线程池的任务分配也可看作**生产者消费者模型**，任务队列中的元素是**产品**，线程池中的每一个线程都是**消费者**，向线程池中添加任务的是**生产者**（一般为主线程）。也就是说，线程池可以看作**单生产者、多消费者模型**。

一个任务只需要也只能分配给一个任务线程处理！因此，需要配合条件变量和互斥锁来实现任务分配。

- **消费者**：没有任务时，任务线程都阻塞在条件变量`queue_not_empty`上，等待新任务到来。
- **生产者**：任务队列满时，主线程`pthreadpool_add`阻塞在条件变量`queue_not_full`上，等待任务队列空间可用。

- `thread_counter`：记录忙状态线程个数的锁！

*此处可以先回顾基于条件变量实现的生产者消费者模型！此处的任务线程处理逻辑与消费者一致！*

## threadpool_create 函数

根据输入的参数创建一个线程池，并返回线程池的指针。

```c
threadpool_t *threadpool_create(int min_thr_num, int max_thr_num, int queue_max_size);
```

- `min_thr_num`：线程池中最少含有线程数
- `max_thr_num`：线程池中最多最多线程数
- `queue_max_size`：线程池中任务队列的最大容量
- 返回值：线程池指针，如果创建失败，返回NULL

## threadpool_add 函数

添加一个任务到线程池的任务队列中，如果线程池任务队列已满，则阻塞等待。

```c
int threadpool_add(threadpool_t *pool, void*(*function)(void *arg), void *arg);
```

- `pool`：线程池指针
- `function`：任务函数
- `arg`：任务函数参数
- 返回值：0，添加成功；不会失败，只会阻塞等待！

## threadpool_destroy 函数

销毁管理线程、通知所有空闲线程结束、等待忙线程结束，释放线程池所占空间。

```c
int threadpool_destroy(threadpool_t *pool);
```

- `pool`：线程池指针
- 返回值：0，销毁成功；pool 为NULL 时，返回-1。

## 实现代码

下面的简单实现可以用来理解线程池的原理，但性能并不理想！

```c
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <assert.h>
#include <stdio.h>
#include <string.h>
#include <signal.h>
#include <stdbool.h>
#include <errno.h>
#include "threadpool.h"

#define DEFAULT_TIME 10                 /* 10s检测一次 */
#define MIN_WAIT_TASK_NUM 10            /* 如果queue_size > MIN_WAIT_TASK_NUM 添加新的线程到线程池 */
#define DEFAULT_THREAD_VARY 10          /* 每次创建和销毁线程的个数 */

typedef struct {
    void *(*function)(void *);          /* 函数指针，回调函数 */
    void *arg;                          /* 上面函数的参数 */
} threadpool_task_t;                    /* 各子线程任务结构体 */

/* 描述线程池相关信息 */

struct threadpool_t {
    pthread_mutex_t lock;           /* 用于锁住本结构体 */
    pthread_mutex_t thread_counter; /* 记录忙状态线程个数的锁 */

    pthread_cond_t queue_not_full;  /* 当任务队列满时，添加任务任务的线程阻塞，等待此条件变量 */
    pthread_cond_t queue_not_empty; /* 任务队列不为空时，通知等待任务的线程 */

    pthread_t *threads;             /* 存放线程池中每个线程的tid 数组 */
    pthread_t adjust_tid;           /* 存管理线程tid */
    threadpool_task_t *task_queue;  /* 任务队列，数组首地址 */

    int min_thr_num;                /* 线程池最小线程数 */
    int max_thr_num;                /* 线程池最大线程数 */
    int live_thr_num;               /* 当前存活线程个数 */
    int busy_thr_num;               /* 忙状态线程个数 */
    int wait_exit_thr_num;          /* 要销毁的线程个数 */

    int queue_front;                /* task_queue 队头下标 */
    int queue_rear;                 /* task_queue 队尾下标 */
    int queue_size;                 /* task_queue 队中实际任务数 */
    int queue_max_size;             /* task_queue 队中可容纳的任务数上限 */

    bool shutdown;                  /* 标志位，线程池使用状态，true或false */
};

void *threadpool_thread(void *threadpool);
void *adjust_thread(void *threadpool);

int is_thread_alive(pthread_t tid);
int threadpool_free(threadpool_t *pool);

threadpool_t *threadpool_create(int min_thr_num, int max_thr_num, int queue_max_size)
{
    int i;
    threadpool_t *pool = NULL;          /* 线程池 结构体 */

    do {
        if ((pool = (threadpool_t *)malloc(sizeof(threadpool_t))) == NULL) {
            printf("malloc threadpool failed!\n");
            break;
        }

        pool->min_thr_num = min_thr_num;
        pool->max_thr_num = max_thr_num;
        pool->busy_thr_num = 0;
        pool->live_thr_num = min_thr_num;               /* 活着的线程数 初值=最小线程数 */
        pool->wait_exit_thr_num = 0;
        pool->queue_size = 0;                           /* 有0个产品 */
        pool->queue_max_size = queue_max_size;          /* 最大任务队列数 */
        pool->queue_front = 0;
        pool->queue_rear = 0;
        pool->shutdown = false;                         /* 不关闭线程池 */

        /* 根据最大线程上限数， 给工作线程数组开辟空间, 并清零 */
        pool->threads = (pthread_t *)malloc(sizeof(pthread_t)*max_thr_num);
        if (pool->threads == NULL) {
            printf("malloc threads failed!\n");
            break;
        }
        memset(pool->threads, 0, sizeof(pthread_t)*max_thr_num);

        /* 给 任务队列 开辟空间 */
        pool->task_queue = (threadpool_task_t *)malloc(sizeof(threadpool_task_t)*queue_max_size);
        if (pool->task_queue == NULL) {
            printf("malloc task failed\n");
            break;
        }

        /* 初始化互斥琐、条件变量 */
        if (pthread_mutex_init(&(pool->lock), NULL) != 0
                || pthread_mutex_init(&(pool->thread_counter), NULL) != 0
                || pthread_cond_init(&(pool->queue_not_empty), NULL) != 0
                || pthread_cond_init(&(pool->queue_not_full), NULL) != 0)
        {
            printf("init lock or cond failed!\n");
            break;
        }

        /* 启动 min_thr_num 个 work thread */
        for (i = 0; i < min_thr_num; i++) {
            pthread_create(&pool->threads[i], NULL, threadpool_thread, (void *)pool);
            printf("start thread 0x%x ... \n", (unsigned int)pool->threads[i]);
        }
        pthread_create(&(pool->adjust_tid), NULL, adjust_thread, (void *)pool);     /* 创建管理者线程 */

        return pool;
    } while(0);

    threadpool_free(pool);      /* 前面代码调用失败时，释放poll存储空间 */

    return NULL;
}

/* 向线程池中 添加一个任务 */
int threadpool_add(threadpool_t *pool, void*(*function)(void *arg), void *arg)
{
    pthread_mutex_lock(&(pool->lock));

    /* == 为真，队列已经满， 调wait阻塞 */
    while ((pool->queue_size == pool->queue_max_size) && (!pool->shutdown)) {
        pthread_cond_wait(&(pool->queue_not_full), &(pool->lock));
    }

    if (pool->shutdown) {
        pthread_cond_broadcast(&(pool->queue_not_empty));
        pthread_mutex_unlock(&(pool->lock));
        return 0;
    }

    /* 清空 工作线程 调用的回调函数 的参数arg */
    if (pool->task_queue[pool->queue_rear].arg != NULL) {
        pool->task_queue[pool->queue_rear].arg = NULL;
    }

    /* 添加任务到任务队列里 */
    pool->task_queue[pool->queue_rear].function = function;
    pool->task_queue[pool->queue_rear].arg = arg;
    pool->queue_rear = (pool->queue_rear + 1) % pool->queue_max_size;       /* 队尾指针移动, 模拟环形 */
    pool->queue_size++;

    /* 添加完任务后，队列不为空，唤醒线程池中 等待处理任务的线程 */
    pthread_cond_signal(&(pool->queue_not_empty));
    pthread_mutex_unlock(&(pool->lock));

    return 0;
}

/* 线程池中各个工作线程 */
void *threadpool_thread(void *threadpool)
{
    threadpool_t *pool = (threadpool_t *)threadpool;
    threadpool_task_t task;

    while (true) {
        /* 刚创建出线程，等待任务队列里有任务，否则阻塞等待任务队列里有任务后再唤醒接收任务 */
        pthread_mutex_lock(&(pool->lock));

        /* queue_size == 0 说明没有任务，调 wait 阻塞在条件变量上, 若有任务，跳过该while */
        while ((pool->queue_size == 0) && (!pool->shutdown)) {
            printf("thread 0x%x is waiting\n", (unsigned int)pthread_self());
            pthread_cond_wait(&(pool->queue_not_empty), &(pool->lock));

            /* 清除指定数目的空闲线程，如果要结束的线程个数大小0，结束线程 */
            if (pool->wait_exit_thr_num > 0) {
                pool->wait_exit_thr_num--;

                /* 如果线程池中线程个数大于最小值时，可以结束线程 */
                if (pool->live_thr_num > pool->min_thr_num) {
                    printf("thread 0x%x is exiting\n", (unsigned int)pthread_self());
                    pool->live_thr_num--;
                    pthread_mutex_unlock(&(pool->lock));
                    pthread_exit(NULL);
                }
            }
        }

        /* 如果指定了true，要关闭线程池里的每个线程，自行退出处理---销毁线程池 */
        if (pool->shutdown) {
            pthread_mutex_unlock(&(pool->lock));
            printf("thread 0x%x is exiting\n", (unsigned int)pthread_self());
            pthread_detach(pthread_self());
            pthread_exit(NULL);     /* 线程自行结束 */
        }

        /* 从任务队列里获取任务, 是一个出队操作 */
        task.function = pool->task_queue[pool->queue_front].function;
        task.arg = pool->task_queue[pool->queue_front].arg;

        pool->queue_front = (pool->queue_front + 1) % pool->queue_max_size;       /* 出队，模拟环形队列 */
        pool->queue_size--;

        /* 任务取出后，立即将 线程池琐 释放 */
        pthread_mutex_unlock(&(pool->lock));

        /* 通知可以有新的任务添加进来 */
        pthread_cond_broadcast(&(pool->queue_not_full));

        /* 执行任务 */
        printf("thread 0x%x start working\n", (unsigned int)pthread_self());
        pthread_mutex_lock(&(pool->thread_counter));                            /* 忙状态线程数变量琐 */
        pool->busy_thr_num++;                                                   /* 忙状态线程数+1 */
        pthread_mutex_unlock(&(pool->thread_counter));

        (*(task.function))(task.arg);                                           /* 执行回调函数任务 */
        //task.function(task.arg);                                              /* 执行回调函数任务 */

        /* 任务结束处理 */
        printf("thread 0x%x end working\n", (unsigned int)pthread_self());
        pthread_mutex_lock(&(pool->thread_counter));
        pool->busy_thr_num--;                                       /* 处理掉一个任务，忙状态数线程数-1 */
        pthread_mutex_unlock(&(pool->thread_counter));
    }

    pthread_exit(NULL);
}

/* 管理线程 */
void *adjust_thread(void *threadpool)
{
    int i;
    threadpool_t *pool = (threadpool_t *)threadpool;

    while(!pool->shutdown) {
        sleep(DEFAULT_TIME);                        /* 定时 对线程池管理 */

        pthread_mutex_lock(&(pool->lock));
        int queue_size = pool->queue_size;          /* 关注 任务数 */
        int live_thr_num = pool->live_thr_num;      /* 存活 线程数 */
        pthread_mutex_unlock(&(pool->lock));

        pthread_mutex_lock(&(pool->thread_counter));
        int busy_thr_num = pool->busy_thr_num;      /* 忙着的线程数 */
        pthread_mutex_unlock(&(pool->thread_counter));

        /* 创建新线程 算法： 任务数大于最小线程池个数, 且存活的线程数少于最大线程个数时 如：30>=10 && 40<100*/
        if (queue_size >= MIN_WAIT_TASK_NUM && live_thr_num < pool->max_thr_num) {
            pthread_mutex_lock(&(pool->lock));
            int add = 0;

            /*一次增加 DEFAULT_THREAD 个线程*/
            for (i = 0; i < pool->max_thr_num && add < DEFAULT_THREAD_VARY
                    && pool->live_thr_num < pool->max_thr_num; i++) {
                if (pool->threads[i] == 0 || !is_thread_alive(pool->threads[i])) {
                    pthread_create(&(pool->threads[i]), NULL, threadpool_thread, (void *)pool);
                    add++;
                    pool->live_thr_num++;
                }
            }

            pthread_mutex_unlock(&(pool->lock));
        }

        /* 销毁多余的空闲线程 算法：忙线程X2 小于 存活的线程数 且 存活的线程数 大于 最小线程数时*/
        if ((busy_thr_num * 2) < live_thr_num  &&  live_thr_num > pool->min_thr_num) {

            /* 一次销毁DEFAULT_THREAD个线程, 隨機10個即可 */
            pthread_mutex_lock(&(pool->lock));
            pool->wait_exit_thr_num = DEFAULT_THREAD_VARY;      /* 要销毁的线程数 设置为10 */
            pthread_mutex_unlock(&(pool->lock));

            for (i = 0; i < DEFAULT_THREAD_VARY; i++) {
                /* 通知处在空闲状态的线程, 他们会自行终止*/
                pthread_cond_signal(&(pool->queue_not_empty));
            }
        }
    }

    return NULL;
}

int threadpool_destroy(threadpool_t *pool)
{
    int i;
    if (pool == NULL) {
        return -1;
    }
    pool->shutdown = true;

    /*先销毁管理线程*/
    pthread_join(pool->adjust_tid, NULL);

    for (i = 0; i < pool->live_thr_num; i++) {
        /*通知所有的空闲线程*/
        pthread_cond_broadcast(&(pool->queue_not_empty));
    }
    for (i = 0; i < pool->live_thr_num; i++) {
        pthread_join(pool->threads[i], NULL);
    }
    threadpool_free(pool);

    return 0;
}

int threadpool_free(threadpool_t *pool)
{
    if (pool == NULL) {
        return -1;
    }

    if (pool->task_queue) {
        free(pool->task_queue);
    }
    if (pool->threads) {
        free(pool->threads);
        pthread_mutex_lock(&(pool->lock));
        pthread_mutex_destroy(&(pool->lock));
        pthread_mutex_lock(&(pool->thread_counter));
        pthread_mutex_destroy(&(pool->thread_counter));
        pthread_cond_destroy(&(pool->queue_not_empty));
        pthread_cond_destroy(&(pool->queue_not_full));
    }
    free(pool);
    pool = NULL;

    return 0;
}

int threadpool_all_threadnum(threadpool_t *pool)
{
    int all_threadnum = -1;                 // 总线程数

    pthread_mutex_lock(&(pool->lock));
    all_threadnum = pool->live_thr_num;     // 存活线程数
    pthread_mutex_unlock(&(pool->lock));

    return all_threadnum;
}

int threadpool_busy_threadnum(threadpool_t *pool)
{
    int busy_threadnum = -1;                // 忙线程数

    pthread_mutex_lock(&(pool->thread_counter));
    busy_threadnum = pool->busy_thr_num;
    pthread_mutex_unlock(&(pool->thread_counter));

    return busy_threadnum;
}

int is_thread_alive(pthread_t tid)
{
    int kill_rc = pthread_kill(tid, 0);     //发0号信号，测试线程是否存活
    if (kill_rc == ESRCH) {
        return false;
    }
    return true;
}
```

## 相关资料

- [线程池原理分析](https://www.bilibili.com/video/BV1iJ411S7UA?p=91)
- [A simple C++11 Thread Pool implementation](https://github.com/progschj/ThreadPool)
- [github threadpool](https://github.com/search?q=threadpool)

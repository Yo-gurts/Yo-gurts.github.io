---
title: Linux系统编程-同步与锁
top_img: transparent
date: 2022-05-06 22:00:59
updated: 2022-05-06 22:00:59
tags:
  - Linux
  - 同步
  - 锁
categories: Linux
keywords:
description: Linux 线程同步与锁的介绍
---

编程中、通信中所说的同步与生活中大家印象中的同步概念略有差异。“同”字应是指协同、协助、互相配合。**主旨在协同步调，按预定的先后次序运行**

线程同步，指一个线程发出某一功能调用时，在没有得到结果之前，该调用不返回。同时其它线程为保证数据一致性，不能调用该功能。

线程同步可以通过锁来实现！与锁相关的部分函数的`man page`需要单独安装。

```bash
sudo apt install manpages-posix-dev
```

## mutex 互斥量（互斥锁）

每个线程在对资源操作前都尝试先加锁，成功加锁才能操作，操作结束解锁。

资源还是共享的，线程间也还是**竞争**的，但通过“锁”就将资源的访问变成互斥操作。

互斥锁实质上是操作系统提供的一把“建议锁”（又称“协同锁”），建议程序中有多线程访问共享资源的时候使用该机制。但，并没有强制限定。因此，即使有了 mutex，如果有线程不按规则来访问数据，依然会造成数据混乱。

**尽量保证锁的粒度，越小越好（访问共享数据前，加锁。访问结束立即解锁）**

```c
#include <pthread.h>

// 静态初始化
pthead_mutex_t muetx = PTHREAD_MUTEX_INITIALIZER;
// 下面几个函数都是 成功返回 0， 失败返回错误号
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
        const pthread_mutexattr_t *restrict attr);
// 销毁锁
int pthread_mutex_destroy(pthread_mutex_t *mutex);
// 加锁，若当前锁已被其它线程加锁，则阻塞，直到被其它线程解锁
int pthread_mutex_lock(pthread_mutex_t *mutex);
// trylock 尝试加锁，成功返回0，失败返回 EBUSY，不会阻塞
int pthread_mutex_trylock(pthread_mutex_t *mutex);
// 解锁
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

### pthread_mutex_init 函数

初始化互斥锁。

如果互斥锁 mutex 是静态分配的（定义在全局，或加了 static 关键字修饰），可以直接使用宏进行初始化。

`pthead_mutex_t muetx = PTHREAD_MUTEX_INITIALIZER;`

局部变量应采用动态初始化，即使用 `pthread_mutex_init(&mutex, NULL);` 初始化。

```c
#include <pthread.h>

int pthread_mutex_init(pthread_mutex_t *restrict mutex,
        const pthread_mutexattr_t *restrict attr);
```

- mutex：互斥锁的地址
- attr：互斥锁的属性，如果为 NULL，则使用默认属性（线程间共享）
- 返回值：0（成功），非 0（失败）

- restrict：限定该指针不能拷贝到其他指针，`*restrict p = &a, *p2 = *p`会报错。

### 死锁

1. 线程试图对同一个互斥量 A 加锁两次。
2. 线程 1 拥有 A 锁，请求获得 B 锁；线程 2 拥有 B 锁，请求获得 A 锁

### example

测试1：在不加锁的情况下，下面的`i++`和`printf`之间可能被另一个线程打断，导致输出的`i`的大小不连续！

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>
#include <unistd.h>

int i = 0;

void *thread_func(void *args) {
    while(1) {
        i++;
        printf("thread_func: %d \n", i);
        usleep(10);
    }
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    while(1) {
        i++;
        printf("main_func: %d \n", i);
        usleep(10);
    }
    return 0;
}

// result:
// thread_func: 17982
// main_func: 17982
// thread_func: 17984
// main_func: 17984
// thread_func: 17986
// main_func: 17986
// thread_func: 17988
// main_func: 17988
// thread_func: 17990
// main_func: 17990
// thread_func: 17992
// main_func: 17992
// thread_func: 17994
```

加锁保证数据不被另一个线程修改！

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>
#include <unistd.h>


int i = 0;
pthread_mutex_t mutex_i = PTHREAD_MUTEX_INITIALIZER;

void *thread_func(void *args) {
    while(1) {
        pthread_mutex_lock(&mutex_i);  // 上锁
        i++;
        printf("thread_func: %d \n", i);
        pthread_mutex_unlock(&mutex_i);  // 解锁
        usleep(10);
    }
    return NULL;
}

int main() {

    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);

    while(1) {
        pthread_mutex_lock(&mutex_i);  // 上锁
        i++;
        printf("main_func: %d \n", i);
        pthread_mutex_unlock(&mutex_i);  // 解锁
        usleep(10);
    }
    return 0;
}

// result:
// main_func: 19946
// thread_func: 19947
// main_func: 19948
// thread_func: 19949
// main_func: 19950
// thread_func: 19951
// main_func: 19952
// thread_func: 19953
// main_func: 19954
// thread_func: 19955
// main_func: 19956
```

## rwlock 读写锁

- 读共享，写独占。
- 写锁优先级高于读锁（读锁写锁同时来，优先处理写锁。读锁加锁成功后，写锁也必须等待读锁释放）。
- 锁还是只有一把，但分**以读模式加锁**和**以写模式加锁**两种。

```c
#include <pthread.h>

// 静态初始化
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
// 初始化读写锁
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
        const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
// 加读锁，
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);

int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

## cond 条件变量

条件变量本身不是锁！但它也可以造成线程阻塞。通常与互斥锁配合使用。给多线程提供一个会合的场所。

```c
#include <pthread.h>

// 静态初始化
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
// 初始化条件变量
int pthread_cond_init(pthread_cond_t *restrict cond,
        const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);

// 阻塞等待条件变量满足
int pthread_cond_wait(pthread_cond_t *restrict cond,
        pthread_mutex_t *restrict mutex);
// 阻塞等待条件变量满足，最长等待到 abstime 时刻
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
        pthread_mutex_t *restrict mutex,
        const struct timespec *restrict abstime);
// 唤醒（至少）一个阻塞在该条件变量上的线程，一般就理解为唤醒一个线程。
int pthread_cond_signal(pthread_cond_t *cond);
// 唤醒所有阻塞在该条件变量上的线程
int pthread_cond_broadcast(pthread_cond_t *cond);
```

### pthread_cond_wait 函数

阻塞等待一个条件变量！

```c
int pthread_cond_wait(pthread_cond_t *restrict cond,
        pthread_mutex_t *restrict mutex);
```

- cond：条件变量的地址
- mutex：互斥锁的地址
- 返回值：0（成功），非 0（失败）

函数作用：
1. 阻塞等待条件变量 `cond` 满足
2. 释放已掌握的互斥锁（解锁互斥量）相当于 `pthread_mutex_unlock(&mutex)`; **1.2.两步为一个原子操作**。
3. 当被唤醒，`pthread_cond_wait` 函数返回时，解除阻塞并重新申请获取互斥锁 `pthread_mutex_lock(&mutex)`;

此函数的原理不太好理解，建议看视频介绍[条件变量原理](https://www.bilibili.com/video/BV1KE411q7ee?p=176&t=290.8)！

### pthread_cond_timedwait 函数

阻塞等待一个条件变量，最长等待到 abstime 时刻。

```c
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
        pthread_mutex_t *restrict mutex,
        const struct timespec *restrict abstime);
```

- cond：条件变量的地址
- mutex：互斥锁的地址
- abstime：等待的时间，绝对时间（即从1970年1月1日零时起的纳秒数）
- 返回值：0（成功），非 0（失败）

### 生产者消费者实现

此处实现的产品是后生产的产品先被消费（栈）！

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

struct msg {
    int num;
    struct msg *next;
};

struct msg *product;
pthread_cond_t has_product = PTHREAD_COND_INITIALIZER;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *consumer(void *arg) {
    struct msg *p;
    while(1) {
        pthread_mutex_lock(&lock);
        while(product == NULL) {    // 多消费者情况下，防止产品被其他消费者消费
            // 注意理解此函数会进行的操作（解锁+阻塞，收到唤醒解除阻塞+加锁）
            pthread_cond_wait(&has_product, &lock);
        }

        p = product;
        product = product->next;    // 模拟消费一个产品
        // 将 printf 也放在锁的范围，这样才能观察到产品是后生产先被消费
        printf("consumer %lu --- product %d\n", pthread_self(), p->num);
        pthread_mutex_unlock(&lock);

        free(p);
        sleep(rand() % 5);
    }
    return NULL;
}

void *producer(void *arg) {
    struct msg *p;
    while(1) {
        p = malloc(sizeof(struct msg));
        p->num = rand() % 1000 + 1;    // 模拟生产一个产品

        pthread_mutex_lock(&lock);
        // 将 printf 也放在锁的范围，这样才能观察到产品是后生产先被消费
        printf("producer %lu --- product %d\n", pthread_self(), p->num);
        p->next = product;
        product = p;    // 这种实现是栈的方式，后生产的产品先出
        pthread_mutex_unlock(&lock);

        // 将一个阻塞在该条件变量上的线程唤醒，当前代码同时唤醒多个线程，会导致错误
        pthread_cond_signal(&has_product);
        sleep(rand() % 5);
    }
    return NULL;
}

int main() {

    pthread_t cons, prod;
    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);
    return 0;
}
```

## sem 信号量

进化版的互斥锁，加锁的线程数量 N 可以自定义，即同时访问数据的线程数量可以为 N。

同样是建议锁，虽然支持多次加锁，但信号量本身并不能保证数据不紊乱，而是需要底层数据结构支持并发操作。

**信号和信号量毫无关系！！**

```c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_destroy(sem_t *sem);

int sem_wait(sem_t *sem);
int sem_trywait(sem_t *sem);
int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);

int sem_post(sem_t *sem);
```

信号量可以应用与线程、**进程**之间的同步！

### sem_init 函数

初始化信号量，可以设置是否共享，以及初始值。

```c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
```

- sem：信号量的地址
- pshared：是否共享，0（用于线程间同步），1（用于进程间同步，在进程间共享）
- value：N 值，可同时访问数据的线程数量
- 返回值：0（成功），-1（失败）

### 生产者消费者实现

这个与上面基于条件变量的实现逻辑不同，建议先看视频[基于信号量实现生产者消费者模型](https://www.bilibili.com/video/BV1KE411q7ee?p=183&spm_id_from=pageDriver)
理解实现的原理，再看具体实现代码。

注意，对信号量初始化时，将 `product_num` 初始化为0，但仍然可以调用`sem_post`, `sem_wait`
等函数，这与上面所说的最多只能有 N (这里为0) 是不是矛盾？怎么实现多个消费者？

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

#define NUM 10

sem_t blank_num;
sem_t product_num;

int products[NUM];

void *producer(void *arg) {
    int i = 0;
    while(1) {
        sem_wait(&blank_num);   // 空格为空时阻塞 blank_num--

        // 生产一个产品
        products[i] = rand() % 1000;
        printf("produce product: %d in index: %d\n", products[i], i);
        sem_post(&product_num); // product_num++

        i = (i+1) % NUM;    // 环形队列
        sleep(rand() % 3);
    }
    return NULL;
}

void *consumer(void *arg) {
    int i = 0;
    while(1) {
        sem_wait(&product_num); // 当产品为空时阻塞 product_num--

        // 消费产品
        printf("consume product: %d in index: %d\n", products[i], i);
        sem_post(&blank_num);   // blank_num++

        i = (i+1) % NUM;
        sleep(rand() % 3);
    }
    return NULL;
}

int main() {
    pthread_t prod, cons;

    sem_init(&product_num, 0, 0);
    sem_init(&blank_num, 0, NUM);

    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);
    return 0;
}
```

## 对比总结

| pthread_mutex_t       | pthread_rwlock_t                                       | pthread_cond_t                                  | sem_t         |
| --------------------- | ------------------------------------------------------ | ----------------------------------------------- | ------------- |
| pthread_mutex_init    | pthread_rwlock_init                                    | pthread_cond_init                               | sem_init      |
| pthread_mutex_lock    | pthread_rwlock_wrlock<br />pthread_rwlock_rdlock       | pthread_cond_wait                               | sem_wait      |
| pthread_mutex_trylock | pthread_rwlock_trywrlock<br />pthread_rwlock_tryrdlock |                                                 | sem_trywait   |
|                       |                                                        | pthread_cond_timedwait                          | sem_timedwait |
| pthread_mutex_unlock  | pthread_rwlock_unlock                                  | pthread_cond_signal<br />pthread_cond_broadcast | sem_post      |
| pthread_mutex_destroy | pthread_rwlock_destroy                                 | pthread_cond_destroy                            | sem_destroy   |

## 相关资料

- [读写锁优先级](https://www.bilibili.com/video/BV1KE411q7ee?p=172&spm_id_from=pageDriver)
- [条件变量原理](https://www.bilibili.com/video/BV1KE411q7ee?p=176&t=290.8)
- [生产者-多个消费者](https://www.bilibili.com/video/BV1KE411q7ee?p=180&t=471.6)
- [基于信号量实现生产者消费者模型](https://www.bilibili.com/video/BV1KE411q7ee?p=183&spm_id_from=pageDriver)
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

## rwlock 读写锁

- 读共享，写独占。
- 写锁优先级高于读锁（读锁写锁同时来，优先处理写锁。读锁加锁成功后，写锁也必须等待读锁释放）。
- 锁还是只有一把，但分**以读模式加锁**和**以写模式加锁**两种。

```c
#include <pthread.h>

// 初始化读写锁
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
        const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
// 加读锁，
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);

int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

## 相关资料

- [读写锁优先级](https://www.bilibili.com/video/BV1KE411q7ee?p=172&spm_id_from=pageDriver)

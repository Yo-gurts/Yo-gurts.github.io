---
title: Linux网络编程-并发服务器
top_img: transparent
date: 2022-08-02 14:07:42
updated: 2022-08-02 14:07:42
tags:
  - Linux
  - socket
  - select
  - poll
  - epoll
categories: Linux
keywords:
description:
---

前面`socket`中创建的`server`只有一个主线程，只能连接一个客户端。要实现可同时连接多个客户端，有几种方法：

- 为每一个连接创建一个**子进程**处理。
- 为每一个连接创建一个**子线程**处理。
- 单线程，但使用`select, poll, epoll`等 **`IO` 复用**算法。

## 多进程并发

每产生一个连接时，就创建一个子进程去处理，父进程只负责监听以及**回收子进程**！

回收子进程不能放在父进程的循环逻辑中，一种办法是通过注册信号`SIGCHLD`捕捉函数，在信号处理函数中完成子进程回收。

```c
1. socket()             // 创建监听套接字 lfd
2. bind()
3. listen()
4. while(1) {
    cfd = accept()      // 与客户端通信的 socket fd
    pid = fork()        // 创建子进程
    if (pid == 0) {     // 子进程
        close(lfd)      // 关闭用于建立连接的套接字 lfd
        read()          // 读数据，并处理
        ....
        write()
    } else if (pid > 0) {   // 父进程
        close(cfd)      // 父进程中不需要 真连接 的 fd
        continue
    }
}

5. while(waitpid());    // 父进程回收子进程，通过信号捕捉函数：`SIGCHLD`
```

## 多线程并发

与多进程并发类似，只是为每一个连接创建一个线程。与多进程相比，回收子线程更简单：

- 可直接使用`pthread_detach`分离子线程，自动回收，该方案无法接收线程返回值。
- 也可创建一个子线程，专门用于回收用于处理连接的子线程（兄弟线程可互相回收）。

```c
1. socket()             // 创建监听套接字 lfd
2. bind()
3. listen()
4. while(1) {
    cfd = accept()      // 与客户端通信的 socket fd
    pthread_create(&tid, NULL, thread_main, NULL)   // 创建子进程
    pthread_detach(tid)    // 分离子线程，自动回收
}

5. thread_main() {      // 子线程处理函数
    close(lfd)          // 关闭用于建立连接的套接字 lfd
    read()              // 读数据，并处理
    ....
    write()
}
```

## 多路IO复用

在上面的多进程或多线程并发服务器中，父进程总是阻塞等待连接（阻塞在`lfd`）、子进程大部分时间阻塞等待消息（阻塞在`cfd`），无法做其他事情。多路IO转接服务器也叫做多任务IO服务器，该类服务器实现的主旨思想是，不再由应用程序自己监视客户端连接，取而代之**由内核替应用程序监视文件**。

与多进程和多线程技术相比，`I/O`多路复用技术的**最大优势是系统开销小**，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

`select, poll, epoll`都是监听**文件描述符**，传递给这些监听函数的参数都是需要监听的文件描述符列表，返回值为有对应监听事件的文件描述符列表。**这些函数监听文件描述符，没有限制必须是`socket`建立的文件描述符，普通文件、管道等所有文件描述符都可以被监听**。

## Select

如下图所示，`select`监听对应的`socket fd`，连接请求也就是`lfd`上有消息到达，其他客户端的请求也就是对应的`cfd`上有消息到达。

![image-20220802151005095](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8/image-20220802151005095.png)

`select`函数中，有三个传入传出参数，**传入**表示要监听的读、写、异常文件描述符，**传出**表示有对应事件的文件描述符。

![img](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8/v2-320be0c91e2a376199b1d5eef626758e_b.gif)

```c
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, struct timeval *timeout);
```

- `nfds`：要监听的最大文件描述符`+1`
- `readfds`：**传入传出参数**，要监听的读集合
- `writefds`：**传入传出参数**，要监听的写集合，一般传 NULL
- `exceptfds`：**传入传出参数**，要监听的异常集合，一般传 NULL
- `timeout`：
  - `>0` 设置监听超时时长
  - `NULL` 阻塞监听
  - `0` 非阻塞监听，轮询
- 返回值：
  - `>=0` 所有监听集合（3个）中，满足对应事件的总数
  - `-1` 异常，`errno`

`fd_set` 是一个二进制集合，在c标准库中其长度是固定的（1024），每一位表示对应位的文件描述符，相关的修改只能通过提供的操作函数。

```c
void FD_CLR(int fd, fd_set *set);   // 将一个描述符移出监听集合
int  FD_ISSET(int fd, fd_set *set); // 判断描述符是否在监听集合中
void FD_SET(int fd, fd_set *set);   // 将一个描述符加入监听集合
void FD_ZERO(fd_set *set);          // 清零监听集合
```

### Select 基本流程

```c
lfd = socket();     // 创建监听套接字 lfd
bind();
listen();
fd_set rset;        // 创建读监听集合
fd_set allset;      // 由于作为select参数的rset每次调用都会被修改，用allset作为监听集合
FD_ZERO(&allset);   // 清空
FD_SET(lfd, allset) // 将 lfd 加入监听集合
while(1) {
    rset = allset;  // 保存监听集合
    ret = select(lfd+1, &rset, NULL, NULL, NULL);   // 监听对应集合
    if (ret > 0) {
        if (FD_ISSET(lfd, &rset)) {
            cfd = accept();
            FD_SET(cfd, &allset);
        }

        for (i = lfd+1; i <= maxfd; i++) {  // 遍历集合，处理对应事件
            if (FD_ISSET(lfd, &rset)) {
                read()
                ....    // 处理数据
            }
        }
    }
}
```

### Select 优缺点

**优点**：跨平台！

**缺点**：

- 监听上限受文件描述符限制，最大1024（不是数量最大1024，而是文件描述符的大小不能超过1024）。
- 检测满足条件的`fd`比较麻烦，提高了编码难度！**但检测逻辑写好了，性能并不比后面的`poll, epoll`低**。

检测条件这里，如果文件描述符很分散，比如 3 和 1023，如果遍历`fd_set`，效率很低，最好的办法是用一个数组来记录需要监听的文件描述符，只不过需要自己实现，比较麻烦。

## Poll

`Poll`只是一个半成品，对上面的`Select`进行了一点优化：

- 它本身就用数组来记录文件描述符，方便处理很分散的文件描述符。
- 不在受限与文件描述符的大小，监听文件描述符的大小不受限。

```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;         // 待监听的文件描述符
    short events;     // 待监听的文件描述符对应的监听时间：POLLIN, POLLOUT, POLLERR
    short revents;    // 传入时给0。如果满足对应时间，返回非0：POLLIN, POLLOUT, POLLERR
}
```

- `fds`：监听的文件描述符【数组】
- `nfds`：监听数组的，实际有效监听个数
- `timeout`：单位 `ms`
  - `>0` 设置监听超时时长
  - `-1` 阻塞监听
  - `0` 非阻塞监听
- 返回值：- 返回值：
  - `>=0` 满足对应监听事件的总数。
  - `-1` 异常，`errno`

### Poll 基本流程

```c
lfd = socket();     // 创建监听套接字 lfd
bind();
listen();
struct pollfd fds[1024];    // 监听数组
fds[0].fd = lfd;
fds[0].events = POLLIN;
fds[0].revents = 0;

while(1) {
    ret = poll(fds, maxi+1, -1);   // 监听对应集合
    if (ret > 0) {
        for (i = 0; i <= maxi; i++) {
            // 遍历数组，依次判断是否有对应事件
        }
    }
}
```

### Poll 优缺点

优点：

- 自带数组结构，可将监听事件集合和返回事件集合分离。
- 可拓展监听上限，超出1024限制。

缺点：

- 不能跨平台，只能在 `Linux / Unix` 下使用。
- 仍然无法直接定位满足监听事件的文件描述符。
- 编码难度较大。

### 突破1024限制

默认情况下，一个进程可打开的文件描述符上限就`1024`，当服务器连接的客户端超过1024时，就无法再为其分配文件描述符，也就无法建立连接。

**查看限制**：

- `cat /proc/sys/fs/file-max`：当前计算机所能打开的最大文件个数。受硬件影响。
- `ulimit -a`（`open files`）：当前用户下的进程，默认可打开文件描述符个数。缺省为 1024

**修改限制**：

```bash
# sudo vim /etc/security/limits.conf    # 在文件末尾加入一下内容
*               soft    nofile          65536   # 设置默认值，可以直接借助命令ulimit修改。【注销用户，使其生效】
*               hard    nofile          100000  # 设置命令上限
```

接着就可使用`ulimit -n num`修改进程可打开的文件描述符上限，受到上面设置的`hard`限制。

## Epoll

在`man epoll`中对`epoll`有较详细的描述。其在内核中的实现采用了两种数据结构：

- **监听红黑树**，用于维护监听的描述符列表。
- **就绪链表**，维护一个有监听事件响应的描述符列表。

如下图所示，使用`epoll_ctl()`向监听红黑树中添加、删除文件描述符。有数据到达时，通过回调函数，将对应的文件描述符插入到就绪链表中；当使用`epoll_wait()`时，只判断该**就绪链表**是否为空，不为空就返回就绪链表。

![img](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8/v2-70f8e9bc1a028d252c01c32329e49341_b.gif)

使用红黑树只是为了管理需要监听的文件描述符，原理上来讲，使用普通的数组也可以记录需要监听的文件描述符，比如方案一：添加监听的描述符通过遍历数组找到第一个空位，删除操作符也需要遍历数组确定位置（而且会留下空位）。这种方式使用的空间复杂度低，但时间复杂度高。方案二：直接使用一个足够长的数组，用文件描述符下标做映射。添加、删除的时间复杂度都是`O(1)`，单空间复杂度很高，因为并不仅仅是记录文件描述符`int`，还需要记录需要监听的事件等信息。而红黑树就是对时间、空间复杂度折中的一个方案，空间复杂度低，时间复杂度`O(\log n)`。

还有一种方案：用哈希表来管理需要监听的文件描述符，但哈希表有个致命缺陷，没有人知道哈希表增删的时候会在什么时候扩容/缩容，这可能会导致某个 `epoll_ctl` 操作增删文件描述符的时候，会比其它操作长几百倍。而为了保证对外提供的服务的质量，每次 `epoll_ctl` 的时间应该至少是平均的。

### epoll_create 函数

创建一棵监听红黑树。

```c
#include <sys/epoll.h>

int epoll_create(int size);
```

- `size`：创建的红黑树的监听节点数量，仅供内核参考。
- 返回值：
  - `>0` 指向新创建的红黑树的根节点的 `fd`。
  - `-1` 失败，`errno`。

### epoll_ctl 函数

操作监听红黑树。

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

- `epfd`：`epoll_create`函数的返回值，红黑树的根节点
- `op`：对该监听红黑数所做的操作：
  - `EPOLL_CTL_ADD` 添加`fd`到 监听红黑树；
  - `EPOLL_CTL_MOD` 修改`fd`在 监听红黑树上的监听事件；
  - `EPOLL_CTL_DEL` 将一个`fd`从 监听红黑树上摘下（取消监听）。
- `fd`：待监听的 `fd`。
- `event`：本质 `struct epoll_event` 结构体 变量地址

```c
typedef union epoll_data {  // union 联合体
    void        *ptr;
    int          fd;    // 传出参数，对应监听时间的 fd
    uint32_t     u32;   // 无用
    uint64_t     u64;   // 无用
} epoll_data_t;

struct epoll_event {
    uint32_t     events;    // 监听事件类型 EPOLLIN/ EPOLLOUT/ EPOLL
    epoll_data_t data;      /* User data variable */
};
```

### epoll_wait 函数

判断该**就绪链表**是否为空，不为空就返回就绪链表，否则阻塞等待。

```c
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events,
                int maxevents, int timeout);
```

- `epfd`：`epoll_create`函数的返回值，红黑树的根节点。
- `events`：传出参数，**数组**，满足监听条件的那些 `fd` 结构体。数组大小预先就给定`LEN`！
- `maxevents`：`events`数组 元素的总个数，也就是上面的`LEN`。
- `timeout`：单位 `ms`
  - `>0` 设置监听超时时长
  - `-1` 阻塞监听
  - `0` 非阻塞监听
- 返回值：
  - `>=0` 满足对应监听事件的总数，也就是`events`中有效成员的数量。
  - `-1` 异常，`errno`

### ET和LT模式

`epoll`是`Linux`下多路复用IO接口`select/poll`的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，另一点原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入`Ready`队列的描述符集合就行了。

`epoll`事件有两种模型：

- `Edge Triggered (ET)` 边缘触发（上升沿）只有数据到来才触发，不管缓存区中是否还有数据。
- `Level Triggered (LT)` 水平触发（高电平）只要有数据都会触发。

![image-20220802175026443](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-%E5%B9%B6%E5%8F%91%E6%9C%8D%E5%8A%A1%E5%99%A8/image-20220802175026443.png)

## 相关资料

- [深入浅出理解select、poll、epoll的实现](https://zhuanlan.zhihu.com/p/367591714)
- [Linux网络编程](https://www.bilibili.com/video/BV1iJ411S7UA?p=75&vd_source=ae9b8ed3bda4e74634d81a57039d0b6e)

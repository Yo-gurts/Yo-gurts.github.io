---
title: Linux系统编程-进程通信
top_img: transparent
date: 2022-04-25 21:24:09
updated: 2022-08-23 23:09:09
tags:
  - Linux
  - 进程
categories: Linux
keywords:
description: 介绍Linux进程通信的几种方式
---

## 进程通信

进程之间是存在内存隔离的，所以进程间通信相对于线程间通信比较麻烦，对于两个完全隔离的事物肯定是没办法进行通信的！不过好在进程并不是完全隔离的，每个进程的地址空间中都有一段共用的内核空间，这就相当于在两个进程之间架了一座桥。所以下面介绍的很多通信方法都是基于这座桥（内核）实现的。

![img](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1/4-%E8%BF%9B%E7%A8%8B%E7%A9%BA%E9%97%B4.jpg)

> 本文中的部分图片来源于：[小林coding](https://xiaolincoding.com/os/4_process/process_commu.html#%E7%AE%A1%E9%81%93)

### 标识

不过内核地址空间也不能随便给各个进程用，而且空间那么大，进程怎么知道要去哪里读写呢？第一个问题好解决，每当'我'需要和其他进程通信，我就先向内核申请一块“中继内存”，然后就可以向该地址中写数据了。不过第二个问题，其他需要读数据的进程怎么知道去哪里读呢？给读进程指定内存地址？但进程未启动的时候是不可能知道'我'向内核申请的内存地址信息的。唉，圆不了话了，反正就是想一想文件的使用，我同样不知道文件的内存地址，但我只要知道文件的路径，不管开多少个进程去打开该文件，读到的都是相同的内容（虽然文件是存储在磁盘而不是内存上，不过道理类似）。

这里的“路径+文件名”就构成了这个文件对象的唯一标识，我们在进程通信时，**也需要一个能唯一标识“中继内存”的标识符**，实际上，`Unix Socket`、`fifo`就是采用了“路径+文件名”作为标识！而信号量、消息队列、共享内存这三种通信方式在Linux上有两种标准实现，`System V`和`POSIX`，前者通过在使用**一个整数**`key`来标识对象，后者使用**对象名**来实现（跟文件很像了）。`Socket`使用`ip+port`作为标识。

虽然大部分进程间通信都需要通过“唯一标识”去获取对应的中继对象，不过也不绝对，由于在`fork`时，子进程会复制父进程的信息，所以也可以基于此设计，比如管道`pipe`。

| 方式                                | 标识                             | 限制           |
| ----------------------------------- | -------------------------------- | -------------- |
| pipe                                | 无（只能用于有血缘关系的进程间） | 容量           |
| fifo                                | 路径+文件名`/tmp/myfifo`         | 容量           |
| System V 信号量、消息队列、共享内存 | 整数                             | ipcs -l        |
| POSIX 消息队列                      |                                  | 消息大小、数量 |
| POSIX 信号量、消息队列、共享内存    | 对象名`/myobject`                |                |
| 文件映射（内存映射）                | 路径+文件名`/tmp/myfile`         |                |
| 匿名映射（内存映射）                | 无（只能用于有血缘关系的进程间） |                |
| Unix Domain Socket                  | 路径+文件名 `/tmp/mysock.sock`   |                |
| Socket                              | IP + port                        |                |

### 方式汇总

Linux 下一切皆文件，用于进程通信的对象也不例外，所以`pipe`、`fifo`、`socket`甚至消息队列，打开后都会分配一个文件描述符，通过文件描述符来读写数据（消息队列有点特殊）。

![processIPC](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1/processIPC.png)

## pipe

> 标识：无，通过 fork 时复制文件描述符的原理。

`Pipe` 管道**并不会**在内核或文件系统上产生唯一的标识，它基于`fork()`调用时，子进程会拷贝父进程的文件描述符列表，达到多个进程共享同一个管道的文件描述符，从而实现进程间通信。也就限制了`Pipe`管道只能用于有血缘关系的进程间通信。

`fork()`之前创建管道`pipe(fd[2])`，这样每个子进程都有读端`fd[0]`和写端`fd[1]`。与所有文件描述符一样，可以使用 `read()`和`write()`系统调用来在管道上执行 I/O。

![img](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1/7-%E7%AE%A1%E9%81%93-pipe-fork-%E5%8D%95%E5%90%91%E9%80%9A%E4%BF%A1.jpg)

```c
#include <unistd.h>

int pipe(int pipefd[2]);
```

- `pipefd[0]` 读端
- `pipefd[1]` 写端
- 返回值：
  - 成功返回 `0`
  - 失败翻译 `-1` ，并设置 `errno`

### example

```c
#include <stdio.h>
#include <sys/types.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

int main() {
    int fd[2];
    pipe(fd);   // 创建 pipe

    char *str = "child write date!\n";
    char buf[100];  // = "";  // 不设置为空可能会出现异常
    if (fork() == 0) {  // child process
        printf("child: %d, %d\n", fd[0], fd[1]);
        close(fd[0]);   // 关闭读端
        int ret = write(fd[1], str, strlen(str));
        printf("child write bytes ret: %d, len: %ld\n", ret, strlen(str));
        close(fd[1]);
    } else {
        printf("parent: %d, %d\n", fd[0], fd[1]);
        close(fd[1]);   // 关闭写端
        sleep(1);
        int ret = read(fd[0], buf, 18);
        printf("parent read bytes ret: %d, len: %ld\n", ret, strlen(buf));
        printf("read buf: %s", buf);    // 对比两种输出方式
        write(STDOUT_FILENO, buf, ret);
        close(fd[0]);
    }
    return 0;
}
```

### 管道的读写行为

**最好同时只有一个读端和一个写端**！

- 一个管道是一个字节流，不存在消息或消息边界的概念。

- 通过管道传递的数据是顺序的，在管道中无法使用 `lseek()` 来随机地访问数据。

- **管道的存储容量是有限的**，一般为 `64kbytes`，容量满之后，写端被阻塞。

    > 虽然管道的容量大小被限制为`64kbytes`，但如果需要，进程也可通过一定方法`fcntl`来修改管道的容量。

- 读管道：
  1. 管道中有数据，`read` 返回实际读取的字节数，并将数据写入缓冲区
  2. 管道无数据：
      - 无写端：`read` 返回 0（类似读到文件末尾）
      - 有写端：`read` 阻塞等待，直到有数据写入管道

- 写管道：
  1. 无读端：异常终止。(`SIGPIPE` 导致)
  2. 有读端：
      - 管道已满，`write` **阻塞等待，直到管道有空间**
      - 管道未满，`write` 返回实际写入的字节数

> 虽然父进程和子进程都可以从管道中读取和写入数据，但这种做法并不常见。也可以有多个进程向单个管道中写入数据，但通常只存在一个写者。**同时只有一个读端和一个写端，关闭其他不用的端口**。否则可能出现以下情况：
>
> - 占用文件描述符；
> - 多个进程同时读，竞争带来不确定性；
> - 其他进程都关闭后，如果写入进程没有关闭管道的读取端，它仍然能够写入；
> - 如果多个进程写入同一个管道，那么如果它们在一个时刻写入的数据量不超过 `PIPE_BUF` (Linux 下为4096，`getconf PIPE_BUF /`)字节，那么就可以确保写入的数据不会发生相互混合的情况（即一次原子写入的最大长度为4096 bytes）。当写入管道的数据块的大小超过了 `PIPE_BUF` 字节，那么内核可能会将数据分割成几个较小的片段来传输，就容易出现数据交叉，导致数据混乱。

## fifo

> 标识：路径+文件名

`FIFO`与管道`Pipe`类似，但**FIFO 会在文件系统上创建一个管道类型(p)的文件**，其他进程可以通过该文件获取该管道，`FIFO`也因此可以用于无血缘关系的进程间通信。**注：创建`FIFO`的进程不一定会使用该管道**，也可以使用`mkfifo`命令手动创建管道文件供其他进程使用该文件。

```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);

// example
const char *pathname = "/tmp/myfifo";
int res = mkfifo(pathname, 0664);
```

- `pathname`：管道文件的路径 + 文件名
- `mode`：管道文件的权限，类似`0644`
- 返回值：
  - 成功返回 `0`
  - 失败翻译 `-1` ，并设置 `errno`

### FIFO的读写行为

FIFO也是一种管道，所以上面**管道的读写行为也基本适用于FIFO**！

打开`FIFO`文件和普通文件的区别有2点：

1. 第一**不能以`O_RDWR`模式打开**`FIFO`文件进行读写操作。这样做的行为是未定义的。只能一个进程以**只读方式`O_RDONLY`打开**，另一个进程以**只写方式`O_WRONLY`打开**该文件。与管道一样，当所有引用 `FIFO` 的描述符都被关闭之后，所有未被读取的数据会被丢弃。

2. 第二是对标志位的`O_NONBLOCK`选项的用法，使用这个选项不仅改变`open`调用的处理方式，还会改变对这次`open`调用返回的文件描述符进行的读写请求的处理方式。`O_RDONLY`、`O_WRONLY`和`O_NONBLOCK`标志共有四种合法的组合方式：

   - `flags=O_RDONLY`：`open`将会**调用阻塞**，直到另一个进程以`O_WRONLY`打开同一个`FIFO`。
   - `flags=O_WRONLY`：`open`将会**调用阻塞**，直到另一个进程以`O_RDONLY`打开同一个`FIFO`。
   - `flags=O_RDONLY|O_NONBLOCK`：**立即返回**，若此时没有其他进程以`O_WRONLY`打开，`open`会成功返回，此时`FIFO`**被读打开**，不会返回错误。
   - `flags=O_WRONLY|O_NONBLOCK`：**立即返回**，若此时没有其他进程以`O_RDONLY`打开，`open`会失败返回，此时`FIFO`**没有被打开**，返回-1。

**当`FIFO`的其中一端关闭后，另一端也会立刻关闭**，可以通过下面的例子测试：

### example

```c
// write fifo
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <string.h>
#include <time.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char *pathname = "/tmp/myfifo";
    int fd = open(pathname, O_WRONLY);

    char buf[100] = "hello, fifo!\n";
    while(1) {
        write(fd, buf, strlen(buf));
        sleep(1);
    }

    return 0;
}

// read fifo
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <string.h>
#include <time.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char *pathname = "/tmp/myfifo";
    int fd = open(pathname, O_RDONLY);

    char buf[100] = "";
    int bytes = 0;
    while(1) {
        memset(buf, 0, 100);
        bytes = read(fd, buf, 100);
        if (bytes == 0) break;
        printf("read from fifo: %s\n", buf);
    }

    return 0;
}
```

`FIFO`文件也可以直接使用命令行进行输入输出：

```bash
# 首先用cat命令读取刚才创建的FIFO文件：
# 这个时候，cat命令将一直挂起，直到终端或者有数据发送到FIFO中。
cat < /tmp/myfifo

# 然后尝试向FIFO中写数据（在另外一个终端执行这个命令）
# 这个时候cat将会输出内容。
echo "FIFO test" > /tmp/myfifo
```

## UNIX Domain Socket

> 标识：路径+文件名

又称**本地套接字**`AF_LOCAL` 、`UNIX Domain Socket(AF_UNIX)`，是一种专门用于**同一主机上进程间**相互通信的`Socket`，同样可使用流`Socket`和数据报`Socket`。使用`UNIX Domain`发送的报文不会经过协议栈，效率更高。

`AF_UNIX` 与 `AF_LOCAL` 的主要区别在于，`AF_UNIX` 是套接字族的官方名称，而 `AF_LOCAL` 是一个历史原因使用的同义词。两个常量都在 `<sys/socket.h>` 头文件中定义，在大多数情况下可以互换使用。

### sockaddr_un

在 `UNIX domain` 中，`socket` 地址以**路径名**来表示，而不是传统的`IP+PORT`。**会在指定的路径上创建一个文件，以便其他进程建立连接**。

为将一个 `UNIX domain socket` 绑定到一个地址上，需要初始化一个 `sockaddr_un` 结构，然后将指向这个结构的一个（转换）指针作为 `addr` 参数传入 `bind()`并将 `addrlen` 指定为这个结构的大小。

```c
#include <sys/un.h>

struct sockaddr_un {
    sa_family_t sun_family;   /* Always AF_UNIX */
    char sun_path[108];       /* socket 对应的路径名字（最好小于92字节） */
}

// example
const char *path = "/tmp/test.sock";
int sfd = socket(AF_UNIX, SOCK_STREAM, 0);    // 指定domain 为 AF_UNIX

struct sockaddr_un addr;
memset(&addr, 0, sizeof(struct sockaddr_un)); // 清零
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, path, sizeof(addr.sun_path)-1);  // 最好使用 strncpy

bind(sfd, (struct sockaddr *)&addr, sizoef(addr));  // 会在文件系统中创建一个条目

listen(sfd, 128);

int client_fd = accept(sfd, NULL, NULL); // 注意这里传入参数为NULL
```

在调用`bind()`时，会依据`sun_path`创建`socket`类型的文件，该目录需要可访问可写。此外还需注意：

- 无法将一个 `socket` 绑定到一个既有路径名上（`bind()`会失败并返回 `EADDRINUSE` 错误）。
- 通常会将一个 `socket` 绑定到一个**绝对路径名**上。
- 一个 `socket` 只能绑定到一个路径名上，相应地，一个路径名只能被一个 `socket` 绑定。
- 无法使用 `open()` 打开一个 `socket`。
- 当不再需要一个 `socket` 时可以使用 `unlink()`（或 `remove()`）**删除其路径名条目**（通常也应该这样做）。

### example

**服务端**：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <string.h>
#include <sys/un.h>

int main() {
    int bytes, res, sfd, cfd;
    char buf[1024];
    const char *path = "/tmp/test.sock";

    sfd = socket(AF_UNIX, SOCK_STREAM, 0);  // stream
    if (sfd == -1) perror("socket() 调用失败");

    struct sockaddr_un addr;    // 设置地址（文件路径标识）
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, path, sizeof(addr.sun_path)-1);

    res = bind(sfd, (struct sockaddr *)&addr, sizeof(addr));  // 绑定地址（创建文件）
    if (res == -1) perror("bind() 调用失败");

    res = listen(sfd, 128);     // 设置监听上限
    if (res == -1) perror("listen() 调用失败");

    cfd = accept(sfd, NULL, NULL);  // 等待连接，这里只演示了一个连接的情况

    while(1) {
        memset(buf, 0, 1024);
        bytes = read(cfd, buf, 1024);
        if (bytes > 0) {
            printf("recv msg: %s\n", buf);
            buf[0] = 'S';
            write(cfd, buf, sizeof(buf));
        } else
            break;
    }

    res = unlink(path);     // 用完后，就删掉绑定的文件
    if (res == -1) perror("unlink() 调用失败");

    return 0;
}
```

**客户端**：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <string.h>
#include <sys/un.h>

int main() {
    int res, bytes, cfd;
    char buf[1024];
    const char *path = "/tmp/test.sock";

    cfd = socket(AF_UNIX, SOCK_STREAM, 0);  // stream
    if (cfd == -1) perror("socket() 调用失败");

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, path, sizeof(addr.sun_path)-1);

    res = connect(cfd, (struct sockaddr*)&addr, sizeof(addr));
    if (res == -1) perror("connect() 调用失败");

    while(1) {
        scanf("%s", buf);
        write(cfd, buf, sizeof(buf));
        read(cfd, buf, 1024);
        printf("client recv msg: %s\n", buf);
    }

    return 0;
}
```

## System V IPC

> 标识：IPC key（一个整数）+ 其他参数 才能唯一标识一个IPC对象。key 可以由 ftok 得到。

`System V IPC` 包含三种方式，消息队列、共享内存、信号量。一般通信流程为：

1. 基于`IPC key`创建一个对象（消息队列、共享内存、信号量）。`IPC key`类似于文件路径，可用于标识对象。
2. 其他进程基于`IPC key`打开该对象。`1,2`步合并在了`get`系统调用中。
3. 多个进程操作该对象实现进程通信。
4. 从系统中删除该对象。（即使没有进程使用，该对象也不会自动删除）

| 接口      | 消息队列                      | 共享内存             | 信号量        |
| --------- | --------------------------- | ------------------ | ------------- |
| 头文件    | `<sys/msg.h>`                | `<sys/shm.h>`        | `<sys/sem.h>` |
| 数据结构  | `msqid_ds`                   | `shmid_ds`             | `semid_ds`    |
| 创建/打开 | `msgget()`                   | `shmget()+shmat()`   | `semget()`      |
| 控制      | `msgctl()`                  | `semctl()`             | `shmctl()`     |
| 执行IPC   | `msgsnd()` 写入<br />`msgrcv()` 接受 | 访问共享区域中的内存 | `semop()`    |

### get 调用

每种 `System V IPC` 机制都有一个相关的 `get` 系统调用，它**与文件上的 `open()`系统调用类似**。给定一个整数 `key`（类似于文件名），`get` 调用完成下列某个操作：

- 使用给定的 key **创建**一个新 IPC 对象并**返回一个唯一的标识符**来标识该对象。
- 返回一个拥有给定的 key 的既有 IPC 对象的标识符（就像文件描述符）。

后续的消息操作收发使用的是**标识符**，`key`也只是为了获取该**标识符**。其他进程若想使用该 IPC 对象，**要么知道key（通过get取获取标识符）**，**要么知道该标识符**，直接使用。

IPC **标识符则是对象本身的一个属性并且对系统全局可见**。也就是不同的进程可以通过对应的 key 获得同一个对象，且获得的标识符大小始终相等（这点与文件描述符不同）。

> 这里容易误解为只要 key 相同的，就能唯一标识一个对象，实际上除了 key ，还需要 get 调用中的其他参数也一致才能唯一标识一个IPC对象，如果使用相同的 key，但其他参数不同，得到的也是不同的标识符。

### IPC Key

消息队列、信号量、共享内存分别有一个数据结构管理其 key，或者说 shmget(100) 和 msgget(100) 返回的是不同的对象。

对于每种 IPC 机制（共享内存、消息队列、或信号量），内核都会维护一个关联的 ipc_ids 结构，它记录着该 IPC 机制的所有实例的各种全局信息，包括一个大小会动态变化的指针数组 entries，数组中的每个元素指向一个对象实例的关联数据结构（在信号量中是 semid_ds 结构）。

![image-20221217105737164](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1/image-20221217105737164.png)

在执行一个 IPC get 调用时，Linux 所采用的算法近似如下：

1. 在关联数据结构列表（entries 数组中的元素指向的结构）中搜索 key 字段与 get 调用中指定的参数匹配的结构。
    1. 没有找到，且没有指定 IPC_CREAT，返回 ENOENT 错误
    2. 找到了，但指定了 IPC_CREAT | IPC_EXCL ，返回 EEXIST 错误
    3. 否则在找到一个匹配的结构的情况下跳过下面的步骤。
2. 没有找到且指定了 IPC_CREAT ，分配一个新的 IPC 对象并对其初始化，在这个操作中还会更新 ipc_ids 结构中的各个字段，并且可能还会重新设定 entries 数组的大小。
3. 使用公式计算 IPC 对象的标识符。

### ftok

如果单纯用数字 key 来标识（当然还有参数），数字不能像字符串那样具备一定含义，不那么方便，而且容易重复。所以也提供了一个函数 ftok 从一个文件 + 数字 来生成 key。

```c
#include <sys/types.h>
#include <sys/ipc.h>

key_t ftok(const char *pathname, int proj_id);
```

- `pathname`：一个实际存在的文件路径。
- `proj_id`：一个数字，目的仅仅是允许从同一个文件中生成多个 key 。
- 返回值：
  - 成功，在 Linux 上，返回的是一个 32 位的值。
  - 失败返回 -1。

在 Linux 上，返回值是取 proj 参数的最低 8 个有效位、包含该文件所属的文件系统的设备的设备号（即次要设备号）的最低 8 个有效位以及 pathname 所引用的文件的 i-node 号（**并不是基于pathname这个字符串来计算**）的最低 16 个有效位组合而成。

### 对象删除

System V IPC 对象具备内核持久性。一旦被创建之后，一个对象就一直存在直到它被显式地删除或系统被关闭。每种 IPC 都有相关的控制操作，其中 IPC_RMID 控制操作可以实现删除 IPC 对象，**对于消息队列和信号量来讲，IPC 对象的删除是立即生效的，对象中包含的所有信息都会被销毁，不管是否有其他进程仍然在使用该对象**（这带来了一定的缺陷，有时候并不容易确定哪个进程是最后一个退出的）。共享内存对象的删除的操作是不同的。在 shmctl(id,IPC_RMID, NULL)调用之后，只有当所有使用该内存段的进程与该内存段分离之后（使用 shmdt()）才会删除该共享内存段。

### ipcs, ipcrm

ipcs 和 ipcrm 命令是 System V IPC 领域中类似于 ls 和 rm 文件命令的命令。

使用 ipcs 能够获取系统上 IPC 对象的信息：

> /proc/sysvipc 目录中的文件也会列出所有 IPC 对象！

```c
------ Message Queues --------              队列数据字节数 消息数量
key        标识符      所有者      权限        对象特有信息
key        msqid      owner      perms      used-bytes   messages

------ Shared Memory Segments --------      区域大小    使用进程数   状态标记
key        shmid      owner      perms      bytes      nattch     status
0x510715a7 7          yogurt     600        152        1
0x00000000 12         yogurt     600        1048576    2          dest
0x00000000 16         yogurt     600        524288     2          dest
0x00000000 20         yogurt     600        524288     2          dest
0x00000000 21         yogurt     600        524288     2          dest
0x00000000 24         yogurt     600        524288     2          dest
0x00000000 27         yogurt     600        524288     2          dest
0x00000000 30         yogurt     600        1048576    2          dest
0x00000000 33         yogurt     600        134217728  2          dest

------ Semaphore Arrays --------            信号集大小
key        semid      owner      perms      nsems
0x510715a6 1          yogurt     600        1
```

使用 ipcrm 删除对象：

```bash
 - Delete a shared memory segment by ID:
   ipcrm --shmem-id {{shmem_id}}

 - Delete a shared memory segment by key:
   ipcrm --shmem-key {{shmem_key}}

 - Delete an IPC queue by ID:
   ipcrm --queue-id {{ipc_queue_id}}

 - Delete an IPC queue by key:
   ipcrm --queue-key {{ipc_queue_key}}

 - Delete a semaphore by ID:
   ipcrm --semaphore-id {{semaphore_id}}

 - Delete a semaphore by key:
   ipcrm --semaphore-key {{semaphore_key}}

 - Delete all IPC resources:
   ipcrm --all
```

### 限制

由于 System V IPC 对象会消耗系统资源，因此内核对各种 IPC 对象进行了各式各样的限制以防止资源被耗尽。

在 Linux 上，ipcs –l 命令可以用来列出各种 IPC 机制上的限制。

|          | 限制    | 解释                                                       |
| -------- | ------- | ---------------------------------------------------------- |
|          |         | 部分限制变量位于 /proc/sys/kernel/ ，变量名为文件名        |
| 消息队列 | MSGMNI  | 系统级，系统中所能创建的消息队列的数量，32000              |
|          | MSGMAX  | 系统级，单条消息中最多可写入的字节数（mtext），8192        |
|          | MSGMNB  | 系统级，一个消息队列中一次最多保存的字节数（mtext），16384 |
|          | MSGTQL  | 系统级，系统中所有消息队列所能存放的消息总数               |
|          | MSGPOOL | 系统级，用来存放系统中所有消息队列中的数据的缓冲池的大小   |
| 信号量   | SEMMNI  | 系统级，限制了所能创建的信号量标识符的数量，32000          |
|          | SEMMSL  | 一个信号量集中能分配的信号量的最大数量，32000              |
|          | SEMMNS  | 系统级，所有信号量集中的信号量数量，1024000000             |
|          | SEMOPM  | 每个 semop()调用能够执行的操作的最大数量，500              |
|          | SEMVMX  | 一个信号量能取的最大值                                     |

## Sys 消息队列

System V 消息队列中的消息是以**块**的形式存储的，这些块由**消息类型**和**数据**两部分组成。消息类型是一个整数，可以用来区分不同类型的消息，而数据则是一段二进制数据，可以用来存储消息的具体内容。（**每个块的大小可以是不相同的**！）

消息队列本身是以**链表**的形式组织的，所有的消息都按照消息类型和发送时间的顺序依次排列。进程可以通过指定消息类型来发送或接收特定类型的消息。

**消息队列本身就具有同步功能**，可以保证在同一时刻只有一个进程能够访问队列。

### msgget

创建一个新消息队列或取得一个既有队列的标识符。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

int msgget(key_t key, int msgflg);
```

- `key`：上面的 IPC key。
- `msgflg`：指定施加于新消息队列之上的权限或检查一个既有队列的权限的位掩码，以及 IPC_CREAT | IPC_EXCL。
- 返回值：
  - 成功返回标识符。
  - 失败返回 -1。

### msgsnd

消息队列的 IO 函数。msgsnd msgrcv 这两个系统调用接收的第一个参数是消息队列标识符（msqid）。第二个参数 msgp 是一个**由程序员定义的**结构的指针，该结构用于存放被发送或接收的消息。

```c
/* 下面这个结构需要你自己定义！mtext也可以设置为定长 */
struct msgbuf {
    long mtype;       /* message type, must be > 0 */
    char mtext[1];    /* message data */
};
```

消息的第一个部分包含了消息类型，它用一个类型为 long 的整数来表示，而消息的剩余部分则是由程序员定义的一个结构，其长度和内容可以是任意的，而无需是一个字符数组。因此 mgsp 参数的类型为 void *，这样就允许传入任意结构的指针了。mtext 字段长度可以为零，当对于接收进程来讲所需传递的信息仅通过消息类型就能表示或只需要知道一条消息本身是否存在时，这种做法有时候就变得非常有用了。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

将 msgp 指向的消息内容复制进内核消息队列链表中，存在内存拷贝。

- msgp ：待发送的消息 msgbuf 地址；其中 mtype 必须指定为 >0 的数。
- msgsz ：指定了 msgbuf.mtext 字段中包含的字节数；不是 msgbuf 的总长度；
- msgflg ：一组标记的位掩码，用于控制 msgsnd()的操作，目前定义了：
  - IPC_NOWAIT，执行非阻塞发送；通常当消息队列满时，msgsnd()会阻塞直到队列中有足够的空间来存放这条消息。但如果指定了这个标记，那么 msgsnd()就会立即返回 EAGAIN 错误。
- 返回值：
  - 成功返回 0；（注意不是返回发送字节数）
  - 失败返回 -1；阻塞发送时，被信号处理器中断设置 EINTR 错误。

### msgrcv

```c
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp,
               int msgflg);
```

从消息队列中取出并删除一条消息，将其内容复制到 msgp 指向的缓冲区中。

- msgsz ：预分配的缓冲区 msgbuf 中 mtext 字段的大小；如果队列中待删除的消息体的大小超过了msgsz 字节，那么就不会从队列中删除消息，并且 msgrcv() 会返回错误 E2BIG。
- msgtyp ：指定接收的消息类型；
  - 如果 >0，则返回并删除指定消息类型的第一条消息。
  - 如果 =0，则删除队列中的第一条消息并将其返回给调用进程。
  - 如果 <0，则将等待消息当成优先队列来处理！队列中 mtype 最小并且其值小于或等于 msgtyp 的绝对值的第一条消息会被删除并返回给调用进程。
  - 通过指定不同的 msgtyp 值，多个进程能够从同一个消息队列中读取消息而不会出现竞争读取同一条消息的情况。比较有用的一项技术是让各个进程选取与自己的进程 ID 匹配的消息。
- msgflg ：位掩码；
  - IPC_NOWAIT 执行一个非阻塞接收；
  - MSG_EXCEPT只有当 msgtyp 大于 0 时这个标记才会起作用，它会强制对常规操作进行补足，即将队列中第一条 mtype 不等于 msgtyp 的消息删除并将其返回给调用者。
  - MSG_NOERROR在默认情况下，当消息的 mtext 字段的大小超过了可用空间时（由 maxmsgsz 参数定义），msgrcv()调用会失败。如果指定了 MSG_NOERROR 标记，那么 msgrcv()将会从队列中删除消息并将其 mtext 字段的大小截短为maxmsgsz 字节，然后将消息返回给调用者。被截去的数据将会丢失。
- 返回值：
  - 成功返回接收到的消息中 mtext 字段的大小；
  - 错误返回 -1；

### msgctl

在标识符为 msqid 的消息队列上执行控制操作。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

int msgctl(int msqid, int cmd, struct msqid_ds *buf);

/* 删除消息队列 */
int err = msgctl(msqid, IPC_RMID, NULL);
if (err == -1) {
    exit(-1);
}
```

- cmd ：指定了在队列上执行的操作：
  - IPC_RMID 立即删除消息队列对象及其关联的 msqid_ds 数据结构。**队列中所有剩余的消息都会丢失**！
  - IPC_STAT 将与这个消息队列关联的 msqid_ds 数据结构的副本放到 buf 指向的缓冲区中。
  - IPC_SET 使用 buf 指向的缓冲区提供的值更新与这个消息队列关联的 msqid_ds 数据结构中被选中的字段。
- buf ：部分控制操作会用到，没有用到的操作传入 NULL 即可。
- 返回值：
  - 成功执行的返回值与对应的操作有关，IPC_STAT, IPC_SET, IPC_RMID 返回 0。
  - 失败返回 -1 。

### example

发送段：在发送过程中可以通过 ipcs 查看消息队列中的消息数。

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <unistd.h>

struct msgbuf {
    long mtype;
    char mtext[100];
};

int main() {
    int i = 100, err;
    struct msgbuf buf;

    key_t key = ftok("./ipc_sysv_msg_snd.c", 1);

    /* 创建 System V 消息队列 */
    int msgid = msgget(key, 0664 | IPC_CREAT | IPC_EXCL);
    if (msgid == -1) {
        perror("msgget error: ");
        exit(-1);
    }

    /* 发送消息 */
    buf.mtype = 1;
    while(i--) {
        sleep(1);
        sprintf(buf.mtext, "%d", i);
        err = msgsnd(msgid, &buf, 100, 0);
        if (err == -1) {
            perror("msgsnd failed!");
            continue;
        }
    }

    getchar();
    /* 删除消息队列 */
    err = msgctl(msgid, IPC_RMID, NULL);
    if (err == -1) {
        perror("msgctl RMID failed!");
        exit(-1);
    }

    return 0;
}
```

接收端：

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <unistd.h>

struct msgbuf {
    long mtype;
    char mtext[100];
};

int main() {
    int i = 100, err;
    struct msgbuf buf;

    key_t key = ftok("./ipc_sysv_msg_snd.c", 1);

    /* 打开 System V 消息队列 */
    int msgid = msgget(key, 0664);
    if (msgid == -1) {
        perror("msgget error: ");
        exit(-1);
    }

    /* 接收消息 */
    buf.mtype = 1;
    while(i--) {
        err = msgrcv(msgid, &buf, 100, 1, 0);
        if (err == -1) {
            perror("msgsnd failed!");
            continue;
        }
        printf("msg.type: %ld, data: %s\n", buf.mtype, buf.mtext);
    }

    return 0;
}

```

## Sys 信号量

> System V 信号量是一种持久性的数据类型，意味着它们在系统重启之后仍然存在。但是，System V 信号量不会被保存到磁盘上，因此在系统重启之后，所有的信号量都将被初始化为 0。

在多线程中，如果多个线程要访问同一个变量，为了避免多个线程同时写导致数据异常，会采用互斥锁、读写锁、信号量等同步机制，在进程通信时，也会出现两个进程同时访问某段内存的情况，同样需要一种同步机制，而且其实现应该由内核完成，这样才能让多个进程都能访问到。

在上面介绍的消息队列中，并不需要专门去同步，它本身就具有同步功能。不过下面的共享内存，大多数情况下都得配合信号量使用！

一个信号量是一个由内核维护的整数，**其值被限制为大于或等于 0**。在一个信号量上可以执行各种操作（即系统调用），包括：

- 将信号量设置成一个绝对值；
- 在信号量当前值的基础上加上一个数量；
- 在信号量当前值的基础上减去一个数量；（试图使信号量降低到 0 之下时**阻塞**）
- 等待信号量的值等于 0。（信号量的当前值不为 0 时**阻塞**）

> 语义上来讲，增加信号量值对应于使一种资源变得可用以便其他进程可以使用它，而减小信号量值则对应于预留（互斥地）进程需使用的资源。在减小一个信号量值时，如果信号量的值太低——即其他一些进程已经预留了这个资源——那么操作就会被阻塞。

使用 System V 信号量的常规步骤如下：

1. 创建信号量：使用 semget() 函数创建一个新的**信号量集**，或者打开一个现有的信号量集。
2. 初始化信号量：使用 semctl() 函数初始化信号量的值。
3. 获取信号量：使用 semop() 函数获取信号量。这会使信号量的值减少 1。
4. 释放信号量：使用 semop() 函数释放信号量。这会使信号量的值增加 1。
5. 删除信号量：使用 semctl() 函数 IPC_RMID 删除信号量集。

> 上面提到的信号量集是什么意思，一个信号量不应该就一个整数值吗？
>
> (chatgpt) 答：在 System V 信号量中，一个信号量集是一组信号量的集合。每个信号量集都有一个唯一的标识符，称为信号量集标识符。每个信号量集中可以包含多个信号量。每个信号量都有一个整数值，可以用来控制进程的访问。使用信号量集的好处在于，你可以通过一个信号量集管理多个信号量，而不是一个一个地管理。这样，你就可以使用一个函数调用来管理多个信号量，而不是多个函数调用。

### semget

创建一个新信号量集或获取一个既有集合的标识符。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

int semget(key_t key, int nsems, int semflg);
```

- key ：前面的 IPC key；
- nsems ：指定集合中信号量的数量，并且其值必须大于 0。如果是获取既有集合的标识符，nsems 必须小于等于集合的大小。
- semflg ：位掩码，指定权限 + IPC_CREAT | IPC_EXCL
- 返回值：
  - 成功返回标识符；
  - 失败返回 -1；

### semctl

在一个信号量集或集合中的单个信号量上执行各种控制操作。初始化信号量、删除信号量集等。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

int semctl(int semid, int semnum, int cmd, ... /* union semun arg */);

union semun {
    int              val;    /* Value for SETVAL */
    struct semid_ds *buf;    /* Buffer for IPC_STAT, IPC_SET */
    unsigned short  *array;  /* Array for GETALL, SETALL */
    struct seminfo  *__buf;  /* Buffer for IPC_INFO
                                           (Linux-specific) */
};
```

- semid ：信号量集的标识符；
- semnum ：标识要操作的集合中具体的信号量，不需要该变量的操作会忽略它，可以设置为0。
- cmd ：指定了需执行的操作，某些操作会用到第四个参数 `union semun`：
  - **IPC_RMID** 立即**删除信号量集**及其关联的 semid_ds 数据结构。无需 arg 参数。
  - GETVAL 返回由 semid 指定的信号量集中第 semnum 个信号量的值。
  - **SETVAL** 将由 semid 指定的信号量集中第 semnum 个信号量的值**初始化**为 arg.val。
  - GETALL 获取由 semid 指向的信号量集中所有信号量的值并将它们放在 arg.array 指向的数组中。
      程序员必须要确保该数组具备足够的空间。
  - **SETALL** 使用 arg.array 指向的数组中的值**初始化** semid 指向的集合中的所有信号量。

### semop

在 semid 标识的信号量集中的信号量上执行一个或**多个**操作。

```c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

int semop(int semid, struct sembuf *sops, size_t nsops);
int semtimedop(int semid, struct sembuf *sops, size_t nsops,
               const struct timespec *timeout);
struct sembuf {
    unsigned short sem_num;  /* semaphore number */
    short          sem_op;   /* semaphore operation */
    short          sem_flg;  /* operation flags */
}
```

- sops ：指向数组的指针，数组中包含了需要执行的操作。
  - sem_num ：指定操作的信号量；
  - sem_op ：指定需要执行的操作：
    - 如果 sem_op 大于 0，那么就将 sem_op 的值加到信号量值上，其结果是其他等待减小信号量值的进程可能会被唤醒并执行它们的操作。
    - 如果 sem_op 等于 0，那么就对信号量值进行检查以确定它当前是否等于 0。如果等于 0，那么操作将立即结束，否则 semop()就会阻塞直到信号量值变成 0 为止。
    - 如果 sem_op 小于 0，那么就将信号量值减去 sem_op。如果信号量的当前值大于或等于 sem_op 的绝对值，那么操作会立即结束。否则 semop()会阻塞直到信号量值增长到在执行操作之后不会导致出现负值的情况为止。
- nsops ：数组的大小，数组至少需要一个元素！操作将会**按照在数组中的顺序以原子的方式被执行**。

## Sys 共享内存

todo

## mmap 内存映射

`mmap`映射方式有两种，一是文件映射，二是匿名映射。映射属性也有两种：私有`MAP_PRIVATE` 和共享`MAP_SHARED`。其四种组合如下：

- **私有文件**：分配一段内存，其内容用文件的内容初始化，文件的作用也只在于初始化，后续在内存上的修改也不会再同步到文件上。多个进程映射同一个文件同一区域时，一开始会共享同一个物理内存分页，采用**写时复制技术**保证各个进程不共享。
- **私有匿名**：分配一段内存，会用0填充内存。每次调用都会创建一个新的映射，不会共享同一个物理页。通过`fork()`创建子进程时，也会采用**写时复制技术**保证不共享。
- **共享文件**：分配一段内存，其内容用文件的内容初始化，对内存的修改内核也会选择合适的时机同步到文件上。多个进程映射同一个文件同一区域时，共享同一个物理内存分页，也就是**共享内存机制**。
- **共享匿名**：分配一段内存，会用0填充内存。当通过`fork()`创建子进程时，父子进程共享同样的物理分页，而不会采用写时复制技术。也就实现了有血缘关系的进程间的共享内存（类似与`pipe`，只能用于有血缘关系的进程）。

一个进程在执行 `exec()` 时映射会丢失，但通过 `fork()` 创建的子进程会继承映射，映射类型（`MAP_PRIVATE` 或 `MAP_SHARED`）也会被继承

### mmap 函数

`mmap()`系统调用在调用进程的虚拟地址空间中创建一个新映射。

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
            int fd, off_t offset);
```

- `addr`：指定映射被放置的虚拟地址（仅作内核参考，还涉及分页对齐），一般为`NULL`，由内核自动选定
- `length`：指定映射的字节数。实际映射大小会向上提升为分页大小的倍数
- `prot`：位掩码，指定施加于映射之上的保护信息，要么是`PROT_NONE`，要么是其他三个标记的组合
  - `PROT_NONE` 区域无法访问
  - `PROT_READ` 区域内容可读取
  - `PROT_WRITE` 区域内容可修改
  - `PROT_EXEC` 区域内容可执行
- `flag`：控制映射操作各个方面的选项的位掩码，这个掩码必须只包含下列值中一个：
  - `MAP_PRIVATE` 创建一个私有映射
  - `MAP_SHARED` 创建一个共享映射
- 剩余的参数 `fd` 和 `offset` 是用于文件映射的（匿名映射将忽略它们）
- 返回值：
  - 成功：映射的起始地址
  - 失败：返回`MAP_FAILED`

一个文件映射的例子：

```c

```

### munmap 函数

与`mmap()`相反，从进程虚拟地址中删除一个映射。

```c
#include <sys/mman.h>

int munmap(void *addr, size_t length);
```

- `addr`：需要解除映射的地址范围的起始地址。通常传`mmap`的返回值。
- `length`：需要解除映射的地址范围的大小，一般与`mmap`中的大小一致，也会自动页对齐。
- 返回值：

## POSIX IPC

- 消息队列：类似与数据包，每个消息独立。与System V 标准中的消息队列不同，POSIX不区分消息类型，但每个消息有一个优先级，优先级高的消息先被处理。
- 信号量：也是由内核维护的整数，其值永远都不会小于0。
- 共享内存

### 标识

在 SUSv3 中规定的唯一一种用来标识 POSIX IPC 对象的可移植的方式是使用**以斜线打头后面跟着一个或多个非斜线字符的名字**，如 /myobject，注意**不能**是 `/tmp/myobject`！

对象名称看起来像在根目录下的文件名，在某些系统上，也确实会创建该文件，而非特权用户又无法在根目录下创建文件，所以要保证程序的可移植性需要注意对象名的选择。

在 Linux 上，POSIX 共享内存和消息队列对象的名字的最大长度为 NAME_MAX（255）个字符，而信号量的名字的最大长度要少 4 个字符，这是因为实现会在信号量名字前面加上字符串 sem.

| 接口      | 消息队列                                        | 信号量 | 共享内存 |
| --------- | ----------------------------------------------- | ------ | -------- |
| 头文件    | <mqueue.h>                                      |        |          |
| 对象句柄  | mqd_t                                           |        |          |
| 创建/打开 | mq_open()                                       |        |          |
| 关闭      | mq_close()                                      |        |          |
| 断开链接  | mq_unlink()                                     |        |          |
| 执行 IPC  | mq_send()<br />mq_receive()                     |        |          |
| 其他操作  | mq_setattr()<br />mq_getattr()<br />mq_notify() |        |          |

### 消息队列

在 Linux 上，POSIX 消息队列被实现成了虚拟文件系统中的 i-node，并且消息队列描述符也是文件描述符。

![image-20221216152710119](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1/image-20221216152710119.png)

验证文件描述符与消息队列描述符的关系：

```c
#include <fcntl.h> /* For O_* constants */
#include <mqueue.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h> /* For mode constants */
#include <time.h>
#include <unistd.h>

const char* ipc_object = "/msgqueue";

int main() {
    struct mq_attr attr;
    attr.mq_maxmsg = 100;
    attr.mq_msgsize = 1500;

    int fd1, fd2;
    fd1 = open("./test1", O_CREAT|O_WRONLY, 0664);
    printf("随便打开了个文件，其fd = %d\n", fd1);

    mqd_t obj = mq_open(ipc_object, O_CREAT|O_EXCL|O_RDWR, 0664, &attr);
    if (obj < 0) {
        perror("msg_queue open failed!");
        exit(-1);
    }
    printf("创建并打开了消息队列，其msg_queue id: %d\n", obj);

    fd2 = open("./test2", O_CREAT|O_WRONLY, 0664);
    printf("随便打开了个文件，其fd = %d\n", fd2);

    close(fd1);
    close(fd2);
    unlink("./test1");
    unlink("./test2");
    mq_close(obj);
    mq_unlink(ipc_object);
    return 0;
}
// gcc test.c -o test && ./test
```

#### 队列特性

也可以说是队列参数，先自己思考一下，实现一个队列需要指定哪些参数呢？队列的大小（可以容纳的消息数量）、每个消息的长度、当前队列中有多少消息。

mq_open()、mq_getattr()以及 mq_setattr()函数都会接收一个参数，它是一个指向 mq_attr 结构的指针。

```c
struct mq_attr {
    long mq_flags;       /* Flags (ignored for mq_open()) */
    long mq_maxmsg;      /* Max. # of messages on queue */
    long mq_msgsize;     /* Max. message size (bytes) */
    long mq_curmsgs;     /* # of messages currently in queue
                                       (ignored for mq_open()) */
};

/* 查看默认参数：
    mq_flags: 0, O_NONBLOCK: 2048
    mq_maxmsg: 10
    mq_msgsize: 8192
    mq_curmsgs: 1
*/
void get_default_attr(void) {
    const char *ipc_key = "/default_attr";
    mqd_t obj = mq_open(ipc_object, O_CREAT|O_EXCL|O_RDWR, 0666, NULL);
    if (obj == -1) {
        perror("msg_queue open failed!");
        exit(-1);
    }
    struct mq_attr attr;
    int err = mq_getattr(obj, &attr);
    if (err == -1) {
        perror("getattr error!");
        exit(-1);
    }

    printf("mq_flags: %lld, O_NONBLOCK: %lld\n", attr.mq_flags, O_NONBLOCK);
    printf("mq_maxmsg: %lld\n", attr.mq_maxmsg);
    printf("mq_msgsize: %lld\n", attr.mq_msgsize);
    printf("mq_curmsgs: %lld\n", attr.mq_curmsgs);

    mq_close(obj);
    mq_unlink(ipc_key);
}
```

#### mq_open

创建一个新的消息队列或者打开现有的消息队列。

```c
#include <fcntl.h>           /* For O_* constants */
#include <sys/stat.h>        /* For mode constants */
#include <mqueue.h>

mqd_t mq_open(const char *name, int oflag);
mqd_t mq_open(const char *name, int oflag, mode_t mode,
              struct mq_attr *attr);
```

- `name`：`IPC key`，对象标识；
- `oflag`：和`open`中的`flag`类似，控制打开参数，读写、创建、非阻塞等。
  - O_CREAT | O_EXCL
  - O_RDONLY | O_WRONLY | O_RDWR
  - O_NONBLOCK
- 如果`oflag`中指定的`O_CREAT`，就还需要`mode`和`attr`，分别控制文件权限，和消息队列的特性。
  - `mode`：文件权限，`0664`，类似与`chmod`修改权限时用的值。
  - `attr`：指定消息队列的参数，也可以用`NULL`使用默认值。
  - 指定队列参数时，最好使用`O_EXCL`，否则由于队列已经存在，其参数若与指定的参数不一致，可能会导致后续的`mq_receive`出错。

- 返回值：
  - 调用成功，返回一个消息队列描述符（`>0`），后续用它来引用该消息队列。
  - 调用失败，返回 `-1`。

#### mq_close

```c
#include <mqueue.h>

int mq_close(mqd_t mqdes);
```

关闭一个消息队列，进程应该主动关闭不使用的消息队列，避免消息队列描述符耗尽。注意，关闭并不会删除该消息队列。删除需要使用`mq_unlink`。

#### mq_unlink

```c
#include <mqueue.h>

int mq_unlink(const char *name);
```

删除指定`name`标识的消息队列，并将队列标记为在所有进程使用完后销毁。

#### mq_send

将位于 msg_ptr 指向的缓冲区中的消息添加到描述符 mqdes 所引用的消息队列中，还需要指定消息的长度、消息的优先级，另外注意该过程存在内存拷贝。

```c
#include <mqueue.h>

int mq_send(mqd_t mqdes, const char *msg_ptr,
            size_t msg_len, unsigned int msg_prio);
```

- msg_ptr：指向待发送消息地址；
- msg_len：待发送消息长度；
- msg_prio：待发送消息优先级；
- 返回值：
  - 成功返回 `0`；
  - 失败返回 `-1`；

每条消息都拥有一个用非负整数表示的优先级，0表示最低优先级，优先级的上限在不同系统上也有差异，SUSv3 要求这个上限至少是 32，这可以通过定义常量 MQ_PRIO_MAX指定，在Linux上，为32768。

> `getconf -a | grep MQ_PRIO_MAX`

当一条消息被添加到队列中时，它会被放置在队列中具有相同的优先级的所有消息之后。如果无需使用，将其指定为0。

如果队列满了，`mq_send`会被阻塞，除非`mq_open`时设置非阻塞。

#### mq_receive

从 mqdes 引用的消息队列中删除一条优先级最高、存在时间最长的消息并将删除的消息放置在 msg_ptr 指向的缓冲区，该过程同样存在内存拷贝。

```c
#include <mqueue.h>

ssize_t mq_receive(mqd_t mqdes, char *msg_ptr,
                   size_t msg_len, unsigned int *msg_prio);
```

- `msg_ptr`：预分配的内存首地址。
- `msg_len`：调用者使用 msg_len 参数来指定 msg_ptr 指向的缓冲区中的可用字节数，不管消息的实际大小是什么，msg_len（即 msg_ptr 指向的缓冲区的大小）必须要大于或等于队列的 mq_msgsize 特性。
- `msg_prio`：传出参数，接收到的消息的优先级会被复制到 msg_prio 指向的位置处。
- 返回值：
  - 正常情况返回接收的消息的大小！
  - 失败返回 `-1`。

如果消息队列为空，mq_receive 会阻塞直到存在可用的消息。

#### 收发超时

与上面的收发函数相同，只是参数多了个超时时间`abs_timeout`，为调用阻塞的时间指定一个上限。abs_timeout 参数是一个 timespec 结构，它将超时时间描述为自新纪元到现在的一个**绝对值**，其单位为秒数和纳秒数。如果要指定相对值，可用通过获取当前时间，将当前时间+相对值赋值给`abs_timeout`。

```c
#include <time.h>
#include <mqueue.h>

int mq_timedsend(mqd_t mqdes, const char *msg_ptr,
                 size_t msg_len, unsigned int msg_prio,
                 const struct timespec *abs_timeout);

ssize_t mq_timedreceive(mqd_t mqdes, char *msg_ptr,
                        size_t msg_len, unsigned int *msg_prio,
                        const struct timespec *abs_timeout);
```

因超时而无法完成操作，那么调用就会失败并返回 ETIMEDOUT 错误。

#### example

**send**:

```c
#include <fcntl.h> /* For O_* constants */
#include <mqueue.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h> /* For mode constants */
#include <time.h>
#include <unistd.h>

// const char* ipc_object = "/home/yogurt/mymsg";
const char* ipc_object = "/tmpmyobsdfject2";

/* send 每秒往消息队列中写入计数值 */
int main() {
    struct mq_attr attr;
    attr.mq_maxmsg = 100;
    attr.mq_msgsize = 1500;

    /* 总是报权限不足 */
    mqd_t obj = mq_open(ipc_object, O_CREAT|O_RDWR, 0666, NULL);
    if (obj < 0) {
        perror("msg_queue open failed!");
        exit(-1);
    }
    printf("msg_queue id: %d\n", obj);

    char msg[1500];
    int i = 0;
    while(1) {
        sprintf(msg, "%d", i++);
        sleep(1);
        mq_send(obj, msg, strlen(msg), 0);
    }

    mq_close(obj);
    mq_unlink(ipc_object);
    return 0;
}
```

**receive**:

```c
#include <fcntl.h> /* For O_* constants */
#include <mqueue.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h> /* For mode constants */
#include <time.h>
#include <unistd.h>

// const char* ipc_object = "/home/yogurt/mymsg";
const char* ipc_object = "/tmpmyobsdfject2";

/* send 每秒往消息队列中写入计数值 */
int main() {
    mqd_t obj = mq_open(ipc_object, O_RDONLY);
    if (obj == -1) {
        perror("msg_queue open failed!");
        exit(-1);
    }

    printf("msg_queue id: %u\n", obj);
    char msg[10000];
    unsigned int prio;
    int bytes;
    while(1) {
        bytes = mq_receive(obj, msg, 10000, &prio);
        if (bytes == -1) {
            perror("receive error! ");
            continue;
        }
        printf("msg: %s, prio: %u\n", msg, prio);
    }

    mq_close(obj);
    return 0;
}
```

## 文件

与 `fifo` 行为类似，两个进程分别以只读和只写方式打开文件。理论上可行，但不推荐。效率低下，而且共享偏移量可能导致问题。

只有通过 `write` 写入到磁盘文件中的内容才可读取。

## mmap 内存映射

内存映射也称**共享内存**，映射的方式分为**文件映射**和**匿名映射**。

存储映射 I/O(Memory-mapped I/O) 使一个磁盘文件与存储空间中的一个缓冲区相映射。于是从缓冲区中取数据，就相当于读文件中的相应字节。与此类似，将数据存入缓冲区，则相应的字节就自动写入文件。这样，就可在不使用 `read` 和 `write` 函数的情况下，使地址指针完成 I/O 操作。

使用这种方法，首先应该通知内核，将一个指定文件映射到存储区域中。这个映射工作可以通过 `mmap` 函数来实现。

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
            int fd, off_t offset);
```

- `addr`：映射的起始地址，通常传 NULL，让系统会自动选择一个合适的地址
- `length`：共享内存映射区的大小（要 <= 文件的实际大小）
- `prot`：共享内存映射区的读写属性。`PROT_READ`、`PROT_WRITE`、`PROT_READ|PROT_WRITE`
- `flags`：标注共享内存的共享属性。`MAP_SHARED`、`MAP_PRIVATE`（修改不会反应到磁盘上，很少用）
- `fd`：用于创建共享内存映射区的那个文件的 文件描述符。
- `offset`：偏移位置，需是 4k 的整数倍。默认 0，表示映射文件全部。
- 返回值：
  - 成功：映射区的首地址。
  - 失败：`MAP_FAILED (void*(-1))`， errno

### munmap 函数

释放映射区。

```c
#include <sys/mman.h>

int munmap(void *addr, size_t length);
```

- `addr`：映射区的首地址
- `length`：映射区的大小
- 返回值：
  - `0`：成功
  - `-1`：失败，errno

### mmap 注意事项

1. 用于创建映射区的文件大小为 0，实际指定非 0 大小创建映射区，出 “总线错误”。
2. 用于创建映射区的文件大小为 0，实际制定 0 大小创建映射区， 出 “无效参数”。
3. 用于创建映射区的文件读写属性为，只读。映射区属性为 读、写。 出 “无效参数”。
4. 创建映射区，需要 read 权限。当访问权限指定为 “共享”MAP_SHARED 时， mmap 的读写权限，应该 <=文件的 open 权限。 只写不行。
5. 文件描述符 fd，在 mmap 创建映射区完成即可关闭。后续访问文件，用 地址访问。
6. offset 必须是 4096 的整数倍。（MMU 映射的最小单位 4k ）
7. 对申请的映射区内存，不能越界访问。
8. munmap 用于释放的 地址，必须是 mmap 申请返回的地址。
9. 映射区访问权限为 “私有”MAP_PRIVATE, 对内存所做的所有修改，只在内存有效，不会反应到物理磁盘上。
10. 映射区访问权限为 “私有”MA_PRIVATE, 只需要 open 文件时，有读权限，用于创建映射区即可。

------

1. 创建映射区的过程中，隐含着一次对映射文件的读操作
2. 当 MAP_SHARED 时，要求：映射区的权限应该<=文件打开的权限（出于对映射区的保护）。而MAP_PRIVATE 则无所谓，因为 mmap 中的权限是对内存的限制
3. 映射区的释放与文件关闭无关。只要映射建立成功，文件可以立即关闭
4. 特别注意，当映射文件大小为 0 时，不能创建映射区。所以：用于映射的文件必须要有实际大小！！
mmap 使用时常常会出现总线错误，通常是由于共享文件存储空间大小引起的。如，400 字节大小的文件，在简历映射区时，offset4096 字节，则会报出总线错误
5. munmap 传入的地址一定是 mmap 返回的地址。坚决杜绝指针++操作
6. 文件偏移量必须为 4K 的整数倍
7. mmap 创建映射区出错概率非常高，一定要检查返回值，确保映射区建立成功再进行后续操作

### example

有血缘关系的两个进程间通信的例子：

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mman.h>

int main() {
    int fd = open("mm", O_RDWR);
    ftruncate(fd, 100);

    char *p = mmap(NULL, 100, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) {
        printf("mmap error\n");
        exit(1);
    }
    close(fd);  // 映射区建立完毕,即可关闭文件
    if (fork() == 0) {
        char str[100] = "asdfghjkl\0";
        memcpy(p, str, strlen(str)); // 将内容写入指针地址！而不是让指针指向内容！
        printf("child: %s \n", p);
    } else {
        sleep(1);
        printf("parent: %s \n", p);
        wait(NULL);
        munmap(p, 100);
    }
    return 0;
}
```

无血缘关系的进程间通信时，只需要分别做两次 mmap 即可。

## 参考资料

> - [进程通信常用方式](https://xiaolincoding.com/os/4_process/process_commu.html)
> - [进程间通信-命名管道FIFO](https://blog.csdn.net/xiajun07061225/article/details/8471777)
> - [linux下，pipe的容量的讨论与查看](https://blog.51cto.com/momo462/1825852)

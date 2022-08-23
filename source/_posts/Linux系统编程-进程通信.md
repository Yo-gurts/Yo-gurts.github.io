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

进程间通信的方式有：

- 管道 `pipe`：简单，但只能用于有血缘关系的进程间
- 命名管道 `fifo`：可以用于任意进程间
- 文件：和 `fifo` 类似
- 消息队列：可以设置订阅 `topic`，只接收订阅的消息
- 套接字：最稳定
- 信号：使用最简单
- 共享内存：可重复读数据
- 信号量：配合共享内存使用，用作同步

## pipe 管道

fork之前创建管道，这样每个子进程都有读端`fd[0]`和写端`fd[1]`。

```c
#include <unistd.h>

int pipe(int pipefd[2]);

// example
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

- `pipefd[0]` 读端
- `pipefd[1]` 写端
- 返回值：
  - `0` 成功
  - `-1` 失败并设置 `errno`

### 管道的读写行为

- 一个管道是一个字节流，不存在消息或消息边界的概念。
- 通过管道传递的数据是顺序的，在管道中无法使用 `lseek()` 来随机地访问数据。
- **管道的容量是有限的**，一般为 `64k` (65536byte)。
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
- 如果多个进程写入同一个管道，那么如果它们在一个时刻写入的数据量不超过 `PIPE_BUF` (Linux为4096)字节，那么就可以确保写入的数据不会发生相互混合的情况。当写入管道的数据块的大小超过了 `PIPE_BUF` 字节，那么内核可能会将数据分割成几个较小的片段来传输，就容易出现数据交叉，导致数据混乱。

虽然父进程和子进程都可以从管道中读取和写入数据，但这种做法并不常见。也可以有多个进程向单个管道中写入数据，但通常只存在一个写者。**同时只有一个读端和一个写端，关闭其他不用的端口**。否则可能出现以下情况：

- 占用文件描述符；
- 多个进程同时读，竞争带来不确定性；
- 其他进程都关闭后，如果写入进程没有关闭管道的读取端，它仍然能够写入；

## mkfifo 命名管道

**该函数会在文件系统上创建一个管道类型(p)的文件**（创建`FIFO`的进程不一定会使用该管道，也可以使用`mkfifo`命令手动创建管道文件供其他进程使用该文件）。

```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);

// example
const char *pathname = "/tmp/myfifo";
int res = mkfifo(pathname, 0664);
```

- `pathname`：管道文件的路径
- `mode`：管道文件的权限，类似`0644`
- 返回值：
  - `0` 成功
  - `-1` 失败并设置 `errno`

打开FIFO文件和普通文件的区别有2点：

1. 第一**不能以`O_RDWR`模式打开**`FIFO`文件进行读写操作。这样做的行为是未定义的。只能一个进程以**只读方式`O_RDONLY`**打开，另一个进程以**只写方式`O_WRONLY`**打开该文件。与管道一样，当所有引用 `FIFO` 的描述符都被关闭之后，所有未被读取的数据会被丢弃。

2. 第二是对标志位的`O_NONBLOCK`选项的用法，使用这个选项不仅改变`open`调用的处理方式，还会改变对这次`open`调用返回的文件描述符进行的读写请求的处理方式。`O_RDONLY`、`O_WRONLY`和`O_NONBLOCK`标志共有四种合法的组合方式：

- `flags=O_RDONLY`：`open`将会调用阻塞，除非有另外一个进程以写的方式打开同一个`FIFO`，否则一直等待。
- `flags=O_WRONLY`：`open`将会调用阻塞，除非有另外一个进程以读的方式打开同一个`FIFO`，否则一直等待。
- `flags=O_RDONLY|O_NONBLOCK`：如果此时没有其他进程以写的方式打开`FIFO`，此时`open`也会成功返回，此时`FIFO`被读打开，而不会返回错误。
- `flags=O_WRONLY|O_NONBLOCK`：立即返回，如果此时没有其他进程以读的方式打开，`open`会失败打开，此时`FIFO`没有被打开，返回-1。

当管道的其中一端关闭后，另一端也会同时关闭，可以通过下面的例子测试：

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

管道文件也可以直接使用命令行进行输入输出：

```bash
# 首先用cat命令读取刚才创建的FIFO文件：
# 这个时候，cat命令将一直挂起，直到终端或者有数据发送到FIFO中。
cat < /tmp/myfifo


# 然后尝试向FIFO中写数据（在另外一个终端执行这个命令）
# 这个时候cat将会输出内容。
echo "FIFO test" > /tmp/myfifo
```

## 文件

与 `fifo` 行为类似，两个进程分别以只读和只写方式打开文件。

只有通过 `write` 写入到磁盘文件中的内容才可读取。

## mmap 存储映射

存储映射 I/O(Memory-mapped I/O) 使一个磁盘文件与存储空间中的一个缓冲区相映射。于是从缓冲区中取数据，就相当于读文件中的相应字节。与此类似，将数据存入缓冲区，则相应的字节就自动写入文件。这样，就可在不使用 `read` 和 `write` 函数的情况下，使地址指针完成 I/O 操作。

使用这种方法，首先应该通知内核，将一个指定文件映射到存储区域中。这个映射工作可以通过 `mmap` 函数来实现。

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
            int fd, off_t offset);
```

- `addr`：映射的起始地址，通常传 NULL，让系统会自动选择一个合适的地址
- `length`：共享内存映射区的大小（要 <= 文件的实际大小）
- `prot`：共享内存映射区的读写属性。PROT_READ、PROT_WRITE、PROT_READ|PROT_WRITE
- `flags`：标注共享内存的共享属性。MAP_SHARED、MAP_PRIVATE（修改不会反应到磁盘上，很少用）
- `fd`：用于创建共享内存映射区的那个文件的 文件描述符。
- `offset`：偏移位置，需是 4k 的整数倍。默认 0，表示映射文件全部。
- 返回值：
  - 成功：映射区的首地址。
  - 失败：MAP_FAILED (void*(-1))， errno

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

- [进程通信常用方式](https://www.bilibili.com/video/BV1KE411q7ee?p=100)
- [进程间通信-命名管道FIFO](https://blog.csdn.net/xiajun07061225/article/details/8471777)

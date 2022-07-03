---
title: Linux系统编程-进程
top_img: transparent
date: 2022-04-25 18:23:12
updated: 2022-04-25 18:23:12
tags:
  - Linux
  - 进程
categories: Linux
keywords:
description: Linux 中的进程创建、回收。
---

## 进程

- 进程（process）是一个可执行程序（program）的实例。
- 程序是包含了一系列信息的文件，这些信息描述了如何在运行时创建一个进程，所包括的内容如下所示：
  - 二进制格式标识
  - 机器语言指令：对程序算法进行编码。
  - **程序入口地址**：标识程序开始执行时的起始指令位置（main）。
  - 数据：程序文件包含的**变量初始值**和程序使用的字面常量（literal constant）值（比如字符串）。
  - 符号表及重定位表：描述程序中函数和变量的位置及名称。这些表格有多种用途，其中包括调试和运行时的符号解析（动态链接）。
  - 共享库和动态链接信息：程序文件所包含的一些字段，列出了程序运行时需要使用的共享库，以及加载共享库的动态链接器的路径名。
  - 其他信息：程序文件还包含许多其他信息，用以描述如何创建进程。

### 进程内存布局

每个进程所分配的内存由很多部分组成，通常称之为“段（segment）”。

- **文本段**包含了进程运行的程序机器语言指令。

- **初始化数据段**包含显式初始化的全局变量和静态变量。当程序加载到内存时，从可执行文件中读取这些变量的值。

- **未初始化数据段【BSS段】**包含了未进行显式初始化的全局变量和静态变量。程序启动之前，系统将本段内所有内存初始化为 0。

  将经过初始化的全局变量和静态变量与未经初始化的全局变量和静态变量分开存放，其主要原因在于程序在磁盘上存储时，没有必要为未经初始化的变量分配存储空间。相反，可执行文件只需记录未初始化数据段的位置及所需大小，直到运行时再由程序加载器来分配这一空间。

- 栈（stack）是一个动态增长和收缩的段，由栈帧（stack frames）组成。系统会为每个当前调用的函数分配一个栈帧。栈帧中存储了函数的局部变量（所谓自动变量）、实参和返回值。

- 堆（heap）是可在运行时（为变量）动态进行内存分配的一块区域。堆顶端称作 `program break`。

![image-20220526222503936](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B/image-20220526222503936.png)

在 Linux 操作系统中，虚拟地址空间的内部又被分为**内核空间和用户空间**两部分，不同位数的系统，地址空间的范围也不同。比如最常见的 32 位和 64 位系统，如下所示：

![img](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E8%BF%9B%E7%A8%8B/3a6cb4e3f27241d3b09b4766bb0b1124.png)

### 进程状态

操作系统中，进程是​资源分配的基本单位，每个进程都有自己的内存空间（虚拟内存），进程的状态，进程的资源，进程的输入输出等。这些信息被记录在操作系统中称为进程控制块（PCB）的结构中，在 `linux` 中的 `task_struct` 结构体中。

进程状态：

1. 创建态：进程正在被创建，操作系统需要为其分配资源、初始化PCB。
2. 就绪态：已经具备运行条件，位于进程的就绪队列中，等待进程调度被分配CPU资源。
3. 运行态：当前占据CPU资源的进程的状态。
4. 阻塞态：处于运行态的进程因发出某种资源请求，操作系统将其CPU资源剥夺，等待请求的事件满足，再变成就绪态，加入就绪队列。
5. 终止态：进程正在从系统中撤销，等待操作系统会回收进程拥有的资源、撤销PCB。

### 环境列表

每一个进程都有与其相关的称之为环境列表（environment list）的字符串数组，或简称为环境（environment）。其中每个字符串都以名称=值（name=value）形式定义。因此，环境是“名称-值”的成对集合，可存储任何信息。常将列表中的名称称为环境变量。

**新进程在创建之时，会继承其父进程的环境副本！**可通过命令`printenv`查看当前shell环境变量。

在 C 语言程序中，可以使用全局变量 `char **environ `访问环境列表。（C 运行时启动代码定义了该变量并以环境列表位置为其赋值。）`environ` 与 `argv` 参数类似，指向一个以 `NULL` 结尾的指针列表，每个指针又指向一个以空字节终止的字符串。

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

#### getenv 函数

`getenv()`函数能够从进程环境中检索单个值。

```c
#include <stdlib.h>

char *getenv(const char *name);
```

- name：指向要检索的环境变量名称的字符串。
- 返回值：如果找到该环境变量，则返回该环境变量的值，否则返回 NULL。

#### putenv 函数

有时，对进程来说，修改其环境很有用处。原因之一是这一修改对该进程后续创建的所有子进程均可见。另一个可能的原因在于设定某一变量，以求对于将要载入进程内存的新程序（“execed”）可见。从这个意义上讲，环境不仅是一种进程间通信的形式，还是程序间通信的方法。

```c
#include <stdlib.h>

int putenv(char *string);
```

- string：参数 string 是一指针，指向 name=value 形式的字符串。
- 返回值：如果成功，则返回 0，否则返回 非0值，并设置 errno。

调用 `putenv()` 函数后，该字符串就成为环境的一部分，换言之，`putenv` 函数将设定 `environ` 变量中某一元素的指向与 `string` 参数的指向位置相同，而非 `string` 参数所指向字符串的复制副本。

## 虚拟内存

大多数程序都展现了两种类型的局部性。

- 空间局部性（Spatial locality）：是指程序倾向于访问在最近访问过的内存地址附近的内存（由于指令是顺序执行的，且有时会按顺序处理数据结构）。
- 时间局部性（Temporal locality）：是指程序倾向于在不久的将来再次访问最近刚访问过的内存地址（由于循环）。

虚拟内存的规划之一是将每个程序使用的内存切割成**小型的、固定大小的“页”（page）单元**。相应地，将 RAM 划分成一系列与**虚存页尺寸相同的页帧**。任一时刻，每个程序仅有部分页需要驻留在物理内存页帧中。这些页构成了所谓驻留集（resident set）。程序未使用的拷贝保存在交换区（swap area）内—这是磁盘空间中的保留区域，作为计算机 RAM 的补充— 仅在需要时才会载入物理内存。若进程欲访问的页面目前并未驻留在物理内存中，将会发生页面错误（page fault），内核即刻挂起进程的执行，同时从磁盘中将该页面载入内存。

虚拟内存的详细介绍可看[为什么要有虚拟内存](https://xiaolincoding.com/os/3_memory/vmem.html)

## 进程控制块 PCB

进程控制块的主要内容有：

- 进程标识符（PID）：`pid_t pid;`
- 文件描述符表：`struct files_struct *files;`
- 进程状态：创建、就绪、运行、阻塞、终止。
- 进程的资源：内存、CPU、IO、磁盘等。
- 进程的优先级；
- 等等。

## fork 函数

基于当前进程创建一个子进程。

1. **父子进程相同**：data 段、text 段、堆、栈、环境变量、全局变量、宿主目录位置、进程工作目录位置、信号处理方式。
2. **父子进程不同**：进程 id、返回值、各自的父进程、进程创建时间、闹钟、未决信号集。
3. **父子进程共享**：
    - 读时共享、写时复制 —— 全局变量。
    - 文件描述符、mmap映射区。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);

// example
pid_t pid = fork();
if (pid == 0) {
    // child process
} else if (pid > 0) {
    // parent process
} else {
    // error
}
```

- 返回值：
  - 成功，父进程中返回子进程的 PID。
  - 子进程返回 0。
  - 失败，返回 -1，并设置 errno 。

### 循环创建子进程

```c
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

## getpid、getppid 函数

获取当前进程的 PID 和父进程的 PID。使用`pstree`命令可查看到这一“家族树”。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
```

- 返回值：
  - 成功，返回 PID。
  - 失败，这些函数总是成功的！

## exec 函数族

`fork` 创建子进程后执行的是和父进程相同的程序，但有可能执行不同的代码分支。子进程往往要调用一种 `exec` 函数以执行另一个程序。

`exec` 函数一旦调用成功，即执行新的程序，**不返回**。只有失败才返回，错误值-1，所以通常我们直接在 `exec` 函数调用后直接调用 `perror()`，和 `exit()`，无需 `if` 判断。

当进程调用一种 `exec` 函数时，该进程的用户空间代码和数据完全被新程序替换，从新程序的启动例程开始执行。调用 `exec` 并不创建新进程，所以调用 `exec` 前后该进程的 `id` 并未改变。

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

## 孤儿进程和僵尸进程

孤儿进程：父进程先于子进终止，子进程沦为“孤儿进程”，会被 `init` 进程领养。

僵尸进程：子进程终止，父进程尚未对子进程进行回收，在此期间，子进程为“僵尸进程”。 `kill` 对其无效。这里要注意，**每个进程结束后都必然会经历僵尸态**，时间长短的差别而已。

子进程终止时，子进程残留资源 PCB 存放于内核中，PCB 记录了进程结束原因，**进程回收就是回收 PCB**。回收僵尸进程，得 `kill` 它的父进程，让孤儿院去回收它

**为什么要回收僵尸进程**？僵尸进程虽然已经结束运行，不占CPU资源，但其`task_struct`结构仍然存在，其中保存了进程的`pid`、退出码等。尤其是`pid`，如果僵尸进程过多，最终耗尽了`pid`，那么将无法创建新的进程。

杀死父进程并不会导致子进程死亡，只是子进程的父进程会变为 `init`。

## wait、waitpid 函数

**回收子进程**，父进程阻塞等待子进程终止。

一次调用只能回收一个子进程，如果父进程需要回收多个子进程，可以多次调用 `wait` 函数。

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
```

- wstatus 用来接收子进程的终止状态。可设置为NULL，表示不接收。
- pid 参数：
  - 如果 pid 为 0，则等待任意同组的子进程终止；
  - 如果 pid 为 -1，则等待任意子进程终止；
  - 如果 pid 为 > 0，则等待指定的子进程终止；
  - 如果 pid < -1，则等待任意进程组 id 为 abs(pid) 的子进程终止。
- options 参数：
  - 0：阻塞等待；
  - WNOHANG：如果没有子进程终止，则立即返回，不阻塞父进程。

- 返回值：
  - `>0`，成功，返回子进程的 pid；
  - 0，调用时指定了 WNOHANG，并且没有子进程终止；
  - 失败，返回 -1，并设置 errno。

### waitpid 回收多个子进程

```c
int main() {
    int i;
    for (i = 0; i < 5; i++) {
        if (fork() == 0) { // child process
            break;
        }
    }
    pid_t wpid;
    if (i == 5) {
        while((wpid = waitpid(-1, NULL, WNOHANG)) != -1) {
            if (wpid > 0)
                printf("wait child %d\n", wpid);
            else
                sleep(1);
        }
        printf("this is parent process\n");
    } else {
        sleep(i);
        printf("this is %dth child process\n", i);
    }
    return 0;
}
```

## 进程组与会话

进程组：进程组是一组进程的集合，每个进程组都有一个组长，与进程ID与组ID相同。

默认子进程与父进程属于同一进程组，进程组ID == 第一个进程ID

只要进程组中有一个进程存在，进程组就存在，与组长进程是否终止无关。

```bash
kill -9 -PGID # 给进程组中的所有进程发送 SIGKILL
```

会话：会话是一组**进程组**的集合，每个会话都有一个会话ID(SID)。

同样，也可以给会话中的所有进程发送信号！

终端中运行的程序其会话ID与该终端的进程ID相同，因此下面的命令会将终端也关闭！

```bash
cat | cat | cat | wc -l # 任意启动多个进程

kill -9 -SID # 在另一个终端中发送 SIGKILL
```

### 会话

创建会话的 6 点注意事项：

1. 调用进程不能是进程组组长，该进程变成新会话首进程
2. 该进程成为一个新进程组的组长进程
3. 需要 root 权限（ubuntu 不需要）
4. 新会话丢弃原有的控制终端，该会话没有控制终端
5. 该调用进程是组长进程，则出错返回
6. 建立新会话时，先调用 `fork`，父进程终止，子进程调用 `setsid`

### getsid 函数

获取进程的会话 id

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getsid(pid_t pid);
```

- pid 参数：
  - 如果 pid 为 0，则获取当前进程的会话 id；
  - 如果 pid 为 > 0，则获取指定进程的会话 id；
- 返回值：
  - 返回会话 id，失败返回 -1，并设置 errno。

### setsid 函数

创建一个新会话，并以自己的 ID 设置进程组 ID，同时也是新会话的 ID！

```c
#include <sys/types.h>
#include <unistd.h>

pid_t setsid(void);
```

- 返回值：成功返回调用进程的会话 ID，失败返回-1，设置 error

## 守护进程

`daemon` 进程。通常运行于操作系统后台，脱离控制终端。一般不与用户直接交互。周期性的等待某个事件发生或周期性执行某一动作。

**不受用户登录注销影响**。通常采用以 d 结尾的命名方式。

创建守护进程，最关键的一步是调用 `setsid` 函数创建一个新的 `Session`，并成为 Session Leader。

创建守护进程的步骤：

1. fork 子进程，让父进程终止。
2. 子进程调用 `setsid()` 创建新会话。
3. 通常根据需要，改变工作目录位置 `chdir()`，防止目录被卸载。
4. 通常根据需要，重设 umask 文件权限掩码，影响新文件的创建权限。022 -- 755
5. 通常根据需要，关闭/重定向 文件描述符（主要是针对 0, 1, 2，一般是重定向到 `/dev/null`）。
6. 守护进程 业务逻辑。`while()`

```c
int main(int argc, char *argv[])
{
    pid_t pid;
    int ret, fd;
    pid = fork();
    if (pid > 0)    // 1. 创建子进程，父进程终止
        exit(0);

    pid = setsid(); // 2. 子进程调用 `setsid()` 创建新会话
    if (pid == -1)
        printf("setsid error");

    ret = chdir("/home/zhcode/Code/code146"); // 3. 通常根据需要，改变工作目录位置
    if (ret == -1)
        printf("chdir error");

    umask(0022);    // 4. 通常根据需要，重设 umask 文件权限掩码

    close(STDIN_FILENO);    // 5. 通常根据需要，关闭/重定向 文件描述符
    fd = open("/dev/null", O_RDWR); // fd --> 0
    if (fd == -1)
        printf("open error");

    dup2(fd, STDOUT_FILENO); // 重定向 stdout 和 stderr
    dup2(fd, STDERR_FILENO);

    while(1);     // 6. 守护进程 业务逻辑
    return 0;
}
```

## 参考资料

- [PCB 进程控制块](https://www.cnblogs.com/yungyu16/p/13024626.html)
- [PCB(task_struct)源码](https://elixir.bootlin.com/linux/v5.4.190/source/include/linux/sched.h#L624)
- [操作系统--进程](https://github.com/szza/LearningNote/blob/master/4.%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/3_%E8%BF%9B%E7%A8%8B.md)
- [物理内存和虚拟内存的映射关系](https://www.bilibili.com/video/BV1KE411q7ee?p=77)
- [小林coding--为什么要有虚拟内存？](https://xiaolincoding.com/os/3_memory/vmem.html#linux-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)
- [小林coding--进程管理](https://xiaolincoding.com/os/4_process/process_base.html)
- 《Linux系统编程——6.4虚拟内存管理》

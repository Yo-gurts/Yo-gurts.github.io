---
title: Linux系统编程-信号
top_img: transparent
date: 2022-05-02 21:49:35
updated: 2022-08-23 21:49:35
tags:
  - Linux
categories: Linux
keywords:
description: Linux 编程中的信号
---

## 信号

1. 信号是软件层面上的“中断”。一旦信号产生，无论程序执行到什么位置，必须立即停止运行，处理信号，处理结束，再继续执行后续指令。
2. 所有信号的产生及处理全部都是由【内核】完成的。
3. 简单、不能携带大量信息、满足条件才发送。

### 信号的生命周期

1. 产生：
2. 未决：产生与递达之间状态。
3. 递达：产生并且送达到进程。直接被内核处理掉。
4. 处理：执行默认处理动作、忽略、捕捉（自定义）

阻塞信号集（信号屏蔽字）：进程控制块PCB中的变量（位图），用来记录信号的屏蔽状态。被屏蔽的信号，在解除屏蔽前，一直处于未决态。

未决信号集：进程控制块PCB中的变量（位图），用来记录信号的处理状态。

- 信号产生时，未决信号集中对应位立刻翻转为1，表示信号处于未决状态。信号被处理后对应位翻转回0，这一过程往往非常短暂。
- 信号产生后，由于某些原因（主要是阻塞）不能递达，一直处理未决状态。
- 设置阻塞后，同一信号多次产生，也只能记录一次（也只会被处理一次）。

```c
struct task_struct {
    /* Signal handlers: */
    struct signal_struct    *signal;
    struct sighand_struct   *sighand;
    sigset_t                blocked;
    sigset_t                real_blocked;

    /* Restored if set_restore_sigmask() was used: */
    sigset_t                saved_sigmask; // 信号屏蔽字
    struct sigpending       pending;    // 未决信号集
    unsigned long           sas_ss_sp;
    size_t                  sas_ss_size;
    unsigned int            sas_ss_flags;
}
```

### 产生信号

1. 按键产生，如：`ctrl+c, ctrl+z, ctrl+\`
2. 系统调用产生，如：`kill, raise, abort`
3. 软件条件产生，如：定时器`alarm`，或者该进程的某个子进程退出
4. 硬件异常产生，如：非法访问内存（段错误）、除0（浮点数例外）、内存对齐出错（总线错误）
5. 命令产生， 如：`kill -9`

### 信号四要素

**编号、名称、事件、默认处理动作**！

总计有64个信号（**可通过`kill -l`查看信号名称及编号**），`man 7 signal`也可查看信号。前32个为常规信号，后32个为实时信号。

信号到达后，进程视具体信号执行如下默认操作之一：

- Term   终止（杀死）进程，这有时是指进程异常终止，而不是进程因调用 `exit()` 而发生的正常终止。
- Ign    忽略信号，也就是说，内核将信号丢弃，信号对进程没有产生任何影响
- Core   产生核心转储文件，同时进程终止：核心转储文件包含对进程虚拟内存的镜像，可将其加载到调试器中以检查进程终止时的状态
- Stop   停止进程：暂停进程的执行
- Cont   于之前暂停后再度恢复进程的执行

```bash
名称       编号      动作      事件
Signal   x86/ARM   Action   Comment
────────────────────────────────────────────────────────────────
SIGHUP       1      Term    Hangup detected on controlling terminal
SIGINT       2      Term    Interrupt from keyboard
SIGQUIT      3      Core    Quit from keyboard
SIGILL       4      Core    Illegal Instruction
SIGTRAP      5      Core    Trace/breakpoint trap
SIGABRT      6      Core    Abort signal from abort(3)
SIGIOT       6      Core    IOT trap. A synonym for SIGABRT
SIGBUS       7      Core    Bus error (bad memory access)
SIGFPE       8      Core    Floating-point exception
SIGKILL      9      Term    Kill signal
SIGUSR1     10      Term    User-defined signal 1
SIGSEGV     11      Core    Invalid memory reference
SIGUSR2     12      Term    User-defined signal 2
SIGPIPE     13      Term    Broken pipe: write to pipe with no
SIGALRM     14      Term    Timer signal from alarm(2)
SIGTERM     15      Term    Termination signal
SIGSTKFLT   16      Term    Stack fault on coprocessor (unused)
SIGCHLD     17      Ign     Child stopped or terminated
SIGCONT     18      Cont    Continue if stopped
SIGSTOP     19      Stop    Stop process
SIGTSTP     20      Stop    Stop typed at terminal see also seccomp(2)
SIGTTIN     21      Stop    Terminal input for background process
SIGTTOU     22      Stop    Terminal output for background process
SIGURG      23      Ign     Urgent condition on socket (4.2BSD)
SIGXCPU     24      Core    CPU time limit exceeded (4.2BSD);
SIGXFSZ     25      Core    File size limit exceeded (4.2BSD);
SIGVTALRM   26      Term    Virtual alarm clock (4.2BSD)
SIGPROF     27      Term    Profiling timer expired
SIGWINCH    28      Ign     Window resize signal (4.3BSD, Sun)
SIGIO       29      Term    I/O now possible (4.2BSD)
SIGPWR      30      Term    Power failure (System V)
SIGSYS      31      Core    Bad system call (SVr4);
SIGUNUSED   31      Core    Synonymous with SIGSYS
```

`SIGKILL(9), SIGSTOP(19)` 不能被捕捉、阻塞、忽略。

重点是下面的信号：

```bash
SIGHUP       1   Term   当用户退出shell时，由该shell启动的所有进程都会收到该信号。
SIGINT       2   Term   用户按下 ctrl+c
SIGQUIT      3   Core   用户按下 ctrl+\
SIGABRT      6   Core   调用 abort() 函数
SIGBUS       7   Core   非法访问内存地址，包括内存对齐出错
SIGFPE       8   Core   浮点数错误、溢出、除数为0等所有运算错误
SIGKILL      9   Term   Kill信号，不能被捕捉、阻塞、忽略
SIGUSR1     10   Term   用户定义的信号1
SIGSEGV     11   Core   进行了无效的内存访问
SIGUSR2     12   Term   用户定义的信号2
SIGPIPE     13   Term   向一个没有读端口的管道写数据
SIGALRM     14   Term   定时器超时
SIGTERM     15   Term   结束信号，kill缺省发送此信号，与KILLSIG不同，可以阻塞、忽略
SIGCHLD     17   Ign    子进程状态发生变化时（进程恢复执行也可能发送），默认忽略此信号
```

### SIGSEGV

段错误，进行了**非法的内存**访问！什么为非法呢？进程中使用的地址是虚拟地址，虚拟地址是按页划分的，每个虚拟页对应一个实际的物理页，但并不是一开始就为每一个虚拟页分配了一个物理页，而是用到了才分配。申请内存时也同样只申请虚拟内存，只要虚拟内存地址有效，就是合法的。

以堆内为例，堆是一段长度可变的**连续虚拟内存**，始于进程的**未初始化数据段末尾**，通常将堆的内存边界（堆顶）称为`program break`，最初，`program break` 正好位于未初始化数据段末尾之后。在堆上分配内存那首先得扩展堆的大小，也就是抬升堆顶`program break`，这样程序就可以访问**新分配区域内**的任何内存地址。（试想一下，如果一开始直接访问堆区域外的地址会发生什么？）

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main() {
    printf("program break: %p\n", sbrk(0)); // sbrk(0) 返回堆顶的地址
    getchar();  getchar();
    printf("访问堆顶以上的内存: %d\n", (int *)(sbrk(0) + 100));

    int *t = malloc(20000*sizeof(int)); // 可以看到堆顶提升
    t[10000] = 10000;
    printf("access array t: %p\n", t);
    printf("access array t[10000]: %d, addr: %p\n", t[10000], t+10000);
    printf("program break: %p\n", sbrk(0));

    getchar();  getchar();

    // 超过128k 不会调整program break, 而是使用mmap
    int *t2 = malloc(50000*sizeof(int));
    t2[10000] = 10000;
    printf("access array t2: %p\n", t2);
    printf("access array t2[10000]: %d, %p\n", t2[10000], t2+10000);
    printf("program break: %p\n", sbrk(0));
    getchar();  getchar();

    return 0;
}
```

通过 `/proc/pid/maps` 文件可以查看进程的内存布局（显示的是虚拟地址）。如果**指针指向的地址不在该文件中任何一个内存段范围内**，就说明该内存地址是“非法的、无效的”。不过即使内存地址合法，如果**没有对应内存段的访问权限**，也会报“非法的内存访问”。使用该地址时，会先通过MMU中查询，如果是出现了页的权限错误，或者是操作系统发现并没有页面可以换入（未分配的），那么它会触发一个“软中断”。在Linux中，所谓的软中断其实就是一个信号(`signal`)，由于访问非法内存地址导致的错误叫作段错误`(segment fault)`，它会发射一个`SIGSEGV`信号，默认的行为就是终止这个程序。

在上面的测试代码中，在并没有调用任何`malloc()`时，通过 `/proc/pid/maps` 文件可以看到堆顶的位置在一个大小为`132k`的内存端的起始位置，此时哪怕直接访问堆顶以上的数据（只要不超过`132k`的范围）都不会报段错误！而进行任何大小的`malloc(1)`调用，使得堆顶提升`132k`。也就是说虽然堆大小为0，但实际上操作系统已经为该进程分配了`132k`的堆内存。

## kill 函数

向指定进程发送指定的信号。默认发送的信号为`SIGTERM`，该信号可以被捕获，所以不一定能杀死进程，但`SIGKILL(9)`总能**一击必杀**。

```c
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int sig);

// example
if (fork() == 0) {
    printf("pid:%d, ppid:%d\n", getpid(), getppid());
    kill(getppid(), SIGBUS); // 向父进程发送SIGBUS信号
} else {
    printf("parent\n");
    sleep(1);
}
```

- sig：信号编号，不推荐直接使用数字，应使用宏名。
- pid：进程编号
  - pid > 0：指定进程编号
  - pid = 0：发给与kill调用者属于同一进程组的所有进程，包括调用进程自身
  - pid = -1：发给进程有权限发送的 所有进程，除去 init（进程 ID 为 1）和调用进程自身。如果特权级进程发起这一调用，那么会发送信号给系统中的所有进程
  - pid < -1：发给进程组编号为abs(pid)的进程组中的 所有进程
- 返回值：成功返回0，失败返回-1

进程组: 每个进程都属于一个进程组，进程组是一个或多个进程集合，他们相互关联，共同完成一个实体任务，每个进程组都有一个进程组长，默认进程组ID 与进程组长ID 相同。

权限保护: super 用户(root)可以发送信号给任意用户，普通用户是不能向系统用户发送信号的。 kill -9 (root 用户的pid) 是不可以的。同样，普通用户也不能向其他普通用户发送信号，终止其进程。 只能向自己创建的进程发送信号。普通用户基本规则是: **发送者实际或有效用户ID == 接收者实际或有效用户ID**

### SIGQUIT

当用户在键盘上键入退出字符（通常为 `Control-\`）时，该信号将发往前台进程组。默认情况下，该信号终止进程，并生成可用于调试的核心转储文件。进程如果陷入无限循环，或者不再响应时，使用 `SIGQUIT` 信号就很合适。键入 `Control-\`，再调用 `gdb` 调试器加载刚才生成的核心转储文件，接着用 `backtrace` 命令来获取堆栈跟踪信息，就能发现正在执行的是程序的哪部分代码。

## alarm 函数

设置定时器（闹钟），指定的 seconds 后，内核给进程发送 SIGALRM 信号，默认动作为终止进程。

**每个进程有且只有一个定时器**。

```c
#include <unistd.h>

unsigned int alarm(unsigned int seconds);
```

- seconds：设置定时器超时时间，单位为秒
- 返回值：返回之前设置的定时器剩余的时间，之前没有设置返回0。无失败！

计时与进程状态无关，自然计时法。

## setitimer 函数

与 alarm 功能类似，精度为微秒，可以实现周期定时。

```c
#include <sys/time.h>

int getitimer(int which, struct itimerval *curr_value);
int setitimer(int which, const struct itimerval *new_value,
                struct itimerval *old_value);

struct itimerval {
    struct timeval it_interval; /* 周期定时 */
    struct timeval it_value;    /* 下一次时间 */
};
struct timeval {
    time_t      tv_sec;         /* seconds */
    suseconds_t tv_usec;        /* microseconds */
};

// example 第一次两秒后执行，后续每隔五秒执行一次
void myfunc(int signo) {
    printf("hello\n");
}

int main() {
    signal(SIGALRM, myfunc); // 注册信号的处理函数
    struct itimerval new_value;
    new_value.it_interval.tv_sec = 5;
    new_value.it_interval.tv_usec = 0;
    new_value.it_value.tv_sec = 2;
    new_value.it_value.tv_usec = 0;

    setitimer(ITIMER_REAL, &new_value, NULL);
    while(1);
    return 0;
}
```

- which：定时器类型
  - ITIMER_REAL：真实时间（自然计时），发送信号 SIGALRM
  - ITIMER_VIRTUAL：虚拟空间计时（用户空间），发送信号 SIGVTALRM。进程占用CPU时间
  - ITIMER_PROF：运行时计时（用户+内核），发送信号 SIGPROF。进程占用CPU时间及系统调用时间
- new_value：设置新的定时器值，如果为 NULL，则不设置新的定时器值，只返回旧的定时器值
- old_value：返回旧的定时器值，如果为 NULL，则不返回旧的定时器值
- 返回值：成功返回0，失败返回-1

`time` 命令查看程序执行时间，real总时间，user用户空间时间，sys系统空间时间。

## raise 函数

有时，进程需要向自身发送信号。`raise()`函数就执行了这一任务。

```c
#include <signal.h>

int raise(int sig);
```

在单线程程序中，调用 `raise()`相当于对 `kill()`的如下调用：

```c
kill(getpid(), sig);
```

支持线程的系统会将 `raise(sig)` 实现为：

```c
pthread_kill(pthread_self(), sig);
```

注意，`raise()`出错将返回非 0 值（不一定为–1）。调用 `raise()` 唯一可能发生的错误为 EINVAL，即 sig 无效。

## abort 函数

函数 `abort()` 终止其调用进程，并生成核心转储。

```c
#include <stdlib.h>

void abort(void);
```

函数 abort()通过产生 SIGABRT 信号来终止调用进程。对 SIGABRT 的默认动作是产生核心转储文件并终止进程。调试器可以利用核心转储文件来检测调用 `abort()` 时的程序状态。

如果 abort()成功终止了进程，那么还将刷新 stdio 流并将其关闭。

## 信号集操作函数

信号集`sigset_t`虽然是位图，但一般不允许直接对PCB中的信号集进行操作，而是先自定义一个信号集，然后操作这个信号集，最后再把信号集通过提供的函数与PCB中的信号集进行运算（与、或、覆盖等）。

```c
#include <signal.h>

sigset_t set;
int sigemptyset(sigset_t *set);     // 全部清零
int sigfillset(sigset_t *set);      // 全部置1
int sigaddset(sigset_t *set, int signum); // 将 signum 置1
int sigdelset(sigset_t *set, int signum); // 将 signum 置0
int sigismember(const sigset_t *set, int signum); // 判断 signum 是否为1
```

## sigprocmask 函数

```c
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

- how：操作方式
  - SIG_BLOCK：设置屏蔽，`mask = mask | set`
  - SIG_UNBLOCK：解除屏蔽，`mask = mask & ~set`
  - SIG_SETMASK：覆盖，不推荐使用，`mask = set`
- set：自定义的传入信号集
- oldset：返回旧的信号集
- 返回值：成功返回0，失败返回-1

## sigpending 函数

读取当前进程的未决信号集。并将其置于 set 指向的 sigset_t 结构中。随后可以使用 `sigismember()`函数来检查 set。

```c
#include <signal.h>

int sigpending(sigset_t *set);

// example
sigset_t set;
sigpending(&set);
```

- set：传出参数，返回未决信号集
- 返回值：成功返回0，失败返回-1

```c
void print_set(sigset_t *set) {
    for (int i=1; i < 33; i++) {
        if (sigismember(set, i))
            printf("has : %d\n", i);
    }
}

int main() {

    sigset_t pend, in;
    sigemptyset(&in);
    sigaddset(&in, SIGINT);
    sigprocmask(SIG_BLOCK, &in, NULL);

    while(1) {
        sigpending(&pend);
        print_set(&pend);
        sleep(2);
    }
    return 0;
}
```

## signal 函数

UNIX 系统提供了两种方法来改变信号处置：`signal()` 和 `sigaction()`。`signal()`的行为在不同 UNIX 实现间存在差异，这也意味着对可移植性有所追求的程序绝不能使用此调用来建立信号处理器函数。故此，`sigaction()`是建立信号处理器的首选 API（强力推）。

注册一个信号捕捉函数，当收到信号时，调用该函数。

```c
#include <signal.h>

typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

- signum：要注册的信号编号，**该参数可以是除去 `SIGKILL` 和 `SIGSTOP` 之外的任何信号**。
- handler：信号处理函数，注意这里是函数指针，函数的参数是信号编号
- 返回值：成功返回之前的信号处理函数，失败返回`SIG_ERR`，并设置errno

在为 `signal()`指定 handler 参数时，可以以如下值来代替函数地址：

- **SIG_DFL** 将信号处置重置为默认值。这适用于将之前 signal()调用所改变的信号处置还原。
- **SIG_IGN** 忽略该信号。如果信号专为此进程而生，那么内核会默默将其丢弃。进程甚至从未知道曾经产生了该信号。

## sigaction 函数

同 signal 函数，注册一个信号捕捉函数，在建立信号处理器程序时，`sigaction()`较之 `signal()`函数可移植性更佳。

`sigaction()`允许在获取信号处置的同时无需将其改变，并且，还可设置各种属性对调用信号处理器程序时的行为施以更加精准的控制。

```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,
                struct sigaction *oldact);
struct sigaction {
    void     (*sa_handler)(int);   // 信号处理函数
    void     (*sa_sigaction)(int, siginfo_t *, void *); // 一般不用
    sigset_t   sa_mask; // 信号屏蔽字，只作用于信号捕捉函数执行期间！
    int        sa_flags; // =0 时，信号捕捉函数执行期间屏蔽当前信号
    void     (*sa_restorer)(void);  // 废弃
};

// example
void print_set(int signo) {
    printf("catch you : %d\n", signo);
}
int main() {
    struct sigaction act, oldact;
    act.sa_handler = print_set;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    sigaction(SIGINT, &act, &oldact);

    while(1);
    return 0;
}
```

### 信号捕捉特性

1. 进程正常运行时，默认 PCB 中有一个信号屏蔽宇，假定为 X，它决定了进程自动屏蔽哪些信号。当注册了某个信号捕捉函数，捕捉到该信号以后，要调用该函数。而该函数有可能执行很长时间，在这期间所屏蔽的信号不由X来指定。而是用 sa_mask 来指定。调用完信号处理函数，再恢复为 X。
2. XXX 信号捕捉函数执行期间，XXX 信号自动被屏蔽（sa_flags=0）。
3. 阻塞的常规信号不支持排队，产生多次只记录一次。(后 32 个实时信号支持排队)
4. XXX 信号捕捉函数执行期间，若信号 B 未被屏蔽，当信号 B 递达时，会从当前的执行函数跳转到 B 信号的处理函数。（信号就像中断，来了就打断当前操作）。

### sa_flags

- **SA_RESTART**：当一个系统调用被信号中断时，信号处理结束恢复执行系统调用。（不幸的是，并非所有的系统调用都可以通过指定 SA_RESTART 来达到自动重启的目的。）
- **SA_NODEFER**：捕获该信号时，不会在执行处理器程序时将该信号自动添加到进程掩码中（即不屏蔽当前信号）。

### 可重入函数

SUSv3 对可重入函数的定义是：**函数由两条或多条线程调用时，即便是交叉执行，其效果也与各线程以未定义顺序依次调用时一致。**

更新全局变量或静态数据结构的函数可能是不可重入的，在 C 语言标准函数库中，这种可能性非常普遍。只用到本地变量的函数肯定是可重入的。

`malloc()`和 `free()`就维护有一个针对已释放内存块的链表，用于从堆中重新分配内存。

将静态数据结构用于内部记账的函数也是不可重入的。其中最明显的例子就是 **stdio 函数库成员（`printf()`、`scanf()`等）**，它们会为缓冲区 I/O 更新内部数据结构。如果在信号处理器函数中调用了 printf()，而主程序又在调用 printf()或其他 stdio 函数期间遭到了处理器函数的中断，那么有时就会看到奇怪的输出，甚至导致程序崩溃或者数据的损坏。

### 异步信号安全函数

异步信号安全的函数是指当从信号处理器函数调用时，可以保证其实现是安全的。如果某一函数是可重入的，又或者信号处理器函数无法将其中断时，就称该函数是异步信号安全的。

## pause 函数

调用 `pause()`将暂停进程的执行，直至信号处理器函数中断该调用为止（即有信号到达）。

```c
#include <unistd.h>

int pause(void);
```

处理信号时，`pause()`遭到中断，并总是返回−1，并将 errno 置为 EINTR。

## SIGCHLD 信号

子进程状态发生变化时，父进程会收到 SIGCHLD 信号。

- 子进程终止时。
- 子进程接收到 SIGSTOP 信号停止时。
- 子进程处在停止态，接受到SIGCONT 后唤醒时

借助 SIGCHILD 信号回收子进程。

```c
#include <fcntl.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>

void catch_chld(int signo) {
    pid_t wpid;
    // 注意这里使用的阻塞的方式回收！当不存在待回收的子进程时，返回-1。
    while((wpid = waitpid(-1, NULL, 0)) != -1) {
        printf("catch child id: %d\n", wpid);
    }
}

int main() {
    pid_t pid;
    sigset_t set;
    // 先阻塞信号，防止父进程还未注册处理函数，子进程就结束了
    sigaddset(&set, SIGCHLD);
    sigprocmask(SIG_BLOCK, &set, NULL);

    int i;
    for (i = 0; i < 15; i++)
        if ((pid = fork()) == 0)
            break;

    if (i == 15) {
        struct sigaction act;
        act.sa_handler = catch_chld;
        sigemptyset(&act.sa_mask);
        act.sa_flags = 0;
        sigaction(SIGCHLD, &act, NULL);
        // 解除阻塞
        // sigdelset(&set, SIGCHLD);
        sigprocmask(SIG_UNBLOCK, &set, NULL);

        printf("i'm parent, pid = %d\n", getpid());
        while(1);
    } else {
        printf("i'm child, pid = %d\n", getpid());
    }

    return 0;
}
```

## 总结

- 了解信号产生条件，信号默认处理方式，修改信号默认处理方式。
- 信号处理函数要简洁！且必须保证为**可重入函数**或**异步信号安全的函数**。
- `SIGKILL, SIGSTOP`无法捕获，无法修改处理方式。
- `SIGCHLD`信号不一定是子进程结束！
- `SIGABRT`信号（`control + \`）可生成核心转储文件，非常方便调试程序。

## 参考资料

- [PCB 中信号相关内容 L913~L923](https://elixir.bootlin.com/linux/v5.4.190/source/include/linux/sched.h#L913)

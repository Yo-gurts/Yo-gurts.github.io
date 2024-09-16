---
title: Linux 内核-中断
top_img: transparent
date: 2023-08-29 14:49:35
updated: 2023-08-29 14:49:35
tags:
  - Linux
categories: Linux
keywords:
description:
---

## Linux 内核 — 中断

## 什么是中断

> 本节更多的是对中断涉及的内容进行简单介绍，后续才会对各个内容详细介绍。

在 Linux 系统中，中断（`Interrupt`）是一种硬件或软件事件，**它可以打断正在执行的 `CPU` 指令序列，使 `CPU` 转而处理某个特定事件**。中断机制允许外部设备或系统组件通过发送信号来通知 `CPU` 某种事件的发生，这可以是来自硬件设备（如键盘、鼠标、硬盘等）的输入、定时器到期、网络数据包到达等等。

中断的主要目的是提供一种异步通信的方式，让 CPU 能够及时响应外部事件，而不需要持续地轮询检查事件是否发生。这样可以大大提高系统的效率和响应速度。

> 但**中断过多反而会降低系统的效率**！在网络收包处理中，往往采用中断+轮询的方式。

在 Linux 系统中，**每种类型的中断都有一个唯一的标识号，被称为中断号**。当一个中断事件发生时，CPU 会立即暂停当前执行的指令，保存当前上下文，然后跳转到与中断号相关联的中断处理程序。中断处理程序是一段预先编写好的代码，负责处理特定类型的中断事件。处理完中断事件后，CPU 会恢复之前保存的上下文，并继续执行被中断的指令。

> 1. 中断号的分配方式。
> 2. 大量中断触发时，上下文保存`_raw_spin_unlock_irq`与恢复`_raw_spin_unlock_irqrestore`往往会成为性能瓶颈。

Linux 内核有一个**中断控制器**（`Interrupt Controller`），它管理着所有中断并决定它们的优先级和处理顺序。中断可以分为两类：**硬件中断**和**软件中断**。硬件中断是由外部设备引发的，例如，当键盘按键被按下时会触发一个硬件中断。软件中断是通过特殊的指令来触发的，通常用于执行系统调用或与内核进行通信。

> 1. 中断有优先级。
> 2. 软件也可以触发中断。

总而言之，中断在 Linux 系统中是一种重要的机制，它允许外部事件以异步的方式通知 CPU，从而使系统能够更高效地响应各种事件。

### 硬件中断和软件中断

硬中断就是由**外部**硬件设备触发的，本质上来说，就是改变某个引脚的电压电平，比如引脚输入由高电平变为低电平（下降沿触发），该变化会被处理器的**中断控制器**检测到，从而跳转到中断处理程序执行。也有一些设备支持上升沿触发、或者**电平触发**，具体的触发方式需要参考设备的技术规格或文档。

> 1. 中断控制器的原理。
> 2. 中断处理程序和中断如何关联，如何切换到中断处理函数执行。
> 3. “外部”：这里所说的外部是针对 `CPU` 而言的，除了传统意义上的外设硬盘、键盘、鼠标、网络接口卡等，`TLB`、`DMA` 等模块也能发起硬件中断。

操作系统收到了中断请求，会打断其他进程的运行，所以**中断请求的响应程序，也就是中断处理程序，要尽可能快的执行完，这样可以减少对正常进程运行调度地影响**。而且中断处理程序可能会暂时关闭中断，这时如果中断处理程序执行时间过长，可能在还未执行完中断处理程序前，会丢失当前其他设备的中断请求

Linux 系统为了解决中断处理程序执行过长和中断丢失的问题，将**中断过程分成了两个阶段**，分别是**「上半部和下半部分」**。

- 上半部用来**快速处理中断**，一般会暂时关闭中断请求，主要负责处理跟硬件紧密相关或者时间敏感的事情。
- 下半部用来**延迟处理**上半部未完成的工作，一般以**「内核线程」**的方式运行。

> 参考这里的例子：[什么是软中断](https://www.xiaolincoding.com/os/1_hardware/soft_interrupt.html#%E4%B8%AD%E6%96%AD%E6%98%AF%E4%BB%80%E4%B9%88)

- **上半部直接处理硬件请求，也就是硬中断**，主要是负责耗时短的工作，特点是快速执行；
- **下半部是由内核触发，也就说软中断**，主要是负责上半部未完成的工作，通常都是耗时比较长的事情，特点是延迟执行；

> ~~注意这里将下半部说为是软中断感觉并不太准确，应该是 `tasklet`~~。
>
> 软中断这个术语存在一定的歧义，这里他是作为**可延迟函数**的总称，既包括了 `softirq`，也包括 `tasklet`。需要根据上下文推断其意思。

还有一个区别，硬中断（上半部）是会打断 CPU 正在执行的任务，然后立即执行中断处理程序，而软中断（下半部）是以内核线程的方式执行，并且每一个 CPU 都对应一个**软中断内核线程**，名字通常为**「`ksoftirqd/CPU` 编号」**，比如 `0` 号 CPU 对应的软中断内核线程的名字是 `ksoftirqd/0`

> 1. 软中断内核线程的作用，与实现
> 2. 中断是与 `CPU` 绑定的！

不过，**软中断不只是包括硬件设备中断处理程序的下半部，一些内核自定义事件也属于软中断**，比如**内核调度**等、RCU 锁（内核里常用的一种锁）等。

> 1. 系统调用也可以看作是一种软中断吧，涉及到特权级别的切换和上下文的保存与恢复。
> 2. 我的理解是只要涉及到上下文的保存与恢复，就可以看作是一种中断。
> 3. 内核调度这里是指线程切换的过程还是调度系统本身呢？

### 同步中断与异步中断

将中断可以按触发源分为硬件中断和软件中断，另一种叫法为**同步中断与异步中断**：

- **同步中断和异常**：这些由 CPU 自身产生，针对当前执行的程序。异常可能因种种原因触发：由于运行时发生的程序设计错误（典型的例子是除 0），或由于出现了异常的情况或条件，致使处理器需要“外部”的帮助才能处理。
    异常情况不见得是由进程直接导致的，但必须借助于内核才能修复。一个可能的例子是缺页异常，在进程试图访问虚拟地址空间的一页，而该页不在物理内存中时，才会发生此类异常。根据第 4 章的讨论，内核必须与 CPU 交互，确保将预期的数据取入物理内存。接下来，进程可以在发生异常的位置恢复执行。由于内核自动恢复了这种情况，进程甚至不会注意到缺页异常的存在。
- **异步中断**。这是经典的中断类型，由外部设备产生，可能发生在任意时间。不同于同步中断，异步中断并不与特定进程关联。它们可能发生在任何时间，而不牵涉系统当前执行的活动。网卡通过发出一个相关的中断来报告新分组的到达。因为数据可能在任意时刻到达系统，所以当前执行的很可能是与数据无关的某个进程或其他东西。为避免损害该进程，内核必须确保中断能够尽快处理完毕（通过缓冲数据），使得 CPU 时间能够返还给当前进程。这也是内核需要延期操作机制的原因。

> 从介绍可以看出，同步中断其实就是软件中断，异步中断为硬件中断，只是说法不同。

两类中断的共同特性是什么？如果 CPU 当前不处于核心态，则发起从用户态到核心态的切换。接下来，在内核中执行一个专门的例程，称为中断服务例程（`interrupt service routine`，简称 **ISR**）或中断处理程序（`interrupt handler`）。

### IRQ 编号与中断号

每个中断都有一个编号。如果中断号 `n` 分配给一个网卡而 `m≠ n` 分配给 `SCSI` 控制器，那么内核即可区分两个设备，并在中断发生时调用对应的 `ISR` 来执行特定于设备的操作。当然，同样的原则也适应于异常，不同的异常指派了不同的编号。

遗憾的是，由于特别设计（通常是历史上的）的“特性”（`IA-32` 体系结构就是一个恰当的特例），情况并不总是像描述的那样简单。因为**只有很少的编号可用于硬件中断，所以必须由几个设备共享一个编号**。在 `IA-32` 处理器上，硬件中断的最大数目通常是 15，这个值可不怎么大，还有考虑到有些中断编号已经永久性地分配给了标准的系统组件（键盘、定时器，等等），因而限制了可用于其他外部设备的中断编号数目。这个过程称为**中断共享**（`interrupt sharing`）。但必须硬件和内核同时支持才能使用该技术，因为必须要识别出中断来源于哪个设备。

> 很自然，精巧设计的总线系统无需该方案。这种系统为硬件设备提供了很多中断，根本不需要共享。

中断不能由处理器外部的外设直接产生，而必须借助于一个称为**中断控制器** (`interrupt controller`) 的标准组件来**请求**，该组件存在于每个系统中。

外部设备（或其槽位），会有电路连接到用于向中断控制器发送中断请求的组件。控制器在执行了各种电工任务（我们对此没有更多兴趣）之后，**将中断请求转发到 CPU 的中断输入**。因为外部设备不能直接发出中断，而必须通过上述组件请求中断，所以这种请求更正确的叫法是**IRQ**，或**中断请求（interrupt request）**。因为就软件而言，IRQ 和中断之间的差别不是那么大，这两个术语通常可替换使用。

对大多数 `CPU` 来说，都只是从可用于处理硬件中断的整个中断号范围抽取一小部分使用。抽取出的范围通常位于所有中断号序列的中部，例如，`IA-32 CPU` 总共提供了 16 个中断号，从 32 到 47。

如果读者曾经在 IA-32 系统上配置过 I/O 扩展卡，或研究过 `/proc/interrupts` 的内容，那么就会了解到，扩展卡的 `IRQ` 编号从 0 开始，到 15 结束，当然，前提是使用了典型的中断控制器 `8256A`。这意味着这里同样有 16 个不同的选项，但数值不同。中**断控制器除了负责 `IRQ` 信号的电工处理之外，还会对 `IRQ` 编号和中断号进行一个“转换”**。在 IA-32 系统上，加 32 即可。如果设备发出 `IRQ` 9，`CPU` 将产生中断 41，在安装中断处理程序时必须考虑到这一点。其他体系结构在中断号和 `IRQ` 编号之间采用其他映射方式，这里不会详细阐述。

> 注意上面这个例子来理解 `IRQ` 编号与中断号的区别！`IRQ` **号主要用于硬件中断的标识，而软件中断则使用其他机制来进行标识和处理**。

### 中断处理程序

在 `CPU` 得知发生中断后，它将进一步的处理委托给一个软件例程，该例程可能会修复故障、提供专门的处理或将外部事件通知用户进程。由于每个中断和异常都有唯一的编号，内核使用一个数组，数组项是指向处理程序函数的指针。相关的中断号根据数组项在数组中的位置判断，如图所示。

![image-20230829210639960](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230829210639960.png)

1. **进入和退出任务**：如下图所示，中断处理划分为 3 部分。首先，必须建立一个适当的环境，使得处理程序函数能够在其中执行，接下来调用处理程序自身，最后将系统复原（在当前程序看来）到中断之前的状态。调用中断处理程序前后的两部分，分别称为**进入路径**（左边方框）和**退出路径**（右边方框）。

    ![image-20230829211951196](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230829211951196.png)

    进入和退出任务还负责确保处理器从用户态切换到核心态。进入路径的一个关键任务是，从用户态栈切换到核心态栈。但是，只有这一点还不够。因为内核还要使用 CPU 资源执行其代码，**进入路径必须保存用户应用程序当前的寄存器状态**，以便在中断活动结束后恢复。这与调度期间用于上下文切换的机制是相同的。

    > 在从用户态切换到内核态时，已经保存了一部分内核会用到的寄存器的状态。但内核没用到的（比如浮点数寄存器），并不保存。在进入中断处理程序前，需要保存这部分寄存器状态。
    >
    > 中断到达时，处理器可能正处于内核态，因而无需用户态切换到内核态等，总之，进入路径也会进行一系列检查。

2. **中断处理程序**：中断处理程序可能会遇到困难，特别是，在处理程序执行期间，发生了其他中断。尽管可以通过在处理程序执行期间禁用中断来防止，但这会引起其他问题，如遗漏重要的中断。**屏蔽**（Masking，这个术语用于表示选择性地禁用一个或多个中断）因而**只能短时间使用**。

   因此 ISR 必须满足如下两个要求。

   1. 实现（特别是在禁用其他中断时）必须包含尽可能少的代码，以支持快速处理。
   2. 可以在其他 ISR 执行期间调用的中断处理程序例程，不能彼此干扰。

   尽管后一个要求可以通过高超的编程和精巧的 ISR 设计来满足，然而前一个要求更难满足。根据具体的中断，必须运行某个程序，来满足中断处理的最低要求。因而代码长度无法任意缩减。

   内核如何解决这种两难问题呢？并非 ISR 的每个部分都同等重要。通常，每个处理程序例程都可以划分为 3 个部分，具有不同的意义。

   1. **关键操作必须在中断发生后立即执行**。否则，无法维持系统的稳定性，或计算机的正确运作。在执行此类操作期间，**必须禁用其他中断**。
   2. 非关键操作也应该尽快执行，但**允许启用中断**（因而可能被其他系统事件中断）。
   3. **可延期操作不是特别重要**，不必在中断处理程序中实现。内核可以延迟这些操作，在时间充裕时进行。**内核提供了 `tasklet`，用于在稍后执行可延期操作**。

> 1. 这里进一步说明了中断的上半部、下半部。但与前面有点矛盾了，前面说的下半部就是软中断，这里说的 `tasklet`，后者并不等同与软中断。
> 2. 关于 `tasklet`，先简单理解为一种“内核线程”。在硬中断的处理函数中，我们创建一个“线程 `tasklet`”去执行较复杂的处理代码，完成线程创建后，中断处理程序就可以退出，进而恢复原系统的进程处理。而我们创建的 “线程`tasklet`” 则进入内核的调度系统，就像普通的线程那样，由内核调度系统来安排时间片进行处理。（仅仅是这样猜测，具体情况看下面）。

### 多核系统中断处理

硬中断和软中断交给哪一个核进行处理的问题！

todo...

### 中断统计信息

在 `/proc` 目录下，有两个文件分别记录了硬中断和软中断的次数：`/proc/interrupts`、`/proc/softirqs`。在多核系统上，会显示出每一个核处理的对应中断的数量，这里只有单核。

```bash
[root@cvitek]~# cat /proc/softirqs
                    CPU0
          HI:          0
       TIMER:      37049
      NET_TX:       1398
      NET_RX:       2344
       BLOCK:        139
    IRQ_POLL:          0
     TASKLET:         19
       SCHED:          0
     HRTIMER:          0
         RCU:      11288
```

软中断的数量是内核代码中固定的，而硬中断则与外设数量有关：

```bash
[root@cvitek]~# cat /proc/interrupts
           CPU0
  2:          0  T-Head PLIC  76  cvi-tpu-tdma
  5:      61141  RISC-V INTC   5  riscv-timer
 11:          0  T-Head PLIC  29  dw_dmac
 16:         60  T-Head PLIC  44  ttyS0
 21:          0  T-Head PLIC  49  4000000.i2c
 22:          0  T-Head PLIC  51  4020000.i2c
 23:          0  T-Head PLIC  52  4030000.i2c
 24:          0  T-Head PLIC  53  4040000.i2c
 25:       1443  T-Head PLIC  31  eth0
 26:       4296  T-Head PLIC  34  mmc0
 27:          0  T-Head PLIC  36  mmc1
 28:          0  T-Head PLIC  40  4100000.i2s
 29:          0  T-Head PLIC  43  4130000.i2s
 30:          0  T-Head PLIC  26  cif-irq0
 31:          0  T-Head PLIC  27  cif-irq1
 32:          0  T-Head PLIC  24  a000000.vi
 33:          0  T-Head PLIC  25  CVI_VIP_SCL
 34:          3  T-Head PLIC  97  a0a0000.ive
 35:          0  T-Head PLIC  28  CVI_VIP_DWA
 36:          0  T-Head PLIC  22  h265
 37:          0  T-Head PLIC  21  h264
 39:          0  T-Head PLIC  20  JPU_CODEC_IRQ
 40:          1  T-Head PLIC 101  mailbox
 49:          0  gpio-dwapb  13  cd-gpio-irq
```

## 硬中断

> 主要关注：
>
> 1. 怎么为某个外设注册中断，包括中断号的分配，中断处理程序的注册。
> 2. 这个过程就会涉及到中断的实现原理，中断的优先级，如何在文件系统上看到中断的相关统计信息 `cat /proc/interrupts` 这种。

### 数据结构

中断技术上的实现有两方面：

- **汇编语言代码**，与处理器高度相关，用于处理特定平台上相关的底层细节；
- **抽象接口**，是设备驱动程序及其他内核代码安装和管理 `IRQ` 处理程序所需的。

本节主要关注第二方面。

![image-20230830114231632](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230830114231632.png)

#### irq_desc

> 每一个中断都对应一个中断描述符 `irq_desc`。

为响应外部设备的 `IRQ`，内核必须为每个潜在的 `IRQ` 提供一个函数。**该函数必须能够动态注册和注**销。`IRQ` 相关信息管理的关键点是一个**全局数组**，每个数组项对应一个 `IRQ` 编号。因为数组位置和中断号是相同的，很容易定位与特定的 `IRQ` 相关的数组项：`IRQ` 0 在位置 0，`IRQ` 15 在位置 15，等等。不过 `IRQ` 最终映射到哪个处理器中断，在这里是不相关的。

> 如前面所说的 `IRQ` 号与中断号的区别。`/proc/interrupts` 中展示的是 `IRQ` 号而不是中断号。
>
> 顺便提一嘴，内核代码都是采用的 `tab` 缩减，并且一个`1tab = 8space`，所以发现内核代码对齐有问题时，看看是不是你的`tab`设置不对。
>
> 本文参考的是内核版本是 `v5.10.186` 。与《深入 Linux 内核架构》使用的略有区别。

```c
// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/irq/irqdesc.c#L554
struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
        [0 ... NR_IRQS-1] = {
                .handle_irq     = handle_bad_irq,
                .depth          = 1,
                .lock           = __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock),
        }
};
```

1. `struct irq_desc irq_desc[NR_IRQS]`: 这是一个中断描述符数组，用于存储系统中所有可能的中断的描述信息。**`NR_IRQS` 表示中断总数**，在不同的体系结构上，支持的中断数并不相同，32、64、128、512 等等。**每个中断都对应一个 [`struct irq_desc`](https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/irqdesc.h#L55) 的实例**（即中断描述符）。
2. `__cacheline_aligned_in_smp`: 这个标志表明这个数组会被在多核系统中**按缓存行对齐**（一般为 `64 byte`），以提高效率。
3. `[0 ... NR_IRQS-1]`: 这是一种初始化数组的语法，表示对数组中的每个元素进行初始化。
4. `.handle_irq = handle_bad_irq`: 这将每个中断描述符的 `handle_irq` 成员初始化为 `handle_bad_irq`。`handle_irq` 是一个函数指针，指向中断处理函数，但在这里它被初始化为 `handle_bad_irq`，即处理“坏”中断的函数。这通常是一个占位符，表明默认情况下所有中断都将使用相同的处理函数。
5. `.depth = 1`: 这将每个中断描述符的 `depth` 成员初始化为 1。**`depth` 表示中断嵌套的深度**，即在处理中断时，如果发生另一个中断，系统可以跟踪中断的嵌套级别。`depth` 有两个任务。它可用于确定 IRQ 电路是启用的还是禁用的。正值表示禁用，而 0 表示启用。为什么用正值表示禁用的 `IRQ` 呢？因为这使得内核能够区分启用和禁用的 `IRQ` 电路，以及重复禁用同一中断的情形。这个值相当于一个计数器，内核其余部分的代码每次禁用某个中断，则将对应的计数器加 1；每次中断被再次启用，则将计数器减 1。在 `depth` 归 0 时，硬件才能再次使用对应的 `IRQ`。这种方法能够支持对嵌套禁用中断的正确处理。
6. `.lock = __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock)`: 这将每个中断描述符的 `lock` 成员初始化为未锁定的自旋锁。自旋锁是一种同步机制，**用于保护对中断描述符的访问**。`__RAW_SPIN_LOCK_UNLOCKED` 宏用于初始化自旋锁。

下面是 `irq_desc` 的结构（有省略）：

```c
struct irq_desc {
        struct irq_common_data  irq_common_data;
        struct irq_data         irq_data;
        /* struct irq_data {
                u32                     mask;
                unsigned int            irq;   // 哪一个是由设备树指定的呢？
                unsigned long           hwirq; // 这
                struct irq_common_data  *common;
                struct irq_chip         *chip;
                struct irq_domain       *domain;
                void                    *chip_data;
        }; */
        unsigned int __percpu   *kstat_irqs;
        irq_flow_handler_t      handle_irq;
        struct irqaction        *action;        /* IRQ action list */
        unsigned int            status_use_accessors;
        unsigned int            depth;          /* nested irq disables */
        unsigned int            irq_count;      /* For detecting broken IRQs */
        unsigned int            irqs_unhandled;
        raw_spinlock_t          lock;
        int                     parent_irq;
        struct module           *owner;
        const char              *name;
} ____cacheline_internodealigned_in_smp;
```

- `action`：提供了一个操作链，需要在中断发生时执行。由中断通知的设备驱动程序，可以将与之相关的处理程序函数放置在此处。有一个专门的数据结构用于表示这些操作。
- `irq_data->chip`：电流处理和芯片相关操作被封装在 `chip` 中。为此引入了一个专门的数据结构。
- `name`：指定了电流层处理程序的名称，对边沿触发中断，通常是 `edge`，对电平触发中断，通常是 `level`。
- `status_use_accessors`：
  - `IRQ_DISABLED` 用于表示被设备驱动程序禁用的 IRQ 电路。该标志通知内核不要进入处理程序。
  - 在 IRQ 处理程序执行期间，状态设置为 `IRQ_INPROGRESS`。与 `IRQ_DISABLED` 类似，这会阻止其余的内核代码执行该处理程序。
  - 在 CPU 注意到一个中断但尚未执行对应的处理程序时，`IRQ_PENDING` 标志位置位。
  - 为正确处理发生在中断处理期间的中断，需要 `IRQ_MASKED` 标志。具体参见 14.1.4 节。
  - 在某个 IRQ 只能发生在一个 CPU 上时，将设置 `IRQ_PER_CPU` 标志位。（在 SMP 系统中，该标志使几个用于防止并发访问的保护机制变得多余）。
  - `IRQ_LEVEL` 用于 Alpha 和 PowerPC 系统，用于区分电平触发和边沿触发的 IRQ。
  - `IRQ_REPLAY` 意味着该 IRQ 已经禁用，但此前尚有一个未确认的中断。
  - `IRQ_AUTODETECT` 和 `IRQ_WAITING` 用于 IRQ 的自动检测和配置。

#### IRQ 控制器抽象

### 注册和分配 IRQ

1. **注册 IRQ**

    由设备驱动程序动态注册 ISR 的工作，可以使所述的数据结构非常简单地进行。

    ```c
    /**
    * request_irq - Add a handler for an interrupt line
    * @irq:     The interrupt line to allocate
    * @handler: Function to be called when the IRQ occurs.
    *           Primary handler for threaded interrupts
    *           If NULL, the default primary handler is installed
    * @flags:   Handling flags
    * @name:    Name of the device generating this interrupt
    * @dev:     A cookie passed to the handler function
    *
    * This call allocates an interrupt and establishes a handler; see
    * the documentation for request_threaded_irq() for details.
    */
    static inline int __must_check
    request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
            const char *name, void *dev)
    {
            return request_threaded_irq(irq, handler, NULL, flags, name, dev);
    }
    ```

    ![image-20230830155455169](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230830155455169.png)

2. **释放 IRQ**
3. **注册中断**

### 处理 IRQ

#### 1. 切换到核心态：

到核心态的切换，是基于每个中断之后由处理器自动执行的汇编语言代码的。只有那些最为必要的操作直接在汇编语言代码中执行。内核试图尽快地返回到常规的 C 代码，因为 C 代码更容易处理。

在 C 语言中调用函数时，需要将所需的数据（返回地址和参数）按一定的顺序放到栈上。在用户态和核心态之间切换时，还需要将最重要的寄存器保存到栈上，以便以后恢复。这两个操作由平台相关的汇编语言代码执行。在大多数平台上，控制流接下来传递到 `C` 函数 `do_IRQ`，其实现也是平台相关的，但情况仍然得到了很大的简化。 根据平台不同，该函数的参数或者是**处理器寄存器集合**、或是中断号和指向处理器寄存器集合的指针。

![image-20230831095645760](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230831095645760.png)

> 这里理解一下如何保存寄存器值的，就是直接读取寄存器的值，然后存放在 `struct pt_regs` 这个结构体中。并且是存放在栈底的，不过寄存器集合也可以被复制到地址空间中栈以外的其他位置。在这种情况下，`do_IRQ` 的一个参数是指向 `pt_regs` 的指针，但这并没有改变以下事实：寄存器的内存已经被保存，可以由 C 代码读取。不同体系结构下，该结构体 `pt_regs` 的定义有区别。[x86_32](https://elixir.bootlin.com/linux/v5.10.186/source/arch/x86/include/asm/ptrace.h#L12)

#### 2. IRQ 栈

1. 调用电流处理程序例程

    ![image-20230830160703773](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230830160703773.png)

2. 调用高层 ISR

3. 实现处理程序例程

## 软中断

> 本节语境下的软中断不再是延迟执行的总称，而是特指 `softirq`！

### 软中断类型

不同于硬件中断（每个设备使用的外设数量和类型不同）需要支持动态地注册，软件中断的类型是已知的，因此是以静态的形式固化在内核代码中。各个软中断都有一个唯一的编号，这表明软中断是相对稀缺的资源，使用其必须谨慎，不能由各种设备驱动程序和内核组件随意使用。默认情况下，系统上只能使用 `32` 个软中断。虽然看起来很有限，但在目前的内核中，仅仅用到了 10 个软中断🛫。

```c
https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/interrupt.h#L545
enum
{
        HI_SOFTIRQ=0,     // 与 TASKLET_SOFTIRQ 类似，用于需要优先处理的 tasklet
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,    // 与块设备（如硬盘）相关的软中断，用于处理块设备的数据传输和 IO 操作
        IRQ_POLL_SOFTIRQ, // 用于轮询处理中断请求（IRQ），通常用于一些特殊的情况
        TASKLET_SOFTIRQ,  // 用于处理一些需要延迟执行的任务，被称为 tasklet，在软中断上下文中执行
        SCHED_SOFTIRQ,    // 与调度器相关的软中断，用于处理进程和线程的调度操作
        HRTIMER_SOFTIRQ,  // 用于高精度定时器的处理，这些定时器用于实现高分辨率的时间间隔
        RCU_SOFTIRQ,    /* Preferable RCU(Read-Copy-Update) should always be the last softirq */

        NR_SOFTIRQS
};

// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L645
const char * const softirq_to_name[NR_SOFTIRQS] = {
        "HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "IRQ_POLL",
        "TASKLET", "SCHED", "HRTIMER", "RCU"
};
```

### 数据结构

内核借助于软中断来获知异常情况的发生，而该情况将在稍后由专门的处理程序例程解决。如上所述，内核在 `do_IRQ` 末尾处理所有待决软中断，因而可以确保软中断能够定期得到处理。

软中断机制的核心部分是一个表，包含 `NR_SOFTIRQS` 个 `softirq_action` 类型的数据项。该数据类型结构非常简单，只包含一个指向处理程序例程的指针，在软中断发生时由内核执行该处理程序例程。

> 软中断是已知的，这些中断的处理程序也是内核中已经实现好了的。搜索 [`open_softirq`](https://elixir.bootlin.com/linux/v5.10.186/C/ident/open_softirq) 就可以看到 `timer`、`RCU` 等在各自的 `.c` 文件中**静态注册**了对应中断的处理程序。

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

// https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/interrupt.h#L559
struct softirq_action
{
        void    (*action)(struct softirq_action *);
};

void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}

// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L636
open_softirq(TASKLET_SOFTIRQ, tasklet_action);
open_softirq(HI_SOFTIRQ, tasklet_hi_action);
```

> 👀这个 `action` 函数指针，它的传入参数也是 `softirq_action`👻（从后面 `tasklet_action` 的例子来看根本没用到这个参数👀）

### 触发软中断

> 触发软中断仅仅是发起了中断请求，会处理，但不一定会马上处理，而是要等调度器安排。

不同于硬件中断直接由外设发起请求（经过中断控制器发起），软中断则是通过调用 `raise_softirq(int nr)` 来引发一个软中断（类似普通中断）。软中断的编号通过参数指定。

```c
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags); // 禁用本地 CPU 的硬件中断，同时将当前中断状态保存在 flags 变量中
        raise_softirq_irqoff(nr); // 触发指定的软中断，仅仅是标志为需要处理，需要等待调度系统安排 CPU cycle
        local_irq_restore(flags); // 在软中断触发完毕后，恢复先前保存的中断状态，即将中断重新打开
}
```

> 触发软中断仅仅是设置对应中断的标志位，并不会像硬件中断那样立刻执行！需要等待调度系统安排 CPU cycle。
> 所以这里虽然禁用了本地 CPU 硬件中断，但设置标志位之后就立马恢复，禁用的时间很短。
>
> 另外，这里禁用的是硬件中断！软件中断的执行是通过内核调度机制来触发和调度的，通常是为了在不阻塞其他操作的情况下执行一些延迟敏感的任务，因此在大多数情况下，它们不需要被显式禁用。

该函数设置各 CPU 变量 `irq_stat[smp_processor_id].__softirq_pending` 中的对应比特位。该函数将相应的软中断标记为执行，但这个执行是延期执行。通过使用特定于处理器的位图，内核确保几个软中断（甚至是相同的）可以同时在不同的 CPU 上执行。

> 这个变量好像和前面的没关系啊，有点奇怪。
>
> todo...

### 优先级

**软中断的编号形成了一个优先顺序**，这并不影响各个处理程序例程执行的频率或它们相当于其他系统活动的优先级，但定义了多个软中断同时活动或待决时处理例程执行的次序。

> 不影响频率是因为在一次 `do_IRQ` 执行中，每个软中断都会被遍历、执行，只是按照“优先级”（顺序）执行而已。
>
> 看看这个函数的代码  [`__do_softirq`](https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L282) ，遍历 `softirq_vec` 的 `actions`。

### 开启软中断处理

有几种方法可开启软中断处理，但这些都归结为调用 `do_softirq` 函数。

```c
// 表示这是一个可见的、用于在汇编语言中调用的函数，用于执行软中断的处理
asmlinkage __visible void do_softirq(void)
{
        __u32 pending;
        unsigned long flags;

        // 检查当前代码是否在一个中断上下文中执行，即不涉及硬件中断，因为软中断用于执行 ISR 中非时间关键部分，
        // 所以其代码本身一定不能在中断处理程序内调用。
        if (in_interrupt())
                return;

        local_irq_save(flags); // 禁用本地 CPU 的硬件中断，以确保在执行软中断处理时不会被其他硬件中断打断

        // 获取当前待处理的软中断标志，即检查哪些软中断需要被执行.
        // 该函数还将原来的位图重置为 0。换句话说，清除所有软中断。
        pending = local_softirq_pending();

        // 首先检查是否有待处理的软中断，并且还会检查是否有专门的内核线程（如 ksoftirqd）
        // 正在运行来处理这些软中断。如果条件满足，就调用
        if (pending && !ksoftirqd_running(pending))
                // 执行软中断的实际处理。这个函数会将软中断放在当前线程的栈上进行处理，
                // 以避免与其他上下文之间的干扰。这个函数会调用 __do_softirq();
                do_softirq_own_stack();

        local_irq_restore(flags);
}
```

![image-20230831101758366](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230831101758366.png)

**`softirq_vec` 中的 `action` 函数在一个 `while` 循环中针对各个待决的软中断被调用**。也就是按照顺序（优先级）遍历所有需要处理的中断，依次处理。

> `__do_softirq` 这里会调用软中断的处理函数，一般来说耗时是比较长的，不理解调用该函数时为什么禁用硬件中断。
>
> 这里需要进入 [`__do_softirq`](https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L282) 内部查看，在正式执行处理函数之前，硬件中断实际上会被重新 `enable`。
>
> ~~不太清楚，不过总之事实就是这样，这也是软中断的一个“缺陷”，基于软中断的 `tasklet` 执行期间同样禁用了硬件中断。因此他们的处理函数执行时间仍然需要尽量短，以保持系统的响应性能。如果需要执行耗时较长的操作，可以考虑使用**工作队列**等其他机制，将这些操作从 `Tasklet` 中移出，以避免阻塞软中断的执行。~~（`chatgpt` 🤮）

在处理了所有标记出的软中断之后，内核检查在此期间是否有新的软中断标记到位图中。要求在前一轮循环中至少有一个没有处理的软中断，而重启的次数没有超过 `MAX_SOFTIRQ_RESTART`（通常设置为 10）。如果是这样，则再次按序处理标记的软中断。这操作会一直重复下去，直至在执行所有处理程序之后没有新的未处理软中断为止。

如果在 `MAX_SOFTIRQ_RESTART` 次重启处理过程之后，仍然有未处理的软中断，那么应该如何？内核将调用 `wakeup_softirqd` 唤醒软中断守护进程。

> 上面描述的是在 [`__do_softirq(void)`](https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L255) 中，重启不仅有次数限制，还有时间限制。

### 软中断守护进程

软中断守护进程的任务是，与其余内核代码异步执行软中断。为此，系统中的每个处理器都分配了自身的守护进程，名为 `ksoftirqd`。

内核中有两处调用 `wakeup_softirqd` 唤醒了该守护进程：

- 在 `do_softirq` 中
- 由 `raise_softirq` 在内部调用

唤醒函数本身只需要几行代码。首先，借助于一些宏，从一个各 `CPU` 变量读取指向当前 `CPU` 软中断守护进程的 `task_struct` 的指针。如果该进程当前的状态不是 `TASK_RUNNING`，则通过 `wake_up_process` 将其放置到**就绪进程的列表末尾**（参见第 2 章）。尽管这并不会立即开始处理所有待决软中断，但只要调度器没有更好的选择，就会选择该守护进程（优先级为 `19`）来执行。

> 软中断的守护进程在内核中，与其余线程一起通过调度器来分配 CPU cycle，不过由于 `ksoftirqd` 的优先级较低，emmm，软中断的优先级这么低？？为什么？那如果系统比较繁忙的时候，这些中断岂不是很难得到处理？（虽然也和调度算法有关，linux 默认的调度算法也始终能保证线程不会饿死，是相对公平的算法。）
>
> 这也是软中断的特点吧，延迟执行，跟硬件中断立刻执行比不了的。

在系统启动时用 `initcall` 机制（见附录 D）调用 `init` 不久，即创建了系统中的软中断守护进程。在初始化之后，各个守护进程都执行以下无限循环：

![image-20230830162923014](../images/Linux%E5%86%85%E6%A0%B8-%E4%B8%AD%E6%96%AD/image-20230830162923014.png)

> 但新版内核似乎已经换了一种实现方式了。

```c
static void run_ksoftirqd(unsigned int cpu)
{
        local_irq_disable();
        if (local_softirq_pending()) {
                /*
                 * We can safely run softirq on inline stack, as we are not deep
                 * in the task stack here.
                 */
                __do_softirq(); // 调用 __do_softirq() 处理
                local_irq_enable();
                cond_resched();
                return;
        }
        local_irq_enable();
}

// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L734
static struct smp_hotplug_thread softirq_threads = {
        .store                  = &ksoftirqd,
        .thread_should_run      = ksoftirqd_should_run,
        .thread_fn              = run_ksoftirqd,
        .thread_comm            = "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
        cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
                                  takeover_tasklets);
        BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

        return 0;
}
early_initcall(spawn_ksoftirqd);
```

每一个 `core` 上都会启动一个 `ksoftirqd` 守护进程。

```c
song.yu@WORKSTATION-PAD3:~$ ps aux | grep softirq
root           9  0.0  0.0      0     0 ?        S    4 月 17  16:58 [ksoftirqd/0]
root          18  0.0  0.0      0     0 ?        S    4 月 17  14:43 [ksoftirqd/1]
root          24  0.0  0.0      0     0 ?        S    4 月 17  13:40 [ksoftirqd/2]
root          30  0.0  0.0      0     0 ?        S    4 月 17  10:04 [ksoftirqd/3]
root          36  0.0  0.0      0     0 ?        S    4 月 17   9:21 [ksoftirqd/4]
root          42  0.0  0.0      0     0 ?        S    4 月 17   9:29 [ksoftirqd/5]
root          48  0.0  0.0      0     0 ?        S    4 月 17  11:24 [ksoftirqd/6]
root          54  0.0  0.0      0     0 ?        S    4 月 17   9:39 [ksoftirqd/7]
root          60  0.0  0.0      0     0 ?        S    4 月 17   9:55 [ksoftirqd/8]
root          66  0.0  0.0      0     0 ?        S    4 月 17   9:12 [ksoftirqd/9]
root          72  0.0  0.0      0     0 ?        S    4 月 17   8:51 [ksoftirqd/10]
root          78  0.0  0.0      0     0 ?        S    4 月 17   9:24 [ksoftirqd/11]
root          84  0.0  0.0      0     0 ?        S    4 月 17   7:59 [ksoftirqd/12]
root          90  0.0  0.0      0     0 ?        S    4 月 17   8:01 [ksoftirqd/13]
root          96  0.0  0.0      0     0 ?        S    4 月 17   8:08 [ksoftirqd/14]
root         102  0.0  0.0      0     0 ?        S    4 月 17   8:30 [ksoftirqd/15]
root         108  0.0  0.0      0     0 ?        S    4 月 17   7:49 [ksoftirqd/16]
root         114  0.0  0.0      0     0 ?        S    4 月 17   8:00 [ksoftirqd/17]
root         120  0.0  0.0      0     0 ?        S    4 月 17   9:33 [ksoftirqd/18]
root         126  0.0  0.0      0     0 ?        S    4 月 17   8:20 [ksoftirqd/19]
root         132  0.0  0.0      0     0 ?        S    4 月 17   8:05 [ksoftirqd/20]
root         138  0.0  0.0      0     0 ?        S    4 月 17   8:28 [ksoftirqd/21]
root         144  0.0  0.0      0     0 ?        S    4 月 17   8:16 [ksoftirqd/22]
root         150  0.0  0.0      0     0 ?        S    4 月 17   8:27 [ksoftirqd/23]
```

### 小结

- 软中断类型是有限的，就 10 来种，软中断的处理函数也是写死了的，不支持动态注册。
- 软中断是通过 `do_softirq` 这个函数调用开始执行的，该函数一般由 `ksoftirqd` 守护进程来调用执行（好像其他位置也可以调用，不过不清楚例子），该进程的优先级很低 `19`，所以**软中断一般不能及时处理**。
- 软中断可以并发运行在多个 CPU 上（即使同一类型的也可以），所以软中断的**处理函数必须设计为可重入的函数**。
- 软中断处理函数执行过程中，硬件中断没有被禁用！

> ✨✨✨✨
>
> 关于中断的时序：
>
> - 硬中断会打断硬中断（当然是不同类型的，以及没有被禁用）；
> - 硬中断会打断软中断；
> - 软中断不会打断硬中断，软中断也不会打断软中断。

## tasklet

软中断是将操作推迟到未来时刻执行的最有效的方法。但该延期机制处理起来非常复杂。因为多个处理器可以同时且独立地处理软中断，**同一个软中断的处理程序例程可以在几个 `CPU` 上同时运行**。对软中断的效率来说，这是一个关键，多处理器系统上的网络实现显然受惠于此。但**处理程序例程的设计必须是完全可重入且线程安全的**。另外，临界区必须用自旋锁保护（或其他 IPC 机制，参见第 5 章），而这需要大量审慎的考虑。

`tasklet` 和工作队列是延期执行工作的机制，其实现基于软中断，但它们更易于使用，因而更适合于设备驱动程序（以及其他一般性的内核代码）。

> 有点类似与线程池的概念✨，线程池的大小就是 `ksoftirqd` 的数量，任务就是 `tasklet` 或者 **工作队列**，单个任务 `tasklet` 可能在任意一个 `ksoftirqd` 上运行，但是一个 `tasklet` 只能在一个 `ksoftirqd` 上运行。`tasklet` 也会用一个单向链表 `tasklet_vec` 来管理，~~每次 `ksoftirqd` 取任务时都需要对该队列进行加锁保护。~~

### 数据结构

#### tasklet_struct

```c
// https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/interrupt.h#L613
/* Tasklets --- multithreaded analogue of BHs.
   Properties:
   * If tasklet_schedule() is called, then tasklet is guaranteed
     to be executed on some cpu at least once after this.
   * If the tasklet is already scheduled, but its execution is still not
     started, it will be executed only once.
   * If this tasklet is already running on another CPU (or schedule is called
     from tasklet itself), it is rescheduled for later.
   * Tasklet is strictly serialized wrt itself, but not
     wrt another tasklets. If client needs some intertask synchronization,
     he makes it with spinlocks.
 */
struct tasklet_struct
{
        struct tasklet_struct *next;
        unsigned long state;
        atomic_t count;
        bool use_callback;
        union {
                void (*func)(unsigned long data);
                void (*callback)(struct tasklet_struct *t);
        };
        unsigned long data;
};
```

- `next` 是一个指针，用于建立 `tasklet_struct` 实例的链表。这容许几个任务排队执行。

- `func` 指向一个函数的地址，该函数的执行将被延期。`data` 用作该函数执行时的参数。

    >  `data` 是 `unsigned long` 类型的，所以注意它可能是作为指针（值为内存地址）来用，而不是单纯的数字。内核中经常这样用。

- `state` 表示任务的当前状态，类似于真正的进程。但只有两个选项，分别由 `state` 中的一个比特位表示，这也是二者可以独立设置/清除的原因。
    - 在 `tasklet` 注册到内核，等待调度执行时，将设置 `TASKLET_STATE_SCHED`。
    - `TASKLET_STATE_RUN` 表示 `tasklet` 当前正在执行。第二个状态只在 `SMP` 系统上有用。用于保护 `tasklet` 在多个处理器上并行执行。

- 原子计数器 `count` 用于禁用已经调度的 `tasklet`。如果其值不等于 0，在接下来执行所有待决的 `tasklet` 时，将忽略对应的 `tasklet`。

这个 `satat` 用于对 `tasklet` 加锁。

```c
// https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/interrupt.h#L678
static inline int tasklet_trylock(struct tasklet_struct *t)
{
        return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}

static inline void tasklet_unlock(struct tasklet_struct *t)
{
        smp_mb__before_atomic();
        clear_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

### 注册 tasklet

`tasklet_schedule` 将一个 `tasklet` 注册到系统中。更准确地来说，是注册到调用 `tasklet_schedule` 这个函数的 `core` 中。每个核会维护一个链表 `tasklet_vec`，它的每个元素都是一个 `tasklet_struct` 实例，表示要执行的任务 `tasklet`。

`tasklet_schedule` 用于将指定的 `tasklet` 添加到链表 `tasklet_vec` 末尾。

```c
// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L502
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);

// https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/interrupt.h#L685
static inline void tasklet_schedule(struct tasklet_struct *t)
{
        if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
                __tasklet_schedule(t);
}

// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L521
void __tasklet_schedule(struct tasklet_struct *t)
{
        __tasklet_schedule_common(t, &tasklet_vec,
                                  TASKLET_SOFTIRQ);
}

static void __tasklet_schedule_common(struct tasklet_struct *t,
                                      struct tasklet_head __percpu *headp,
                                      unsigned int softirq_nr)
{
        struct tasklet_head *head;
        unsigned long flags;

        local_irq_save(flags);
        head = this_cpu_ptr(headp);
        t->next = NULL;
        *head->tail = t;
        head->tail = &(t->next);
        raise_softirq_irqoff(softirq_nr); // 引起一次 TASKLET_SOFTIRQ 软中断，以便执行 tasklet
        local_irq_restore(flags);
}
```

> 关于 `tasklet_vec` 变量，首先它是一个 `per_cpu` 变量，每个核都会维护自己的链表。其次，每个核上的任务 `tasklet` 只能由它自己处理，其他核不会帮忙处理，因此可能就会涉及到如何合理的分配 `tasklet`。
>
> 此外，`tasklet` 执行完毕后是否会从 `tasklet_vec` 上移除？

### 执行 tasklet

`tasklet` 的生命周期中最重要的部分就是其执行。因为 `tasklet` 基于软中断实现，它们总是在处理软中断时执行。

`tasklet` 关联到 `TASKLET_SOFTIRQ` 软中断。因而，调用 `raise_softirq(TASKLET_SOFTIRQ)`，即可在下一个适当的时机执行当前处理器的 `tasklet`。内核使用 `tasklet_action` 作为该软中断的 `action` 函数。

> 这个函数会取 `tasklet_vec` 链表中的 `tasklet` 依次执行

```c
// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L636
open_softirq(TASKLET_SOFTIRQ, tasklet_action);
open_softirq(HI_SOFTIRQ, tasklet_hi_action);

// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L576
static __latent_entropy void tasklet_action(struct softirq_action *a)
{
        tasklet_action_common(a, this_cpu_ptr(&tasklet_vec), TASKLET_SOFTIRQ);
}

static void tasklet_action_common(struct softirq_action *a,
                                  struct tasklet_head *tl_head,
                                  unsigned int softirq_nr)
{
        struct tasklet_struct *list;

        // 这里清空了 tasklet_vec 链表的，用临时变量 list 来访问 ✨
        // 因为预期是会将所有 tasklet 执行的，如果出现了意外情况，再添加回来
        local_irq_disable();
        list = tl_head->head;
        tl_head->head = NULL;
        tl_head->tail = &tl_head->head;
        local_irq_enable();

        while (list) {
                // 遍历单向链表，依次执行
                struct tasklet_struct *t = list;

                list = list->next;

                // 这个加锁、解锁，就是前面提到的 state 标志位的处理
                // https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/interrupt.h#L678
                if (tasklet_trylock(t)) {
                        // 只有这个原子计数为 0 时才会继续执行该 tasklet
                        if (!atomic_read(&t->count)) {
                                if (!test_and_clear_bit(TASKLET_STATE_SCHED,
                                                        &t->state))
                                        BUG();
                                // 调用 tasklet 对应的函数指针
                                if (t->use_callback)
                                        t->callback(t);
                                else
                                        t->func(t->data);
                                tasklet_unlock(t);
                                // 处理完就继续处理下一个 tasklet
                                continue;
                        }
                        tasklet_unlock(t);
                }

                // 只有当 tasklet 已经在其他核处理了时，加锁失败才会到这里。
                local_irq_disable();
                t->next = NULL;
                *tl_head->tail = t;
                tl_head->tail = &t->next;
                __raise_softirq_irqoff(softirq_nr);
                local_irq_enable();
        }
}
```

**因为一个 `tasklet` 只能在一个处理器上执行一次，但其他的 `tasklet` 可以并行运行，所以需要特定于 `tasklet` 的锁**。 `state` 状态用作锁变量。在执行一个 `tasklet` 的处理程序函数之前，内核使用 `tasklet_trylock` 检查 `tasklet` 的状态是否为 `TASKLET_STATE_RUN`。换句话说，它是否已经在系统的另一个处理器上运行

> 🧠🧠这里 `tasklet` 用完之后，没有销毁 `free` 诶？不怕内存泄漏吗？
>
> `tasklet` 不是每一个核执行它自己的吗？为什么会有一个 `tasklet` 可能在其他核上面执行？

### HI_SOFTIRQ

除了普通的 `tasklet` 之外，内核还使用了另一种 `tasklet`，它具有“较高”的优先级。除以下修改之外，其实现与普通的 `tasklet` 完全相同。

1. 使用 `HI_SOFTIRQ` 作为软中断，而不是 `TASKLET_SOFTIRQ`，相关的 `action` 函数是 `tasklet_hi_action`。

2. 注册的 `tasklet` 在 CPU 相关的变量 `tasklet_hi_vec` 中排队。这是使用 `tasklet_hi_schedule` 完成的。

在这里，“较高优先级”是指该软中断的处理程序 `HI_SOFTIRQ` 在所有其他处理程序之前执行，**尤其是在构成了软中断活动主体的网络处理程序之前执行**。

> 所有软中断其实本质上都是由 `ksoftirqd` 决定的较低优先级，只不过不同类型的软中断之间，优先级略有区别，按顺序执行而已。这里说的较高优先级也仅仅是这个意思。毕竟 `HI_SOFTIRQ` 是所有软中断中优先级最高的了。🛫🛫而且软中断的主要开销就是处理网络收发包，只要优先级比网络收发包高，就能够很快地执行了。

当前，大部分声卡驱动程序都利用了这一选项，因为操作延迟时间太长可能损害音频输出的音质。**而用于高速传输的网卡也可以得益于该机制**。

> 😮😮丧心病狂了吧，网络收发包的优先级本来就仅次于 `HI`、`TIMER`，这都还要来抢 `HI` ？

简单看看代码，只能和上面说一模一样✨✨就优先处理了而已。

```c
// https://elixir.bootlin.com/linux/v5.10.186/source/kernel/softirq.c#L581
static __latent_entropy void tasklet_hi_action(struct softirq_action *a)
{
        tasklet_action_common(a, this_cpu_ptr(&tasklet_hi_vec), HI_SOFTIRQ);
}
```

### 小结

- 一种特定类型的 `tasklet` 只能运行在一个 `CPU` 上，不能并行，只能串行执行。
- 多个不同类型的 `tasklet` 可以并行在多个 `CPU` 上。
- 软中断是静态分配的，在内核编译好之后，就不能改变。但 `tasklet` 就灵活许多，可以在运行时改变（比如添加模块时）。
- 软中断中不允许睡眠，而 `tasklet` 同样是在软中断上下文中执行的，因此同样不允许睡眠。而下文所提到的**工作队列**是允许睡眠的。

## 完成量

完成量与信号量有些相似，但是基于等待队列实现的。

## 工作队列

> 工作队列其实和 `ksoftirqd + tasklet` 类似，不过它不是工作在软中断的上下文中，而是普通的内核线程。此外进程的数量也不在局限于 `core` 的数量，几乎可以任意创建。进程名以 `kworker` 为前缀！
>
> 其实还是和线程池的原理相似的。

工作队列是另外一个处理延后函数的概念，它大体上和 `tasklets` 类似。工作队列运行于内核进程上下文，而 `tasklets` 运行于软中断上下文。这意味着工作队列函数不必像 `tasklets` 一样必须是原子性的。`Tasklets` 总是运行于它提交自的那个处理器，工作队列在默认情况下也是这样。工作队列在 `Linux` 内核代码 [kernel/workqueue.c](https://elixir.bootlin.com/linux/v5.10.186/source/kernel/workqueue.c#L200) 中由如下的数据结构表示：

```c
struct pool_workqueue {
        struct worker_pool      *pool;          /* I: the associated pool */
        struct workqueue_struct *wq;            /* I: the owning workqueue */
        int                     work_color;     /* L: current color */
        int                     flush_color;    /* L: flushing color */
        int                     refcnt;         /* L: reference count */
        int                     nr_in_flight[WORK_NR_COLORS];
                                                /* L: nr of in_flight works */
        int                     nr_active;      /* L: nr of active works */
        int                     max_active;     /* L: max active works */
        struct list_head        inactive_works; /* L: inactive works */
        struct list_head        pwqs_node;      /* WR: node on wq->pwqs */
        struct list_head        mayday_node;    /* MD: node on wq->maydays */

        /*
         * Release of unbound pwq is punted to system_wq.  See put_pwq()
         * and pwq_unbound_release_workfn() for details.  pool_workqueue
         * itself is also RCU protected so that the first pwq can be
         * determined without grabbing wq->mutex.
         */
        struct work_struct      unbound_release_work;
        struct rcu_head         rcu;
} __aligned(1 << WORK_STRUCT_FLAG_BITS);
```

工作队列最基础的用法，是作为创建内核线程的接口来处理提交到队列里的工作任务。所有这些内核线程称之为 `worker thread`。工作队列内的任务是由代码 [include/linux/workqueue.h](https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/workqueue.h#L102) 中定义的 `work_struct` 表示的，其定义如下：

```c
struct work_struct {
        atomic_long_t data;
        struct list_head entry;
        work_func_t func;
#ifdef CONFIG_LOCKDEP
        struct lockdep_map lockdep_map;
#endif
};
```

这里有两个字段比较有意思：`func` --将被工作队列调度执行的函数，`data` --这个函数的参数。

> 为什么内核使用 `atomic_long_t` 作为指向任意数据的指针的数据类型，而不是通常的 `void *`？
>
> 这里内核使用了一点小技巧，显然有点近乎于“肮脏”，以便将更多信息放入该结构，而又不付出更多代价。因为指针在所有支持的体系结构上都对齐到 4 字节边界，而前两个比特位保证为 0。因而可以“滥用”这两个比特位，将其用作标志位。剩余的比特位照旧保存指针的信息。以下的宏用于屏蔽标志位：
>
> ```c
> // https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/workqueue.h#L90
> WORK_STRUCT_FLAG_MASK = (1UL << WORK_STRUCT_FLAG_BITS) - 1,
> WORK_STRUCT_WQ_DATA_MASK = ~WORK_STRUCT_FLAG_MASK,
> ```
>
> 当前只定义了一个标志：`WORK_STRUCT_PENDING` 用来查找当前是否有待决（该标志位置位）的可延迟工作项。辅助宏 `work_pending(work)`用来检查该标志位。请注意，将 `data` 设置为原子数据类型，确保对该比特位的修改不会带来并发问题。

### 小结



## 参考资料

- ⭐⭐🔥🔥《深入 Linux 内核架构》第 14 章，书讲得更好一些，下面的可以参考一下。
- ⭐⭐🔥🔥[《Linux 内核揭秘》](https://github.com/MintCN/linux-insides-zh/blob/master/Interrupts/README.md) 这本书讲得也很好，而且更贴近代码讲解。
- [什么是软中断](https://www.xiaolincoding.com/os/1_hardware/soft_interrupt.html#%E4%B8%AD%E6%96%AD%E6%98%AF%E4%BB%80%E4%B9%88)
- [Linux 内核中的软中断、tasklet 和工作队列详解](https://zhuanlan.zhihu.com/p/265705850)
- [理解内核的硬软中断（详细讲解~）](https://www.toutiao.com/article/7153606983776731651/?wid=1693292862137)


# 第8章 异常控制流

*异常控制流（Exceptional Control Flow ，ECF）*

作为程序员，理解ECF很重要，这有很多原因：

- **理解ECF将帮助你理解重要的系统概念。**

> ECF是操作系统用来实现I/O、进程和虚拟内存的基本机制。在能够真正理解这些重要概念之前，你必须理解ECF。

- **理解ECF将帮助你理解应用程序是如何与操作系统交互的。**

> 应用程序通过使用一个叫做陷阱（trap）或者系统调用（systemcall）的ECF形式，向操作系统请求服务。比如，向磁盘写数据、从网络读取数据、创建一个新进程，以及终止当前进程，都是通过应用程序调用系统调用来实现的。理解基本的系统调用机制将帮助你理解这些服务是如何提供给应用的。

- 理解下将帮助你编写有趣的新应用程序
- 理解ECF将帮助你理解并发

> ECF是计算机系统中实现并发的基本机制。在运行中的并发的例子有：中断应用程序执行的异常处理程序，在时间上重叠执行的进程和线程，以及中断应用程序执行的信号处理程序。理解ECF是理解并发的第一步。我们会在第12章中更详细地研究并发。

- 理解ECF将帮助你理解软件异常如何工作

> 像C++和Java这样的语言通过try、 catch以及throw语句来提供软件异常机制。软件异常允许程序进行非本地跳转（即违反通常的调用/返回栈规则的跳转）来响应错误情况。非本地跳转是一种应用层ECF，在C中是通过setjmp和longjmp函数提供的。理解这些低级函数将帮助你理解高级软件异常如得以实现

## 8.1 异常

异常（exception）就是控制流中的突变，用来响应处理器状态中的某些变化。

<img src="https://img2018.cnblogs.com/blog/1539443/201905/1539443-20190505120122945-1667087205.png" alt="img" style="zoom:67%;" />

在任何情况下，当处理器检测到有事件发生时，它就会通过一张叫做**异常表**（exception table）的跳转表，进行一个间接过程调用（异常），到一个专门设计用来处理这类事件的操作系统子程序（**异常处理程序**（exception handler））.当异常处理程序完成处理后，根据引起异常的事件的类型，会发生以下 3 种情况中的一种：

1. 处理程序将控制返回给当前指令 Icurr ，即当事件发生时正在执行的指令。
2. 处理程序将控制返回给 Inext ，如果没有发生异常将会执行的下一条指令。
3. 处理程序终止被中断的程序。



### 8.1.1 异常处理

系统中可能的每种类型的异常都分配了一个唯一的非负整数的**异常号**（exception number）。

在系统启动时（当计算机重启或者加电时），操作系统分配和初始化一张称为**异常表**的跳转表，使得表目 k 包含异常 k 的处理程序的地址。图 8-2 展示了异常表的格式。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-02%20%E5%BC%82%E5%B8%B8%E8%A1%A8.png" alt="图 8-2 异常表。异常表是一张跳转表，其中表目 k 包含异常 k 的处理程序代码的地址" style="zoom:67%;" />

异常号是到异常表中的索引，异常表的起始地址放在一个叫做**异常表基址寄存器**（exception table base register）的特殊 CPU 寄存器里。在运行时（当系统在执行某个程序时），处理器检测到发生了一个事件，并且确定了相应的异常号 k。随后，处理器触发异常，方法是执行间接过程调用，通过异常表的表目 k，转到相应的处理程序。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-03%20%E7%94%9F%E6%88%90%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86%E7%A8%8B%E5%BA%8F%E7%9A%84%E5%9C%B0%E5%9D%80.png" alt="图 8-3 生成异常处理程序的地址。异常号是到异常表中的索引" style="zoom:67%;" />

### 8.1.2 异常的类别

异常可以分为四类：**中断**（interrupt），**陷阱**（trap）、**故障**（fault）和**终止**（abort）。图 8-4 中的表对这些类别的属性做了小结。

| 类别 | 原因                | 异步/同步 | 返回行为             |
| ---- | ------------------- | --------- | -------------------- |
| 中断 | 来自 I/O 设备的信号 | 异步      | 总是返回到下一条指令 |
| 陷阱 | 有意的异常          | 同步      | 总是返回到下一条指令 |
| 故障 | 潜在可恢复的错误    | 同步      | 可能返回到当前指令   |
| 终止 | 不可恢复的错误      | 同步      | 不会返回             |



#### 1. 中断

**中断**是异步发生的，是来自处理器外部的 I/O 设备的信号的结果。

硬件中断不是由任何一条专门的指令造成的，从这个意义上来说它是异步的。硬件中断的异常处理程序常常称为**中断处理程序**（interrupt handler）。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0805-zhong-duan-chu-li-.png" alt="图 8-5 中断处理。中断处理程序将控制返回给应用程序控制流中的下一条指令" style="zoom:50%;" />

> 在当前指令完成执行之后，处理器注意到中断引脚的电压变高了，就从系统总线读取异常号，然后调用适当的中断处理程序。当处理程序返回时，它就将控制返回给下一条指令（也即如果没有发生中断，在控制流中会在当前指令之后的那条指令）。结果是程序继续执行，就好像没有发生过中断一样。

剩下的异常类型（陷阱、故障和终止）是同步发生的，是执行当前指令的结果。我们把这类指令叫做**故障指令**（faulting instruction）。



#### 2. 陷阱和系统调用

陷阱是有意的异常，是执行一条指令的结果。

就像中断处理程序一样，陷阱处理程序将控制返回到下一条指令。陷阱最重要的用途是在用户程序和内核之间提供一个像过程一样的接口，叫做**系统调用**。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-06%20%E9%99%B7%E9%98%B1%E5%A4%84%E7%90%86.png" alt="图 8-6 陷阱处理。陷阱处理程序将控制返回给应用程序控制流中的下一条指令" style="zoom:50%;" />

#### 3. 故障

故障由错误情况引起，它可能能够被故障处理程序修正。

当故障发生时，处理器将控制转移给故障处理程序。如果处理程序能够修正这个错误情况，它就将控制返回到引起故障的指令，从而重新执行它。否则，处理程序返回到内核中的 abort 例程，abort 例程会终止引起故障的应用程序。图 8-7 概述了一个故障的处理。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0807-gu-zhang-chu-li-.png" alt="图 8-7 故障处理。根据故障是否能够被修复，故障处理程序要么重新执行引起故障的指令，要么终止" style="zoom:50%;" />

> 一个经典的故障示例是缺页异常，当指令引用一个虚拟地址，而与该地址相对应的物理页面不在内存中，因此必须从磁盘中取出时，就会发生故障。就像我们将在第 9 章中看到的那样，一个页面就是虚拟内存的一个连续的块（典型的是 4KB）。缺页处理程序从磁盘加载适当的页面，然后将控制返回给引起故障的指令。当指令再次执行时，相应的物理页面已经驻留在内存中了，指令就可以没有故障地运行完成了。

#### 4. 终止

终止是不可恢复的致命错误造成的结果，通常是一些硬件错误，比如 DRAM 或者 SRAM 位被损坏时发生的奇偶错误。

终止处理程序从不将控制返回给应用程序。如图 8-8 所示，处理程序将控制返回给一个 abort 例程，该例程会终止这个应用程序。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-08%20%E7%BB%88%E6%AD%A2%E5%A4%84%E7%90%86.png" alt="图 8-8 终止处理。终止处理程序将控制传递给一个内核 abort 例程，该例程会终止这个应用程序" style="zoom:50%;" />

### 8.1.3 Linux/x86-64 系统中的异常

| 异常号   | 描述               | 异常类别   |
| -------- | ------------------ | ---------- |
| 0        | 除法错误           | 故障       |
| 13       | 一般保护故障       | 故障       |
| 14       | 缺页               | 故障       |
| 18       | 机器检查           | 终止       |
| 32 ~ 255 | 操作系统定义的异常 | 中断或陷阱 |

> 有高达 256 种不同的异常类型【50】。0 ∼ 31 的号码对应的是由 Intel 架构师定义的异常，因此对任何 x86-64 系统都是一样的。32 ∼ 255 的号码对应的是操作系统定义的中断和陷阱。

#### 1. Linux/x86-64 故障和终止

**一般保护故障。**许多原因都会导致不为人知的一般保护故障（异常 13），通常是因为一个程序引用了一个未定义的虚拟内存区域，或者因为程序试图写一个只读的文本段。Linux 不会尝试恢复这类故障。Linux shell 通常会把这种一般保护故障报告为“段故障（Segmentation fault）”

**缺页**（异常 14）是会重新执行产生故障的指令的一个异常示例。处理程序将适当的磁盘上虚拟内存的一个页面映射到物理内存的一个页面，然后重新执行这条产生故障的指令。我们将在第 9 章中看到缺页是如何工作的细节。

**机器检查。**机器检查（异常 18）是在导致故障的指令执行中检测到致命的硬件错误时发生的。机器检查处理程序从不返回控制给应用程序。

#### 2. Linux/86-64 系统调用

| 编号 | 名字  | 描述               | 编号 | 名字   | 描述                 |
| ---- | ----- | ------------------ | ---- | ------ | -------------------- |
| 0    | read  | 读文件             | 33   | pause  | 挂起进程直到信号到达 |
| 1    | write | 写文件             | 37   | alarm  | 调度告警信号的传送   |
| 2    | open  | 打开文件           | 39   | getpid | 获得进程ID           |
| 3    | close | 关闭文件           | 57   | fork   | 创建进程             |
| 4    | stat  | 获得文件信息       | 59   | execve | 执行一个程序         |
| 9    | mmap  | 将内存页映射到文件 | 60   | _exit  | 终止进程             |
| 12   | brk   | 重置堆顶           | 61   | wait4  | 等待一个进程终止     |
| 32   | dup2  | 复制文件描述符     | 62   | kill   | 发送信号到一个进程   |

C 程序用 syscall 函数可以直接调用任何系统调用。然而，实际中几乎没必要这么做。对于大多数系统调用，标准 C 库提供了一组方便的包装函数。这些包装函数将参数打包到一起，以适当的系统调用指令陷入内核，然后将系统调用的返回状态传递回调用程序。

在 X86-64 系统上，系统调用是通过一条称为 syscall 的陷阱指令来提供的。

Linux 系统调用的参数都是通过通用寄存器而不是栈传递的。

- 按照惯例，寄存器 ％rax 包含系统调用号，寄存器 %rdi、%rsi、%rdx、%r10、%r8 和 ％r9 包含最多 6 个参数。第一个参数在 ％rdi 中，第二个在 ％rsi 中，以此类推。
- 从系统调用返回时，寄存器 %rcx 和 ％r11 都会被破坏，％rax 包含返回值。-4095 到 -1 之间的负数返回值表明发生了错误，对应于负的 errno。

例如，考虑大家熟悉的 hello 程序的下面这个版本，用系统级函数 write（见 10.4 节）来写，而不是用 printf：

```c
int main()
{
    write(1, "hello, world\n", 13);
    _exit(0);
}
```

> write 函数的第一个参数将输出发送到 stdout。第二个参数是要写的字节序列，而第三个参数是要写的字节数。

下面给出的是 hello 程序的汇编语言版本,直接使用 syscall 指令来调用 write 和 exit 系统调用

```assembly
.section .data
string:
  .ascii "hello, world\n"
string_end:
  .equ len, string_end - string
.section .text
.globl main
main:
  # First, call write(1, "hello, world\n", 13)
  movq $1, %rax                 # write is system call 1
  movq $1, %rdi                 # Arg1: stdout has descriptor 1
  movq $string, %rsi            # Arg2: hello world string
  movq $len, %rdx               # Arg3: string length
  syscall                       # Make the system call

  # Next, call _exit(0)
  movq $60, %rax                # _exit is system call 60
  movq $0, %rdi                 # Arg1: exit status is 0
  syscall                       # Make the system call
```




## 8.2 进程

异常是允许操作系统内核提供**进程**（process）概念的基本构造块，进程是计算机科学中最深刻、最成功的概念之一。

进程的经典定义就是一个执行中程序的实例。系统中的每个程序都运行在某个进程的**上下文**（context）中。上下文是由程序正确运行所需的状态组成的。这个状态包括存**<u>放在内存中的程序的代码和数据，它的栈、通用目的寄存器的内容、程序计数器、环境变量以及打开文件描述符的集合。</u>**

每次用户通过向 shell 输入一个可执行目标文件的名字，运行程序时，shell 就会创建一个新的进程，然后在这个新进程的上下文中运行这个可执行目标文件。应用程序也能够创建新进程，并且在这个新进程的上下文中运行它们自己的代码或其他应用程序。

进程提供给应用程序的关键抽象：

- **一个独立的逻辑控制流，它提供一个假象，好像我们的程序独占地使用处理器。**
- **一个私有的地址空间，它提供一个假象，好像我们的程序独占地使用内存系统。**

### 8.2.1 逻辑控制流

如果想用调试器单步执行程序，我们会看到一系列的程序计数器（PC）的值，这些值唯一地对应于包含在程序的可执行目标文件中的指令，或是包含在运行时动态链接到程序的共享对象中的指令。

<u>这个 PC 值的序列叫做**逻辑控制流**，或者简称**逻辑流**。</u>

如图 8-12 所示。处理器的一个物理控制流被分成了三个逻辑流，每个进程一个。进程为每个程序提供了一种假象，好像程序在独占地使用处理器。每个竖直的条表示一个进程的逻辑控制流的一部分。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0812-luo-ji-kong-zhi-liu-.png" alt="img" style="zoom: 67%;" />

> 每个进程执行它的流的一部分，然后被**抢占**（preempted）（暂时挂起），然后轮到其他进程。对于一个运行在这些进程之一的上下文中的程序，它看上去就像是在独占地使用处理器。

### 8.2.2 并发流

异常处理程序、进程、信号处理程序、线程和 Java 进程都是逻辑流的例子。

一个逻辑流的执行在时间上与另一个流重叠，称为**并发流**（concurrent flow），这两个流被称为**并发地运行**。

多个流并发地执行的一般现象被称为**并发**（concurrency）。一个进程和其他进程轮流运行的概念称为**多任务**（multitasking）o 一个进程执行它的控制流的一部分的每一时间段叫做**时间片**（time slice）。因此，多任务也叫做**时间分片**（timeslicing）。例如，图 8-12 中，进程 A 的流由两个时间片组成。

如果两个流并发地运行在不同的处理器核或者计算机上，那么我们称它们为**并行流**（parallel flow），它们**并行地运行**（running in parallel），且**并行地执行**（parallel execution）。

### 8.2.3 私有地址空间

进程为每个程序提供它自己的**私有地址空间**。一般而言，和这个空间中某个地址相关联的那个内存字节是不能被其他进程读或者写的，从这个意义上说，这个地址空间是私有的。

尽管和每个私有地址空间相关联的内存的内容一般是不同的，但是每个这样的空间都有相同的通用结构。比如，图 8-13 展示了一个 x86-64 Linux 进程的地址空间的组织结构。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-13%20%E8%BF%9B%E7%A8%8B%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4.png" alt="图 8-13 进程地址空间" style="zoom:80%;" />

> 地址空间底部是保留给用户程序的，包括通常的代码、数据、堆和栈段。
>
> 代码段总是从地址 0x400000 开始。
>
> 地址空间顶部保留给内核（操作系统常驻内存的部分）。地址空间的这个部分包含内核在代表进程执行指令时（比如当应用程序执行系统调用时）使用的代码、数据和栈。

### 8.2.4 用户模式和内核模式

为了使操作系统内核提供一个无懈可击的进程抽象，处理器必须提供一种机制，限制一个应用可以执行的指令以及它可以访问的地址空间范围。

处理器通常是用某个控制寄存器中的一个**模式位**（mode bit）来提供这种功能的，该寄存器描述了进程当前享有的特权。

- 当设置了模式位时，进程就运行在**内核模式**中（有时叫做**超级用户模式**）。一个运行在内核模式的进程可以执行指令集中的任何指令，并且可以访问系统中的任何内存位置。
- 没有设置模式位时，进程就运行在用户模式中。用户模式中的进程不允许执行**特权指令**（privileged instruction），比如停止处理器、改变模式位，或者发起一个 I/O操作。也不允许用户模式中的进程直接引用地址空间中内核区内的代码和数据。任何这样的尝试都会导致致命的保护故障。反之，用户程序必须通过系统调用接口间接地访问内核代码和数据。

<u>**进程从用户模式变为内核模式的唯一方法是通过诸如中断、故障或者陷入系统调用这样的异常。**</u>

> 当异常发生时，控制传递到异常处理程序，处理器将模式从用户模式变为内核模式。处理程序运行在内核模式中，当它返回到应用程序代码时，处理器就把模式从内核模式改回到用户模式。

Linux 提供了一种聪明的机制，叫做 /proc 文件系统，它允许用户模式进程访问内核数据结构的内容。

> Linux 的 **`/proc`** 文件系统是一个虚拟文件系统，它提供了内核和进程的运行时信息。这个文件系统位于 **`/proc`** 目录下，包含了大量的文件和目录，允许用户查看和修改操作系统的内核信息、进程信息、硬件状态等。
>
> ### `/proc` 文件系统概述
>
> - **虚拟文件系统**：`/proc` 不是一个实际的磁盘文件系统，而是内存中的虚拟文件系统，提供内核信息，数据并不是存储在磁盘上，而是通过内核动态生成。
> - **信息访问**：`/proc` 使用户能够直接访问内核和系统状态信息，或者以特定格式读取系统参数。它主要是用于调试、监视、配置操作系统和应用程序。
>
> ### 主要目录和文件
>
> `/proc` 下包含许多子目录和文件，下面是一些常见的目录和文件：
>
> #### 1. **`/proc/[pid]`**
>
> 每个正在运行的进程在 `/proc` 中都有一个对应的子目录，目录名称就是进程的 PID。比如，进程 ID 为 1234 的进程的信息可以在 `/proc/1234` 中找到。
>
> 常见的文件：
>
> - **`/proc/[pid]/cmdline`**：进程的启动命令行参数。
> - **`/proc/[pid]/status`**：进程的状态信息，包括内存使用情况、线程数、信号等。
> - **`/proc/[pid]/stat`**：进程的运行统计信息。
> - **`/proc/[pid]/fd/`**：进程打开的文件描述符目录。
> - **`/proc/[pid]/maps`**：进程的虚拟内存映射。
>
> #### 2. **`/proc/cpuinfo`**
>
> 提供了 CPU 的详细信息，如型号、核心数、频率、缓存等。例如：
>
> ```bash
> $ cat /proc/cpuinfo
> processor	: 0
> vendor_id	: GenuineIntel
> cpu family	: 6
> model		: 158
> model name	: Intel(R) Core(TM) i7-8650U CPU @ 1.90GHz
> cpu MHz		: 2112.000
> ...
> ```
>
> #### 3. **`/proc/meminfo`**
>
> 显示系统内存的详细信息，包括总内存、空闲内存、缓存、交换空间等。例如：
>
> ```bash
> $ cat /proc/meminfo
> MemTotal:        16301544 kB
> MemFree:          6745984 kB
> MemAvailable:     10656088 kB
> Buffers:           702904 kB
> Cached:           4174028 kB
> SwapCached:            0 kB
> ...
> ```
>
> #### 4. **`/proc/uptime`**
>
> 显示系统启动以来的运行时间，以及系统空闲时间。例如：
>
> ```bash
> $ cat /proc/uptime
> 12345.67 9876.54
> ```
>
> - 第一个值是系统的运行时间（秒）。
> - 第二个值是系统的空闲时间（秒）。
>
> #### 5. **`/proc/version`**
>
> 显示内核版本、操作系统类型等信息。例如：
>
> ```bash
> $ cat /proc/version
> Linux version 5.4.0-42-generic (buildd@lgw01-amd64-046) (gcc version 9.3.0 (Ubuntu 9.3.0-17ubuntu1~20.04)) #46-Ubuntu SMP Fri Jul 10 12:10:31 UTC 2020
> ```
>
> #### 6. **`/proc/partitions`**
>
> 列出系统中的所有磁盘分区及其大小。例如：
>
> ```bash
> $ cat /proc/partitions
> major minor  #blocks  name
> 
>   8     0  1953514584 sda
>   8     1     524288  sda1
>   8     2  1952992384 sda2
> ```
>
> #### 7. **`/proc/net`**
>
> 显示网络相关信息。比如：
>
> - **`/proc/net/dev`**：网络接口的统计信息。
> - **`/proc/net/tcp`**：TCP 连接的详细信息。
>
> ```bash
> $ cat /proc/net/dev
> Inter-|   Receive                                                    |  Transmit
>  face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
>     lo: 55608746 469968 0 0 0 0 0 0 55608746 469968 0 0 0 0 0 0
>   eth0: 342235392 2203848 0 0 0 0 0 0 42186329 418178 0 0 0 0 0 0
> ```
>
> #### 8. **`/proc/sys`**
>
> 该目录下包含了许多内核参数，可以通过这些文件动态地查看和修改内核配置。例如：
>
> - **`/proc/sys/net/ipv4/ip_forward`**：查看和修改 IP 转发设置。
> - **`/proc/sys/vm/swappiness`**：控制交换空间的使用策略。
>
> 通过这些文件可以直接调整系统行为，而不需要重新启动或修改配置文件。
>
> #### 9. **`/proc/stat`**
>
> 显示系统级的统计信息，如 CPU 使用率、上下文切换、系统调用等。例如：
>
> ```
> bash复制编辑$ cat /proc/stat
> cpu  2401206 8950 1534466 257550426 115731 0 27758 0 0 0
> cpu0 1357533 3363 740510 128776017 61775 0 13871 0 0 0
> ...
> ```
>
> #### 10. **`/proc/interrupts`**
>
> 显示系统中的硬件中断信息。例如：
>
> ```bash
> $ cat /proc/interrupts
>            CPU0       CPU1       CPU2       CPU3       
>   0:    4302372    4101806    4136948    4177746   IO-APIC   2-edge      timer
>   1:          0          0          0          0   IO-APIC   1-edge      i8042
> ...
> ```
>
> #### 11. **`/proc/loadavg`**
>
> 显示系统的负载情况（类似于 `uptime` 命令）。它显示的是系统的 1 分钟、5 分钟和 15 分钟的平均负载。例如：
>
> ```bash
> $ cat /proc/loadavg
> 0.10 0.08 0.06 1/129 9823
> ```
>
> ### 访问 `/proc` 的常见操作
>
> 1. **读取信息**：使用 `cat` 或 `less` 查看信息，如 `/proc/meminfo`、`/proc/cpuinfo`。
>
> 2. **修改内核参数**：通过 `echo` 命令向 `/proc/sys` 下的文件写入新的值，动态修改内核行为。例如：
>
>    ```
>    echo 1 > /proc/sys/net/ipv4/ip_forward
>    ```
>
>    这会启用 IP 转发。
>
> 3. **查看进程信息**：查看特定进程的详细信息，可以使用 `cat` 命令查看 `/proc/[pid]/status`、`/proc/[pid]/cmdline` 等文件。
>
> 4. **监控系统**：利用 `/proc/uptime`、`/proc/loadavg` 等文件实时监控系统的健康状况。

### 8.2.5 上下文切换

操作系统内核使用一种称为**上下文切换**（context switch）的较高层形式的异常控制流来实现多任务。上下文切换机制是建立在 8.1 节中已经讨论过的那些较低层异常机制之上的。

内核为每个进程维持一个**上下文**（context）。上下文就是内核重新启动一个被抢占的进程所需的状态。它由一些对象的值组成，这些对象包括**<u>通用目的寄存器、浮点寄存器、程序计数器、用户栈、状态寄存器、内核栈和各种内核数据结构</u>**，比如描述地址空间的页表、包含有关当前进程信息的进程表，以及包含进程已打开文件的信息的文件表。

在进程执行的某些时刻，内核可以决定抢占当前进程，并重新开始一个先前被抢占了的进程。这种决策就叫做**调度**（scheduling），是由内核中称为**调度器**（scheduler）的代码处理的。

> 当内核选择一个新的进程运行时，我们说内核调度了这个进程。在内核调度了一个新的进程运行后，它就抢占当前进程，并使用一种称为上下文切换的机制来将控制转移到新的进程，

**上下文切换的过程**

> 1. **保存当前进程的状态（上下文）**：
>    - 操作系统需要保存当前进程的执行状态，包括寄存器的值、程序计数器（PC）的值、栈指针（SP）、标志寄存器等，这些都构成了进程的上下文。
> 2. **加载下一个进程的状态**：
>    - 操作系统加载下一个进程的上下文，这包括恢复它的寄存器值、程序计数器等信息，从而让新进程能从它停止的地方继续执行。
> 3. **切换到新进程**：
>    - 操作系统将 CPU 控制权交给新的进程，新的进程开始执行。

**上下文切换的代价**

> - **时间成本**：
>   上下文切换会消耗一定的 CPU 时间，因为操作系统需要保存和加载进程的状态。特别是在进程数目多的情况下，频繁的上下文切换会显著影响系统的性能。
> - **缓存失效**：
>   进程的上下文切换会导致 CPU 缓存中的数据失效，因为每个进程有自己独立的地址空间。当切换到另一个进程时，CPU 缓存中可能存储的是前一个进程的数据，导致下一个进程的数据无法有效利用缓存。
> - **系统开销**：
>   上下文切换增加了系统的负担，尤其是在处理器的核心数较少的情况下，频繁的上下文切换会带来较高的系统开销。

**上下文切换的触发条件**

> 1. **时间片用完**：
>    在时间片轮转调度算法下，操作系统会给每个进程分配一个时间片。时间片到期时，操作系统会强制进行一次上下文切换，将 CPU 控制权交给下一个进程。
> 2. **进程阻塞**（中断）：
>    如果一个进程请求某些资源（例如 I/O 操作）并且阻塞，操作系统会将其暂停，然后选择其他就绪的进程运行。
> 3. **进程终止**：
>    当一个进程完成任务或被终止时，操作系统需要进行上下文切换，选择另一个进程来继续执行。
> 4. **系统调用**：
>    当一个进程执行系统调用（例如请求操作系统服务时），可能会导致上下文切换。某些系统调用会导致进程的调度，而有些会让操作系统选择一个新的进程来执行。

图 8-14 展示了一对进程 A 和 B 之间上下文切换的示例。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-14%20%E8%BF%9B%E7%A8%8B%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2%E7%9A%84%E5%89%96%E6%9E%90.png" alt="图 8-14 进程上下文切换的剖析" style="zoom:67%;" />



## 8.3 系统调用错误处理

当 Unix 系统级函数遇到错误时，它们通常会返回 —1，并设置全局整数变量 errno 来表示什么出错了。

程序员应该总是检査错误，但是不幸的是，许多人都忽略了错误检查，因为它使代码变得臃肿，而且难以读懂。比如，下面是我们调用 Unix fork 函数时会如何检査错误：

```c
if ((pid = fork()) < 0) {
    fprintf(stderr, "fork error: %s\n", strerror(errno));
    exit(0);
}
```

strerror 函数返回一个文本串，描述了和某个 errno 值相关联的错误。通过定义下面的错误报告函数，我们能够在某种程度上简化这个代码：

```c
void unix_error(char *msg) /* Unix-style error */
{
    fprintf(stderr, "%s: %s\n", msg, strerror(errno));
    exit(0);
}
```

给定这个函数，我们对 fork 的调用从 4 行缩减到 2 行：

```c
if ((pid = fork()) < 0)
    unix_error("fork error");
```

通过使用错误处理包装函数，我们可以更进一步地简化代码,包装函数调用基本函数，检査错误，如果有任何问题就终止。比如，下面是 fork 函数的错误处理包装函数：

```c
pid_t Fork(void)
{
    pid_t pid;
  
    if ((pid = fork()) < 0)
        unix_error("Fork error");
    return pid;
}
```

给定这个包装函数，我们对 fork 的调用就缩减为 1 行：

```c
pid = Fork();
```





许多Linux异步信号安全的函数都会在出错返回时设置 errno。在处理程序中调用这样的函数可能会干扰主程序中其他依赖于errno的部分。解决方法是在进人处理程序时把errno保存在一个局部变量中，在处理程序返回前恢复它。

c++示例

```c++
#include <iostream>
#include <unistd.h>
#include <signal.h>
#include <errno.h>
#include <cstring>
#include <fcntl.h>

// 信号处理程序
void signal_handler(int signo) {
    // 保存当前errno的值
    int saved_errno = errno;

    // 处理信号，演示一个使用异步信号安全函数的场景
    const char* msg = "Signal received!\n";
    ssize_t bytesWritten = write(STDOUT_FILENO, msg, strlen(msg));

    if (bytesWritten == -1) {
        std::cerr << "写入失败: " << strerror(errno) << std::endl;
    }

    // 恢复之前保存的errno值
    errno = saved_errno;
}

int main() {
    // 注册SIGINT信号处理程序
    if (signal(SIGINT, signal_handler) == SIG_ERR) {
        std::cerr << "注册信号处理程序失败: " << strerror(errno) << std::endl;
        return 1;
    }

    // 主程序部分
    std::cout << "程序正在运行... 按Ctrl+C中断程序" << std::endl;

    // 模拟一些需要依赖errno的操作
    int fd = open("nonexistent_file.txt", O_RDONLY);
    if (fd == -1) {
        std::cerr << "打开文件失败: " << strerror(errno) << std::endl;
    }

    // 无限循环，等待信号
    while (true) {
        // 主程序可以在这里执行其他任务
    }

    return 0;
}

```


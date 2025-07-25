## 8.5 信号

在本节中，我们将研究一种更高层的软件形式的异常，称为 **Linux 信号**，它允许进程和内核中断其他进程。

一个**信号**就是一条小消息，它通知进程系统中发生了一个某种类型的事件。比如，图 8-26 展示了 Linux 系统上支持的 30 种不同类型的信号。

| 序号 | 名称      | 默认行为                    | 相应事件                       |
| ---- | --------- | --------------------------- | ------------------------------ |
| 1    | SIGHUP    | 终止                        | 终端线挂断                     |
| 2    | SIGINT    | 终止                        | 来自键盘的中断                 |
| 3    | SIGQUIT   | 终止                        | 来自键盘的退出                 |
| 4    | SIGILL    | 终止                        | 非法指令                       |
| 5    | SIGTRAP   | 终止并转储内存$$^①$$        | 跟踪陷阱                       |
| 6    | SIGABRT   | 终止并转储内存$$^①$$        | 来自 abort 函数的终止信号      |
| 7    | SIGBUS    | 终止                        | 总线错误                       |
| 8    | SIGFPE    | 终止并转储内存$$^①$$        | 浮点异常                       |
| 9    | SIGKILL   | 终止$$^②$$                  | 杀死程序                       |
| 10   | SIGUSR1   | 终止                        | 用户定义的信号 1               |
| 11   | SIGSEGV   | 终止并转储内存$$^①$$        | 无效的内存引用（段故障）       |
| 12   | SIGUSR2   | 终止                        | 用户定义的信号 2               |
| 13   | SIGPIPE   | 终止                        | 向一个没有读用户的管道做写操作 |
| 14   | SIGALRM   | 终止                        | 来自 alarm 函数的定时器信号    |
| 15   | SIGTERM   | 终止                        | 软件终止信号                   |
| 16   | SIGSTKFLT | 终止                        | 协处理器上的栈故障             |
| 17   | SIGCHLD   | 忽略                        | 一个子进程停止或者终止         |
| 18   | SIGCONT   | 忽略                        | 继续进程如果该进程停止         |
| 19   | SIGSTOP   | 停止直到下一个SIGCONT$$^②$$ | 不是来自终端的停止信号         |
| 20   | SIGTSTP   | 停止直到下一个SIGCONT       | 来自终端的停止信号             |
| 21   | SIGTTIN   | 停止直到下一个SIGCONT       | 后台进程从终端读               |
| 22   | SIGTTOU   | 停止直到下一个SIGCONT       | 后台进程向终端写               |
| 23   | SIGURG    | 忽略                        | 套接字上的紧急情况             |
| 24   | SIGXCPU   | 终止                        | CPU 时间限制超出               |
| 25   | SIGXFSZ   | 终止                        | 文件大小限制超出               |
| 26   | SIGVTALRM | 终止                        | 虚拟定时器期满                 |
| 27   | SIGPROF   | 终止                        | 剖析定时器期满                 |
| 28   | SIGWINCH  | 忽略                        | 窗口大小变化                   |
| 29   | SIGIO     | 终止                        | 在某个描述符上可执行 I/O 操作  |
| 30   | SIGPWR    | 终止                        | 电源故障                       |

每种信号类型都对应于某种系统事件。低层的硬件异常是由内核异常处理程序处理的，正常情况下，对用户进程而言是不可见的。信号提供了一种机制，通知用户进程发生了这些异常。

> 比如，如果一个进程试图除以 0，那么内核就发送给它一个 SIGFPE 信号（号码 8）。如果一个进程执行一条非法指令，那么内核就发送给它一个 SIGILL 信号（号码 4）。如果进程进行非法内存引用，内核就发送给它一个 SIGSEGV 信号（号码 11）。
>
> 其他信号对应于内核或者其他用户进程中较高层的软件事件。比如，如果当进程在前台运行时，你键入 Ctrl+C（也就是同时按下 Ctrl 键和 C 键），那么内核就会发送一个 SIGINT 信号（号码 2）给这个前台进程组中的每个进程。一个进程可以通过向另一个进程发送一个 SIGKILL 信号（号码 9）强制终止它。当一个子进程终止或者停止时，内核会发送一个 SIGCHLD 信号（号码 17）给父进程。

### 8.5.1 信号术语

传送一个信号到目的进程是由两个不同步骤组成的：

- **发送信号。**内核通过更新目的进程上下文中的某个状态，发送（递送）一个信号给目的进程。发送信号可以有如下两种原因：

  - 1）内核检测到一个系统事件，比如除零错误或者子进程终止。
  - 2）一个进程调用了 kill 函数（在下一节中讨论），显式地要求内核发送一个信号给目的进程。

  一个进程可以发送信号给它自己。

- **接收信号。当目的进程被内核强迫以某种方式对信号的发送做出反应时，它就接收了信号。进程可以忽略这个信号，终止或者通过执行一个称为信号处理程序**（signal handler）的用户层函数捕获这个信号。图 8-27 给出了信号处理程序捕获信号的基本思想：接收到信号会触发控制转移到信号处理程序。在信号处理程序完成处理之后，它将控制返回给被中断的程序

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-27%20%E4%BF%A1%E5%8F%B7%E5%A4%84%E7%90%86.png" alt="img" style="zoom:67%;" />

一个发出而没有被接收的信号叫做**待处理信号**（pending signal）。在任何时刻，一种类型至多只会有一个待处理信号。

> 如果一个进程有一个类型为A的待处理信号，那么任何接下来发送到这个进程的类型为A的信号都不会排队等待；它们只是被简单地丢弃。

一个进程可以有选择性地阻塞接收某种信号。当一种信号被阻塞时，它仍可以被发送，但是产生的待处理信号不会被接收，直到进程取消对这种信号的阻塞。

一个待处理信号最多只能被接收一次。内核为每个进程在 pending 位向量中维护着待处理信号的集合，而在✦ blocked 位向量✦中维护着被阻塞的信号集合。只要传送了一个类型为 k 的信号，内核就会设置 pending 中的第 k 位，而只要接收了一个类型为 k 的信号，内核就会清除 pending 中的第 k 位。

### 8.5.2 发送信号



Unix 系统提供了大量向进程发送信号的机制。所有这些机制都是基于进程组（process group）这个概念的。

#### 1. 进程组

每个进程都只属于一个进程组，进程组是由一个正整数进程组 ID 来标识的。getpgrp 函数返回当前进程的进程组 ID：

```c
#include <unistd.h>
pid_t getpgrp(void);

// 返回：调用进程的进程组 ID。
```

默认地，一个子进程和它的父进程同属于一个进程组。一个进程可以通过使用 setpgid 函数来改变自己或者其他进程的进程组：

```c
#include <unistd.h>
int setpgid(pid_t pid, pid_t pgid);

// 返回：若成功则为o，若错误则为 -1。
```

etpgid 函数将进程 pid 的进程组改为 pgid。如果 pid 是 0，那么就使用当前进程的 PID。如果 pgid 是 0，那么就用 pid 指定的进程的 PID 作为进程组 ID。例如，如果进程 15213 是调用进程，那么

```c
setpgid(0, 0);
```

会创建一个新的进程组，其进程组 ID 是 15213，并且把进程 15213 加入到这个新的进程组中。

#### 2. 用 /bin/kill 程序发送信号

/bin/kill 程序可以向另外的进程发送任意的信号。比如，命令

```sh
linux> /bin/kill -9 15213
```

发送信号 9（SIGKILL）给进程 15213。一个为负的 PID 会导致信号被发送到进程组 PID 中的每个进程。比如，命令

```
linux> /bin/kill -9 -15213
```

发送一个 SIGKILL 信号给进程组 15213 中的每个进程。注意，在此我们使用完整路径 /bin/kill，因为有些 Unix shell 有自己内置的 kill 命令。

#### 3. 从键盘发送信号

Unix shell 使用**作业**（job）这个抽象概念来表示为对一条命令行求值而创建的进程。在任何时刻，至多只有一个前台作业和 0 个或多个后台作业。比如，键入

```sh
linux> ls | sort
```

会创建一个由两个进程组成的前台作业，这两个进程是通过 Unix 管道连接起来的：一个进程运行 ls 程序，另一个运行 sort 程序。

shell 为每个作业创建一个独立的进程组。进程组 ID 通常取自作业中父进程中的一个。

> 比如，图 8-28 展示了有一个前台作业和两个后台作业的 shell。前台作业中的父进程 PID 为 20，进程组 ID 也为 20。父进程创建两个子进程，每个也都是进程组 20 的成员。
>
> <img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-28%20%E5%89%8D%E5%8F%B0%E5%92%8C%E5%90%8E%E5%8F%B0%E8%BF%9B%E7%A8%8B%E7%BB%84.png" alt="图 8-28 前台和后台进程组" style="zoom:67%;" />

在键盘上输入 Ctrl+C 会导致内核发送一个 SIGINT 信号到前台进程组中的每个进程。默认情况下，结果是**终止前台作业**。

类似地，输入 Ctrl+Z 会发送一个 SIGTSTP 信号到前台进程组中的每个进程。默认情况下，结果是**停止（挂起）前台作业**。

#### 4. 用 kill 函数发送信号

进程通过调用 kill 函数发送信号给其他进程（包括它们自己）。

```c
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int sig);

// 返回：若成功则为 0，若错误则为 -1。
```

- 如果 pid 大于零，那么 kill 函数发送信号号码 sig 给进程 pid。
- 如果 pid 等于零，那么 kill 发送信号 sig 给调用进程所在进程组中的每个进程，包括调用进程自己。
- 如果 pid 小于零，kill 发送信号 sig 给进程组 |pid|（pid 的绝对值）中的每个进程。

图 8-29 展示了一个示例，父进程用 kill 函数发送 SIGKILL 信号给它的子进程。

```c
#include "csapp.h"

int main()
{
    pid_t pid;

    /* Child sleeps until SIGKILL signal received, then dies */
    if ((pid = Fork()) == 0) {
        Pause(); /* Wait for a signal to arrive */
        printf("control should never reach here!\n");
        exit(0);
    }

    /* Parent sends a SIGKILL signal to a child */
    Kill(pid, SIGKILL);
    exit(0);
}
```

#### 5. 用 alarm 函数发送信号

进程可以通过调用 alarm 函数**<u>向它自己发送 SIGALRM 信号</u>**。

```c
#include <unistd.h>
unsigned int alarm(unsigned int secs);
//参数：seconds 是定时器的时间，单位为秒。当定时器到期时，进程会接收到 `SIGALRM` 信号。
// 返回：前一次闹钟剩余的秒数，若以前没有设定闹钟，则为0。
```

进程可以使用 `alarm` 函数来设置定时器，使得在指定的时间之后，向进程发送一个 `SIGALRM` 信号。这个信号通常用来控制进程的超时行为，或者在特定的时间点触发一些操作。

如果你多次调用 `alarm` 函数，后续的调用会覆盖之前的定时器设置。这意味着如果在第一次定时器还没有触发时，再次调用 `alarm` 会取消之前的定时器并设置一个新的定时器。

- 在 C++ 中，`alarm` 函数的使用与 C 语言相同，通过设置定时器来向进程发送 `SIGALRM` 信号。
- 可以使用 `signal` 函数注册一个自定义的信号处理函数来响应 `SIGALRM` 信号。
- 使用 `alarm(0)` 可以取消当前的定时器。
- 通过返回值可以查询之前设置的定时器时间。
- `alarm` 函数会覆盖之前的定时器设置，因此如果你多次调用 `alarm`，只有最后设置的定时器会生效。

这种机制适用于需要超时控制、定时任务或者时间到期后触发某些操作的场景。

python示例：使用 `signal.alarm` 实现超时控制

```python
import signal
import time

# 定义信号处理函数
def handler(signum, frame):
    print("Time's up! SIGALRM received.")

# 注册信号处理函数
signal.signal(signal.SIGALRM, handler)

# 设置定时器，10秒后发送 SIGALRM 信号
signal.alarm(10)

# 模拟长时间运行的任务
print("Task started. Waiting for signal...")

# 模拟任务工作
time.sleep(20)

print("Task completed.")

```

C++示例：

```c++
#include <iostream>
#include <unistd.h>
#include <signal.h>
#include <cstdlib>

// 信号处理函数
void handle_alarm(int sig) {
    std::cout << "Time's up! SIGALRM signal received." << std::endl;
    exit(0);  // 退出程序
}

int main() {
    // 注册 SIGALRM 信号处理函数
    signal(SIGALRM, handle_alarm);

    // 设置定时器，10秒后发送 SIGALRM 信号
    alarm(10);

    // 模拟长时间运行的任务
    std::cout << "Task started. Waiting for SIGALRM..." << std::endl;

    // 取消定时器
    alarm(0);
    std::cout << "Timer cancelled. No SIGALRM will be sent." << std::endl;

    // 模拟任务执行
    sleep(20);

    std::cout << "Task completed without timeout." << std::endl;

    return 0;
}

```
### 8.5.3 接收信号

当内核把进程 p 从内核模式切换到用户模式时（从系统调用返回或是完成了一次上下文切换），它会检查进程 p 的未被阻塞的待处理信号的集合（pending &~blocked）。

- 如果这个集合为空（通常情况下），那么内核将控制传递到 p 的逻辑控制流中的下一条指令（*I~next~*）。
- 如果集合是非空的，那么内核选择集合中的某个信号 k （通常是最小的 k），并且强制 p 接收信号 k。收到这个信号会触发进程采取某种行为。一旦进程完成了这个行为，那么控制就传递回 p 的逻辑控制流中的下一条指令（*I~next~*）

每个信号类型都有一个预定义的**默认行为**，是下面中的一种：

- 进程终止。
- 进程终止并转储内存。
- 进程停止（挂起）直到被 SIGCONT 信号重启。
- 进程忽略该信号。

> 图 8-26 展示了与每个信号类型相关联的默认行为。比如，收到 SIGKILL 的默认行为就是终止接收进程。另外，接收到 SIGCHLD 的默认行为就是忽略这个信号。

进程可以通过使用 signal 函数修改和信号相关联的默认行为。**<u>唯一的例外是 SIGSTOP 和 SIGKILL，它们的默认行为是不能修改的。</u>**

```c
#include <signal.h>
typedef void (*sighandler_t)(int);

sighandler_t signal(int signum, sighandler_t handler);

// 返回：若成功则为指向前次处理程序的指针，若出错则为 SIG_ERR（不设置 errno）。
```

signal 函数可以通过下列三种方法之一来改变和信号 signum 相关联的行为：

- 如果 handler 是 SIG_IGN，那么忽略类型为 signum 的信号。
- 如果 handler 是 SIG_DFL，那么类型为 signum 的信号行为恢复为默认行为。
- 否则，handler 就是用户定义的函数的地址，这个函数被称为**信号处理程序**，只要进程接收到一个类型为 signum 的信号，就会调用这个程序。通过把处理程序的地址传递到 signal 函数从而改变默认行为，这叫做**设置信号处理程序**（installing the handler）。调用信号处理程序被称为**捕获信号**。执行信号处理程序被称为**处理信号**。

当一个进程捕获了一个类型为 k 的信号时，会调用为信号 k 设置的处理程序，一个整数参数被设置为 k。这个参数允许同一个处理函数捕获不同类型的信号。(sighandler_t的一个int参数会被设置为k)

当处理程序执行它的 return 语句时，控制（通常）传递回控制流中进程被信号接收中断位置处的指令。我们说“通常”是因为在某些系统中，被中断的系统调用会立即返回一个错误。

图 8-30 展示了一个程序，它捕获用户在键盘上输入 Ctrl+C 时发送的 SIGINT 信号。SIGINT 的默认行为是立即终止该进程。在这个示例中，我们将默认行为修改为捕获信号，输出一条消息，然后终止该进程。

```c
#include "csapp.h"

void sigint_handler(int sig) /* SIGINT handler */
{
    printf("Caught SIGINT!\n");
    exit(0);
}

int main()
{
    /* Install the SIGINT handler */
    if (signal(SIGINT, sigint_handler) == SIG_ERR)
    	unix_error("signal error");

    pause(); /* Wait for the receipt of a signal */

    return 0;
}
```

信号处理程序可以被其他信号处理程序中断。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-31%20%E4%BF%A1%E5%8F%B7%E5%A4%84%E7%90%86%E7%A8%8B%E5%BA%8F%E5%8F%AF%E4%BB%A5%E8%A2%AB%E5%85%B6%E4%BB%96%E4%BF%A1%E5%8F%B7%E5%A4%84%E7%90%86%E7%A8%8B%E5%BA%8F%E4%B8%AD%E6%96%AD.png" alt="图 8-31 信号处理程序可以被其他信号处理程序中断" style="zoom:67%;" />

> 如图 8-31 所示。在这个例子中，主程序捕获到信号 s，该信号会中断主程序，将控制转移到处理程序 S。S 在运行时，程序捕获信号 t≠s，该信号会中断 S，控制转移到处理程序 T。当 T 返回时，S 从它被中断的地方继续执行。最后，S 返回，控制传送回主程序，主程序从它被中断的地方继续执行。
### 8.5.4 阻塞和解除阻塞信号

Linux 提供阻塞信号的隐式和显式的机制：

- **隐式阻塞机制。**内核默认阻塞任何当前处理程序正在处理信号类型的待处理的信号。

  > 例如，图 8-31 中，假设程序捕获了信号 s，当前正在运行处理程序 S。如果发送给该进程另一个信号 s，那么直到处理程序 S 返回，s 会变成待处理而没有被接收。

- **显式阻塞机制。**应用程序可以使用 sigprocmask 函数和它的辅助函数，明确地阻塞和解除阻塞选定的信号。

```c
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signum);
int sigdelset(sigset_t *set, int signum);
//返回；如果成功则为 0，若出错则为 -1。

int sigismember(const sigset_t *set, int signum);
// 返回：若 signum 是 set 的成员则为 1，如果不是则为 0，若出错则为 -1。
```

#### sigprocmask

> `sigprocmask` 是一个用于操作信号掩码（signal mask）的系统调用。它允许你查看、修改或设置进程的信号掩码，进而控制哪些信号可以被递送到进程。
>
> ##### 函数原型：
>
> ```c
> int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
> 
> ```
>
> ##### 参数说明：
>
> 1. **`how`**：指定如何修改信号掩码，有三个选项：
>    - **`SIG_BLOCK`**：将 `set` 中的信号添加到当前的信号掩码中，从而阻塞这些信号。即阻止这些信号的递送。
>    - **`SIG_UNBLOCK`**：将 `set` 中的信号从当前信号掩码中移除，解除阻塞这些信号。解除对这些信号的屏蔽，允许信号递送。
>    - **`SIG_SETMASK`**：将进程的信号掩码替换为 `set` 中的信号集合，完全覆盖当前信号掩码。
> 2. **`set`**：指向 `sigset_t` 类型的信号集，用于指定要操作的信号集合。该集合包含要被阻塞、解除阻塞或用于替换当前信号掩码的信号。
> 3. **`oldset`**：指向一个 `sigset_t` 变量，用来保存操作之前的信号掩码。如果不关心旧信号掩码，可以传递 `NULL`。
>
> ##### 返回值：
>
> - **成功时**：返回 `0`。
> - **失败时**：返回 `-1`，并设置 `errno` 以指示错误。
>
> ### `sigset_t` 类型：
>
> `sigset_t` 是一个用于表示信号集合的数据类型，通常它是一个位掩码，用于表示哪些信号是“有效”的。你可以使用 `sigemptyset`、`sigaddset` 和 `sigfillset` 等函数来操作 `sigset_t` 变量。
>
> ##### 常用信号操作函数：
>
> - **`sigemptyset(sigset_t \*set)`**：初始化一个信号集合，使其为空（没有任何信号）。
>
> - **`sigaddset(sigset_t \*set, int signo)`**：将信号 `signo` 添加到信号集合 `set` 中。
>
> - **`sigdelset(sigset_t \*set, int signo)`**：从信号集合 `set` 中删除信号 `signo`。

例如，图 8-32 展示了如何用 sigprocmask 来临时阻塞接收 SIGINT 信号。

```c
sigset_t mask, prev_mask;

Sigemptyset(&mask);
Sigaddset(&mask, SIGINT);

/* Block SIGINT and save previous blocked set */
Sigprocmask(SIG_BLOCK, &mask, &prev_mask);
.
.  // Code region that will not be interrupted by SIGINT
.
/* Restore previous blocked set, unblocking SIGINT */
Sigprocmask(SIG_SETMASK, &prev_mask, NULL);
```

> ##### 📌 **1️⃣ 什么是“阻塞进程的信号”？**
>
> **阻塞信号**是指 **暂时屏蔽（不处理）某些信号** 的机制。
>
> - **信号**（`signal`） → 操作系统发送给进程的异步通知
> - **阻塞信号** → **信号来了，但暂时不处理，等解除阻塞后再处理**
>
> ##### 📌 **2️⃣ 为什么要阻塞信号？**
>
> **目的是**：
>
> - 避免**在关键代码区**（临界区）**被异步中断**
> - 保证 **数据一致性**
> - **防止竞争条件（Race Condition）**
> - 保持进程状态**可控**
>
> ##### 📌 **3️⃣ 应用场景**
>
> | 场景                             | 原因                                     |
> | -------------------------------- | ---------------------------------------- |
> | **修改全局变量时**               | 避免信号处理函数同时访问导致数据不一致   |
> | **信号处理函数内部再接收到信号** | 避免嵌套处理，导致栈溢出或逻辑混乱       |
> | **事务型操作**                   | 保证一系列操作要么全部完成，要么全部不做 |
>
> ##### 📌 **5️⃣ 常见 API**
>
> **在 C 语言/Linux 中：**
>
> ```c
> sigprocmask(SIG_BLOCK, &set, NULL);    // 阻塞指定信号
> sigprocmask(SIG_UNBLOCK, &set, NULL);  // 解除阻塞
> sigpending(&set);                      // 查看哪些信号被挂起
> ```
>
> **Python 示例（仅部分信号支持）：**
>
> ```python
> import signal
> 
> signal.pthread_sigmask(signal.SIG_BLOCK, {signal.SIGINT})
> # 关键代码
> signal.pthread_sigmask(signal.SIG_UNBLOCK, {signal.SIGINT})
> ```
### 8.5.5 编写信号处理程序

信号处理是 Linux 系统编程最棘手的一个问题。处理程序有几个属性使得它们很难推理分析：

1. 处理程序与主程序并发运行，共享同样的全局变量，因此可能与主程序和其他处理程序互相干扰；
2. 如何以及何时接收信号的规则常常有违人的直觉；
3. 不同的系统有不同的信号处理语义。

#### 1. 安全的信号处理

信号处理程序很麻烦是因为它们和主程序以及其他信号处理程序并发地运行，正如我们在图 8-31 中看到的那样。如果处理程序和主程序并发地访问同样的全局数据结构，那么结果可能就不可预知，而且经常是致命的。

这里我们的目标是给你一些保守的编写处理程序的原则，使得这些处理程序能安全地并发运行。

- **G0. 处理程序要尽可能简单。**避免麻烦的最好方法是保持处理程序尽可能小和简单。例如，处理程序可能只是简单地设置全局标志并立即返回；所有与接收信号相关的处理都由主程序执行，它周期性地检查（并重置）这个标志。
- **G1. 在处理程序中只调用异步信号安全的函数。所谓异步信号安全**的函数（或简称安全的函数）能够被信号处理程序安全地调用，原因有二：要么它是可重入的（例如只访问局部变量，见 12.7.2 节），要么它不能被信号处理程序中断。图 8-33 列出了 Linux 保证安全的系统级函数。注意，许多常见的函数（例如 printf、sprintf、malloc 和 exit）都不在此列。

| _Exit         | fexecve     | poll              | sigqueue         |
| ------------- | ----------- | ----------------- | ---------------- |
| _exit         | fork        | posix_trace_event | sigset           |
| abort         | fstat       | pselect           | sigsuspend       |
| accept        | fstatat     | raise             | sleep            |
| access        | fsync       | read              | sockatmark       |
| aio_error     | ftruncate   | readlink          | socket           |
| aio_return    | futimens    | readlinkat        | socketpair       |
| aio_suspend   | getegid     | recv              | stat             |
| alarm         | geteuid     | recvfrom          | symlink          |
| bind          | getgid      | recvmsg           | symlinkat        |
| cfgetispeed   | getgroups   | rename            | tcdrain          |
| cfgetospeed   | getpeername | renameat          | tcflow           |
| cfsetispeed   | getpgrp     | rmdir             | tcflush          |
| cfsetospeed   | getpid      | select            | tcgetattr        |
| chdir         | getppid     | sem_post          | tcgetpgrp        |
| chmod         | getsockname | send              | tcsendbreak      |
| chown         | getsockopt  | sendmsg           | tcsetattr        |
| clock_gettime | getuid      | sendto            | tcsetpgrp        |
| close         | kill        | setgid            | time             |
| connect       | link        | setpgid           | timer_getoverrun |
| creat         | linkat      | setsid            | timer_gettime    |
| dup           | listen      | setsockopt        | timer_settime    |
| dup2          | lseek       | setuid            | times            |
| execl         | lstat       | shutdown          | umask            |
| execle        | mkdir       | sigaction         | uname            |
| execv         | mkdirat     | sigaddset         | unlink           |
| execve        | mkfifo      | sigdelset         | unlinkat         |
| faccessat     | mkfifoat    | sigemptyset       | utime            |
| fchmod        | mknod       | sigfillset        | utimensat        |
| fchmodat      | mknodat     | sigismember       | utimes           |
| fchown        | open        | signal            | wait             |
| fchownat      | openat      | sigpause          | waitpid          |
| fcntl         | pause       | sigpending        | write            |
| fdatasync     | pipe        | sigprocmask       |                  |

*图 8-33 异步信号安全的函数（来源：man 7 signal。数据来自 Linux Foundation）*

信号处理程序中产生输出唯一安全的方法是使用 write 函数（见 10.1 节）。<u>特别地，调用 printf 或 sprintf 是不安全的</u>。为了绕开这个不幸的限制，我们开发一些安全的函数，称为 SIO（安全的 I/O）包，可以用来在信号处理程序中打印简单的消息。

```c
#include "csapp.h"

ssize_t sio_putl(long v);
ssize_t sio_puts(char s[]);
// 返回：如果成功则为传送的字节数，如果出错，则为 -1。

void sio_error(char s[]);
// 返回：空。

ssize_t sio_puts(char s[]) /* Put string */
{
    return write(STDOUT_FILENO, s, sio_strlen(s));
}

ssize_t sio_putl(long v) /* Put long */
{
    char s[128];

    sio_ltoa(v, s, 10); /* Based on K&R itoa() */
    return sio_puts(s);
}

void sio_error(char s[]) /* Put error message and exit */
{
    sio_puts(s);
    _exit(1);
}
```

图 8-35 给出了图 8-30 中 SIGINT 处理程序的一个安全的版本。

```c
#include "csapp.h"

void sigint_handler(int sig) /* Safe SIGINT handler */
{
    Sio_puts("Caught SIGINT!\n"); /* Safe output */
    _exit(0);                     /* Safe exit */
}
```

- **G2. 保存和恢复 errno。**许多 Linux 异步信号安全的函数都会在出错返回时设置 errno。在处理程序中调用这样的函数可能会干扰主程序中其他依赖于 errno 的部分。解决方法是在进入处理程序时把 errno 保存在一个局部变量中，在处理程序返回前恢复它。**<u>注意，只有在处理程序要返回时才有此必要</u>**。如果处理程序调用 _exit 终止该进程，那么就不需要这样做了。

- **G3. 阻塞所有的信号，保护对共享全局数据结构的访问。**<u>如果处理程序和主程序或其他处理程序共享一个全局数据结构，那么在访问（读或者写）该数据结构时，你的处理程序和主程序应该暂时阻塞所有的信号。</u>这条规则的原因是从主程序访问一个数据结构 d 通常需要一系列的指令，如果指令序列被访问次的处理程序中断，那么处理程序可能会发现 d 的状态不一致，得到不可预知的结果。在访问 d 时暂时阻塞信号保证了处理程序不会中断该指令序列。

- **G4. 用 volatile 声明全局变量。**考虑一个处理程序和一个 main 函数，它们共享一个全局变量 g。处理程序更新 g，main 周期性地读 g。对于一个优化编译器而言，main 中 g 的值看上去从来没有变化过，因此使用缓存在寄存器中 g 的副本来满足对 g 的每次引用是很安全的。如果这样，main 函数可能永远都无法看到处理程序更新过的值。

  <u>可以用 volatile 类型限定符来定义一个变量，告诉编译器不要缓存这个变量</u>。例如：

  **`volatile int g;`**

  > volatile 限定符强迫编译器每次在代码中引用 g 时，都要从内存中读取 g 的值。一般来说，和其他所有共享数据结构一样，应该暂时阻塞信号，保护每次对全局变量的访问。

- **G5. 用 sig_atomic_t 声明标志。在常见的处理程序设计中，处理程序会写全局标志**来记录收到了信号。主程序周期性地读这个标志，响应信号，再清除该标志。对于通过这种方式来共享的标志，<u>C 提供一种整型数据类型 sig_atomic_t，对它的读和写保证会是原子的（不可中断的）</u>，因为可以用一条指令来实现它们：

  **`volatile sig_atomic_t flag;`**

  #### 2. 正确的信号处理

未处理的信号是不排队的。因为 pending 位向量中每种类型的信号只对应有一位，所以每种类型最多只能有一个未处理的信号。因此，如果两个类型 k 的信号发送给一个目的进程，而因为目的进程当前正在执行信号 k 的处理程序，所以信号 k 被阻塞了，那么第二个信号就简单地被丢弃了；它不会排队。关键思想是如果存在一个未处理的信号就表明**至少**有一个信号到达了。

要了解这样会如何影响正确性，来看一个简单的应用，它本质上类似于像 shell 和 Web 服务器这样的真实程序。

> 基本的结构是父进程创建一些子进程，这些子进程各自独立运行一段时间，然后终止。要求是：
>
> - 父进程必须回收子进程以避免在系统中留下僵死进程。
> - 但是我们还希望父进程能够在子进程运行时自由地去做其他的工作。
>
> 我们决定用 SIGCHLD 处理程序来回收子进程，而不是显式地等待子进程终止。（回想一下，只要有一个子进程终止或者停止，内核就会发送一个 SIGCHLD 信号给父进程。）

初次尝试：父进程设置了一个 SIGCHLD 处理程序，然后创建了 3 个子进程。同时，父进程等待来自终端的一个输入行，随后处理它。这个处理被模型化为一个无限循环。当每个子进程终止时，内核通过发送一个 SIGCHLD 信号通知父进程。父进程捕获这个 SIGCHLD 信号，回收一个子进程，做一些其他的清理工作（模型化为 sleep 语句），然后返回。

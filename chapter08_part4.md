```c
/* WARNING: This code is buggy! */

void handler1(int sig)
{
    int olderrno = errno;

    if ((waitpid(-1, NULL, 0)) < 0)
        sio_error("waitpid error");
    Sio_puts("Handler reaped child\n");
    Sleep(1);
    errno = olderrno;
}

int main()
{
    int i, n;
    char buf[MAXBUF];

    if (signal(SIGCHLD, handler1) == SIG_ERR)
        unix_error("signal error");

    /* Parent creates children */
    for (i = 0; i < 3; i++) {
        if (Fork() == 0) {
            printf("Hello from child %d\n", (int)getpid());
            exit(0);
        }
    }

    /* Parent waits for terminal input and then processes it */
    if ((n = read(STDIN_FILENO, buf, sizeof(buf))) < 0)
        unix_error("read");

    printf("Parent processing input\n");
    while (1)
        ;

    exit(0);

```

程序signal1是有缺陷的，因为它假设信号是排队的。

图 8-36 中的 signal1 程序看起来相当简单。然而，当在 Linux 系统上运行它时，我们得到如下输出：

```c
linux> ./signal1
Hello from child 14073
Hello from child 14074
Hello from child 14075
Handler reaped child
Handler reaped child
CR
Parent processing input
```

从输出中我们注意到，尽管发送了 3 个 SIGCHLD 信号给父进程，但是其中只有两个信号被接收了，因此父进程只是回收了两个子进程。如果挂起父进程，我们看到，实际上子进程 14075 没有被回收，它成了一个僵死进程（在 ps 命令的输出中由字符串 “defunct” 表明）：

```c
Ctrl+Z
Suspended
linux> ps t
  PID TTY   STAT   TIME COMMAND
  .
  .
  .
14072 pts/3 T      0:02 ./signal1
14075 pts/3 Z      0:00 [signal1] <defunct>
14076 pts/3 R+     0:00 ps t
```

> 哪里出错了呢？问题就在于我们的代码没有解决信号不会排队等待这样的情况。所发生的情况是：父进程接收并捕获了第一个信号。当处理程序还在处理第一个信号时，第二个信号就传送并添加到了待处理信号集合里。然而，因为 SIGCHLD 信号被 SIGCHLD 处理程序阻塞了，所以第二个信号就不会被接收。此后不久，就在处理程序还在处理第一个信号时，第三个信号到达了。因为已经有了一个待处理的 SIGCHLD，第三个 SIGCHLD 信号会被丢弃。一段时间之后，处理程序返回，内核注意到有一个待处理的 SIGCHLD 信号，就迫使父进程接收这个信号。父进程捕获这个信号，并第二次执行处理程序。在处理程序完成对第二个信号的处理之后，已经没有待处理的 SIGCHLD 信号了，而且也绝不会再有，因为第三个 SIGCHLD 的所有信息都已经丢失了。

**由此得到的重要教训是，不可以用信号来对其他进程中发生的事件计数。**

为了修正这个问题，我们必须回想一下，存在一个待处理的信号只是暗示自进程最后一次收到一个信号以来，至少已经有一个这种类型的信号被发送了。所以我们必须修改 SIGCHLD 的处理程序，<u>使得每次 SIGCHLD 处理程序被调用时，回收尽可能多的僵死子进程</u>。图 8-37 展示了修改后的 SIGCHLD 处理程序。

```c
void handler2(int sig)
{
    int olderrno = errno;

    while (waitpid(-1, NULL, 0) > 0) {
        Sio_puts("Handler reaped child\n");
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    Sleep(1);
    errno = olderrno;
}
```

*图 8-37 signal2：图 8-36 的一个改进版本，它能够正确解决信号不会排队等待的情况*

当我们在 Linux 系统上运行 signal2 时，它现在可以正确地回收所有的僵死子进程了：

```c
linux> ./signal2
Hello from child 15237
Hello from child 15238
Hello from child 15239
Handler reaped child
Handler reaped child
Handler reaped child
CR
Parent processing input
```
##### 练习题 8.8

下面这个程序的输出是什么？

```c
volatile long counter = 2;

void handler1(int sig)
{
    sigset_t mask, prev_mask;

    Sigfillset(&mask);
    Sigprocmask(SIG_BLOCK, &mask, &prev_mask);  /* Block sigs */
    Sio_putl(--counter);
    Sigprocmask(SIG_SETMASK, &prev_mask, NULL); /* Restore sigs */

    _exit(0);
}

int main()
{
    pid_t pid;
    sigset_t mask, prev_mask;

    printf("%ld", counter);
    fflush(stdout);

    signal(SIGUSR1, handler1);
    if ((pid = Fork()) == 0) {
        while (1) {};
    }
    Kill(pid, SIGUSR1);
    Waitpid(-1, NULL, 0);

    Sigfillset(&mask);
    Sigprocmask(SIG_BLOCK, &mask, &prev_mask);  /* Block sigs */
    printf("%ld", ++counter);
    Sigprocmask(SIG_SETMASK, &prev_mask, NULL); /* Restore sigs */

    exit(0);
}
```

> 这个程序打印字符串 “213”，这是卡内基—梅隆大学 CS：APP 课程的缩写名。父进程开始时打印 “2”，然后创建子进程，子进程会陷入一个无限循环。然后父进程向子进程发送一个信号，并等待它终止。子进程捕获这个信号（中断这个无限循环），对计数器值（从初始值 2）减一，打印 “1”，然后终止。在父进程回收子进程之后，它对计数器值（从初始值 2）加一，打印 “3”，并且终止。

#### 3. 可移植的信号处理



Unix 信号处理的另一个缺陷在于不同的系统有不同的信号处理语义。例如：

- **signal 函数的语义各有不同。**有些老的 Unix 系统在信号 k 被处理程序捕获之后就把对信号 k 的反应恢复到默认值。在这些系统上，每次运行之后，处理程序必须调用 signal 函数，显式地重新设置它自己。
- **系统调用可以被中断。像 read、write 和 accept 这样的系统调用潜在地会阻塞进程一段较长的时间，称为慢速系统调用**。在某些较早版本的 Unix 系统中，当处理程序捕获到一个信号时，被中断的慢速系统调用在信号处理程序返回时不再继续，而是立即返回给用户一个错误条件，并将 errno 设置为 EINTR。在这些系统上，程序员必须包括手动重启被中断的系统调用的代码。

要解决这些问题，Posix 标准定义了 sigaction 函数，它允许用户在设置信号处理时，明确指定他们想要的信号处理语义。

```c
#include <signal.h>

int sigaction(int signum, struct sigaction *act,
              struct sigaction *oldact);
// 返回：若成功则为 0，若出错则为 -1。
```

sigaction 函数运用并不广泛，因为它要求用户设置一个复杂结构的条目。一个更简洁的方式，最初是由 W. Richard Stevens 提出的【110】，就是定义一个包装函数，称为 Signal，它调用 sigaction。图 8-38 给出了 Signal 的定义，它的调用方式与 signal 函数的调用方式一样。

```c
handler_t *Signal(int signum, handler_t *handler)
{
    struct sigaction action, old_action;

    action.sa_handler = handler;
    sigemptyset(&action.sa_mask); /* Block sigs of type being handled */
    action.sa_flags = SA_RESTART; /* Restart syscalls if possible */
    
    if (sigaction(signum, &action, &old_action) < 0)
        unix_error("Signal error");
    return (old_action.sa_handler);
}
```

*图 8-38 Signal：sigaction 的一个包装函数，它提供在 Posix 兼容系统上的可移植的信号处理*

Signal 包装函数设置了一个信号处理程序，其信号处理语义如下：

- 只有这个处理程序当前正在处理的那种类型的信号被阻塞。
- 和所有信号实现一样，信号不会排队等待。
- 只要可能，被中断的系统调用会自动重启。
- 一旦设置了信号处理程序，它就会一直保持，直到 Signal 带着 handler 参数为 SIG_IGN 或者 SIG_DFL 被调用。

我们在所有的代码中实现 Signal 包装函数。

### 8.5.6 同步流以避免讨厌的并发错误



如何编写读写相同存储位置的并发流程序的问题，困扰着数代计算机科学家。一般而言，流可能交错的数量与指令的数量呈指数关系。这些交错中的一些会产生正确的结果，而有些则不会。基本的问题是以某种方式同步并发流，从而得到最大的可行的交错的集合，每个可行的交错都能得到正确的结果。

并发编程是一个很深且很重要的问题，我们将在第 12 章中更详细地讨论。不过，在本章中学习的有关异常控制流的知识，可以让你感觉一下与并发相关的有趣的智力挑战。例如，考虑图 8-39 中的程序，它总结了一个典型的 Unixshell 的结构。父进程在一个全局作业列表中记录着它的当前子进程，每个作业一个条目。addjob 和 deletejob 函数分别向这个作业列表添加和从中删除作业。

```c
/* WARNING: This code is buggy! */
void handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;

    Sigfillset(&mask_all);
    while ((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap a zombie child */
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        deletejob(pid); /* Delete the child from the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    errno = olderrno;
}

int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, prev_all;

    Sigfillset(&mask_all);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (1) {
        if ((pid = Fork()) == 0) { /* Child process */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all); /* Parent process */
        addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    exit(0);
}
```

> 图 8-39 一个具有细微同步错误的 shell 程序。如果子进程在父进程能够开始运行前就结束了，那么 addjob 和 deletejob 会以错误的方式被调用

当父进程创建一个新的子进程后，它就把这个子进程添加到作业列表中。当父进程在 SIGCHLD 处理程序中回收一个终止的（僵死）子进程时，它就从作业列表中删除这个子进程。

信一看，这段代码是对的。不幸的是，可能发生下面这样的事件序列：

1. 父进程执行 fork 函数，内核调度新创建的子进程运行，而不是父进程。
2. 在父进程能够再次运行之前，子进程就终止，并且变成一个僵死进程，使得内核传递一个 SIGCHLD 信号给父进程。
3. 后来，当父进程再次变成可运行但又在它执行之前，内核注意到有未处理的 SIGCHLD 信号，并通过在父进程中运行处理程序接收这个信号。
4. 信号处理程序回收终止的子进程，并调用 deletejob，这个函数什么也不做，因为父进程还没有把该子进程添加到列表中。
5. 在处理程序运行完毕后，内核运行父进程，父进程从 fork 返回，通过调用 add-job 错误地把（不存在的）子进程添加到作业列表中。

因此，对于父进程的 main 程序和信号处理流的某些交错，可能会在 addjob 之前调用 deletejob。这导致作业列表中出现一个不正确的条目，对应于一个不再存在而且永远也不会被删除的作业。另一方面，也有一些交错，事件按照正确的顺序发生。例如，如果在 fork 调用返回时，内核刚好调度父进程而不是子进程运行，那么父进程就会正确地把子进程添加到作业列表中，然后子进程终止，信号处理函数把该作业从列表中删除。

这是一个称为**竞争**（race）的经典同步错误的示例<u>。在这个情况中，main 函数中调用 addjob 和处理程序中调用 deletejob 之间存在竞争。</u>如果 addjob 赢得进展，那么结果就是正确的。如果它没有，那么结果就是错误的。这样的错误非常难以调试，因为几乎不可能测试所有的交错。你可能运行这段代码十亿次，也没有一次错误，但是下一次测试却导致引发竞争的交错。

图 8-40 展示了消除图 8-39 中竞争的一种方法。**通过在调用 fork 之前，阻塞 SIGCHLD 信号，然后在调用 addjob 之后取消阻塞这些信号，我们保证了在子进程被添加到作业列表中之后回收该子进程。**注意，子进程继承了它们父进程的被阻塞集合，所以我们必须在调用 execve 之前，小心地解除子进程中阻塞的 SIGCHLD 信号。



```c
void handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;

    Sigfillset(&mask_all);
    while ((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap a zombie child */
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        deletejob(pid); /* Delete the child from the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    errno = olderrno;
}

int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, mask_one, prev_one;

    Sigfillset(&mask_all);
    Sigemptyset(&mask_one);
    Sigaddset(&mask_one, SIGCHLD);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */
    
    while (1) {
        Sigprocmask(SIG_BLOCK, &mask_one, &prev_one); /* Block SIGCHLD */
        if ((pid = Fork()) == 0) { /* Child process */
            Sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */
        addjob(pid); /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
    }
    exit(0);
}
```

*图 8-40 用 sigprocmask 来同步进程。在这个例子中，父进程保证在相应的 deletejob 之前执行 addjob*

> 针对上面的程序，有如下两个问题：
>
> - 在main函数中创建子进程之前已经阻塞了mask_one，为什么在addjob前还要阻塞？
>
> 虽然 **`SIGCHLD`** 已经通过：
>
> ```
> Sigprocmask(SIG_BLOCK, &mask_one, &prev_one);
> ```
>
> **阻塞**了 → 可以防止 **子进程终止** 时 `handler()` 异步执行，避免和 `addjob(pid)` **同时操作作业表**。但是：还有其他信号可能中断 `addjob()` 的执行！所以**进一步屏蔽“所有信号”**，**彻底保护“临界区”**。总之：**阻塞 `SIGCHLD` 是为了防止子进程退出引发的 `handler()` 异步执行；**阻塞 `mask_all` 是为了防止其他未知/未来的信号干扰临界区 → 保证作业表的完整性。
>
> - 为什么addjob之后是使用&perv_one来恢复阻塞，而不是用mask_all来恢复？
>
> **`mask_one`** → 只包含 `SIGCHLD`
>
> **`mask_all`** → 所有信号
>
> **`prev_one`** → **在 `mask_one` 阻塞前的信号屏蔽状态**，通过第一次 `Sigprocmask` 得到
>
> **主要原因：**要保证程序恢复到“调用 `Sigprocmask(SIG_BLOCK, &mask_one, &prev_one);` 前的状态”，而不是盲目取消所有阻塞。这样可以避免误伤，如果在程序其他地方也主动屏蔽了某些信号（例如 `SIGINT`），盲目恢复可能会导致信号提前到达。
>
> 
## 8.6 非本地跳转

`longjmp` 和 `siglongjmp` 是两种用于非正常流程控制的函数，它们通常用于从一个 `setjmp` 调用返回，并可以跳转到 `setjmp` 被调用时保存的程序状态。这些函数对于实现错误处理、异常处理和信号处理等功能非常有用。

### 1. **`longjmp` 和 `setjmp`**

`longjmp` 和 `setjmp` 是标准库函数，主要用于非局部跳转。在 C 语言中，`setjmp` 和 `longjmp` 提供了一种类似于异常处理的机制，允许你从一个深层嵌套的函数调用中跳出，并且返回到先前保存的状态。

#### `setjmp`：

`setjmp` 用来保存当前的程序状态（堆栈、寄存器等），并将该状态存储到 `jmp_buf` 类型的变量中。如果 `setjmp` 返回 0，表示程序正常执行，如果跳转回来则返回非 0 值。

```c
int setjmp(jmp_buf env);
```

- **`env`**：保存程序状态的缓冲区（`jmp_buf` 类型），用于以后恢复程序的状态

#### `longjmp`：

`longjmp` 用来跳回到先前通过 `setjmp` 保存的状态。跳回时，`setjmp` 会返回一个非 0 的值，并且 `longjmp` 会将控制权交给调用 `setjmp` 的地方。

```c
void longjmp(jmp_buf env, int retval);
```

- **`env`**：存储程序状态的缓冲区，`longjmp` 会恢复到该状态。

- **`retval`**：`longjmp` 返回时给 `setjmp` 的返回值。通常，`retval` 是非零值，表示发生了跳转。

使用示例：

```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf env;

void func() {
    printf("In func, calling longjmp\n");
    longjmp(env, 1);  // 从 func 跳回 setjmp
}

int main() {
    if (setjmp(env) == 0) {
        printf("First time through setjmp\n");
        func();  // 会跳转到 longjmp
    } else {
        printf("Returned from longjmp\n");
    }
    return 0;
}

```

输出：

```c
First time through setjmp
In func, calling longjmp
Returned from longjmp
```

2. **`siglongjmp` 和 `sigsetjmp`**

`siglongjmp` 和 `sigsetjmp` 是与信号处理相关的版本，特别用于在处理信号时进行非局部跳转。它们与 `longjmp` 和 `setjmp` 类似，但是增加了信号掩码的管理，这在信号处理程序中尤其重要。`siglongjmp` 会跳转到一个由 `sigsetjmp` 保存的点，并可以恢复信号掩码。

`sigsetjmp` 是一个扩展的版本，它除了保存程序的状态（如 `setjmp`）外，还会保存当前的信号掩码。这使得我们可以控制信号在跳转时的行为，特别是在信号处理程序中。

```c
int sigsetjmp(sigjmp_buf env, int savesigs);
```

- **`env`**：保存程序状态的缓冲区，`sigjmp_buf` 类型。它保存了堆栈信息和信号掩码。
- **`savesigs`**：如果 `savesigs` 是非零值，`sigsetjmp` 会保存当前的信号掩码（也就是信号阻塞的状态）。如果 `savesigs` 为零，信号掩码不会被保存。

#### `siglongjmp`：

`siglongjmp` 与 `longjmp` 类似，但是它在跳转时会恢复信号掩码，从而确保信号的处理状态是正确的。

```c
void siglongjmp(sigjmp_buf env, int retval);
```

- **`env`**：保存程序状态的缓冲区，`sigjmp_buf` 类型，保存堆栈信息和信号掩码。
- **`retval`**：跳转时传给 `sigsetjmp` 的返回值，通常是非零值。

使用示例：当用户在键盘上键入Ctrl十C时，这个程序用信号和非本地跳转来实现软重启

```c
/* $begin restart */
#include "csapp.h"

sigjmp_buf buf;

void handler(int sig) 
{
    siglongjmp(buf, 1);
}

int main() 
{
    if (!sigsetjmp(buf, 1)) {
        Signal(SIGINT, handler);
	Sio_puts("starting\n");
    }
    else 
	Sio_puts("restarting\n");

    while(1) {
	Sleep(1);
	Sio_puts("processing...\n");
    }
    exit(0); /* Control never reaches here */
}
/* $end restart */
```

输出：

```
linux> ./restart 
starting
processing... 
processing.. 
Ctrl+c
restarting 
processing... 
Ctrl+c
restarting 
processing..
```

示例：通过 `SIGUSR1` 控制其他进程执行某些功能

假设我们有两个进程，一个作为“控制进程”来发送信号，另一个作为“目标进程”来接收信号并执行特定功能。

#### 1. 目标进程：接收信号并执行功能

```c++
#include <iostream>
#include <csignal>
#include <unistd.h>

void signal_handler(int sig) {
    if (sig == SIGUSR1) {
        std::cout << "Received SIGUSR1: Executing custom function!" << std::endl;
        // 这里可以执行其他功能，如修改变量、执行特定任务等
    }
}

int main() {
    // 注册 SIGUSR1 信号的处理程序
    signal(SIGUSR1, signal_handler);

    std::cout << "Target process running. PID: " << getpid() << std::endl;
    
    // 持续运行，等待信号
    while (true) {
        pause();  // 暂停，等待信号
    }

    return 0;
}

```

python版本：

```python
import signal
import os
import time


# 信号处理函数
def signal_handler(signum, frame):
    if signum == signal.SIGUSR1:
        print("Received SIGUSR1: Executing custom function!")
        # 执行自定义功能，比如更改变量或执行任务


def target_process():
    # 注册 SIGUSR1 信号的处理程序
    signal.signal(signal.SIGUSR1, signal_handler)

    print(f"Target process running. PID: {os.getpid()}")

    # 持续运行，等待信号
    while True:
        time.sleep(1)  # 让进程持续运行，等待信号


if __name__ == "__main__":
    target_process()
```

2. 控制进程：发送信号到目标进程

```c++
#include <iostream>
#include <csignal>
#include <unistd.h>

int main() {
    pid_t target_pid;

    // 获取目标进程的 PID
    std::cout << "Enter target process PID: ";
    std::cin >> target_pid;

    // 发送 SIGUSR1 信号给目标进程
    if (kill(target_pid, SIGUSR1) == 0) {
        std::cout << "Signal SIGUSR1 sent to process " << target_pid << std::endl;
    } else {
        std::cerr << "Failed to send signal to process " << target_pid << std::endl;
    }

    return 0;
}

```

python版本：

```python
import signal
import os
import time


# 信号处理函数
def signal_handler(signum, frame):
    if signum == signal.SIGUSR1:
        print("Received SIGUSR1: Executing custom function!")
        # 执行自定义功能，比如更改变量或执行任务


def target_process():
    # 注册 SIGUSR1 信号的处理程序
    signal.signal(signal.SIGUSR1, signal_handler)

    print(f"Target process running. PID: {os.getpid()}")

    # 持续运行，等待信号
    while True:
        time.sleep(1)  # 让进程持续运行，等待信号


if __name__ == "__main__":
    target_process()

```



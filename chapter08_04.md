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


关于errno的延伸：

> #### 什么是errno
>
> `errno` 是一个全局变量，定义在 `<errno.h>` 头文件中，用于表示系统调用和库函数在执行过程中出现的错误。它用于存储最近一次发生的错误的错误码。通过检查 `errno`，程序可以获知导致函数失败的具体原因。不同的函数会根据其执行过程中的不同错误情况，设置不同的 `errno` 值。
>
> #### 常见的 `errno` 错误码：
>
> - `EPERM`：操作不允许
> - `ENOENT`：没有这样的文件或目录
> - `ESRCH`：没有这样的进程
> - `EIO`：输入/输出错误
> - `ENOMEM`：内存不足
> - `EAGAIN`：资源暂时不可用（如 `EWOULDBLOCK`）
> - `EINVAL`：无效参数
>
> #### 为什么 `errno` 是一个全局变量？
>
> `errno` 是一个全局变量，并且它的设计具有一些特定的原因和历史背景。理解这些原因，可以帮助你更好地理解它的工作原理。
>
> #### 1. **系统调用和库函数的错误报告机制**
>
> 许多系统调用和库函数在执行失败时都会设置 `errno`。然而，由于这些函数通常会返回错误码（例如 -1 或 NULL），程序员通常需要检查这个返回值并通过 `errno` 获取更详细的错误信息。如果 `errno` 是局部变量，那么在每个函数中都需要传递错误状态，这将增加不必要的复杂性。使用全局变量 `errno` 使得错误信息可以由操作系统在系统调用之间传递，不需要通过函数的返回值传递。
>
> #### 2. **保证线程安全性（针对多线程环境）**
>
> 尽管 `errno` 是一个全局变量，但它通常在多线程程序中是**线程本地存储**的。这意味着每个线程都有自己独立的 `errno`，从而避免了线程之间对 `errno` 的竞争。在 POSIX 标准中，`errno` 被设计成线程局部存储（Thread-Local Storage, TLS），每个线程都有一份独立的 `errno`，不会被其他线程影响。
>
> 在 Linux 和其他支持 POSIX 的操作系统中，`errno` 并不是全局的，而是每个线程独立的。使用线程局部存储可以保证不同线程之间不会互相干扰，并且每个线程都能保持其独立的错误状态。
>
> #### 3. **操作系统设计的简便性**
>
> 全局 `errno` 的设计使得操作系统和标准库的实现更加简便。许多系统调用直接将错误码写入 `errno`，而不需要通过复杂的机制将错误信息从系统调用传递到应用程序。这样的设计使得库函数和系统调用的错误处理变得简单高效。
>
> #### 4. **历史遗留原因**
>
> `errno` 的设计始于 Unix 操作系统，最初的 Unix 系统是单线程的。在单线程环境下，全局变量是一个简单有效的错误报告机制。随着多线程编程的出现，`errno` 被调整为线程局部存储（TLS），以适应多线程环境的需求。
>
> **在单线程程序中**：可以直接使用 `errno`，它仍然是一个全局变量。
>
> **在多线程程序中**：每个线程都有自己的 `errno` 副本，线程间互不干扰，因此你可以在每个线程中直接使用 `errno`。
>
> **手动保存和恢复 `errno`**：在某些情况下，如果你希望显式管理 `errno`，可以保存和恢复它，但通常在多线程环境下这不是必要的，因为每个线程都有独立的 `errno` 副本。
>
> ```c
> #include <stdio.h>
> #include <pthread.h>
> #include <errno.h>
> #include <string.h>
> #include <unistd.h>
> 
> void* thread_func(void* arg) {
>     int saved_errno = errno; // 保存当前线程的 errno
> 
>     // 线程中调用系统函数，可能会设置 errno
>     if (access("nonexistent_file.txt", F_OK) == -1) {
>         // 在每个线程中独立访问 errno
>         printf("Thread %ld: Error - %s\n", (long)arg, strerror(errno));
>     }
> 
>     // 恢复原始 errno 值
>     errno = saved_errno;
>     return NULL;
> }
> 
> int main() {
>     pthread_t threads[3];
> 
>     // 创建多个线程
>     for (long i = 0; i < 3; i++) {
>         if (pthread_create(&threads[i], NULL, thread_func, (void*)i) != 0) {
>             perror("pthread_create");
>             return 1;
>         }
>     }
> 
>     // 等待所有线程结束
>     for (int i = 0; i < 3; i++) {
>         pthread_join(threads[i], NULL);
>     }
> 
>     return 0;
> }
> 
> ```
>
> 
## 8.4 进程控制

### 8.4.1 获取进程 ID（getpid）

每个进程都有一个唯一的正数（非零）进程 ID（PID）。getpid 函数返回调用进程的 PID。getppid 函数返回它的父进程的 PID（创建调用进程的进程）。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);

// 返回：调用者或其父进程的 PID。
```

> getpid 和 getppid 函数返回一个类型为 pid_t 的整数值，在 Linux 系统上它在 types.h 中被定义为 int。

### 8.4.2 创建和终止进程（fork）

从程序员的角度，我们可以认为进程总是处于下面三种状态之一：

- **运行。**进程要么在 CPU 上 执行，要么在等待被执行且最终会被内核调度。
- **停止。**进程的执行被挂起（suspended），且不会被调度。当收到 SIGSTOP、SIGTSTP、SIGTTIN 或者 SIGTTOU 信号时，进程就停止，并且保持停止直到它收到一个 SIGCONT 信号，在这个时刻，进程再次开始运行。（信号是一种软件中断的形式，将在 8.5 节中详细描述。）
- **终止。**进程永远地停止了。进程会因为三种原因终止：
  - 1）收到一个信号，该信号的默认行为是终止进程；
  - 2）从主程序返回；
  - 3）调用 exit 函数。

```c
#include <stdlib.h>

void exit(int status);

// 该函数不返回。
```

> exit 函数以 status 退出状态来终止进程（另一种设置退出状态的方法是从主程序中返回一个整数值）。

父进程通过调用 fork 函数创建一个新的运行的子进程。

```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);

// 返回：子进程返回 0，父进程返回子进程的 PID，如果出错，则为 -1。
```

新创建的子进程几乎但不完全与父进程相同。父进程和新创建的子进程之间最大的区别在于它们有不同的 PID。

> 子进程得到与父进程用户级虚拟地址空间相同的（但是独立的）一份副本，包括代码和数据段、堆、共享库以及用户栈。
>
> 子进程还获得与父进程任何打开文件描述符相同的副本，这就意味着当父进程调用 fork 时，子进程可以读写父进程中打开的任何文件。

fork 函数是有趣的（也常常令人迷惑），因为它只被调用一次，却会返回两次：一次是在调用进程（父进程）中，一次是在新创建的子进程中。**<u>返回值就提供一个明确的方法来分辨程序是在父进程还是在子进程中执行。</u>**

- 在父进程中，fork 返回子进程的 PID。
- 在子进程中，fork 返回 0。因为子进程的 PID 总是为非零

```c
int main()
{
    pid_t pid;
    int x = 1;

    pid = Fork();
    if (pid == 0) { /* Child */
        printf("child : x=%d\n", ++x);
        exit(0);
    }
    
    /* Parent */
    printf("parent: x=%d\n", --x);
    exit(0);
}
```

当在 Unix 系统上运行这个程序时，我们得到下面的结果：

````
linux> ./fork
parent：x=0
child ：x=2
````

> 这个简单的例子有一些微妙的方面。
>
> - **调用一次，返回两次。**fork 函数被父进程调用一次，但是却返回两次。一次是返回到父进程，一次是返回到新创建的子进程。
> - **并发执行。**父进程和子进程是并发运行的独立进程。内核能够以任意方式交替执行它们的逻辑控制流中的指令。
> - **相同但是独立的地址空间。**如果能够在 fork 函数在父进程和子进程中返回后立即暂停这两个进程，我们会看到两个进程的地址空间都是相同的。然而，因为父进程和子进程是独立的进程，它们都有自己的私有地址空间。父进程和子进程对 x 所做的任何改变都是独立的，不会反映在另一个进程的内存中。这就是为什么当父进程和子进程调用它们各自的 printf 语句时，它们中的变量 x 会有不同的值。
> - **共享文件。**当运行这个示例程序时，我们注意到父进程和子进程都把它们的输出显示在屏幕上。原因是子进程继承了父进程所有的打开文件。当父进程调用 fork 时，stdout 文件是打开的，并指向屏幕。子进程继承了这个文件，因此它的输出也是指向屏幕的。

如果你是第一次学习 fork 函数，画进程图通常会有所帮助，进程图是刻画程序语句的偏序的一种简单的前趋图。

- 每个顶点 a 对应于一条程序语句的执行。
- 有向边 a → b 表示语句 a 发生在语句 b 之前。边上可以标记出一些信息，例如一个变量的当前值。对应于 printf 语句的顶点可以标记上 printf 的输出。
- 每张图从一个顶点开始，对应于调用 main 的父进程。这个顶点没有入边，并且只有一个出边。
- 每个进程的顶点序列结束于一个对应于 exit 调用的顶点。这个顶点只有一条入边，没有出边

图 8-16 展示了图 8-15 中示例程序的进程图。初始时，父进程将变量 x 设置为 1。父进程调用 fork，创建一个子进程，它在自己的私有地址空间中与父进程并发执行。

![图 8-16   图 8-15 中示例程序的进程图](https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-16%20%E5%9B%BE8-15%E4%B8%AD%E7%A4%BA%E4%BE%8B%E7%A8%8B%E5%BA%8F%E7%9A%84%E8%BF%9B%E7%A8%8B%E5%9B%BE.png)



对于运行在单处理器上的程序，对应进程图中所有顶点的**拓扑排序**（topological sort）表示程序中语句的一个可行的全序排列。

> 下面是一个理解拓扑排序概念的简单方法：给定进程图中顶点的一个排列，把顶点序列从左到右写成一行，然后画出每条有向边。排列是一个拓扑排序，当且仅当画出的每条边的方向都是从左往右的。因此，在图 8-15 的示例程序中，父进程和子进程的 printf 语句可以以任意先后顺序执行，因为每种顺序都对应于图顶点的某种拓扑排序。

进程图特别有助于理解带有嵌套 fork 调用的程序。例如，图 8-17 中的程序源码中两次调用了 fork。对应的进程图可帮助我们看清这个程序运行了四个进程，每个都调用了—次 printf，这些 printf 可以以任意顺序执行。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/08-17%20%E5%B5%8C%E5%A5%97fork%E7%9A%84%E8%BF%9B%E7%A8%8B%E5%9B%BE.png" alt="图 8-17 嵌套 fork 的进程图" style="zoom:67%;" />

#### 练习题 8.2

考虑下面的程序：

```c
int main()
{
    int x = 1;
    
    if (Fork() == 0)
        printf("p1: x=%d\n", ++x);
    printf("p2: x=%d\n", --x);
    exit(0);
}
```

A. 子进程的输出是什么？

![图 8-47 练习题 8.2 的进程图](https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0847-lian-xi-ti-8.2-de-jin-cheng-tu-.png)

> p1: x=2
> p2: x=1

B. 父进程的输出是什么？

```c
p2: x=0
```

### 8.4.3 回收子进程（waitpid）

**当一个进程由于某种原因终止时，内核并不是立即把它从系统中清除。相反，进程被保持在一种已终止的状态中，直到被它的父进程回收（reaped）。**

当父进程回收已终止的子进程时，内核将子进程的退出状态传递给父进程，然后抛弃已终止的进程，从此时开始，该进程就不存在了。

一个终止了但还未被回收的进程称为**僵死进程**（zombie）。

如果一个父进程终止了，内核会安排 init 进程成为它的孤儿进程的养父。init 进程的 PID 为 1，是在系统启动时由内核创建的，它不会终止，是所有进程的祖先。如果父进程没有回收它的僵死子进程就终止了，那么内核会安排 init 进程去回收它们。不过，长时间运行的程序，比如 shell 或者服务器，总是应该回收它们的僵死子进程。即使僵死子进程没有运行，它们仍然消耗系统的内存资源。

**一个进程可以通过调用 waitpid 函数来等待它的子进程终止或者停止。**

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *statusp, int options);

// 返回：如果成功，则为子进程的 PID，如果 WNOHANG，则为 0，如果其他错误，则为 -1。
```

参数解析：

1. 第一个参数pid：一个 进程ID (PID)，指定父进程要等待的子进程。
   - 如果它是 `-1`，表示父进程等待 **任何一个子进程** 终止，也就是父进程不关心哪个子进程结束，只要有子进程结束就会返回。
   - 如果是具体的进程 ID，比如 `pid`，则父进程只会等待该进程终止。
2. **第二个参数：`statusp`**
   - 这是一个指向 **状态信息** 的指针，用来存储子进程的退出状态。如果你不关心子进程的退出状态，可以将它设置为 `NULL`，即不收集退出状态。
   - 如果你希望获取子进程的退出码（比如 `exit()` 返回值或者信号），你可以传递一个指针（如 `int status`）来保存返回的状态信息。wait.h 头文件定义了解释 status 参数的几个宏：
     - **WIFEXITED**(status)：如果于进程通过调用 exit 或者一个返回（return）正常终止，就返回真。
     - **WEXITSTATUS**(status)：返回一令正常终止的子进程的退出状态。只有在 WIFEXITED() 返回为真时，才会定义这个状态。
     - **WIFSIGNALED**(status)：如果子进程是因为一个未被捕获的信号终止的，. 那么就返回真。
     - **WTERMSIG**(status)：返回导致子进程终止的信号的编号。只有在 WIFSIGNALED() 返回为真时，才定义这个状态。
     - **WIFSTOPPED**(status)：如果引起返回的子进程当前是停止的，那么就返回真。
     - **WSTOPSIG**(status)：返回引起子进程停止的信号的编号。只有在 WIFSTOPPED() 返回为真时，才定义这个状态。
     - **WIFCONTINUED**(status)：如果子进程收到 SIGCONT 信号重新启动，则返回真。
3. **第三个参数：`options`**
   - 这是 **选项** 参数，通常设置为 `0`，表示 `waitpid` 会按照默认行为执行。
   - 还可以使用其他选项，如：
     - `WNOHANG`：如果没有子进程终止，立即返回，而不会阻塞。waitpid默认是阻塞式的等待子进程终止的。这个参数可以取消阻塞状态。让我们可以尝试多次轮询的方式来调用waitpid
     - `WUNTRACED`：如果子进程因信号暂停（例如通过 `kill` 或者 `Ctrl+Z`），父进程也会收到通知。默认的行为是只返回已终止的子进程。当你想要检査已终止和被停止的子进程时，这个选项会有用。
     - `WCONTINUED`：挂起调用进程的执行，直到等待集合中一个正在运行的进程终止或等待集合中一个被停止的进程收到 SIGCONT 信号重新开始执行
   - 可以用或运算把这些选项组合起来。例如：
     - **WNOHANG | WUNTRACED：**立即返回，如果等待集合中的子进程都没有被停止或终止，则返回值为 0；如果有一个停止或终止，则返回值为该子进程的 PID。

`waitpid` 的返回值：

- `waitpid` 的返回值是终止的子进程的进程 ID。
  - 如果调用成功且有子进程终止，它返回终止子进程的 PID。
  - 如果调用成功但没有子进程终止，且 `WNOHANG` 被指定，它将返回 0。
  - 如果调用进程没有子进程，它会返回 -1，并设置 `errno` 为 `ECHILD`。如果 waitpid 函数被一个信号中断，那么它返回 -1，并设置 `errno `为 `EINTR`。

> 每个 Unix 函数的 man 页列出了无论何时你在代码中使用那个函数都要包含的头文件。同时，为了检查诸如 ECHILD 和 EINTR 之类的返回代码，你必须包含 errno.h。为了简化代码示例，我们包含了一个称为 csapp.h 的头文件，它包括了本书中使用的所有函数的头文件。csapp.h 头文件可以从 CS：APP 网站在线获得。

#### 练习题 8.3

列出下面程序所有可能的输出序列：

```c
int main()
{
    if (Fork() == 0) {
        printf("a"); fflush(stdout);
    }
    else {
        printf("b"); fflush(stdout);
        waitpid(-1, NULL, 0);
    }
    printf("c"); fflush(stdout);
    exit(0);
 }
```

> ![图 8-48 练习题 8.3 的进程图](https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0848-lian-xi-ti-8.3-de-jin-cheng-tu-.png)
>
> 序列 acbc、abcc 和 bacc 是可能的，因为它们对应有进程图的拓扑排序（图 8-48）。而像 bcac 和 cbca 这样的序列不对应有任何拓扑排序，因此它们是不可行的。

#### wait 函数

wait 函数是 waitpid 函数的简单版本：

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *statusp);
// 返回：如果成功，则为子进程的 PID，如果出错，则为 -1。
```

调用 **wait(&status)** 等价于调用 **waitpid(-1, &status, 0)**。

#### 使用 waitpid 的示例

```c
#include "csapp.h"
#define N 2

int main()
{
    int status, i;
    pid_t pid;

    /* Parent creates N children */
    for (i = 0; i < N; i++)
        if ((pid = Fork()) == 0) /* Child */
            exit(100+i);

    /* Parent reaps N children in no particular order */
    while ((pid = waitpid(-1, &status, 0)) > 0) {
        if (WIFEXITED(status))
            printf("child %d terminated normally with exit status=%d\n",
                   pid, WEXITSTATUS(status));
        else
            printf("child %d terminated abnormally\n", pid);
    }

    /* The only normal termination is if there are no more children */
    if (errno != ECHILD)
        unix_error("waitpid error");
   
    exit(0);
}
```

> 在第 15 行，父进程用 waitpid 作为 while 循环的测试条件，等待它所有的子进程终止。因为第一个参数是 -1，所以对 waitpid 的调用会阻塞，直到任意一个子进程终止。

注意，程序不会按照特定的顺序回收子进程。子进程回收的顺序是这台特定的计算机系统的属性。这是非确定性行为。图 8-19 展示了一个简单的改变，它消除了这种不确定性，按照父进程创建子进程的相同顺序来回收这些子进程。

```c
#include "csapp.h"
#define N 2

int main()
{
    int status, i;
    pid_t pid[N], retpid;

    /* Parent creates N children */
    for (i = 0; i < N; i++)
        if ((pid[i] = Fork()) == 0) /* Child */
            exit(100+i);

    /* Parent reaps N children in order */
    i = 0;
    while ((retpid = waitpid(pid[i++], &status, 0)) > 0) {
        if (WIFEXITED(status))
            printf("child %d terminated normally with exit status=%d\n",
                   retpid, WEXITSTATUS(status));
    else
        printf("child %d terminated abnormally\n", retpid);
    }

    /* The only normal termination is if there are no more children */
    if (errno != ECHILD)
        unix_error("waitpid error");
    
    exit(0);
}
```

#### 练习题 8.4

考虑下面的程序：

```c
int main()
{
    int status;
    pid_t pid;
    
    printf("Hello\n");
    pid = Fork();
    printf("%d\n", !pid);
    if (pid != 0) {
        if (waitpid(-1, &status, 0) > 0) {
            if (WIFEXITED(status) != 0)
                printf("%d\n", WEXITSTATUS(status));
        }
    }
    printf("Bye\n");
    exit(2);
}
```

A. 这个程序会产生多少输出行？

> 进程图：
>
> ![图 8-49 练习题 8.4 的进程图](https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0849-lian-xi-ti-8.4-de-jin-cheng-tu-.png)
>
>  只简单地计算进程图（图 8-49）中 printf 顶点的个数就能确定输出行数。在这里，有 6 个这样的顶点，因此程序会打印 6 行输出。

B. 这些输出行的一种可能的顺序是什么？

> 任何对应有进程图的拓扑排序的输出序列都是可能的。例如：Hello、1、0、Bye、2、Bye 是可能的。
### 8.4.4 让进程休眠

sleep函数将一个进程挂起一段指定的时间。

```c
#include <unistd.h>
unsigned int sleep(unsigned int secs);

// 返回：还要休眠的秒数。
```

> 如果请求的时间量已经到了，sleep 返回 0，如果因为 sleep 函数被一个信号中断而过早地返回，则返回还剩下的要休眠的秒数。

另一个很有用的函数是 pause 函数，**<u>该函数让调用函数休眠，直到该进程收到一个信号。</u>**

```c
#include <unistd.h>
int pause(void);

// 总是返回 -1。
```

#### 练习题 8.5

编写一个 sleep 的包装函数，叫做 snooze，带有下面的接口：

```c
unsigned int snooze(unsigned int secs);
```

snooze 函数和 sleep 函数的行为完全一样，除了它会打印出一条消息来描述进程实际休眠了多长时间：

```c
Slept for 4 of 5 secs.
```

> ```c
> unsigned int snooze(unsigned int sec){
>     unsigned int rc = sleep(secs);
>     printf("Sleot for %d of %d secs.",secs-rc,secs);
>     return rc;
> }
> ```

### 8.4.5 加载并运行程序（execve）

execve 函数在当前进程的上下文中加载并运行一个新程序。

```c
#include <unistd.h>
int execve(const char *filename, const char *argv[],
           const char *envp[]);

// 如果成功，则不返回，如果错误，则返回 -1。
```

execve 函数加载并运行可执行目标文件 filename，且带参数列表 argv 和环境变量列表 envp。只有当出现错误时，例如找不到 filename，execve 才会返回到调用程序。所以，与 fork—次调用返回两次不同，execve 调用一次并从不返回。

参数列表是用图 8-20 中的数据结构表示的。argv 变量指向一个以 null 结尾的指针数组，其中每个指针都指向一个参数字符串。按照惯例，argv[0] 是可执行目标文件的名字。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0820-can-shu-lie-biao-de-zu-zhi-jie-gou-.png" alt="图 8-20 参数列表的组织结构" style="zoom:67%;" />

环境变量的列表是由一个类似的数据结构表示的，如图 8-21 所示。envp 变量指向一个以 null 结尾的指针数组，其中每个指针指向一个环境变量字符串，每个串都是形如 “name=value” 的名字—值对。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0821-huan-jing-bian-liang-lie-biao-de-zu-zhi-jie-gou-.png" alt="图 8-21 环境变量列表的组织结构" style="zoom:67%;" />

在 execve 加载了 filename 之后，它调用 7.9 节中描述的启动代码。启动代码设置栈，并将控制传递给新程序的主函数，该主函数有如下形式的原型

```c
int main(int argc, char **argv, char **envp);
```

或者等价的

```c
int main(int argc, char *argv[], char *envp[]);
```

main 函数有 3 个参数：

1. argc，它给出 argv[ ] 数组中非空指针的数量；
2. argv，指向 argv[ ] 数组中的第一个条目；
3. envp，指向 envp[ ] 数组中的第一个条目。

当 main 开始执行时，用户栈的组织结构如图 8-22 所示。

<img src="https://github.com/wangfud/CSAPP_note_0611/raw/master/.gitbook/assets/0822-yi-ge-xin-cheng-xu-kai-shi-shi-yong-hu-zhan-de-dian-xing-zu-zhi-jie-gou-.png" alt="图 8-22 一个新程序开始时，用户栈的典型组织结构" style="zoom:50%;" />

在栈的顶部是系统启动函数 libc_start_main（见 7.9 节）的栈帧。

Linux 提供了几个函数来操作环境数组：

```c
#include <stdlib.h>
char *getenv(const char *name);

// 返回：若存在则为指向 name 的指针，若无匹配的，则为 NULL。
//getenv 函数在环境数组中搜索字符串 “name=value”。如果找到了，它就返回一个指向 value 的指针，否则它就返回 NULL。


//如果环境数组包含一个形如 “name=oldva1ue” 的字符串，那么 unsetenv 会删除它，而 setenv 会用 newvalue 代替 oldvalue，但是只有在 overwirte 非零时才会这样。如果 name 不存在，那么 setenv 就把 “name=newvalue” 添加到数组中。
int setenv(const char *name, const char *newvalue, int overwrite);
// 返回：若成功则为 0，若错误则为 -1。

void unsetenv(const char *name);
// 返回：无。

```









WUNTRACED 是 Unix/Linux 系统中 waitpid() 系统调用的一个选项常量，用于让父进程能够接收到子进程被暂停（stopped）时的通知。

> 当子进程被 信号（如 SIGSTOP）暂停，默认情况下 waitpid() 并不会返回。但如果你传入了 WUNTRACED，则可以捕捉这个暂停事件。 

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <signal.h>

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        // 子进程休眠，然后被手动暂停（例如用 kill -STOP <pid>）
        while (1) pause();  // 一直等待信号
    } else {
        int status;
        printf("父进程等待子进程暂停...\n");

        // 等待子进程被暂停（例如被 SIGSTOP 信号暂停）
        waitpid(pid, &status, WUNTRACED);

        if (WIFSTOPPED(status)) {
            printf("子进程已被暂停，信号编号: %d\n", WSTOPSIG(status));
        }
    }

    return 0;
}

```

### 8.4.6 利用 fork 和 execve 运行程序

图 8-23 展示了一个简单 shell 的 main 例程。shell 打印一个命令行提示符，等待用户在 stdin 上 输入命令行，然后对这个命令行求值。

```c
#include "csapp.h"
#define MAXARGS 128

/* Function prototypes */
void eval(char *cmdline);
int parseline(char *buf, char **argv);
int builtin_command(char **argv);

int main()
{
    char cmdline[MAXLINE]; /* Command line */

    while (1) {
        /* Read */
        printf("> ");
        Fgets(cmdline, MAXLINE, stdin);
        if (feof(stdin))
            exit(0);

        /* Evaluate */
        eval(cmdline);
    }
}
```

图 8-24 展示了对命令行求值的代码。

```c
/* eval - Evaluate a command line */
void eval(char *cmdline)
{
    char *argv[MAXARGS]; /* Argument list execve() */
    char buf[MAXLINE];   /* Holds modified command line */
    int bg;              /* Should the job run in bg or fg? */
    pid_t pid;           /* Process id */

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);
    if (argv[0] == NULL)
        return;   /* Ignore empty lines */

    if (!builtin_command(argv)) {
        if ((pid = Fork()) == 0) {   /* Child runs user job */
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found.\n", argv[0]);
                exit(0);
            }
        }

        /* Parent waits for foreground job to terminate */
        if (!bg) {
            int status;
            if (waitpid(pid, &status, 0) < 0)
                unix_error("waitfg: waitpid error");
            }
        else
            printf("%d %s", pid, cmdline);
    }
    return;
}

/* If first arg is a builtin command, run it and return true */
int builtin_command(char **argv)
{
    if (!strcmp(argv[0], "quit")) /* quit command */
        exit(0);
    if (!strcmp(argv[0], "&"))    /* Ignore singleton & */
        return 1;
    return 0;                     /* Not a builtin command */
}
```

图 8-25 parseline 解析 shell 的一个输入行

```c
/* parseline - Parse the command line and build the argv array */
int parseline(char *buf, char **argv)
{
    char *delim;         /* Points to first space delimiter */
    int argc;            /* Number of args */
    int bg;              /* Background job? */

    buf[strlen(buf)-1] = ' ';  /* Replace trailing '\n' with space */
    while (*buf && (*buf == ' ')) /* Ignore leading spaces */
        buf++;

    /* Build the argv list */
    argc = 0;
    while ((delim = strchr(buf, ' '))) {
        argv[argc++] = buf;
        *delim = '\0';
        buf = delim + 1;
        while (*buf && (*buf == ' ')) /* Ignore spaces */
            buf++;
    }
    argv[argc] = NULL;

    if (argc == 0) /* Ignore blank line */
        return 1;

    /* Should the job run in the background? */
    if ((bg = (*argv[argc-1] == '&')) != 0)
        argv[--argc] = NULL;

    return bg;
}
```

如果最后一个参数是一个 “&” 字符，那么 parseline 返回 1，表示应该在后台执行该程序（shell 不会等待它完成）。否则，它返回 0，表示应该在前台执行这个程序（shell 会等待它完成）

在解析了命令行之后，eval 函数调用 builtin_command 函数，该函数检查第一个命令行参数是否是一个内置的 shell 命令。如果是，它就立即解释这个命令，并返回值 1。否则返回 0。简单的 shell 只有一个内置命令—— quit 命令，该命令会终止 shell。实际使用的 shell 有大量的命令，比如 pwd、jobs 和 fg。

如果 builtin_cornnand 返回 0，那么 shell 创建一个子进程，并在子进程中执行所请求的程序。如果用户要求在后台运行该程序，那么 shell 返回到循环的顶部，等待下一个命令行。否则，shell 使用 waitpid 函数等待作业终止。当作业终止时，shell 就开始下一轮迭代。


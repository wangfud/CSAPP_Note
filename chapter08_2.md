
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


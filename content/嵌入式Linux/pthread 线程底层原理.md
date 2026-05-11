# 1 不懂 pthread 线程底层，别说你会 Linux多线程开发


不少 Linux 后台开发者日常频繁使用多线程编程，熟练掌握 pthread 各类 API 的调用方式，能够完成基础的并发业务开发。但绝大多数人仅停留在表层使用层面，对 pthread 线程的底层运行逻辑一无所知。一旦线上遭遇线程僵死、资源泄露、随机死锁、并发调度异常等疑难问题，只能盲目调试、无从根治，这也是多数人多线程开发能力停滞不前的核心原因。

API 调用只是表象，底层原理才是 Linux 多线程开发的核心精髓。不理解线程创建、分离、join 的内核逻辑，看不懂互斥锁、条件变量的底层实现，就无法写出稳定、高效、高并发的工业级代码。本文摒弃碎片化的基础用法讲解，深度拆解 pthread 线程核心底层原理，帮你打通 Linux 多线程知识闭环，彻底摆脱只会套代码、不懂底层逻辑的开发短板。

## 1.1 一、认识 pthread 线程


### 1.1.1 线程与进程的区别

在操作系统的世界里，进程是资源分配的基本单元 。每一个进程都拥有独立的内存空间、文件描述符表、环境变量等一系列资源。就好比一个独立的小王国，有着自己独立的一套 “基础设施”。当你启动一个程序，比如打开一个浏览器，操作系统就会为这个浏览器创建一个进程，这个进程会拥有自己独立的内存空间，用于存储浏览器的各种数据，像网页缓存、用户浏览记录等，它与其他进程相互隔离，互不干扰，一个进程的崩溃不会影响到其他进程的正常运行。

而线程则是进程中的执行单元，是 CPU 调度的最小单位 。一个进程可以包含多个线程，这些线程共享进程的资源，像是内存空间、打开的文件描述符等。举个形象的例子，如果把进程比作是一个工厂，那么线程就像是工厂里的工人，所有工人共享工厂的场地、设备等资源，但每个工人又有自己独立的工作流程和任务。线程的创建和切换开销比进程要小得多，这使得线程在处理并发任务时更加高效。但由于线程共享进程资源，一旦某个线程出现错误，比如访问了非法内存地址，就可能导致整个进程崩溃 。

### 1.1.2 什么是 pthread 库？

pthread 库是 POSIX 线程标准（Portable Operating System Interface for Threads）在 Linux 等类 Unix 系统上的具体实现，为开发者提供了一套操作线程的 API，是 C/C++ 多线程编程中极为常用的库。POSIX 标准致力于为不同操作系统提供统一的 API 规范，确保应用程序能在遵循该标准的系统间具有良好的可移植性 。

在使用 pthread 库时，我们需要在代码中包含<pthread.h>头文件，它声明了 pthread 库中各种函数、类型和常量，让编译器知晓我们使用的函数和数据结构。例如：

```
#include <pthread.h>
```

编译包含 pthread 库函数的代码时，要链接-lpthread 库，它包含了 pthread 库的实现代码，告诉链接器去查找并链接这些代码。在命令行中使用 gcc 编译时，通过-pthread 选项指定链接 pthread 库，如：

```
gcc -o my_program my_program.c -pthread
```

这样，我们就能顺利使用 pthread 库提供的功能，开启多线程编程之旅。

## 1.2 二、pthread API 详解

面试题写作模版

pthread 库提供了一系列丰富且强大的 API，这些 API 是我们进行多线程编程的得力工具，涵盖了线程从诞生到消亡的整个生命周期，以及线程间同步协作等关键方面 。熟练掌握这些 API 的使用方法和内在原理，是编写出高效、稳定多线程程序的基础 。下面，我们就来深入探究 pthread 库中那些核心 API。

### 1.2.1 线程创建：pthread\_create

在 pthread 库中，pthread\_create 函数用于创建一个新线程 。它的函数原型如下：

```
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);
```

*   thread：是一个指向 pthread\_t 类型变量的指针，用于存储新创建线程的标识符，就像是新线程的 “身份证”，通过这个标识符可以对线程进行后续的操作，比如等待线程结束、取消线程等 。
    
*   attr：用于设置线程的属性，例如线程的栈大小、调度策略、分离状态等 。如果传入 NULL，则表示使用默认属性 。比如，默认情况下线程的栈大小是由系统设定的一个合适值，若不进行特殊设置，就采用这个默认栈大小 。
    
*   start\_routine：是一个函数指针，指向线程启动后要执行的函数，这个函数是线程的执行体，线程创建成功后就会从这个函数开始执行代码 。
    
*   arg：是传递给start\_routine函数的参数 ，它是一个void\*类型的指针，因此可以传递任何类型的数据指针，但在使用时需要进行类型转换 。
    

该函数成功创建线程时返回 0，失败时返回错误码，错误码表示线程创建失败的具体原因，如 EAGAIN 表示系统资源不足，无法创建新线程；EINVAL 表示传入的参数无效 。下面通过一个简单的示例代码来展示 pthread\_create 的用法：

```
#include <stdio.h>#include <pthread.h>// 线程执行函数void* thread_function(void* arg) {    int num = *(int*)arg;    printf("子线程执行，参数为: %d\n", num);    return NULL;}int main() {    pthread_t tid;    int arg = 10;    int ret = pthread_create(&tid, NULL, thread_function, &arg);    if (ret != 0) {        printf("线程创建失败: %s\n", strerror(ret));        return1;    }    printf("主线程继续执行\n");    // 等待子线程结束    pthread_join(tid, NULL);    return0;}
```

在上述代码中，首先定义了 thread\_function 作为线程执行函数，它接收一个 void\*类型的参数，并将其转换为 int 类型后打印 。在 main 函数中，使用 pthread\_create 创建线程，将变量 arg 的地址作为参数传递给线程函数 。创建线程后，检查返回值以确保线程创建成功 。这里设置线程属性为 NULL，表示使用默认属性 。最后通过 pthread\_join 等待子线程结束 。

### 1.2.2 线程等待：pthread\_join

pthread\_join 函数的功能是阻塞当前线程，直到指定的线程结束，并获取指定线程的退出状态 。它的函数原型为：

```
int pthread_join(pthread_t thread, void **retval);
```

*   thread：指定要等待的线程标识符，通过这个标识符来确定等待哪个线程结束 。
    
*   retval：是一个指向指针的指针，用于获取被等待线程的返回值 。如果不关心线程的返回值，可以将其设置为 NULL 。
    

该函数成功时返回 0，失败时返回错误码，比如 ESRCH 表示没有找到对应的线程；EINVAL 表示传入的线程是一个分离状态的线程，不能被等待 。下面通过示例展示 pthread\_join 的使用方法：

```
#include <stdio.h>#include <pthread.h>#include <stdlib.h>// 线程执行函数void* thread_function(void* arg) {    int* result = (int*)malloc(sizeof(int));    *result = 42;    return (void*)result;}int main() {    pthread_t tid;    void* thread_result;    int ret = pthread_create(&tid, NULL, thread_function, NULL);    if (ret != 0) {        printf("线程创建失败: %s\n", strerror(ret));        return1;    }    // 等待线程结束并获取返回值    ret = pthread_join(tid, &thread_result);    if (ret != 0) {        printf("等待线程失败: %s\n", strerror(ret));        return1;    }    int* result = (int*)thread_result;    printf("子线程返回值: %d\n", *result);    free(result);    return0;}
```

在这个示例中，thread\_function 函数在堆上分配内存存储返回值，然后返回 。在 main 函数中，使用 pthread\_join 等待子线程结束，并获取其返回值 。获取返回值后，将其转换为 int\*类型并打印，最后释放分配的内存 。如果在等待过程中出现错误，会打印错误信息 。

### 1.2.3 线程退出：pthread\_exit

pthread\_exit 函数用于线程主动退出 。它的函数原型是：

```c
void pthread_exit(void *retval);
```

retval：是线程的返回值，这个返回值可以被 pthread\_join 获取，用于传递线程执行的结果等信息 。如果线程不需要返回值，可以将其设置为 NULL 。

当线程调用 pthread\_exit 时，它会立即停止执行，并释放线程栈等相关资源，但不会释放线程所占用的堆内存等资源，这些需要开发者手动管理 。例如，如果线程在执行过程中分配了堆内存，在调用 pthread\_exit 之前，应该确保释放这些内存，否则会导致内存泄漏 。需要注意的是，不要返回指向栈上变量的指针作为 retval，因为线程退出后，栈上的变量会被销毁，返回的指针将指向一块已无效的内存区域，这会导致未定义行为 。比如下面这种情况就是错误的示范：

```c
#include <stdio.h>#include <pthread.h>void* thread_func(void* arg) {int local_var = 10;// 错误做法，返回栈上变量的指针return (void*)&local_var;}int main() {pthread_t tid;void* result;pthread_create(&tid, NULL, thread_func, NULL);pthread_join(tid, &result);// 这里访问 result 指向的内存会导致未定义行为int* value = (int*)result;printf("获取到的值: %d\n", *value);return0;}
```

在 thread\_func 函数中，local\_var 是栈上的局部变量，当线程调用 pthread\_exit 返回其指针后，local\_var 所在的栈内存已被销毁，main 函数中通过 pthread\_join 获取到的指针指向的是一块无效内存，访问这块内存会引发严重错误 。正确的做法是使用堆内存来存储需要返回的数据，或者确保返回的数据在主线程中仍然有效 。

### 1.2.4 线程标识：pthread\_self

pthread\_self 函数用于获取当前线程的标识符 。它的函数原型为：

```
pthread_t pthread_self(void);
```

该函数返回当前线程的 pthread\_t 类型标识符 。在多线程程序中，通过 pthread\_self 可以方便地区分不同的线程，进行线程特定的操作，比如在日志记录中添加线程标识符，以便追踪每个线程的执行情况 。下面通过一个示例来说明 pthread\_self 的作用：

```
#include <stdio.h>#include <pthread.h>void* thread_function(void* arg) {    pthread_t self = pthread_self();    printf("子线程 ID: %lu\n", (unsigned long)self);    return NULL;}int main() {    pthread_t tid;    pthread_create(&tid, NULL, thread_function, NULL);    pthread_t main_self = pthread_self();    printf("主线程 ID: %lu\n", (unsigned long)main_self);    pthread_join(tid, NULL);    return0;}
```

在这个示例中，thread\_function 函数和 main 函数分别使用 pthread\_self 获取自己的线程标识符并打印 。通过输出的线程 ID，可以清晰地看到主线程和子线程的区别，这在调试和分析多线程程序的执行流程时非常有用 。

### 1.2.5 线程分离：pthread\_detach

pthread\_detach 函数的作用是将指定线程设置为分离状态 。处于分离状态的线程在结束时会自动释放所有资源，无需其他线程调用 pthread\_join 来等待它结束 。它的函数原型是：

```
int pthread_detach(pthread_t thread);
```

thread：指定要设置为分离状态的线程标识符 。

该函数成功时返回 0，失败时返回错误码 。pthread\_detach 与 pthread\_join 是两种不同的线程资源管理方式 。pthread\_join 需要等待线程结束并获取其返回值，适用于需要知道线程执行结果的场景；而 pthread\_detach 适用于那些不需要关心线程执行结果，且希望线程结束后自动释放资源的场景，比如一些后台线程，它们默默执行任务，完成后直接释放资源，不需要主线程进行额外的处理 。例如：

```
#include <stdio.h>#include <pthread.h>void* thread_function(void* arg) {    printf("分离线程执行\n");    return NULL;}int main() {    pthread_t tid;    int ret = pthread_create(&tid, NULL, thread_function, NULL);    if (ret != 0) {        printf("线程创建失败: %s\n", strerror(ret));        return1;    }    // 将线程设置为分离状态    ret = pthread_detach(tid);     if (ret != 0) {        printf("设置线程分离失败: %s\n", strerror(ret));        return1;    }    printf("主线程继续执行\n");    return0;}
```

在上述代码中，创建线程后，使用 pthread\_detach 将线程设置为分离状态，然后主线程继续执行，无需等待子线程结束 。子线程结束时会自动释放资源 。

### 1.2.6 线程取消：pthread\_cancel

pthread\_cancel 函数用于取消指定线程的执行 。它的函数原型是：

```
int pthread_cancel(pthread_t thread);
```

thread：指定要取消的线程标识符 。

该函数成功时返回 0，失败时返回错误码 。线程取消并不是立即生效的，而是有一个取消机制 。当调用 pthread\_cancel 时，系统会向目标线程发送一个取消请求 。线程在运行过程中会检查是否有取消请求，这个检查点被称为取消点 。常见的取消点包括一些系统调用、pthread 库函数等 。

当线程到达取消点时，如果有取消请求，就会根据设置的取消状态和类型来决定如何响应 。取消类型分为两种：延迟取消（默认）和异步取消 。延迟取消是指线程到达取消点时才响应取消请求；异步取消则是线程随时响应取消请求 。例如，在下面的代码中，我们尝试取消一个线程：

```
#include <stdio.h>#include <pthread.h>#include <unistd.h>void* thread_function(void* arg) {    while (1) {        printf("线程正在执行...\n");        sleep(1); // sleep 是一个取消点    }    return NULL;}int main() {    pthread_t tid;    int ret = pthread_create(&tid, NULL, thread_function, NULL);    if (ret != 0) {        printf("线程创建失败: %s\n", strerror(ret));        return1;    }    sleep(3); // 主线程等待 3 秒    // 取消线程    ret = pthread_cancel(tid);     if (ret != 0) {        printf("取消线程失败: %s\n", strerror(ret));        return1;    }    // 等待线程结束    pthread_join(tid, NULL);     printf("线程已被取消\n");    return0;}
```

在这个示例中，thread\_function 函数不断打印信息并睡眠 1 秒，sleep 函数是一个取消点 。主线程创建子线程后，等待 3 秒，然后调用 pthread\_cancel 取消子线程 。子线程在下次执行到sleep 函数时，检测到取消请求，从而响应取消操作，结束线程 。最后主线程通过pthread\_join 等待子线程结束，并打印提示信息 。

## 1.3 三、深入 pthread 底层内核原理

面试题写作模版

### 1.3.1 Linux 线程实现机制

在 Linux 操作系统中，线程的本质是轻量级进程（Light Weight Process，LWP） 。从内核的角度来看，它并没有为线程专门设计一套独立的数据结构，而是复用了进程的内核数据结构，即 task\_struct 。这意味着在 Linux 内核中，进程和线程都被视为一个可调度的任务（task），都由 task\_struct 来进行管理 。这种设计方式的精妙之处在于，它极大地减少了内核的代码量，提高了系统的精简程度和运行效率 。

线程作为轻量级进程，与所属进程共享很多内核数据结构和资源 。例如，同一进程中的多个线程共享虚拟内存空间，这使得它们可以直接访问进程中的全局变量、堆内存等数据 。它们还共享文件系统信息，像当前工作目录、文件描述符表等，这意味着一个线程打开的文件，其他线程也可以直接访问 。共享这些资源使得线程间的通信和数据共享变得高效，但也带来了线程同步的问题，需要开发者使用同步机制来确保数据的一致性 。

线程通过 clone 系统调用创建 。clone系统调用是一个非常强大且灵活的调用，它可以创建进程，也可以创线程 。当创建线程时，会设置一些特定的标志位，来决定新创建的线程与父进程（调用者）共享哪些资源 。例如，设置 CLONE\_VM 标志位表示共享虚拟内存空间，设置 CLONE\_FS 标志位表示共享文件系统信息等 。通过这些标志位的组合，内核能够精确地控制线程与进程之间的资源共享关系 。

### 1.3.2 clone 系统调用剖析

clone 函数是 Linux 系统中用于创建新进程或线程的底层系统调用 ，它的函数原型如下：

```
#define _GNU_SOURCE#include <sched.h>int clone(int (*fn)(void *), void *child_stack, int flags, void *arg, ... /* pid_t *parent_tid, void *tls, pid_t *child_tid */ );
```

*   fn：是一个函数指针，指向新创建的进程或线程要执行的函数，新的执行流从这个函数开始 。
    
*   child\_stack：指定新进程或线程使用的栈空间，栈的生长方向与系统相关，通常栈是向下生长的，所以这里一般传入栈空间的最高地址 。
    
*   flags：是一组标志位，这是 clone 系统调用的关键参数，通过设置不同的标志位，可以控制新创建的进程或线程与父进程之间的资源共享关系 。
    
*   arg：是传递给 fn 函数的参数 。
    
*   后面的可选参数 parent\_tid、tls、child\_tid 分别用于指定父进程获取子进程线程 ID 的位置、线程局部存储（TLS）的位置以及子进程获取自身线程 ID 的位置 。
    

在创建线程时，一些关键的标志位如下：

*   CLONE\_VM：表示共享虚拟内存 ，使得线程与父进程共享同一套地址空间，这样线程可以直接访问父进程的全局变量、堆内存等，极大地提高了线程间数据共享的效率 。
    
*   CLONE\_FS：共享文件系统信息 ，包括当前工作目录、根目录等文件系统相关的状态，这意味着一个线程对文件系统的操作，如改变当前工作目录，其他线程也能感知到 。
    
*   CLONE\_FILES：共享打开的文件描述符表 ，一个线程打开的文件，其他线程也可以直接操作，无需再次打开 。
    
*   CLONE\_SIGHAND：共享信号处理函数和信号掩码 ，即进程内所有线程对信号的处理方式是一致的 。
    
*   CLONE\_THREAD：将新创建的线程放入与父进程相同的线程组 ，同一线程组内的线程共享同一个进程 ID（PID），并且当组内最后一个线程退出或主线程调用 exit()时，整个线程组（进程）退出 。
    

当 flags 中设置了这些共享资源的标志位时，创建出来的就是线程；如果没有设置这些共享标志位，或者只设置了一些进程独有的标志位，如 SIGCHLD（用于通知父进程子进程状态的改变，常用于进程创建场景），那么创建出来的就是一个独立的进程 。例如，下面的代码展示了使用 clone 创建线程的示例：

```
#define   _GNU_SOURCE#include  <stdio.h>#include  <sched.h>#include  <unistd.h>#include  <sys/wait.h>#include  <stdlib.h>int child_func(void* arg) {    printf("子线程执行，参数为: %d\n", *(int*)arg);    return0;}int main() {    int arg = 10;    void* stack = malloc(1024 * 1024); // 分配 1MB 栈空间    if (!stack) {        perror("malloc");        return1;    }    // 使用 clone 创建线程，设置共享资源标志位    int pid = clone(child_func, (char*)stack + 1024 * 1024, CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD, &arg);     if (pid == -1) {        perror("clone");        free(stack);        return1;    }    // 等待子线程结束    waitpid(pid, NULL, 0);     free(stack);    return0;}
```

在上述代码中，通过 clone 系统调用创建线程，设置了多个共享资源的标志位，使得新创建的线程与父进程共享多种资源 。为线程分配了 1MB 的栈空间，并在创建线程后等待其结束，最后释放栈空间 。

### 1.3.3 线程的内核数据结构

在 Linux 内核中，task\_struct 结构体是管理进程和线程的核心数据结构 。对于线程来说，task\_struct 同样起着至关重要的作用，它记录了线程运行所需的各种信息 。

每个线程都有一个唯一的 task\_struct 结构体实例 ，这个结构体中包含了线程的状态、调度信息、内存管理信息、文件系统相关信息、信号处理信息等 。例如，线程的状态字段 state 记录了线程当前是运行态（TASK\_RUNNING）、睡眠态（TASK\_INTERRUPTIBLE 或 TASK\_UNINTERRUPTIBLE）还是其他状态 。调度信息包括线程的优先级、调度策略等，这些信息决定了内核如何调度线程运行 。

线程 ID（LWP，Light Weight Process ID）和线程组 ID（TGID，Thread Group ID）是 task\_struct 结构体中与线程标识相关的重要概念 。线程 ID（LWP）是内核为每个线程分配的唯一标识，在系统范围内是唯一的，内核通过这个 ID 来识别和调度线程 。线程组 ID（TGID）则标识了线程所属的线程组，同一线程组内的线程共享同一个 TGID，并且 TGID 与该线程组的主线程的 PID 相同 。

在用户态，我们使用 pthread\_t 类型的标识符来标识线程，它是 POSIX 线程库定义的线程 ID，仅在进程内部有效 。而在内核态，使用的是 LWP 来标识线程 。pthread\_t 与 LWP 之间存在映射关系，POSIX 线程库通过维护这种映射关系，实现了用户态线程 ID 到内核态线程 ID 的转换 。

例如，当我们在用户态调用 pthread\_create 创建线程时，线程库会在内核中通过 clone 系统调用创建一个新的线程，并将返回的 LWP 与新创建的 pthread\_t 进行关联 。在用户态通过 pthread\_self 获取线程 ID 时，实际上是获取到线程库维护的 pthread\_t，而线程库可以通过内部机制将其转换为对应的 LWP，以便在内核中进行相关操作 。

### 1.3.4 线程调度与上下文切换

内核调度线程的过程是多线程系统高效运行的关键 。内核调度器负责决定在某一时刻哪个线程可以获得 CPU 资源并执行 。在 Linux 系统中，内核调度器采用了多种调度算法，其中完全公平调度器（CFS，Completely Fair Scheduler）是默认的针对普通进程和线程的调度算法 。

时间片分配是线程调度的重要环节 。CFS调度算法并不像传统调度算法那样为每个线程分配固定的时间片 ，而是为每个线程分配一个虚拟运行时间（vruntime） 。线程的优先级决定了其获得CPU时间的比例，优先级高的线程，其虚拟运行时间增长得相对较慢，这样就能优先获得CPU资源 。例如，一个高优先级线程和一个低优先级线程同时竞争 CPU，高优先级线程在相同的实际时间内，其虚拟运行时间增加得比低优先级线程少，因此内核调度器会更频繁地调度高优先级线程运行 。

上下文切换是指当内核决定暂停当前线程的执行，转而执行另一个线程时，需要保存当前线程的执行状态，并恢复要执行线程的执行状态的过程 。在上下文切换过程中，内核需要保存当前线程的寄存器状态，包括通用寄存器（如 eax、ebx 等）、程序计数器（PC，指向当前正在执行的指令地址）等，这些寄存器保存了线程执行的中间结果和下一步要执行的指令位置 。还要保存栈指针，栈指针指向线程的栈顶，栈中存储了函数调用的参数、局部变量等信息，保存栈指针可以确保线程下次恢复执行时，能够正确地访问这些数据 。

当切换到另一个线程时，内核会从该线程对应的 task\_struct 结构体中读取保存的寄存器状态和栈指针等信息，并将这些信息恢复到 CPU 的寄存器中，使得该线程能够从上次暂停的位置继续执行 。上下文切换虽然是多线程系统实现并发的必要操作，但也会带来一定的开销，包括保存和恢复寄存器状态、刷新缓存等操作都会消耗 CPU 时间 。因此，在编写多线程程序时，应尽量减少不必要的上下文切换，以提高系统性能 。

### 1.3.5 线程同步原语的内核实现

线程同步原语是多线程编程中确保数据一致性和避免竞态条件的重要工具 ，它们在内核层面有着复杂而精妙的实现机制 。

以互斥锁为例，它是最常用的线程同步工具之一 。在 pthread 库中，pthread\_mutex\_t 类型的互斥锁用于保护临界区，确保同一时间只有一个线程能够进入临界区访问共享资源 。在内核层面，互斥锁的实现利用了原子操作和 futex（快速用户空间互斥量，Fast Userspace Mutex） 。

原子操作是指不可被中断的操作，在 CPU 执行原子操作时，不会被其他线程或进程打断 。对于互斥锁的加锁和解锁操作，会使用原子操作来确保其原子性 。当一个线程尝试加锁时，会通过原子操作检查锁的状态，如果锁处于未被占用状态，就将其设置为已占用状态，这个过程是原子的，不会出现多个线程同时认为锁未被占用而同时获取锁的情况 。

当多个线程竞争锁时，如果锁已被占用，其他线程就需要等待 。这时就会用到 futex 。futex 是一种用户空间和内核空间协作的同步机制 。当一个线程发现锁已被占用时，它会通过 futex 系统调用进入内核态，将自己放入等待队列中，并将当前线程的状态设置为睡眠状态，让出 CPU 资源 。当持有锁的线程释放锁时，会通过 futex 唤醒等待队列中的一个线程，被唤醒的线程会重新竞争锁 。这种机制减少了线程在用户空间的无效循环等待，提高了系统的效率 。

条件变量（pthread\_cond\_t）也是常用的线程同步原语 ，它通常与互斥锁配合使用，用于线程间的通信和协作 。在内核实现中，条件变量同样依赖于等待队列和信号机制 。当一个线程调用 pthread\_cond\_wait 等待某个条件成立时，它会先释放持有的互斥锁（这是为了避免死锁，因为如果不释放锁，其他线程无法获取锁来修改条件，导致该线程永远等待），然后将自己放入条件变量的等待队列中，并进入睡眠状态 。

当另一个线程满足条件后，调用pthread\_cond\_signal 或 pthread\_cond\_broadcast唤醒等待队列中的一个或多个线程 。被唤醒的线程会重新获取互斥锁，然后检查条件是否满足，如果满足则继续执行，否则再次等待 。

## 1.4 四、 pthread 实战案例分析

面试题写作模版

### 1.4.1 案例一：线程僵死问题

在一个分布式文件系统中，有多个线程负责文件的读写操作。部分线程在执行文件读取操作时，由于文件系统的某些异常（如磁盘 I/O 错误、文件损坏等），导致线程进入一种无限等待的状态，最终造成线程僵死。从表面上看，开发者可能只是发现某些文件读取请求长时间没有响应，但通过简单的调试很难找到问题的根本原因。

```
#include <stdio.h>#include <stdlib.h>#include <pthread.h>#include <unistd.h>#include <fcntl.h>#include <errno.h>// 线程函数：模拟出现 I/O 异常导致线程僵死void *file_read_routine(void *arg) {    // 打开一个不存在或异常的设备/文件    int fd = open("/dev/exception_device", O_RDONLY);    if (fd == -1) {        perror("文件打开失败");        return NULL;    }    char buffer[1024];    // 无超时、无错误处理，异常时会永久阻塞    ssize_t ret = read(fd, buffer, sizeof(buffer));    // 若 I/O 异常卡死，以下代码永远不会执行    printf("读取完成，返回值：%ld\n", ret);    close(fd);    return NULL;}int main() {    pthread_t tid;    pthread_create(&tid, NULL, file_read_routine, NULL);    // 主线程等待僵死线程，永远无法返回    pthread_join(tid, NULL);    return0;}
```

运用底层原理进行调试时，首先可以通过系统工具（如 ps 命令结合 gdb 调试器）查看线程的状态，确定哪些线程处于僵死状态。了解线程创建的内核逻辑后，知道每个线程都有对应的 task\_struct 结构体，通过分析这个结构体中的状态信息以及相关的内核日志，可以发现这些僵死线程在等待某些资源（如文件锁、磁盘 I/O 完成信号等）时进入了死等状态，且由于异常情况，这些资源永远无法被释放。

**代码示例：修复线程僵死

**

```
void *file_read_safe_routine(void *arg) {    // 以非阻塞方式打开文件    int fd = open("/dev/exception_device", O_RDONLY | O_NONBLOCK);    if (fd == -1) {        perror("文件打开失败");        return NULL;    }    // 设置 1 秒超时    struct timeval tv = {.tv_sec = 1, .tv_usec = 0};    fd_set read_fds;    FD_ZERO(&read_fds);    FD_SET(fd, &read_fds);    // 等待数据可读，超时则退出    int ready = select(fd + 1, &read_fds, NULL, NULL, &tv);    if (ready == -1) {        perror("select 错误");        close(fd);        return NULL;    } elseif (ready == 0) {        printf("读取超时，线程安全退出，避免僵死\n");        close(fd);        return NULL;    }    // 正常读取    char buffer[1024];    read(fd, buffer, sizeof(buffer));    printf("文件读取成功\n");    close(fd);    return NULL;}
```

根据这个分析结果，在代码中添加更完善的错误处理机制，当文件读取出现异常时，及时释放线程占用的资源，并设置合适的错误返回值，通知调用者处理异常情况。这样，通过对底层原理的理解，成功解决了线程僵死问题，提高了文件系统的稳定性和可靠性。

### 1.4.2 案例二：死锁问题

考虑一个银行转账系统，假设有两个账户 A 和 B，有两个线程分别负责从账户 A 向账户 B 转账和从账户 B 向账户 A 转账的操作。每个线程在进行转账操作时，都需要先获取两个账户的锁，以保证转账过程的原子性和数据一致性。但如果代码编写不当，就可能出现死锁。比如，线程 1 先获取了账户 A 的锁，然后试图获取账户 B 的锁；与此同时，线程 2 先获取了账户 B 的锁，然后试图获取账户 A 的锁。这样，两个线程就相互等待对方释放锁，形成了死锁。

```
#include <stdio.h>#include <pthread.h>#include <unistd.h>// 两个账户对应的互斥锁pthread_mutex_t accountA = PTHREAD_MUTEX_INITIALIZER;pthread_mutex_t accountB = PTHREAD_MUTEX_INITIALIZER;// 线程 1：A → Bvoid *transfer_A2B(void *arg) {    pthread_mutex_lock(&accountA);    printf("线程 1：已锁住账户 A\n");    sleep(1); // 放大死锁概率    // 等待线程 2 释放 accountB，形成死锁    pthread_mutex_lock(&accountB);    printf("线程 1：转账完成\n");    pthread_mutex_unlock(&accountB);    pthread_mutex_unlock(&accountA);    return NULL;}// 线程 2：B → Avoid *transfer_B2A(void *arg) {    pthread_mutex_lock(&accountB);    printf("线程 2：已锁住账户 B\n");    sleep(1);    // 等待线程 1 释放 accountA，形成死锁    pthread_mutex_lock(&accountA);    printf("线程 2：转账完成\n");    pthread_mutex_unlock(&accountA);    pthread_mutex_unlock(&accountB);    return NULL;}
```

在解决这个问题时，理解互斥锁的底层实现原理至关重要。通过分析死锁发生时的线程堆栈信息和互斥锁的状态，可以发现两个线程对互斥锁的获取顺序不一致。

**代码示例：统一锁顺序，解决死锁

**

```
// 所有线程都按照固定顺序获取锁：先 A 后 Bvoid *transfer_safe(void *arg) {    // 统一顺序：先锁 A，再锁 B    pthread_mutex_lock(&accountA);    pthread_mutex_lock(&accountB);    printf("线程：执行转账操作\n");    // 释放顺序与获取顺序相反    pthread_mutex_unlock(&accountB);    pthread_mutex_unlock(&accountA);    return NULL;}
```

根据互斥锁的原子操作和内核调度机制，对代码进行修改，规定所有线程获取锁的顺序必须一致，例如都先获取账户 A 的锁，再获取账户 B 的锁。这样，当一个线程获取了账户 A 的锁后，其他线程就无法再获取账户 A 的锁，从而避免了死锁的发生。同时，在获取锁时，添加超时机制，当线程在一定时间内无法获取到所需的锁时，主动释放已获取的锁并进行错误处理，进一步增强了系统的健壮性。

### 1.4.3 案例三：并发调度异常

以一个在线游戏服务器为例，多个线程负责处理玩家的游戏操作请求，如移动、攻击、交易等。在高并发情况下，由于线程调度的不确定性，可能会出现并发调度异常，导致玩家的操作顺序被打乱，影响游戏的公平性和体验。比如，玩家 A 先发起攻击操作，然后发起移动操作，但由于并发调度异常，服务器可能先处理了移动操作，后处理攻击操作，这就使得游戏中的实际情况与玩家的预期不符。

```
#include <stdio.h>#include <pthread.h>#include <unistd.h>// 无同步控制，调度无序void *attack(void *arg) {    printf("玩家执行攻击\n");    return NULL;}void *move(void *arg) {    printf("玩家执行移动\n");    return NULL;}// 结果可能出现：先攻击后移动，逻辑错误
```

深入理解条件变量和线程同步的底层原理后，我们可以在代码中使用条件变量来控制线程的执行顺序。在处理玩家操作请求的线程中，为每个操作类型设置对应的条件变量。当一个线程接收到玩家的操作请求时，先判断该操作是否依赖于其他操作的完成（如攻击操作可能依赖于玩家的位置信息，即移动操作先完成）。

**代码示例：条件变量保证执行顺序

**

```
#include <stdio.h>#include <pthread.h>pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;pthread_cond_t cond = PTHREAD_COND_INITIALIZER;int move_finished = 0;// 线程：移动void *move_thread(void *arg) {    pthread_mutex_lock(&mutex);    printf("玩家移动完成\n");    move_finished = 1;    // 唤醒等待的攻击线程    pthread_cond_signal(&cond);    pthread_mutex_unlock(&mutex);    return NULL;}// 线程：攻击（必须等待移动完成）void *attack_thread(void *arg) {    pthread_mutex_lock(&mutex);    // 等待移动完成    while (!move_finished) {        pthread_cond_wait(&cond, &mutex);    }    printf("玩家发起攻击\n");    pthread_mutex_unlock(&mutex);    return NULL;}
```

如果依赖，线程就等待相应的条件变量被触发。当相关操作完成后，通过条件变量唤醒等待的线程，确保操作按照正确的顺序执行。同时，结合互斥锁保证共享数据（如玩家的状态信息）的安全访问，避免数据不一致的问题。通过这种方式，有效地解决了并发调度异常，提升了游戏服务器的性能和稳定性。

## 1.5 五、pthread 实战避坑指南

面试题写作模版

### 1.5.1 线程创建与资源管理

**（1）线程创建失败原因与解决——在使用 pthread\_create 创建线程时，可能会遭遇创建失败的情况

。

**这就好比在一场体育赛事中，招募新选手（创建新线程）时却遇到了阻碍。其中一个常见原因是系统资源限制，当达到最大线程数时，就像赛事的参赛名额已满，无法再接纳新选手。例如在 Linux 系统中，每个进程所能创建的线程数量是有限制的，这个限制可以通过 ulimit -u 命令查看 。当线程创建失败返回 EAGAIN 错误码时，很可能就是因为达到了这个限制。此时，我们可以通过调整系统参数来解决，比如使用 ulimit -u new\_limit 命令临时提高每个用户可创建的最大线程数 new\_limit，或者修改/etc/security/limits.conf 文件进行永久设置 。

内存不足也是导致线程创建失败的一个重要因素。线程的创建需要分配一定的内存空间，包括线程栈、线程控制块等，如果系统内存紧张，无法满足这些内存需求，线程创建就会失败。这就如同建造房屋（创建线程）时，没有足够的建筑材料（内存）。当遇到这种情况时，我们需要优化程序的内存使用，释放不必要的内存资源，或者增加系统的物理内存。还有一种情况是线程属性设置错误，比如设置了不支持的调度策略、非法的栈大小等，这就像给选手制定了不合理的比赛规则，导致招募失败。此时，我们需要仔细检查线程属性的设置，确保其符合系统的要求和规范。

**（2）栈空间配置问题——线程的栈空间就像是选手比赛时的专属场地，用于存储局部变量、函数调用帧等信息。



**如果栈空间配置过小，就像场地过于狭小，选手施展不开；或者递归深度过大，不断地往场地里堆放物品，最终都可能导致栈溢出，程序崩溃。例如下面这段简单的递归代码：

```
#include <stdio.h>#include <pthread.h>void recursive_function() {    int local_variable[10000];  // 占用较大栈空间的局部数组    recursive_function();  // 递归调用}void* thread_function(void* arg) {    recursive_function();    return NULL;}int main() {    pthread_t thread;    if (pthread_create(&thread, NULL, thread_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_join(thread, NULL) != 0) {        perror("pthread_join");        return1;    }    return0;}
```

在这段代码中，recursive\_function 函数不断递归调用自身，并且每次调用都创建一个占用较大栈空间的局部数组 local\_variable，很容易导致栈溢出。

为了预防栈溢出问题，我们需要合理设置线程栈大小。在 Linux 下，可以通过 pthread\_attr\_setstacksize 函数来调整线程栈大小。例如：

```
#include <stdio.h>#include <pthread.h>void* thread_function(void* arg) {    // 线程函数内容    return NULL;}int main() {    pthread_t thread;    pthread_attr_t attr;    pthread_attr_init(&attr);    size_t stack_size = 1024 * 1024;  // 设置栈大小为 1MB    pthread_attr_setstacksize(&attr, stack_size);    if (pthread_create(&thread, &attr, thread_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_join(thread, NULL) != 0) {        perror("pthread_join");        return1;    }    pthread_attr_destroy(&attr);    return0;}
```

通过上述代码，我们在创建线程前设置了线程的栈大小为 1MB，从而避免了因栈空间过小导致的栈溢出问题 。同时，在编写代码时，也要注意避免过度递归，合理控制函数调用的深度。

**（3）线程分离与连接误区——线程分离（pthread\_detach）和连接（pthread\_join）是线程生命周期管理中的重要操作，但在使用时容易出现误区。



**线程分离就像是让选手独自参赛，比赛结束后自行离开赛场，不需要等待其他线程（如主线程）来处理后续事宜；而线程连接则像是等待选手比赛结束，获取比赛结果（线程返回值）并清理赛场（回收线程资源）。

如果对一个需要获取返回值或者需要确保其执行完毕的线程使用了 pthread\_detach，就像在一场重要比赛中，不关心选手的比赛成绩就放走了选手，会导致无法获取线程的执行结果，造成信息丢失。例如在一个计算任务中，子线程负责复杂的计算，计算结果对主线程后续的操作至关重要，如果将该子线程分离，主线程就无法得到计算结果，影响整个程序的逻辑。

相反，如果对已经分离的线程调用 pthread\_join，就像去寻找一个已经离开赛场的选手要成绩，这是不合理的，会导致错误。例如下面的代码：

```
#include <stdio.h>#include <pthread.h>#include <string.h>void* thread_function(void* arg) {    printf("子线程正在运行\n");    return (void*)1;}int main() {    pthread_t thread;    if (pthread_create(&thread, NULL, thread_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_detach(thread) != 0) {        perror("pthread_detach");        return1;    }    void* result;    int ret = pthread_join(thread, &result);    if (ret != 0) {        printf("join 失败，错误代码: %d，错误信息: %s\n", ret, strerror(ret));    } else {        printf("线程返回值: %ld\n", (long)result);    }    return0;}
```

在这段代码中，我们先将线程分离，然后又尝试对其调用 pthread\_join，运行时会发现 pthread\_join 失败，并返回错误信息 “Invalid argument”，因为已经分离的线程不再是可连接的状态。

为了避免这些误区，我们需要明确线程分离和连接的正确使用场景。当我们不关心线程的返回值，且希望线程结束后自动释放资源时，使用 pthread\_detach，比如在一些后台日志记录线程、监控线程等场景中；当我们需要等待线程执行完毕，并获取其返回值时，使用 pthread\_join，比如在一些计算任务线程、数据处理线程等场景中 。

### 1.5.2 线程同步与互斥

**（1）锁争用问题



**——在多线程编程中，当多个线程频繁访问同一共享资源时，就像多个选手都想同时使用同一个比赛道具，过度使用互斥锁（mutex）会导致线程阻塞，形成锁争用瓶颈。互斥锁就像是这个比赛道具的唯一钥匙，同一时间只有拿到钥匙（获取锁）的线程才能使用道具（访问共享资源），其他线程只能等待。

例如在一个多线程的银行账户操作程序中，多个线程可能同时对账户余额进行取款、存款等操作，如果每个操作都使用互斥锁进行保护，当线程并发量较大时，就会有很多线程因为等待锁而处于阻塞状态，大大降低了程序的执行效率。为了解决锁争用问题，我们可以采取以下优化方式。首先，减少临界区范围，就像缩小选手使用比赛道具的时间和范围，只在真正需要访问共享资源的代码段加锁，而不是将整个函数都置于锁的保护之下。例如：

```
#include <pthread.h>#include <stdio.h>pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;int shared_variable = 0;void* thread_function(void* arg) {    // 其他非共享资源操作，不需要加锁    for (int i = 0; i < 1000; ++i) {        // 临界区，访问共享资源时加锁        pthread_mutex_lock(&mutex);        shared_variable++;        pthread_mutex_unlock(&mutex);    }    return NULL;}int main() {    pthread_t thread1, thread2;    if (pthread_create(&thread1, NULL, thread_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_create(&thread2, NULL, thread_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_join(thread1, NULL) != 0) {        perror("pthread_join");        return1;    }    if (pthread_join(thread2, NULL) != 0) {        perror("pthread_join");        return1;    }    pthread_mutex_destroy(&mutex);    printf("最终共享变量的值: %d\n", shared_variable);    return0;}
```

在这段代码中，我们只在对 shared\_variable 进行操作时加锁，而不是将整个 for 循环都加锁，从而减少了锁的持有时间，降低了锁争用的可能性。

其次，在一些读多写少的场景中，可以使用读写锁（sync.RWMutex）。读写锁允许多个线程同时进行读操作，就像多个选手可以同时观看比赛道具展示（读操作），只有在进行写操作时才需要独占锁，就像只有一个选手可以对比赛道具进行修改（写操作）。这样可以大大提高并发性能。例如：

```
#include  <pthread.h>#include  <stdio.h>pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;int shared_data = 0;void* read_thread(void* arg) {    pthread_rwlock_rdlock(&rwlock);    printf("读线程读取数据: %d\n", shared_data);    pthread_rwlock_unlock(&rwlock);    return NULL;}void* write_thread(void* arg) {    pthread_rwlock_wrlock(&rwlock);    shared_data++;    printf("写线程修改数据: %d\n", shared_data);    pthread_rwlock_unlock(&rwlock);    return NULL;}int main() {    pthread_t read_thread1, read_thread2, write_thread1;    if (pthread_create(&read_thread1, NULL, read_thread, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_create(&read_thread2, NULL, read_thread, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_create(&write_thread1, NULL, write_thread, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_join(read_thread1, NULL) != 0) {        perror("pthread_join");        return1;    }    if (pthread_join(read_thread2, NULL) != 0) {        perror("pthread_join");        return1;    }    if (pthread_join(write_thread1, NULL) != 0) {        perror("pthread_join");        return1;    }    pthread_rwlock_destroy(&rwlock);    return0;}
```

在这个例子中，读线程可以同时获取读锁进行数据读取，而写线程在获取写锁时会独占资源，其他读线程和写线程都需要等待，这样在保证数据一致性的同时，提高了读操作的并发性能。此外，还可以考虑使用无锁数据结构，如 std::atomic 等，这些数据结构通过硬件指令实现原子操作，避免了锁的使用，从而提高了并发性能，但使用时需要对其原理和适用场景有深入的理解 。

**（2）死锁的产生与避免



**——死锁是多线程编程中一个非常棘手的问题，就像几个选手在比赛中相互抢夺对方手中的道具，并且都不愿意先放手，最终导致谁也无法继续比赛。死锁产生的原因主要是多个线程以不同顺序获取多个锁，形成循环等待。例如，线程 A 持有锁 1，等待获取锁 2；而线程 B 持有锁 2，等待获取锁 1，这样两个线程就会永远等待下去，形成死锁。

为了避免死锁，我们可以采取以下方法。首先，按固定顺序获取锁，就像给选手们制定一个统一的道具获取顺序，避免循环等待。例如，假设有两个锁 mutex1 和 mutex2，我们可以规定所有线程都先获取 mutex1，再获取 mutex2。代码示例如下：

```
#include  <pthread.h>#include  <stdio.h>pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;pthread_mutex_t mutex2 = PTHREAD_MUTEX_INITIALIZER;void* thread1_function(void* arg) {    pthread_mutex_lock(&mutex1);    printf("线程 1 获取锁 1\n");    // 模拟业务操作    sleep(1);    pthread_mutex_lock(&mutex2);    printf("线程 1 获取锁 2\n");    // 操作共享资源    pthread_mutex_unlock(&mutex2);    printf("线程 1 释放锁 2\n");    pthread_mutex_unlock(&mutex1);    printf("线程 1 释放锁 1\n");    return NULL;}void* thread2_function(void* arg) {    pthread_mutex_lock(&mutex1);    printf("线程 2 获取锁 1\n");    // 模拟业务操作    sleep(1);    pthread_mutex_lock(&mutex2);    printf("线程 2 获取锁 2\n");    // 操作共享资源    pthread_mutex_unlock(&mutex2);    printf("线程 2 释放锁 2\n");    pthread_mutex_unlock(&mutex1);    printf("线程 2 释放锁 1\n");    return NULL;}int main() {    pthread_t thread1, thread2;    if (pthread_create(&thread1, NULL, thread1_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_create(&thread2, NULL, thread2_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_join(thread1, NULL) != 0) {        perror("pthread_join");        return1;    }    if (pthread_join(thread2, NULL) != 0) {        perror("pthread_join");        return1;    }    pthread_mutex_destroy(&mutex1);    pthread_mutex_destroy(&mutex2);    return0;}
```

在这段代码中，线程 1 和线程 2 都按照先获取 mutex1，再获取 mutex2 的顺序进行操作，从而避免了死锁的发生。其次，使用超时锁（pthread\_mutex\_timedlock），给获取锁的操作设置一个超时时间，就像给选手设定一个抢夺道具的时间限制，超过时间就放弃。如果在超时时间内没有获取到锁，线程可以执行其他操作，避免无限等待。例如：

```
#include  <pthread.h>#include  <stdio.h>#include  <time.h>pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;void* thread_function(void* arg) {    struct timespec timeout;    clock_gettime(CLOCK_REALTIME, &timeout);    timeout.tv_sec += 2;  // 设置超时时间为 2 秒    if (pthread_mutex_timedlock(&mutex, &timeout) == 0) {        printf("线程获取到锁\n");        // 操作共享资源        pthread_mutex_unlock(&mutex);        printf("线程释放锁\n");    } else {        printf("线程获取锁超时\n");    }    return NULL;}int main() {    pthread_t thread;    if (pthread_create(&thread, NULL, thread_function, NULL) != 0) {        perror("pthread_create");        return1;    }    if (pthread_join(thread, NULL) != 0) {        perror("pthread_join");        return1;    }    pthread_mutex_destroy(&mutex);    return0;}
```

在这个例子中，线程尝试获取锁时设置了 2 秒的超时时间，如果 2 秒内没有获取到锁，就会打印 “线程获取锁超时”，从而避免了线程因为等待锁而无限阻塞。

另外，要尽量避免嵌套锁，减少死锁的风险。嵌套锁就像一个选手在已经持有一个道具的情况下，又去抢夺另一个道具，而且可能与其他选手的抢夺顺序不一致，容易引发死锁。如果必须使用嵌套锁，一定要严格按照固定顺序获取，并且做好注释，以便后续维护。

**（3）条件变量使用不当



**——条件变量（pthread\_cond\_t）是线程同步中的重要工具，它通常与互斥锁配合使用，用于线程间的通信和同步。然而，如果使用不当，就会导致线程同步问题，影响程序的正确性和稳定性。在生产者 - 消费者模型中，生产者线程负责生产数据并将其放入共享缓冲区，消费者线程则从缓冲区中取出数据进行处理。在这个模型中，条件变量用于协调生产者和消费者之间的操作。

如果条件变量使用不当，就可能出现线程等待或唤醒不合理的情况。生产者在向缓冲区中放入数据后，没有及时调用 pthread\_cond\_signal 或 pthread\_cond\_broadcast 来通知消费者线程，那么消费者线程就会一直处于等待状态，导致缓冲区中的数据无法被及时处理。相反，如果消费者在没有获取互斥锁的情况下调用 pthread\_cond\_wait，就会导致未定义行为，程序可能会出现崩溃或者数据不一致等问题。

为了正确使用条件变量，我们需要遵循一些基本原则。在调用 pthread\_cond\_wait 之前，必须先获取对应的互斥锁，因为 pthread\_cond\_wait 会自动释放锁并将线程挂起，当线程被唤醒时，又会重新获取锁。所以在等待条件变量时，一定要确保锁的正确使用，避免出现线程安全问题。我们应该使用 while 循环来检查条件，而不是使用 if 判断。这是因为在多线程环境下，可能会出现虚假唤醒的情况，即线程被无缘无故地唤醒。如果使用 if 判断，就可能会因为虚假唤醒而跳过等待条件的检查，导致程序逻辑错误。使用 while 循环可以确保在条件真正满足之前，线程会一直等待。

### 1.5.3 线程调度与优先级

**（1）调度策略理解误区



**——在 POSIX 系统中，线程的调度策略主要有 SCHED\_FIFO（先来先服务）、SCHED\_RR（时间片轮转）和 SCHED\_OTHER（普通调度）。不同的调度策略适用于不同的应用场景，但如果开发者对这些策略理解和使用不当，就可能导致问题的出现，尤其是在实时任务调度方面。

SCHED\_FIFO 调度策略会优先调度最先进入就绪队列的线程，并且该线程会一直运行，直到它主动放弃 CPU 或者被更高优先级的线程抢占。这种策略适用于对实时性要求极高的任务，如工业控制系统中的实时数据采集和处理任务。如果在使用 SCHED\_FIFO 时，没有合理设置线程的优先级，或者没有正确处理线程的阻塞和唤醒，就可能导致低优先级的线程长时间得不到执行，出现 “饿死” 的情况。

SCHED\_RR 调度策略则是为每个线程分配一个时间片，当时间片用完后，线程会被暂停，调度器会将其放入就绪队列的末尾，等待下一次调度。这种策略适用于对公平性要求较高的场景，如交互式应用程序，它可以确保每个线程都有机会得到执行。但如果对时间片的大小设置不合理，或者没有考虑到不同任务的执行特点，也可能会影响程序的性能。

SCHED\_OTHER 是默认的调度策略，它适用于普通的非实时任务，调度器会根据系统的负载情况和其他因素来进行线程调度。如果在需要实时性保障的场景中错误地使用了 SCHED\_OTHER 策略，就无法满足任务对时间的严格要求，导致系统响应迟缓。

为了正确使用线程调度策略，开发者需要深入理解每种策略的特点和适用场景。在实时任务调度中，应根据任务的实时性要求和重要程度，合理选择调度策略，并正确设置线程的优先级。对于关键的实时任务，优先选择 SCHED\_FIFO 策略，并为其分配较高的优先级；对于一些对公平性要求较高的普通任务，则可以选择 SCHED\_RR 策略。同时，要注意在不同调度策略下，线程的阻塞、唤醒和优先级调整等操作的正确处理，以确保系统的稳定运行和任务的高效执行。

**（2）优先级设置陷阱



**——设置线程优先级是优化多线程程序性能和实现特定调度需求的重要手段，但在设置过程中也存在一些陷阱。如果优先级范围设置不合理，就可能导致线程的实际调度效果与预期不符。在不同的调度策略下，线程优先级的范围是不同的。在 SCHED\_FIFO 和 SCHED\_RR 调度策略中，线程优先级的范围通常是有明确规定的，并且需要具备相应的权限才能设置较高的优先级。如果普通用户尝试设置超出其权限范围的优先级，就会导致设置失败，而程序可能不会给出明显的错误提示，这就会使开发者误以为优先级已经设置成功，从而影响程序的正常运行。

缺乏相应权限也是设置线程优先级时常见的问题。在 Linux 系统中，普通用户默认只能设置 SCHED\_OTHER 调度策略下的优先级，并且范围有限。如果需要设置 SCHED\_FIFO 或 SCHED\_RR 调度策略下的较高优先级，通常需要具备 root 权限或者相应的 cap\_sys\_nice 能力。如果在没有获取到这些权限的情况下尝试设置高优先级，就会导致设置失败，线程仍然会按照默认的优先级进行调度。

为了正确获取和设置优先级，我们可以使用 sched\_get\_priority\_min/max 函数来获取不同调度策略下优先级的最小值和最大值，从而确保设置的优先级在合理范围内。在设置线程优先级之前，要先检查当前用户是否具备相应的权限，如果权限不足，需要通过合适的方式获取权限，如使用 sudo 命令或者为程序授予 cap\_sys\_nice 能力。在设置线程优先级时，还可以通过 pthread\_attr\_t 配置线程属性，将优先级设置与线程的创建和管理结合起来，确保优先级设置的正确性和有效性。例如：

```
#include <pthread.h>#include <sched.h>#include <stdio.h>void* thread_function(void* arg) {    // 线程执行的函数体    return NULL;}int main() {    pthread_t thread;    pthread_attr_t attr;    struct sched_param param;    pthread_attr_init(&attr);    // 获取 SCHED_FIFO 调度策略下的最大优先级    int max_priority = sched_get_priority_max(SCHED_FIFO);    if (max_priority == -1) {        perror("sched_get_priority_max failed");        return1;    }    // 设置线程优先级为最大优先级    param.sched_priority = max_priority;    // 设置线程调度策略为 SCHED_FIFO    if (pthread_attr_setschedpolicy(&attr, SCHED_FIFO) != 0) {        perror("pthread_attr_setschedpolicy failed");        return1;    }    // 设置线程优先级    if (pthread_attr_setschedparam(&attr, &param) != 0) {        perror("pthread_attr_setschedparam failed");        return1;    }    if (pthread_create(&thread, &attr, thread_function, NULL) != 0) {        perror("pthread_create failed");        return1;    }    if (pthread_join(thread, NULL) != 0) {        perror("pthread_join failed");        return1;    }    pthread_attr_destroy(&attr);    return0;}
```

在这个示例中，我们首先获取了 SCHED\_FIFO 调度策略下的最大优先级，然后设置线程的调度策略为 SCHED\_FIFO，并将优先级设置为最大优先级，最后根据设置好的属性创建线程。通过这种方式，可以正确地设置线程的优先级，避免因优先级设置不当而导致的问题。

## 1.6 六、附录：经典面试题解析

面试题写作模版

### 1.6.1 描述线程和进程的区别，以及 pthread 线程的特点？

**考点分析

**：这道题主要考察对线程和进程基本概念的理解，以及对 pthread 线程特性的掌握，重点在于清晰阐述它们在资源分配、调度、地址空间等方面的差异 ，以及 pthread 线程基于 POSIX 标准的独特优势。

**参考回答

**：线程和进程是操作系统中的重要概念 。进程是资源分配的基本单位，拥有独立的地址空间、文件描述符、内存等系统资源 ，不同进程之间相互隔离，互不干扰。线程是 CPU 调度的基本单位，是进程的一个执行流，同一进程内的多个线程共享进程的资源，包括代码段、数据段、堆等 。

它们的区别主要体现在以下几个方面：

*   资源分配：进程的创建和销毁需要分配和释放大量的系统资源，开销较大；线程的创建和销毁只需要分配和释放少量的资源，如线程栈和寄存器等，开销相对较小 。
    
*   调度：进程的调度由操作系统负责，切换时需要保存和恢复大量的上下文信息，开销较大；线程的调度也由操作系统负责，但由于线程共享进程资源，切换时只需保存和恢复少量的上下文信息，开销较小 。
    
*   地址空间：进程拥有独立的地址空间，不同进程之间的地址空间相互隔离；线程共享进程的地址空间，同一进程内的线程可以直接访问共享内存 。
    
*   上下文切换开销：进程切换时，需要切换地址空间、页表等，开销较大；线程切换时，只需要切换栈指针、寄存器等，开销较小 。
    
*   通信方式：进程间通信较为复杂，需要使用管道、消息队列、信号量、共享内存等进程间通信（IPC）机制；线程间通信相对简单，可以直接通过共享变量进行通信 ，但需要注意线程同步问题。
    

pthread 线程具有以下特点：

*   基于 POSIX 标准：遵循 POSIX 规范，具备良好的跨平台特性，可在多种类 Unix 系统，如 Linux、macOS 等上运行，甚至在 Windows 系统中，借助 pthreads-win32 等工具和库也能实现移植使用 。
    
*   高效性：创建和销毁开销小，上下文切换迅速，能更灵活地处理并发任务，且同一进程内的线程共享资源，数据共享和通信高效 。
    
*   丰富的函数接口：提供了一系列用于线程创建、销毁、同步、调度等操作的函数，方便开发者进行多线程编程 。
    

### 1.6.2 在多线程编程中，如何解决线程安全问题？

**考点分析

**：本题着重考察对线程安全概念的理解以及解决线程安全问题的实际能力，需要清晰阐述线程安全问题产生的原因和常见解决方案，并通过具体例子进行说明 。

**参考回答

**：在多线程编程中，当多个线程同时访问和修改共享资源时，就可能出现线程安全问题，导致数据不一致或程序运行错误 。比如多个线程同时对一个全局变量进行读写操作，由于线程执行顺序的不确定性，可能会导致最终结果不符合预期 。

解决线程安全问题的常见方法有使用互斥锁、条件变量、信号量等同步机制 。以互斥锁为例，它可以保证同一时间只有一个线程能够进入临界区，访问共享资源 。下面以银行转账为例，展示如何使用互斥锁来实现线程安全：

```
#include <stdio.h>#include <pthread.h>// 银行账户结构体typedef struct {    int balance;    pthread_mutex_t mutex;} Account;// 初始化账户void initAccount(Account *account, int initialBalance) {    account->balance = initialBalance;    pthread_mutex_init(&account->mutex, NULL);}// 转账函数void transfer(Account *from, Account *to, int amount) {    pthread_mutex_lock(&from->mutex);    pthread_mutex_lock(&to->mutex);    if (from->balance >= amount) {        from->balance -= amount;        to->balance += amount;        printf("Transfer %d from account %p to account %p. New balances: from %d, to %d\n", amount, from, to, from->balance, to->balance);    } else {        printf("Insufficient funds in account %p for transfer of %d\n", from, amount);    }    pthread_mutex_unlock(&to->mutex);    pthread_mutex_unlock(&from->mutex);}// 线程执行函数void* transferThread(void* args) {    Account *from = ((Account**)args)[0];    Account *to = ((Account**)args)[1];    int amount = *((int*)(((Account**)args)[2]));    transfer(from, to, amount);    return NULL;}int main() {    Account account1, account2;    initAccount(&account1, 1000);    initAccount(&account2, 500);    int amount1 = 200;    int amount2 = 300;    pthread_t thread1, thread2;    Account* accounts[3] = {&account1, &account2, (Account*)&amount1};    pthread_create(&thread1, NULL, transferThread, (void*)accounts);    accounts[2] = (Account*)&amount2;    pthread_create(&thread2, NULL, transferThread, (void*)accounts);    pthread_join(thread1, NULL);    pthread_join(thread2, NULL);    pthread_mutex_destroy(&account1.mutex);    pthread_mutex_destroy(&account2.mutex);    return0;}
```

在上述代码中，Account结构体包含账户余额和一个互斥锁 。在transfer函数中，通过pthread\_mutex\_lock和pthread\_mutex\_unlock来加锁和解锁，确保在转账操作期间，其他线程无法同时访问账户，从而保证了线程安全 。

### 1.6.3 讲讲条件变量的原理和使用场景，以及可能出现的问题（如虚假唤醒）

**考点分析

**：本题重点考察对条件变量原理、应用场景以及潜在问题的理解，需要详细阐述条件变量的工作机制、在实际场景中的运用，以及如何应对虚假唤醒等问题 。

**参考回答

**：条件变量是一种线程同步机制，用于线程间的等待和通知协作，通常与互斥锁配合使用 。其原理是一个线程等待某个条件满足才继续执行，条件不满足就阻塞休眠；其他线程修改条件后主动唤醒等待线程 。

  

在 pthread 库中，条件变量相关的函数有pthread\_cond\_init（初始化条件变量）、pthread\_cond\_destroy（销毁条件变量）、pthread\_cond\_wait（阻塞等待条件，自动解锁，唤醒后自动加锁）、pthread\_cond\_signal（唤醒一个等待线程）、pthread\_cond\_broadcast（唤醒所有等待线程） 。

常见的使用场景是生产者 - 消费者模型 。在这个模型中，生产者线程负责生产数据并将其放入缓冲区，消费者线程从缓冲区中取出数据进行消费 。当缓冲区为空时，消费者线程需要等待生产者线程生产数据；当缓冲区满时，生产者线程需要等待消费者线程消费数据 。通过条件变量和互斥锁的配合，可以实现生产者和消费者之间的同步 。

下面是一个简单的生产者 - 消费者模型示例：

```
#include <stdio.h>#include <pthread.h>#include <semaphore.h>#define BUFFER_SIZE 5int buffer[BUFFER_SIZE];int in = 0;int out = 0;pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;pthread_cond_t cond_producer = PTHREAD_COND_INITIALIZER;pthread_cond_t cond_consumer = PTHREAD_COND_INITIALIZER;// 生产者线程函数void* producer(void* arg) {    int item = 1;    while (1) {        pthread_mutex_lock(&mutex);        while ((in + 1) % BUFFER_SIZE == out) {            // 缓冲区满，等待            pthread_cond_wait(&cond_producer, &mutex);        }        buffer[in] = item;        printf("Produced: %d\n", item);        in = (in + 1) % BUFFER_SIZE;        pthread_cond_signal(&cond_consumer);        pthread_mutex_unlock(&mutex);        item++;    }    return NULL;}// 消费者线程函数void* consumer(void* arg) {    while (1) {        pthread_mutex_lock(&mutex);        while (in == out) {            // 缓冲区空，等待            pthread_cond_wait(&cond_consumer, &mutex);        }        int item = buffer[out];        printf("Consumed: %d\n", item);        out = (out + 1) % BUFFER_SIZE;        pthread_cond_signal(&cond_producer);        pthread_mutex_unlock(&mutex);    }    return NULL;}int main() {    pthread_t tid1, tid2;    pthread_create(&tid1, NULL, producer, NULL);    pthread_create(&tid2, NULL, consumer, NULL);    pthread_join(tid1, NULL);    pthread_join(tid2, NULL);    pthread_mutex_destroy(&mutex);    pthread_cond_destroy(&cond_producer);    pthread_cond_destroy(&cond_consumer);    return0;}
```

在上述代码中，生产者线程在缓冲区满时等待cond\_producer条件变量，消费者线程消费数据后通过pthread\_cond\_signal唤醒生产者线程 ；消费者线程在缓冲区空时等待cond\_consumer条件变量，生产者线程生产数据后唤醒消费者线程 。

虚假唤醒是指在没有线程调用pthread\_cond\_signal或pthread\_cond\_broadcast的情况下，等待条件变量的线程被唤醒 。这是因为操作系统的调度机制可能会导致线程被意外唤醒 。为了避免虚假唤醒，在使用pthread\_cond\_wait时，应该使用while循环来检查条件，而不是if语句 。例如：

```
pthread_mutex_lock(&mutex);while (condition_is_false) {    pthread_cond_wait(&cond, &mutex);}// 执行条件满足后的操作pthread_mutex_unlock(&mutex);
```

这样，即使发生虚假唤醒，线程也会重新检查条件，若条件不满足则继续等待，从而保证程序的正确性 。

end

  

  

如果这篇文章对你有所启发，欢迎点赞、在看，转发三连。星标⭐账号，还可以第一时间收到推送，感谢你的收看，我们下期再见～

  

往期干货推荐

☑

【专栏

模块

】

[嵌入式Linux](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4NDQ0OTI4Ng==&action=getalbum&album_id=3206967864977817603#wechat_redirect)

☑

【

专栏

模块】

[性能优化](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4NDQ0OTI4Ng==&action=getalbum&album_id=3354529506191245313#wechat_redirect)

☑

【

专栏

模块】

[面试八股文](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4NDQ0OTI4Ng==&action=getalbum&album_id=3601023407971401733#wechat_redirect)

☑

【

专栏

模块

】

[项目实战](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4NDQ0OTI4Ng==&action=getalbum&album_id=3037464324489117698#wechat_redirect)

☑

【硬核干货

】

[缺了这些，别说你懂 Linux内核](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4NDQ0OTI4Ng==&action=getalbum&album_id=4453174433557774341#wechat_redirect)

☑

【

硬核

干货】

[缺了这些，别说你懂 Linux C/C++](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg4NDQ0OTI4Ng==&action=getalbum&album_id=3140091333123276802#wechat_redirect)

☑

【学习思维导图】

[Linux内核源码自主学习路线](https://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247493867&idx=1&sn=848ef81157409d02ab88435843e91083&scene=21#wechat_redirect)
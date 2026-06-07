---
title: pthread线程底层
description: pthread线程底层原理的总结，从API到底层调度
tags:
  - 嵌入式Linux
  - 线程
  - pthread
number headings: first-level 2, start-at 1, max 3, 1.1, auto, contents toc
---

> 来源: [不懂 pthread 线程底层，别说你会 Linux多线程开发](https://mp.weixin.qq.com/s/u9uJPqv1XyDm34GtSlKaLg)

不少 Linux 后台开发者熟练掌握 pthread 各类 API 的调用，但对底层运行逻辑一无所知。一旦遭遇线程僵死、资源泄露、随机死锁、并发调度异常等疑难问题，只能盲目调试、无从根治。

**API 调用只是表象，底层原理才是核心。** 本文深度拆解 pthread 线程底层原理，打通 Linux 多线程知识闭环。

## 1 认识 pthread 线程

### 1.1 线程与进程的区别

- **进程**是资源分配的基本单元，拥有独立的内存空间、文件描述符表、环境变量等资源。进程间相互隔离，一个进程崩溃不会影响其他进程。
- **线程**是 CPU 调度的最小单位，是进程中的执行单元。同一进程的多个线程共享进程资源（内存空间、文件描述符等），创建和切换开销比进程小得多。

线程共享资源使得通信高效，但也带来同步问题；且某个线程的错误（如非法内存访问）可能导致整个进程崩溃。

### 1.2 什么是 pthread 库？

**pthread** 是 POSIX 线程标准在 Linux 等 Unix 系统上的实现，为 C/C++ 多线程编程提供统一 API，具有良好的可移植性。

使用时需包含头文件 `<pthread.h>`，编译时链接 `-pthread`：

```bash
gcc -o my_program my_program.c -pthread
```

## 2 pthread API 详解

### 2.1 线程创建：pthread_create

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```

**参数说明：**

| 参数 | 说明 |
|------|------|
| `thread` | 存储新线程标识符 |
| `attr` | 线程属性（栈大小、调度策略等），`NULL` 使用默认值 |
| `start_routine` | 线程启动后执行的函数指针 |
| `arg` | 传递给线程函数的参数 |

成功返回 0，失败返回错误码（`EAGAIN` 资源不足、`EINVAL` 参数无效）。

**示例：**

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>

void *thread_function(void *arg) {
    int num = *(int *)arg;
    printf("子线程执行，参数为: %d\n", num);
    return NULL;
}

int main() {
    pthread_t tid;
    int arg = 10;
    int ret = pthread_create(&tid, NULL, thread_function, &arg);
    if (ret != 0) {
        printf("线程创建失败: %s\n", strerror(ret));
        return 1;
    }
    printf("主线程继续执行\n");
    pthread_join(tid, NULL);
    return 0;
}
```

### 2.2 线程等待：pthread_join

```c
int pthread_join(pthread_t thread, void **retval);
```

**阻塞当前线程**，直到指定线程结束，并获取其退出状态。`retval` 用于接收返回值，不关心可设为 `NULL`。

**示例：**

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <string.h>

void *thread_function(void *arg) {
    int *result = (int *)malloc(sizeof(int));
    *result = 42;
    return (void *)result;
}

int main() {
    pthread_t tid;
    void *thread_result;

    int ret = pthread_create(&tid, NULL, thread_function, NULL);
    if (ret != 0) {
        printf("线程创建失败: %s\n", strerror(ret));
        return 1;
    }

    ret = pthread_join(tid, &thread_result);
    if (ret != 0) {
        printf("等待线程失败: %s\n", strerror(ret));
        return 1;
    }

    int *result = (int *)thread_result;
    printf("子线程返回值: %d\n", *result);
    free(result);
    return 0;
}
```

### 2.3 线程退出：pthread_exit

```c
void pthread_exit(void *retval);
```

线程主动退出，`retval` 可被 `pthread_join` 获取。

**注意：不要返回指向栈上变量的指针**，线程退出后栈内存被销毁，会导致未定义行为：

```c
// 错误示范
void *thread_func(void *arg) {
    int local_var = 10;
    return (void *)&local_var; // 错误！返回栈上变量的指针
}

// 正确做法：使用堆内存
void *thread_func_safe(void *arg) {
    int *result = malloc(sizeof(int));
    *result = 10;
    return (void *)result;
}
```

### 2.4 线程标识：pthread_self

```c
pthread_t pthread_self(void);
```

返回当前线程的 `pthread_t` 标识符，常用于日志追踪和区分线程。

```c
#include <stdio.h>
#include <pthread.h>

void *thread_function(void *arg) {
    pthread_t self = pthread_self();
    printf("子线程 ID: %lu\n", (unsigned long)self);
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, thread_function, NULL);
    printf("主线程 ID: %lu\n", (unsigned long)pthread_self());
    pthread_join(tid, NULL);
    return 0;
}
```

### 2.5 线程分离：pthread_detach

```c
int pthread_detach(pthread_t thread);
```

将线程设置为**分离状态**，结束后自动释放资源，无需 `pthread_join`。

- **pthread_join**：需要等待线程结束并获取返回值
- **pthread_detach**：不关心返回值，线程结束后自动回收资源

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>

void *thread_function(void *arg) {
    printf("分离线程执行\n");
    return NULL;
}

int main() {
    pthread_t tid;
    int ret = pthread_create(&tid, NULL, thread_function, NULL);
    if (ret != 0) {
        printf("线程创建失败: %s\n", strerror(ret));
        return 1;
    }
    ret = pthread_detach(tid);
    if (ret != 0) {
        printf("设置线程分离失败: %s\n", strerror(ret));
        return 1;
    }
    printf("主线程继续执行\n");
    return 0;
}
```

### 2.6 线程取消：pthread_cancel

```c
int pthread_cancel(pthread_t thread);
```

向目标线程发送取消请求。**取消不是立即生效的**，线程到达**取消点**（如 `sleep`、`read` 等系统调用）时才响应。

取消类型：
- **延迟取消**（默认）：到达取消点时响应
- **异步取消**：随时响应

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <string.h>

void *thread_function(void *arg) {
    while (1) {
        printf("线程正在执行...\n");
        sleep(1); // sleep 是取消点
    }
    return NULL;
}

int main() {
    pthread_t tid;
    int ret = pthread_create(&tid, NULL, thread_function, NULL);
    if (ret != 0) {
        printf("线程创建失败: %s\n", strerror(ret));
        return 1;
    }
    sleep(3); // 主线程等待 3 秒

    ret = pthread_cancel(tid);
    if (ret != 0) {
        printf("取消线程失败: %s\n", strerror(ret));
        return 1;
    }
    pthread_join(tid, NULL);
    printf("线程已被取消\n");
    return 0;
}
```

## 3 深入 pthread 底层内核原理

### 3.1 Linux 线程实现机制

在 Linux 中，**线程的本质是轻量级进程（LWP）**。内核没有为线程设计独立的数据结构，而是**复用了进程的 `task_struct`**。

进程和线程在内核中都被视为可调度的 task，由 `task_struct` 管理。线程通过 **`clone` 系统调用**创建，通过标志位控制资源共享：

| 标志位 | 作用 |
|--------|------|
| `CLONE_VM` | 共享虚拟内存空间 |
| `CLONE_FS` | 共享文件系统信息 |
| `CLONE_FILES` | 共享文件描述符表 |
| `CLONE_SIGHAND` | 共享信号处理函数 |
| `CLONE_THREAD` | 放入同一线程组，共享 PID |

### 3.2 clone 系统调用剖析

```c
#define _GNU_SOURCE
#include <sched.h>

int clone(int (*fn)(void *), void *child_stack, int flags, void *arg,
          ... /* pid_t *parent_tid, void *tls, pid_t *child_tid */);
```

**关键参数：**

| 参数 | 说明 |
|------|------|
| `fn` | 新线程执行的函数 |
| `child_stack` | 新线程的栈空间（通常传最高地址） |
| `flags` | 控制资源共享的标志位组合 |
| `arg` | 传给 `fn` 的参数 |

**使用 clone 创建线程的示例：**

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <sched.h>
#include <unistd.h>
#include <sys/wait.h>
#include <stdlib.h>

int child_func(void *arg) {
    printf("子线程执行，参数为: %d\n", *(int *)arg);
    return 0;
}

int main() {
    int arg = 10;
    void *stack = malloc(1024 * 1024); // 分配 1MB 栈空间
    if (!stack) {
        perror("malloc");
        return 1;
    }

    int pid = clone(child_func, (char *)stack + 1024 * 1024,
                    CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD,
                    &arg);
    if (pid == -1) {
        perror("clone");
        free(stack);
        return 1;
    }

    waitpid(pid, NULL, 0);
    free(stack);
    return 0;
}
```

### 3.3 线程的内核数据结构

每个线程有唯一的 **`task_struct`** 实例，包含状态、调度信息、内存管理、信号处理等信息。

**关键标识概念：**

- **LWP（Light Weight Process ID）**：内核为每个线程分配的唯一标识，系统范围唯一
- **TGID（Thread Group ID）**：线程所属线程组的标识，等于主线程的 PID
- **pthread_t**：用户态线程 ID，仅在进程内部有效

pthread 库维护 `pthread_t` 与 `LWP` 的映射关系，实现用户态到内核态的 ID 转换。

### 3.4 线程调度与上下文切换

Linux 默认使用 **CFS（Completely Fair Scheduler）** 调度普通线程。

**核心机制：** CFS 不分配固定时间片，而是为每个线程维护 **vruntime（虚拟运行时间）**。优先级高的线程 vruntime 增长较慢，从而更频繁地获得 CPU。

**上下文切换**流程：
1. 保存当前线程的寄存器状态（通用寄存器、PC、栈指针等）
2. 从目标线程的 `task_struct` 恢复寄存器状态
3. 目标线程从上次暂停位置继续执行

上下文切换有开销（保存/恢复寄存器、刷新缓存），应尽量减少不必要的切换。

### 3.5 线程同步原语的内核实现

#### 互斥锁（Mutex）

互斥锁的实现基于**原子操作 + futex（Fast Userspace Mutex）**：

1. **加锁**：原子操作检查锁状态，空闲则占用
2. **竞争**：锁已占用时，通过 futex 系统调用进入内核态，将线程放入等待队列并睡眠
3. **解锁**：通过 futex 唤醒等待队列中的一个线程

#### 条件变量（Condition Variable）

条件变量依赖等待队列和信号机制：

1. `pthread_cond_wait`：**先释放互斥锁**，再将线程放入等待队列并睡眠
2. `pthread_cond_signal`/`pthread_cond_broadcast`：唤醒等待线程
3. 被唤醒的线程重新获取互斥锁，检查条件是否满足

## 4 pthread 实战案例分析

### 4.1 案例一：线程僵死问题

**问题场景：** 文件读取线程因 I/O 异常进入无限等待，导致线程僵死。

**错误代码：**

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

void *file_read_routine(void *arg) {
    int fd = open("/dev/exception_device", O_RDONLY);
    if (fd == -1) {
        perror("文件打开失败");
        return NULL;
    }
    char buffer[1024];
    // 无超时、无错误处理，异常时永久阻塞
    ssize_t ret = read(fd, buffer, sizeof(buffer));
    printf("读取完成，返回值：%ld\n", ret);
    close(fd);
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, file_read_routine, NULL);
    pthread_join(tid, NULL); // 永远阻塞
    return 0;
}
```

**修复方案：** 使用非阻塞 I/O + `select` 超时机制：

```c
void *file_read_safe_routine(void *arg) {
    int fd = open("/dev/exception_device", O_RDONLY | O_NONBLOCK);
    if (fd == -1) {
        perror("文件打开失败");
        return NULL;
    }

    struct timeval tv = {.tv_sec = 1, .tv_usec = 0};
    fd_set read_fds;
    FD_ZERO(&read_fds);
    FD_SET(fd, &read_fds);

    int ready = select(fd + 1, &read_fds, NULL, NULL, &tv);
    if (ready == -1) {
        perror("select 错误");
        close(fd);
        return NULL;
    } else if (ready == 0) {
        printf("读取超时，线程安全退出\n");
        close(fd);
        return NULL;
    }

    char buffer[1024];
    read(fd, buffer, sizeof(buffer));
    printf("文件读取成功\n");
    close(fd);
    return NULL;
}
```

### 4.2 案例二：死锁问题

**问题场景：** 银行转账中，两个线程以不同顺序获取锁，形成循环等待。

**错误代码（死锁）：**

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t accountA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t accountB = PTHREAD_MUTEX_INITIALIZER;

// 线程 1：A → B（先锁 A，再锁 B）
void *transfer_A2B(void *arg) {
    pthread_mutex_lock(&accountA);
    printf("线程 1：已锁住账户 A\n");
    sleep(1); // 放大死锁概率
    pthread_mutex_lock(&accountB); // 等待线程 2 释放 B → 死锁
    printf("线程 1：转账完成\n");
    pthread_mutex_unlock(&accountB);
    pthread_mutex_unlock(&accountA);
    return NULL;
}

// 线程 2：B → A（先锁 B，再锁 A）
void *transfer_B2A(void *arg) {
    pthread_mutex_lock(&accountB);
    printf("线程 2：已锁住账户 B\n");
    sleep(1);
    pthread_mutex_lock(&accountA); // 等待线程 1 释放 A → 死锁
    printf("线程 2：转账完成\n");
    pthread_mutex_unlock(&accountA);
    pthread_mutex_unlock(&accountB);
    return NULL;
}
```

**修复方案：** 统一锁获取顺序：

```c
// 所有线程都按固定顺序获取锁：先 A 后 B
void *transfer_safe(void *arg) {
    pthread_mutex_lock(&accountA);
    pthread_mutex_lock(&accountB);
    printf("线程：执行转账操作\n");
    pthread_mutex_unlock(&accountB);
    pthread_mutex_unlock(&accountA);
    return NULL;
}
```

### 4.3 案例三：并发调度异常

**问题场景：** 游戏服务器中，攻击操作依赖移动完成，但调度不确定性导致执行顺序错乱。

**修复方案：** 使用条件变量保证执行顺序：

```c
#include <stdio.h>
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int move_finished = 0;

// 线程：移动
void *move_thread(void *arg) {
    pthread_mutex_lock(&mutex);
    printf("玩家移动完成\n");
    move_finished = 1;
    pthread_cond_signal(&cond); // 唤醒等待的攻击线程
    pthread_mutex_unlock(&mutex);
    return NULL;
}

// 线程：攻击（必须等待移动完成）
void *attack_thread(void *arg) {
    pthread_mutex_lock(&mutex);
    while (!move_finished) {
        pthread_cond_wait(&cond, &mutex);
    }
    printf("玩家发起攻击\n");
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

## 5 pthread 实战避坑指南

### 5.1 线程创建与资源管理

**（1）线程创建失败常见原因**

| 错误码 | 原因 | 解决方式 |
|--------|------|----------|
| `EAGAIN` | 系统资源不足，达到最大线程数 | `ulimit -u` 调整限制 |
| `ENOMEM` | 内存不足 | 优化内存使用 |
| `EINVAL` | 线程属性设置错误 | 检查 attr 参数 |

**（2）栈空间配置问题**

栈空间过小或递归过深会导致**栈溢出**。通过 `pthread_attr_setstacksize` 调整：

```c
#include <stdio.h>
#include <pthread.h>

void *thread_function(void *arg) {
    // 线程函数内容
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    size_t stack_size = 1024 * 1024; // 设置栈大小为 1MB
    pthread_attr_setstacksize(&attr, stack_size);

    if (pthread_create(&thread, &attr, thread_function, NULL) != 0) {
        perror("pthread_create");
        return 1;
    }
    pthread_join(thread, NULL);
    pthread_attr_destroy(&attr);
    return 0;
}
```

**（3）线程分离与连接误区**

- **`pthread_detach`**：不关心返回值，线程结束后自动释放资源（后台线程、监控线程）
- **`pthread_join`**：需要获取返回值，等待线程结束（计算任务、数据处理）

**注意：** 对已分离的线程调用 `pthread_join` 会返回 `EINVAL` 错误。

### 5.2 线程同步与互斥

**（1）锁争用问题**

减少锁争用的策略：
- **缩小临界区范围**：只在访问共享资源时加锁
- **读写锁**：读多写少场景使用 `pthread_rwlock_t`，允许多线程并发读
- **无锁数据结构**：如 `std::atomic`，通过硬件原子指令避免锁

```c
// 缩小临界区示例
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int shared_variable = 0;

void *thread_function(void *arg) {
    for (int i = 0; i < 1000; ++i) {
        pthread_mutex_lock(&mutex);
        shared_variable++; // 只在操作共享变量时加锁
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

**（2）死锁的产生与避免**

避免死锁的方法：
1. **按固定顺序获取锁**：所有线程以相同顺序获取多个锁
2. **超时锁**：`pthread_mutex_timedlock` 设置获取超时
3. **避免嵌套锁**：减少同时持有多个锁的场景

```c
// 超时锁示例
#include <pthread.h>
#include <stdio.h>
#include <time.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *thread_function(void *arg) {
    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += 2; // 2 秒超时

    if (pthread_mutex_timedlock(&mutex, &timeout) == 0) {
        printf("线程获取到锁\n");
        pthread_mutex_unlock(&mutex);
    } else {
        printf("线程获取锁超时\n");
    }
    return NULL;
}
```

**（3）条件变量使用不当**

正确使用条件变量的原则：
- 调用 `pthread_cond_wait` 前必须先获取互斥锁
- **使用 `while` 循环检查条件**（防止虚假唤醒），而非 `if`

```c
pthread_mutex_lock(&mutex);
while (condition_is_false) {
    pthread_cond_wait(&cond, &mutex);
}
// 执行条件满足后的操作
pthread_mutex_unlock(&mutex);
```

### 5.3 线程调度与优先级

**（1）调度策略**

| 策略 | 特点 | 适用场景 |
|------|------|----------|
| `SCHED_FIFO` | 先来先服务，不主动让出则一直运行 | 实时性要求极高的任务 |
| `SCHED_RR` | 时间片轮转，用完后放入队列末尾 | 公平性要求较高的场景 |
| `SCHED_OTHER` | 默认策略，CFS 调度 | 普通非实时任务 |

**（2）优先级设置陷阱**

- 优先级范围因调度策略而异，用 `sched_get_priority_min/max` 查询合法范围
- 设置 `SCHED_FIFO`/`SCHED_RR` 高优先级通常需要 root 权限或 `CAP_SYS_NICE` 能力

```c
#include <pthread.h>
#include <sched.h>
#include <stdio.h>

void *thread_function(void *arg) {
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_attr_t attr;
    struct sched_param param;
    pthread_attr_init(&attr);

    int max_priority = sched_get_priority_max(SCHED_FIFO);
    param.sched_priority = max_priority;

    pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
    pthread_attr_setschedparam(&attr, &param);

    if (pthread_create(&thread, &attr, thread_function, NULL) != 0) {
        perror("pthread_create failed");
        return 1;
    }
    pthread_join(thread, NULL);
    pthread_attr_destroy(&attr);
    return 0;
}
```

## 6 附录：经典面试题解析

### 6.1 线程和进程的区别，以及 pthread 线程的特点？

| 维度 | 进程 | 线程 |
|------|------|------|
| 资源分配 | 独立地址空间、文件描述符等 | 共享进程资源 |
| 调度开销 | 上下文切换开销大（需切换页表等） | 开销小（只切换栈指针、寄存器） |
| 通信方式 | IPC（管道、消息队列、共享内存等） | 直接访问共享变量 |
| 隔离性 | 相互隔离，崩溃不影响其他进程 | 共享资源，一个线程出错可能影响整个进程 |

**pthread 线程特点：**
- 基于 POSIX 标准，跨平台可移植
- 创建/销毁开销小，上下文切换迅速
- 提供丰富的线程管理 API

### 6.2 如何解决线程安全问题？

当多个线程同时访问共享资源时，可能出现数据不一致。常用同步机制：**互斥锁**、**条件变量**、**信号量**。

**示例：银行转账（互斥锁保护）：**

```c
#include <stdio.h>
#include <pthread.h>

typedef struct {
    int balance;
    pthread_mutex_t mutex;
} Account;

void initAccount(Account *account, int initialBalance) {
    account->balance = initialBalance;
    pthread_mutex_init(&account->mutex, NULL);
}

void transfer(Account *from, Account *to, int amount) {
    pthread_mutex_lock(&from->mutex);
    pthread_mutex_lock(&to->mutex);
    if (from->balance >= amount) {
        from->balance -= amount;
        to->balance += amount;
        printf("转账 %d 成功\n", amount);
    } else {
        printf("余额不足\n");
    }
    pthread_mutex_unlock(&to->mutex);
    pthread_mutex_unlock(&from->mutex);
}
```

### 6.3 条件变量的原理、使用场景和虚假唤醒

**原理：** 线程等待某个条件满足，不满足则阻塞休眠；其他线程修改条件后主动唤醒等待线程。

**核心 API：**
- `pthread_cond_wait`：阻塞等待，**自动释放锁，唤醒后自动重新获取锁**
- `pthread_cond_signal`：唤醒一个等待线程
- `pthread_cond_broadcast`：唤醒所有等待线程

**典型场景：** 生产者-消费者模型

```c
#include <stdio.h>
#include <pthread.h>

#define BUFFER_SIZE 5
int buffer[BUFFER_SIZE];
int in = 0, out = 0;

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond_producer = PTHREAD_COND_INITIALIZER;
pthread_cond_t cond_consumer = PTHREAD_COND_INITIALIZER;

void *producer(void *arg) {
    int item = 1;
    while (1) {
        pthread_mutex_lock(&mutex);
        while ((in + 1) % BUFFER_SIZE == out) {
            pthread_cond_wait(&cond_producer, &mutex); // 缓冲区满，等待
        }
        buffer[in] = item;
        printf("Produced: %d\n", item);
        in = (in + 1) % BUFFER_SIZE;
        pthread_cond_signal(&cond_consumer);
        pthread_mutex_unlock(&mutex);
        item++;
    }
    return NULL;
}

void *consumer(void *arg) {
    while (1) {
        pthread_mutex_lock(&mutex);
        while (in == out) {
            pthread_cond_wait(&cond_consumer, &mutex); // 缓冲区空，等待
        }
        int item = buffer[out];
        printf("Consumed: %d\n", item);
        out = (out + 1) % BUFFER_SIZE;
        pthread_cond_signal(&cond_producer);
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

**虚假唤醒：** 操作系统调度机制可能导致线程在没有 `signal`/`broadcast` 时被意外唤醒。**必须使用 `while` 循环检查条件**来防止虚假唤醒导致的逻辑错误。

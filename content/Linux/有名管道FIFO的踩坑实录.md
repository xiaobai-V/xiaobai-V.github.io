---
created: 2026-05-20
updated: 2026-05-20
tags:
  - 嵌入式Linux
  - IPC
  - 有名管道
related:
number headings: first-level 2, start-at 1, max 3, 1.1, auto, contents toc
---

<font color="#f79646">TLDR：</font>




---

# Linux 命名管道（FIFO）踩坑实录

> 从 fork 到独立进程，从死锁到优雅通信，记录学习 FIFO 过程中遇到的每一个坑。

## 1 背景

学习 Linux 进程间通信时，从匿名管道（`pipe`）过渡到命名管道（`mkfifo`）。FIFO 的核心优势是**允许无亲缘关系的进程通信**——它以文件形式存在于文件系统中，任何知道路径的进程都能打开它。


按照学习路径，我先在 `08_Fifo` 中用 `fork` 模拟父子进程通信，然后在 `09_Fifo_Write` / `09_Fifo_Read` 中拆分为两个独立进程。以下是踩过的坑。

---

## 2 坑 1：用 O_RDWR 打开 FIFO 导致死锁

**现象**：父进程用 `O_RDWR` 打开 FIFO 后，程序卡住不动。

```c
// ❌ 错误写法
fd = open(MY_FIFO_PATH, O_RDWR);
```

**原因**：FIFO 的 `open` 是**阻塞**的，内核会等待读端和写端**同时就绪**才返回。用 `O_RDWR` 打开意味着当前进程同时充当读端和写端，内核认为两端都已就绪，`open` 立即返回。但后续的 `read`/`write` 行为会变得不可预测，容易陷入死锁。

**修复**：明确角色，读端用 `O_RDONLY`，写端用 `O_WRONLY`。

```c
// ✅ 正确写法
// 读端
fd = open(MY_FIFO_PATH, O_RDONLY);
// 写端
fd = open(MY_FIFO_PATH, O_WRONLY);
```

**原则**：FIFO 是单向通道，打开时必须明确自己是读者还是写者。

---

## 3 坑 2：read 缓冲区没有给 '\0' 留位置

**现象**：读端打印出来的内容末尾有乱码。

```c
// ❌ 错误写法：缓冲区 10 字节，read 最多读 10 字节，没有空间放 '\0'
char read_buf[10];
ssize_t bytes_read = read(fd, read_buf, sizeof(read_buf));
// 手动补 '\0' 时越界：read_buf[10] = '\0'
```

**原因**：`read` 不关心字符串结尾，它只是搬运原始字节。如果缓冲区全部填满，手动追加 `'\0'` 就会越界写入。

**修复**：读取时少读一个字节，给 `'\0'` 预留位置。

```c
// ✅ 正确写法
char read_buf[10];
ssize_t bytes_read = read(fd, read_buf, sizeof(read_buf) - 1); // -1 给 '\0' 留位
read_buf[bytes_read] = '\0';
```

**原则**：`read` 是系统调用，不是 `fgets`，它不会自动添加 `'\0'`。

---

## 4 坑 3：缓冲区太小导致数据截断

**现象**：发送 `"a message from fifo\n"`（20 字节），但缓冲区只有 10 字节，第一次只读到了 `"a message"`。

**原因**：FIFO 内部有内核缓冲区（通常是 64KB），`write` 一次写入的数据可能需要 `read` 多次才能读完。这不是 bug，是 FIFO 的正常行为——它是**字节流**，不保证一次 write 对应一次 read。

**修复**：循环读取，直到 `read` 返回 0（对端关闭）。

```c
// ✅ 正确写法：循环读取
ssize_t bytes_read;
while ((bytes_read = read(fd, read_buf, sizeof(read_buf) - 1)) > 0)
{
    read_buf[bytes_read] = '\0';
    printf("读取到%zd字节：%s\n", bytes_read, read_buf);
}
```

**原则**：永远不要假设一次 `read` 能读完所有数据，必须循环读取。

---

## 5 坑 4：write 时要不要 +1 把 '\0' 也写入？

**结论：不需要。**

```c
// ✅ 正确：不需要 +1
write(fd, msg, strlen(msg));
```

FIFO 传输的是**原始字节流**，不是 C 字符串。`read` 返回值告诉你读了多少字节，读端自己加 `'\0'` 即可。写入 `'\0'` 反而浪费一个字节，还会导致读端多读一个不可见字符。

---

## 6 坑 5：拆分为独立进程后——谁来创建/删除 FIFO？

从 `08_Fifo`（fork 模式）过渡到 `09_Fifo_Write` / `09_Fifo_Read`（独立进程模式）时，第一个问题是：**FIFO 文件由谁创建、由谁清理？**

### 6.1 创建：写入端和读取端都可以

```c
// 容忍已存在的情况（errno == EEXIST 则忽略）
if (mkfifo(MY_FIFO_PATH, 0666) == -1 && errno != EEXIST)
{
    perror("mkfifo");
    return 1;
}
```

两端都用这种方式创建，先启动的一方负责实际创建，后启动的一方因为 `EEXIST` 直接跳过。这样无论先启动读端还是写端，都不会出错，非常灵活。

### 6.2 删除：写入端负责

写入端清楚何时不再发送数据，由它在关闭前 `unlink` 清理。

```c
close(fd);
unlink(MY_FIFO_PATH); // 写入端负责删除
```

### 6.3 启动顺序：无所谓

- 读取端先启动：`open(O_RDONLY)` 阻塞，等待写入端
- 写入端先启动：`open(O_WRONLY)` 阻塞，等待读取端

两端相互等待，谁先启动都行。

---

## 7 坑 6：读取端如何检测写入端关闭？

当写入端 `close(fd)` 后，读取端的 `read` 会返回 `0`——这不是错误，而是 EOF 信号。

```c
int bytes_read = read(fd, buf, sizeof(buf) - 1);
if (bytes_read == 0)
{
    // 写入端已关闭 FIFO
    printf("发送端已经关闭fifo\n");
    break;
}
```

**注意**：如果写入端没有关闭（比如还在循环写入），`read` 会一直阻塞等待新数据。这是正常的阻塞行为，不是死锁。

---

## 8 总结

| 坑 | 根因 | 解决方案 |
|----|------|----------|
| O_RDWR 死锁 | FIFO 需要读写两端分别就绪 | 读端 `O_RDONLY`，写端 `O_WRONLY` |
| 末尾乱码 | `read` 不自动补 `'\0'` | 缓冲区预留 1 字节，手动补 `'\0'` |
| 数据截断 | FIFO 是字节流，不保证一次读完 | 循环 `read` 直到返回 0 |
| '\0' 是否写入 | FIFO 传字节流不是字符串 | `write` 不需要 `+1` |
| FIFO 谁创建 | 两端都可能先启动 | 两端都 `mkfifo` + 忽略 `EEXIST` |
| 检测对端关闭 | 不知道何时停止读取 | `read` 返回 0 即 EOF |

**核心认知**：FIFO 是内核管理的字节流管道，不是文件。它有阻塞语义，有缓冲区大小限制，需要明确的读写角色分工。理解了这些，踩过的坑都是通向正确用法的路标。

---

*源码参考：[xiaobai-V/KelpBar: 基于小智学长的KelpBar智慧屏项目，个人代码和笔记](https://github.com/xiaobai-V/KelpBar)
`08_Fifo/`（fork 模式）、`09_Fifo_Write/` + `09_Fifo_Read/`（独立进程模式）*

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


## 1 有名管道FIFO简介

不同于匿名管道pipe，有名管道fifo可以通过在文件系统中创建一个文件描述符，通过文件描述符就可以实现父子进程或者独立进程之间的半双工通信。

这么一听是不是很简单，linux一切皆文件嘛，既然fifo已经抽象成一个文件描述符了，岂不是随便读写就可以通信了？

不着急，先来复习一遍FIFO的API
创建有名管道：`mkfifo`
```c
#include <sys/types.h>
#include <sys/stat.h>
int mkfifo(const char *pathname, mode_t mode);
```
`pathname`：fifo的路径名称
`mode`：权限，比如`0666`



但是问题是，假如有两个进程，谁来创建，谁来删除，如果还没写入就读取了会怎么样？如果
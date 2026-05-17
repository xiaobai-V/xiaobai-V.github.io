---
title: sizeof vs strlen：从一次 fwrite 踩坑说起
date: 2026-05-17
tags:
  - C语言
  - Linux系统编程
  - 文件IO
  - 踩坑记录
categories:
  - 嵌入式Linux
number headings: first-level 2, start-at 1, max 3, 1.1, auto, contents toc
---

## 1 起因

写文件练习时，用 `fwrite` 向文件写入字符串数组，写入成功，但用 VS Code 打开文件时报错：

> 此文件是二进制文件或使用了不受支持的文本编码

排查后发现，问题出在 `fwrite` 的第三个参数。

## 2 问题代码

```c
char write_buf[] = "Hello world!\n";

// 错误写法
fwrite(write_buf, sizeof(char), sizeof(write_buf), fp);

// 正确写法
fwrite(write_buf, sizeof(char), strlen(write_buf), fp);
```

就这一个参数的区别。

## 3 根因分析

### 3.1 sizeof vs strlen

对于字符数组 `char write_buf[] = "Hello world!\n"`：

| 表达式 | 值 | 含义 |
|--------|-----|------|
| `sizeof(write_buf)` | 14 | 数组总大小，包含末尾的 `\0` |
| `strlen(write_buf)` | 13 | 字符串长度，不含 `\0` |

`"Hello world!\n"` 有 13 个可见字符，但 C 字符串末尾会自动追加 `\0`，所以数组实际占 14 字节。

关键区别：

- `sizeof` 是编译期运算符，返回数组占用的**总字节数**（含 `\0`）
- `strlen` 是运行时函数，逐个字符计数直到遇到 `\0` 为止，**不包含 `\0`**

### 3.2 fwrite 参数含义

```c
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

- `ptr`：数据缓冲区指针
- `size`：每个数据块的字节数
- `nmemb`：**数据块**的个数
- 总写入字节数 = `size × nmemb`
- return：成功写入的**数据块的个数**，注意不是字节数

代码中 `size = sizeof(char) = 1`，所以 `nmemb` 的值就等于写入的字节数。传 `sizeof(write_buf)` = 14，比实际字符串多写了一个 `\0`。

### 3.3 \0 对文件的影响

用 `xxd` 查看写入的文件内容：

```
00000000: 4865 6c6c 6f20 776f 726c 6421 0a00       Hello world!..
```

最后一个字节 `00` 就是多余的 `\0`。

实测验证，`fopen` + `fread` 可以完整读取含 `\0` 的文件，14 字节一个不差。`\0` 只是一个值为 0 的字节，不会损坏文件结构。真正受影响的是文本编辑器——VS Code 检测到文件中存在 `\0` 字节，判定为二进制文件，拒绝以文本模式显示。

## 4 fwrite 返回值

排查过程中发现教程中的一个错误：

```c
// 错误写法：fwrite 返回 size_t（无符号），永远不会等于 -1
if (blocks_write == -1) { ... }

// 正确写法：检查实际写入块数是否小于预期
if (blocks_write < expected_count) { ... }
```

`fwrite` 返回 `size_t`，是无符号整数。`(size_t)-1` 实际会被解释为 `18446744073709551615`（即 `SIZE_MAX`），所以 `blocks_write == -1` 永远为假，写入失败的情况永远不会被捕获。

这一点要和 Linux 系统调用 `write()` 区分：`write()` 返回 `ssize_t`（有符号），出错时确实返回 `-1`。`fwrite` 是 C 标准库函数，两者返回值类型不同，判断方式不能照搬。

## 5 面试要点速查

| 考点                   | 要点                                                        |
| -------------------- | --------------------------------------------------------- |
| `sizeof` vs `strlen` | `sizeof` 含 `\0`，`strlen` 不含；`sizeof` 编译期求值，`strlen` 运行时遍历 |
| `fwrite` 第三参数        | 是块数（nmemb），不是总字节数；总写入 = `size × nmemb`                    |
| `fwrite` 返回值         | `size_t`（无符号），不能用 `== -1` 判断错误；应与期望写入块数比较                 |
| `\0` 写入文件            | 文件本身合法，`fread` 可正常读取；文本编辑器会误判为二进制                         |

## 6 参考代码

```c
#include <stdio.h> // 包含标准输入输出函数
// #include <fcntl.h>  // 包含 open() 函数的声明
// #include <unistd.h> // 包含 close() 函数的声明
#include <string.h>

int main()
{
    char write_buf[] = "Hello world!\n";

    FILE *fp = fopen("example.txt", "w+");
    if (fp == NULL)
    {
        printf("打开文件失败\n");
        return 1;
    }
    // 写入文件
    // size_t blocks_write = fwrite(write_buf, sizeof(char), sizeof(write_buf), fp);
    size_t len = strlen(write_buf);
    size_t blocks_write = fwrite(write_buf, sizeof(char), len, fp);
    if (blocks_write != len)
    {
        printf("期望写入%zu个数据块，实际写入%zu个数据块\n", len, blocks_write);
        fclose(fp);
        return 1;
    }
    else
    {
        printf("成功写入 %zu 个数据块\n", blocks_write);
    }

    // 定位光标

    // 读取文件

    // 关闭文件
    if (fclose(fp) != 0)
    {
        printf("关闭文件失败\n");
    }

    return 0;
}
```

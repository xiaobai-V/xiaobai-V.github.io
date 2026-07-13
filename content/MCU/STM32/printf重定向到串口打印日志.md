---
title: printf重定向到串口打印日志
date: 2026-07-08
description: 在 GCC 和 MDK 两种工具链下把 printf 重定向到串口，用一个 #ifdef 统一实现
tags:
  - STM32
  - UART
  - printf
  - 调试日志
draft: false
number headings: first-level 2, start-at 1, max 3, 1.1, auto, contents toc
---

## 1 为什么要重定向

`printf` 默认往 `stdout` 输出，底层调 `_write`（GCC）或 `fputc`（MDK）。MCU 没有 `stdout`，也没有终端，所以默认情况下 `printf` 没地方输出。

重定向就是劫持这个底层入口，把字符塞给 `HAL_UART_Transmit`，`printf` 就能从串口打出来。比手写 log 函数省事，还能用格式化。

## 2 两条工具链，两个入口

| 工具链 | 调用链 | 你要实现的函数 |
|---|---|---|
| CMake / GCC | `printf → _write → __io_putchar → HAL_UART_Transmit` | `__io_putchar` |
| MDK / ARMCC | `printf → fputc → HAL_UART_Transmit` | `fputc` |

- **GCC**：newlib 的 `syscalls.c` 里有 `_write()` 框架，遍历字符串逐字符调 `__io_putchar`。这个 `__io_putchar` 是 `__weak` 弱符号，写个强符号覆盖它就行。
- **MDK**：ARM C 库 / MicroLIB 的标准 retarget 入口就是 `fputc`。

## 3 统一实现

一个 `#ifdef` 同时搞定两种工具链：

```c
#include <stdio.h>

// 两种工具链的 retarget 入口不一样，用宏统一
#ifdef __GNUC__
#define PUTCHAR_PROTOTYPE int __io_putchar(int ch)
#else
#define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)
#endif

PUTCHAR_PROTOTYPE
{
    // huart1 换成你 CubeMX 配的串口句柄
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

放哪：
- GCC 工程：`usart.c`，`syscalls.c` 的 `_write` 会调到它。
- MDK 工程：`usart.c` 或 `main.c` 都行。

## 4 前提与注意事项

- **先配串口**：CubeMX 里开 USART1（或你要用的串口）并生成代码，`huart1` 才存在；记得调 `MX_USART1_UART_Init()` 初始化。
- **改句柄**：代码里是 `&huart1`，用别的串口就改成 `&huart2` 之类。
- **MDK 勾 MicroLIB**：Options → Target → 勾 `Use MicroLIB`。不勾的话标准库 `printf` 体积大，浮点（`%f`）可能不支持或链接报错。
- **GCC 的 `_write` 要通**：`syscalls.c` 的 `_write` 得调到 `__io_putchar`（CMake / CubeMX 模板一般自带）。如果它直接返回或没调，`printf` 就没输出。

## 5 要点速查

| 项 | 要点 |
|---|---|
| GCC 入口 | `__io_putchar`（覆盖 `__weak` 同名函数） |
| MDK 入口 | `fputc` |
| 统一手段 | `#ifdef __GNUC__` 切换函数原型 |
| 串口句柄 | `huart1` 换成自己 CubeMX 配的 |
| MDK | 勾 MicroLIB，否则体积大 / `%f` 可能不行 |
| GCC | `syscalls.c` 的 `_write` 要调到 `__io_putchar` |

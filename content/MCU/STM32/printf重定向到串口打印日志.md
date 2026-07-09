---
title: printf重定向到串口打印日志
description: 基于GCC和ARMCC两个版本
tags:
created: 2026-07-08
updated: 2026-07-08
number headings: first-level 1, start-at 1, max 3, 1.1, auto, contents toc, off
---
## Printf重定向串口

### CMake/GCC工程：

`printf `→ `_write `→ `__io_putchar` → `HAL_UART_Transmit`。

> `syscalls.c `中已有 `_write()` 遍历字符串逐字符调用 `__io_putchar` 的框架。该函数原本是 `__weak` 弱符号，我们提供的强符号会覆盖它。
### MDK ARM工程

函数名是 fputc，ARM C 库 / MicroLIB 的标准 retarget 入口，
`printf` → `fputc` → `HAL_UART_Transmit`。

```
#include <stdio.h>
// printf("text") → _write() [syscalls.c] → __io_putchar() [usart.c] → HAL_UART_Transmit(&huart1)
// syscalls.c 中已有 _write() 遍历字符串逐字符调用 __io_putchar 的框架
// 该函数原本是 __weak 弱符号，我们提供的强符号会覆盖它。
#ifdef __GNUC__
#define PUTCHAR_PROTOTYPE int __io_putchar(int ch)
#else
#define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)
#endif

PUTCHAR_PROTOTYPE
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```
---
title: printf重定向到串口打印日志
description: 基于GCC和ARMCC两个版本
tags:
created: 2026-07-08
updated: 2026-07-08
number headings: first-level 1, start-at 1, max 3, 1.1, auto, contents toc
---
# 1 Printf重定向串口
## 1.1 CMake/GCC工程

 CMake/GCC 工程：printf → Newlib 的 _write()（syscalls.c (Core/Src/syscalls.c)）→ __io_putchar()（usart.c (Core/Src/usart.c)）→ HAL_UART_Transmit

在 `usart.c`尾部`/* USER CODE BEGIN 1 */`添加如下代码
```C
/* USER CODE BEGIN 1 */
#include <stdio.h>
// printf("text") → _write() [syscalls.c] → __io_putchar() [usart.c] → HAL_UART_Transmit(&huart1)
// syscalls.c 中已有 _write() 遍历字符串逐字符调用 __io_putchar 的框架
// 该函数原本是 __weak 弱符号，我们提供的强符号会覆盖它。
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 1 */
```


## 1.2 MDK工程

MDK/ARM
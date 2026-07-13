---
title: STM32程序的启动过程
description:
tags:
created: 2026-07-13
updated: 2026-07-13
number headings: first-level 1, start-at 1, max 3, 1.1, auto, contents toc
---

# 1 程序启动流程

## 1.1 ARM-CM3处理器设计结构-**哈弗架构**

![](../../image/STM32/STM32程序的启动过程/STM32程序的启动过程-1783910558790.webp)

## 1.2 M3 内核支持的指令
编译工具链完成
芯片上电触发复位异常
中断向量表，偏移

![](../../image/STM32/STM32程序的启动过程/STM32程序的启动过程-1783910621985.webp)

复位异常最高等级

![](../../image/STM32/STM32程序的启动过程/STM32程序的启动过程-1783910738271.webp)

复位异常调转到中断向量表的特定偏移位置，获取里面的内容执行
在0x04取址
![](../../image/STM32/STM32程序的启动过程/STM32程序的启动过程-1783910762585.webp)
# 2 启动文件解析
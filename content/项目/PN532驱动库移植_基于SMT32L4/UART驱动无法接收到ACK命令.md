---
title:
description: 记录排查PN532驱动bug的过程
tags:
created: 2026-06-28
updated: 2026-06-28
number headings: first-level 1, start-at 1, max 3, 1.1, auto, contents toc, off
---


SPI和I2C测试比较顺利

UART驱动遇到无法接收到ACK的问题

阅读《PN532 User Manual》6.3 Handshake mechanism，怀疑以下问题：

## PN532处于LowVbat模式 

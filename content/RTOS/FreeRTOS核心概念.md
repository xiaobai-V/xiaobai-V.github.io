---
title: FreeRTOS 核心概念
date: 2025-05-10
tags: [RTOS, FreeRTOS, 嵌入式]
description: FreeRTOS 任务、内存、同步机制的核心要点
---

# FreeRTOS 核心概念

裸机用中断 + 超级循环，复杂了就得上 RTOS。FreeRTOS 是最流行的选择。

## 任务状态

```
Running ←→ Ready
  ↓           ↑
Blocked → Ready (事件完成)
```

- **Running**：正在执行，单核同一时刻只有一个
- **Ready**：就绪等调度
- **Blocked**：等待事件（延时、队列、信号量）

## 调度规则

- 优先级数字越大越高
- 同优先级时间片轮转
- 核心规则：**永远运行最高优先级的就绪任务**
- 高优先级不让出 CPU → 低优先级饿死

### 关键 API

| API | 用途 |
|-----|------|
| `xTaskCreate()` | 动态创建任务 |
| `xTaskCreateStatic()` | 静态创建，自配内存 |
| `vTaskDelay()` | 相对延时 |
| `vTaskDelayUntil()` | 绝对延时，精确定周期 |

> 周期任务用 `vTaskDelayUntil`，不受任务执行时间影响。

## 内存管理

FreeRTOS 不用 malloc/free，提供 5 种 heap 方案：

| 方案 | 特点 | 场景 |
|------|------|------|
| heap_1 | 只分配不释放 | 极简 |
| heap_2 | 最佳适配，不合并 | 固定块大小 |
| **heap_4** | **首次适配 + 合并空闲块** | **通用推荐** |
| heap_5 | heap_4 + 多内存区 | 不连续 RAM |

## 同步与通信

| 机制 | 用途 |
|------|------|
| **队列** | 任务间传数据，FIFO，线程安全 |
| **信号量** | 二值：同步/互斥；计数：资源管理 |
| **互斥量** | 带**优先级继承**的互斥锁 |

### 优先级翻转

```
低任务 L 拿锁 → 中任务 M 抢占 → 高任务 H 等锁（被 M 间接阻塞）
```

Mutex 的优先级继承会把 L 临时升到 H 的优先级，尽快释放。二值信号量没这机制——**互斥用 Mutex，不用 Semaphore**。

### 任务通知

FreeRTOS 轻量机制，比队列快 45%：
- 每个任务一个 32 位通知值
- 可当轻量信号量/事件组用
- 限制：只能一对一

## 实战建议

1. `uxTaskGetStackHighWaterMark()` 查栈剩余，**栈大小要实测**
2. 中断里用 `FromISR` 后缀的 API
3. 共享资源必须保护：Mutex 或关中断
4. configASSERT 要打开，开发阶段帮定位问题

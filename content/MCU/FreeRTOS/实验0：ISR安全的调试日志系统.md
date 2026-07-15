---
title:
description:
tags:
number headings: first-level 2, start-at 1, max 3, 1.1, auto, contents toc
---
## 1 基础环境

本实验基于 `FreeRTOS_Start` 模板工程，已具备以下条件：

| 已有设施 | 说明 |
|----------|------|
| LED 任务 | `LEDTask` (osPriorityNormal, 512B 栈)，每 500ms 翻转蓝色 LED (PE9) |
| 串口任务 | `SerialTask` (osPriorityLow, 512B 栈)，每 1s 通过 printf 输出 |
| printf 重定向 | `usart.c` 中 `__io_putchar()` → `HAL_UART_Transmit(&huart1, ..., HAL_MAX_DELAY)` |
| USART1 | 115200-8N1, PA9-TX / PA10-RX |
| FreeRTOS | V10.3.1, heap_4, 抢占式调度 |

## 2 问题描述：为什么不直接用 printf

`printf` 在多任务 + 中断环境中存在三个致命问题：

| 问题 | 原因 | 后果 |
|------|------|------|
| **不可重入** | 内部使用全局 FILE 状态 | 中断打断任务中的 printf → 输出损坏 |
| **阻塞** | `HAL_UART_Transmit` 轮询发送，115200 下 1 字节 ≈ 87us | ISR 中调用阻塞 → 系统卡死 |
| **栈消耗大** | printf 浮点格式化需 1-2KB 栈 | ISR 栈固定（链接脚本 MSP = 1KB），可能溢出 |

直接结论：**printf 只能在一个地方调用，且不能在 ISR 中调用。**

## 3 解决方案：生产者-消费者模型

核心思路：将"产生日志"和"输出日志"解耦。

```
┌──────────────┐    ┌──────────────┐
│  任务上下文    │    │  ISR 上下文   │
│  LOG_INFO()  │    │  LOG_INFO()  │
└──────┬───────┘    └──────┬───────┘
       │                    │
       │ ① snprintf 到本地   │ ① snprintf 到本地
       │ ② 临界区保护写入    │ ② 直接写入（ISR 天然互斥）
       │    环形缓冲区        │    环形缓冲区
       │ ③ xTaskNotifyGive   │ ③ vTaskNotifyGiveFromISR
       ▼                    ▼
  ┌─────────────────────────────────┐
  │        Ring Buffer (512B)        │
  └───────────────┬─────────────────┘
                  │ ④ 任务通知唤醒
                  ▼
  ┌─────────────────────────────────┐
  │   LogTask (优先级 1, 4096B 栈)    │
  │   ulTaskNotifyTake 阻塞等待      │
  │   → 临界区读取缓冲区              │
  │   → printf 输出到串口             │
  └─────────────────────────────────┘
```

**关键约束**：printf 仅在 LogTask 中调用，ISR 和其他任务只写缓冲区。

## 4 关键实现解析

### 4.1 环形缓冲区

```
SIZE = 512 (2^9)

     tail                    head
      │                       │
  [x][x][x][x][ ][ ][ ][ ][ ][ ]
       │←── 已用 ──→│←── 空闲 ──→│

实际位置 = index & (SIZE - 1)    // 位掩码取模，等价于 % SIZE 但更快
已用空间 = head - tail           // uint16_t 自然溢出，数学上正确
空闲空间 = SIZE - 已用空间
```

**为什么不用 FreeRTOS 队列？**

| 环形缓冲区 | FreeRTOS 队列 |
|-----------|--------------|
| 逐字节写入，天然支持变长字符串 | 每项固定大小，变长数据需要封装 |
| 纯数组操作，无动态分配 | 内部是链表 + 拷贝 |
| ISR 中绝对安全 | 需要使用 `FromISR` 变体 |
| 满时丢弃（调试日志可接受） | 满时阻塞或返回失败 |

### 4.2 ISR 上下文检测

```c
uint32_t ipsr = __get_IPSR();  // 读 IPSR 寄存器，编译为单条 MRS 指令

if (ipsr != 0U) {
    // ISR 路径：直接写缓冲区 + vTaskNotifyGiveFromISR
} else {
    // 任务路径：临界区写缓冲区 + xTaskNotifyGive
}
```

`__get_IPSR()` 是 CMSIS 内置函数，读取 Cortex-M 的 **中断程序状态寄存器**：
- 线程模式（任务执行）= 0
- 中断模式（ISR 执行）= 异常号（非零）

零运行时开销，编译为单条 `MRS r0, IPSR` 指令。

### 4.3 临界区 vs ISR 保护

| 场景 | 保护方式 | 原理 |
|------|---------|------|
| 任务写缓冲区 | `taskENTER_CRITICAL()` | 设置 BASEPRI 寄存器，屏蔽优先级 ≥ 5 的中断 |
| ISR 写缓冲区 | 无需保护 | ISR 执行期间任务被挂起，天然互斥 |
| LogTask 读缓冲区 | `taskENTER_CRITICAL()` | 防止读取过程中被写入端修改 |

`taskENTER_CRITICAL()` 在 Cortex-M 上的实现：

```
vPortEnterCritical():
    mov r0, #80          // configMAX_SYSCALL_INTERRUPT_PRIORITY << 4 = 0x50
    msr BASEPRI, r0      // 屏蔽优先级 ≥ 5 的中断
    // 优先级 < 5 的中断不受影响（硬件关键中断仍可响应）
```

### 4.4 任务通知

本实验使用任务通知作为 LogTask 的唤醒机制：

| API | 上下文 | 作用 |
|-----|--------|------|
| `xTaskNotifyGive(handle)` | 任务 | 将 handle 的通知值 +1 |
| `vTaskNotifyGiveFromISR(handle, &woken)` | ISR | ISR 版本，+1 并检查是否需要上下文切换 |
| `ulTaskNotifyTake(pdTRUE, timeout)` | LogTask | 等待并清零通知值，或超时返回 0 |
| `portYIELD_FROM_ISR(woken)` | ISR 返回时 | 如果 woken == pdTRUE，ISR 返回后立即切换到高优先级任务 |

**为什么选任务通知而不是信号量？**

| 任务通知 | 二值信号量 |
|---------|-----------|
| 直接绑定到 TCB，无需创建对象 | 需要调用 `xSemaphoreCreateBinary()` |
| 无数据拷贝，O(1) | 内部创建队列结构体（一切皆队列） |
| RAM: 0 额外字节 | RAM: ~80 字节 |
| 限 1:1 通知（一个任务通知另一个） | 可多对多 |

任务通知是 FreeRTOS 中最轻量的同步原语。实验 5 会深入对比。

### 4.5 LogTask 设计

```c
static void LogTask(void *pvParameters) {
    char read_buf[512];
    for (;;) {
        ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(100)); // 阻塞等待或 100ms 超时

        taskENTER_CRITICAL();
        uint16_t count = ringbuf_read(&log_ring, read_buf, 511);
        taskEXIT_CRITICAL();

        if (count > 0) {
            read_buf[count] = '\0';
            printf("%s", read_buf);  // 只在这里调用 printf，无竞争
        }
    }
}
```

**100ms 超时的意义**：正常情况下 `xTaskNotifyGive` 唤醒。但如果极端情况通知丢失（理论上不应发生），超时确保缓冲区数据不会永远不被刷新。

**栈大小 1024 字 (4096 字节)**：printf + snprintf 的栈消耗很大，特别是浮点格式化时。实测需要 1-2KB，留足余量。

## 5 文件清单

| 文件 | 说明 |
|------|------|
| `Core/Inc/debug_log.h` | 宏定义、日志级别、API 声明 |
| `Core/Src/debug_log.c` | 环形缓冲区、LogTask、上下文检测、同步原语 |
| `Core/Src/freertos.c` | 调用 `DebugLog_Init()`，使用 `LOG_INFO` 输出 |

## 6 配置变更（相对模板工程）

| 参数 | 模板值 | 本实验值 | 原因 |
|------|--------|----------|------|
| `configTOTAL_HEAP_SIZE` | 8192 | 16384 | LogTask 栈 4096B + 其他任务栈 |
| `configMINIMAL_STACK_SIZE` | 128 | 1024 | 用户已调整 |
| `configTIMER_TASK_STACK_DEPTH` | 256 | 2048 | 用户已调整 |

## 7 使用方法

```c
#include "debug_log.h"

// 初始化（在 MX_FREERTOS_Init 最前面）
DebugLog_Init();

// 在任务中使用
LOG_INFO("Sensor value: %d", adc_val);
LOG_WARN("Queue full, dropping data");
LOG_ERROR("I2C timeout, addr=0x%02X", dev_addr);
LOG_DEBUG("tick=%lu, free_heap=%u", xTaskGetTickCount(), xPortGetFreeHeapSize());

// 在 ISR 中同样安全
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    LOG_INFO("Timer interrupt");  // 自动走 ISR 路径
}
```

**编译期过滤**：在 `FreeRTOSConfig.h` 或编译选项中定义 `DEBUG_LOG_LEVEL`：

```c
#define DEBUG_LOG_LEVEL  DEBUG_LOG_ERROR  // 只输出错误，Release 构建用
#define DEBUG_LOG_LEVEL  DEBUG_LOG_DEBUG  // 输出全部，Debug 构建用
```

不满足级别的 `DEBUG_LOG` 宏会被编译器完全消除（死代码优化），零运行时开销。

## 8 学到的 FreeRTOS 机制

| 机制 | API | 将在哪个实验深入 |
|------|-----|-----------------|
| 临界区 | `taskENTER_CRITICAL` / `taskEXIT_CRITICAL` | 实验 11（调度器机制） |
| 任务通知 | `xTaskNotifyGive` / `ulTaskNotifyTake` | 实验 5（任务通知） |
| ISR 安全 API | `vTaskNotifyGiveFromISR` / `portYIELD_FROM_ISR` | 实验 10（中断管理） |
| 任务创建 | `xTaskCreate`（原生 API） | 贯穿所有实验 |

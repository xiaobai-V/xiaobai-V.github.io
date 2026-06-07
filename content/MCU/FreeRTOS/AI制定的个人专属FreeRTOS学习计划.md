# 1 FreeRTOS 系统学习计划

## 1.1 目标

系统掌握 FreeRTOS 核心机制，能独立设计复杂多任务系统。不只是会用 API，而是真正理解调度、内存、同步、中断管理的底层原理。

## 1.2 学习方式

- **逐模块实验**：每个核心机制做一个小实验工程，隔离性好
- **三层深度**：每个实验都达到 "会用 API → 测边界行为 → 读源码"
- **导师模式**：AI 讲原理 + 指出源码关键位置 + 给实验要求 → 自己写代码 → 写完 review

## 1.3 基础设施

- **开发板**：正点原子潘多拉 STM32L496VET6 (Cortex-M4F, 80MHz, 512KB Flash, 320KB SRAM)
- **工具链**：CMake + ARM GCC + STM32CubeMX
- **模板工程**：`FreeRTOS_Start`（已验证 LED + 串口 printf + FreeRTOS）
- **调试系统**：`debug_log.h/c`（ISR 安全的 DEBUG_LOG 宏，实验 0 产物）

## 1.4 实验工程组织

每个实验从 `FreeRTOS_Start` 模板复制为独立工程，带入 `debug_log.h/c`。

```
STM32L496VET6/
├── FreeRTOS_Start/          # 模板（不动）
├── FreeRTOS_Pro/            # 综合项目（后续重构目标）
├── FreeRTOS_00_DebugLog/    # ✅ 已完成
├── FreeRTOS_01_Queue/       # ✅ 已完成
├── FreeRTOS_02_BinSem/	     # ✅ 已完成
├── FreeRTOS_03_CountSem/    # ✅ 已完成
├── FreeRTOS_04_Mutex/       # ✅ 已完成
├── FreeRTOS_05_TaskNotify/  # ✅ 已完成  
├── FreeRTOS_06_EventGroup/
├── FreeRTOS_07_StreamBuf/
├── FreeRTOS_08_SoftTimer/
├── FreeRTOS_09_StaticCreate/
├── FreeRTOS_10_Interrupt/
├── FreeRTOS_11_Scheduler/
├── FreeRTOS_12_MemMgmt/
└── FreeRTOS_13_RunTimeStats/
```

## 1.5 实验进度

### 1.5.1 ✅ 实验 0：DEBUG_LOG 宏系统
- 环形缓冲区 + LogTask + 任务通知 + 临界区 + ISR 检测
- 已验证：多任务交替输出不乱码，ISR 中安全调用
- 文档：`FreeRTOS_00_DebugLog/Docs/00_DebugLog.md`

### 1.5.2 ✅ 实验 1：队列（Queue）
- 核心 IPC 原语，"一切皆队列" 的基础
- 三个任务：Sender1(200ms) + Sender2(300ms) → Queue(深度2) → Receiver
- 边界测试：满队列阻塞/丢弃、空队列超时、多生产者竞争
- 源码重点：`queue.c` 的 `QueueDefinition` 结构体、`xQueueGenericSend`、`xQueueGenericReceive`
- 文档：`FreeRTOS_01_Queue/Docs/01_Queue.md`

### 1.5.3 ✅ 实验 2：二值信号量（Binary Semaphore）
- 理解 "一切皆队列"：深度 1、项大小 0 的特化队列
- 两阶段：任务间同步 + ISR→任务同步（KEY0 PD10 EXTI）
- 边界测试：多次 Give 只记录 1 次 → 引出计数信号量
- 文档：`FreeRTOS_02_BinSem/Docs/02_BinSem.md`

### 1.5.4 ✅ 实验 3：计数信号量（Counting Semaphore）
- 深度 N、项大小 0，解决"多次 Give 丢失"问题
- 两阶段：任务间同步（Producer Give×3 + Consumer Take 全部取到）+ ISR→任务同步（快速连按不丢事件）
- 边界测试：Producer 速率 > Consumer 速率时计数累积到 maxCount 后 Give 失败
- 文档：`FreeRTOS_03_CountSem/Docs/03_CountSem.md`

### 1.5.5 ✅ 实验 4：互斥量（Mutex）
- 优先级继承：解决高优先级被中优先级间接阻塞的问题
- 三任务演示：Low/Mid/High，对比互斥量(1101ms) vs 二值信号量(2001ms)
- 关键概念：所有权、HAL_Delay 忙等 vs vTaskDelay、初始状态差异
- 文档：`FreeRTOS_04_Mutex/Docs/04_Mutex.md`

### 1.5.6 ✅ 实验 5：任务通知（Task Notification）
- 零创建开销：通知值在 TCB 中，直接操作不经过队列
- 三阶段：计数模式(eIncrement) + 位通知(eSetBits) + ISR 通知
- 与实验 0 的联系：debug_log 已在使用 xTaskNotifyGive/ulTaskNotifyTake
- 文档：`FreeRTOS_05_TaskNotify/Docs/05_TaskNotify.md`

### 1.5.7 ⏳ 实验 6：事件组（Event Group）
- 多条件同步，位操作逻辑
- 典型场景：等待多个传感器都就绪后才启动处理

### 1.5.8 ⏳ 实验 7：流/消息缓冲区（Stream/Message Buffer）
- 队列的优化版本：拷贝 vs 指针传递
- 单生产者单消费者场景下的最佳选择

### 1.5.9 ⏳ 实验 8：软件定时器（Software Timer）
- 定时器命令队列、回调模型的限制
- 一次性和周期性定时器

### 1.5.10 ⏳ 实验 9：静态 vs 动态创建
- 内存确定性：`xTaskCreateStatic` + `xQueueCreateStatic`
- 静态分配需要用户提供内存

### 1.5.11 ⏳ 实验 10：中断管理
- `FromISR` 全家桶 + 延迟中断处理模式（Deferred Interrupt Handling）
- 中断优先级分组、嵌套

### 1.5.12 ⏳ 实验 11：调度器机制
- Tick 中断、时间片、同优先级轮转
- 上下文切换的底层实现（PendSV）
- 临界区实现（BASEPRI 寄存器）

### 1.5.13 ⏳ 实验 12：内存管理
- heap_1 ~ heap_5 对比
- heap_4 源码精读：空闲块合并算法

### 1.5.14 ⏳ 实验 13：调试手段
- `vTaskList()`：任务状态列表
- `vTaskGetRunTimeStats()`：各任务 CPU 占用率
- 钩子函数：Idle Hook、Tick Hook、Stack Overflow Hook

## 1.6 终局项目

学完 14 个实验后，**重构 FreeRTOS_Pro**：
- 用互斥量保护共享资源
- 用事件组协调多任务启动
- 用任务通知替换部分信号量
- 用流缓冲区优化 ADC → 处理的数据流
- 最终目标：将裸机项目移植为规范的 FreeRTOS 多任务系统

## 1.7 每个实验的标准流程

```markdown
1. 复制 FreeRTOS_00_DebugLog 为 FreeRTOS_xx_Name
2. AI 讲原理 + 指出 FreeRTOS 源码关键位置 + 给实验要求
3. 自己写代码
4. 编译 → 烧录 → 串口观察
5. 做边界测试（改参数观察极端行为）
6. 读 FreeRTOS 源码（AI 指定的文件和函数）
7. 贴关键代码给 AI review
8. 写实验文档到 Docs/xx_Name.md
9. 下一个实验
```

## 1.8 调试信息格式规范

ARM 32 位平台上 FreeRTOS 类型的 printf 格式：

| FreeRTOS 类型    | 底层类型          | printf 格式 |
|-----------------|------------------|-------------|
| `UBaseType_t`   | `unsigned long`  | `%lu`       |
| `BaseType_t`    | `long`           | `%ld`       |
| `uint32_t`      | `unsigned long`  | `%lu`       |
| `TickType_t`    | `uint32_t`       | `%lu`       |
| `uint16_t`      | `unsigned short` | `%u`        |
| `uint8_t`       | `unsigned char`  | `%u`        |

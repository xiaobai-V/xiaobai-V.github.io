---
title: STM32 开发环境搭建 (VSCode + CMake + GCC)
date: 2025-05-10
tags: [STM32, VSCode, CMake, GCC, 工具链]
description: 用 VSCode + CMake + arm-none-eabi-gcc 搭建现代 STM32 开发环境
---

# STM32 开发环境搭建 (VSCode + CMake + GCC)

告别 Keil，用更现代的工具链开发 STM32。参考 [Keysking 视频](https://www.bilibili.com/video/BV1QfbpzGENy/) 整理。

## 为什么不用 Keil

| 对比   | Keil            | VSCode + CMake + GCC |
| ---- | --------------- | -------------------- |
| 成本   | 商业付费            | 免费开源                 |
| 编辑体验 | 一般              | 智能补全、多插件             |
| 项目管理 | .uvprojx 绑定 IDE | CMakeLists.txt 通用    |
| 跨平台  | 仅 Windows       | Win / Mac / Linux    |
| 版本控制 | 工程文件难 diff      | 纯文本，git 友好           |

## 需要安装的东西

### 1. STM32CubeMX

ST 官方的图形化配置工具，选芯片、配引脚、设时钟、生成初始化代码。

- 下载：ST 官网搜索 [STM32CubeMX](https://www.st.com.cn/zh/development-tools/stm32cubemx.html)
- 安装后下载对应系列的 HAL 库（如 F1、F4）
- 生成代码时 Toolchain 选 **CMake**（新版 CubeMX 已支持）

> 如果 CubeMX 版本较旧不支持 CMake 生成，可选 Makefile，后续手动写 CMakeLists.txt

### 2. VSCode + 插件

安装 VSCode 后装这几个插件：

- **C/C++** (Microsoft) — 代码补全、语法检查
- **CMake Tools** (Microsoft) — CMake 项目管理、编译、调试
- **Cortex-Debug** (marus25) — ARM Cortex 调试支持（可选）
- **Embedded IDE** (CL) — 一键烧录支持（可选）

### 3. GCC 交叉编译工具链

`arm-none-eabi-gcc`，ARM 裸机编译器。

- 下载：ARM 官网 GNU Arm Embedded Toolchain
- 安装后将 `bin` 目录加入系统 PATH
- 验证：

```bash
arm-none-eabi-gcc --version
```

### 4. CMake + Ninja

- CMake：官网下载安装，加入 PATH
- Ninja（可选但推荐）：比 Make 更快的构建工具
- 验证：

```bash
cmake --version
ninja --version
```

### 5. STLink 驱动 + 烧录工具

- 安装 **STM32CubeProgrammer**（自带 STLink 驱动）
- 或者单独安装 STLink 驱动
- 验证：STLink 插上 USB，设备管理器能识别

## 工程创建流程

### Step 1：CubeMX 生成项目

1. 选择芯片型号
2. 配置引脚、外设、时钟树
3. Project Manager → Toolchain/IDE 选 **CMake**
4. Generate Code

### Step 2：VSCode 打开项目

用 VSCode 打开 CubeMX 生成的项目文件夹，CMake Tools 插件会自动检测。

### Step 3：配置工具链

<!-- 这里填你实际的配置方式，比如 CMakePresets.json 或 cmake-kits.json -->

### Step 4：编译

<!-- 填编译命令或快捷键 -->

### Step 5：烧录

<!-- 填烧录命令或配置 -->

## 常见问题

<!-- 你搭建过程中遇到的问题写在这里 -->

- **Q: CubeMX 不支持生成 CMake 项目？**
  A: 版本太旧，升级到 6.x 以上

- **Q: CMake 找不到交叉编译器？**
  A: 检查 arm-none-eabi-gcc 是否在 PATH 中

- **Q: 代码补全不工作？**
  A: 检查 `c_cpp_properties.json` 中的 includePath 配置

## 参考资料

- [Keysking - 爽！手把手教你用VSCode开发STM32](https://www.bilibili.com/video/BV1QfbpzGENy/)
- [基于 Keysking 教程的环境搭建指引 (CSDN)](https://blog.csdn.net/xiaoxu_bjtu/article/details/151644743)

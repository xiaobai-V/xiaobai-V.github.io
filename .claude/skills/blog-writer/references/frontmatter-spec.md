# frontmatter 规范

每篇文章顶部必须有 frontmatter。

## 标准模板（可直接复制）

```yaml
---
title: 标题
date: 2026-07-13
description: 一句话讲清这篇讲什么
tags:
  - STM32
  - DMA
draft: false
number headings: first-level 2, start-at 1, max 3, 1.1, auto, contents toc
---
```

## 字段说明

| 字段 | 必需 | 作用 |
|---|---|---|
| `title` | 是 | 文章标题，覆盖文件名作为页面标题 |
| `date` | 是 | 发布日期，`YYYY-MM-DD` |
| `description` | 强烈建议 | 一句话摘要，用于文章列表、RSS、SEO |
| `tags` | 是 | 2–4 个标签的数组；Quartz 为每个 tag 生成分组页，选有检索价值的 |
| `draft` | 可选 | `true` = 草稿，Quartz 的 RemoveDrafts 会过滤不发布。落盘默认 `false`；想先存草稿再润色设 `true` |
| `number headings` | 是 | Obsidian number-headings 插件配置，保持原值不动 |

## 不用的字段（统一规范）

- ❌ `categories`：Quartz 只认 `tags` 生成分组页，`categories` 没实际作用。
- ❌ `created` / `updated`：让 Quartz 从文件系统 / git 自动取 modified 时间，只留 `date` 作发布日。

## 分类目录（决定文件放哪）

- `MCU/STM32`
- `MCU/FreeRTOS`
- `Linux`
- `项目`（项目连载放对应子目录，如 `项目/PN532驱动库移植_基于SMT32L4/`）
- `硬件基础/硬件与协议`
- `工具与方法`
- `随笔`
- `编程语言/C语言`

分类由 skill 按内容判断，预览时告知用户，用户可改。

## 文件命名

简洁中文描述名，保持博客现有风格：

- `DMA缓存一致性问题.md`
- `sizeof vs strlen：从一次 fwrite 踩坑说起.md`
- `PN532模块基本介绍和接线说明.md`

# blog-writer Skill 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: 用 superpowers:subagent-driven-development 或 superpowers:executing-plans 逐任务执行。步骤用 checkbox（`- [ ]`）跟踪。

**Goal:** 创建 blog-writer skill，把用户随手写的凌乱嵌入式开发草稿整理成符合博客规范、风格统一的专业文章。

**Architecture:** Claude Code skill，progressive disclosure——精简的 SKILL.md（主流程 + 三条红线）+ 3 个 references（writing-style / article-structures / frontmatter-spec）。所有内容来自已批准 spec。

**Tech Stack:** Claude Code skill（纯 markdown）；目标博客 Obsidian + Quartz 4.5.2（文章在 `content/`）。

**Global Constraints:**
- 内容用简体中文（博客语言）。
- **不自动 git 提交**（用户偏好：未经要求不做 git 操作）；用户要提交再手动 commit。
- 严格遵循已批准 spec（`docs/superpowers/specs/2026-07-13-blog-writer-skill-design.md`）。
- 去 AI 味（强制）。
- frontmatter 统一：砍 `categories`、不手写 `created`/`updated`、`number headings: first-level 2`。

---

## 文件结构

```
.claude/skills/blog-writer/
├── SKILL.md                    # 入口：frontmatter + 三条红线 + 7步工作流 + 边界处理
└── references/
    ├── writing-style.md        # 风格指南（标题/代码/表格/笔触/AI味黑名单）
    ├── article-structures.md   # 5 类文章骨架
    └── frontmatter-spec.md     # frontmatter 规范 + 分类 + 命名
```

---

## Task 1: 创建 SKILL.md（skill 入口）

**Files:**
- Create: `.claude/skills/blog-writer/SKILL.md`

**职责：** skill 触发入口。frontmatter 的 `description` 决定 Claude 何时建议触发；正文给三条红线、7 步工作流、边界处理，详细规则指向 references。

**交付内容：**
- frontmatter：`name: blog-writer` + `description`（中英混合写清触发场景：用户给嵌入式主题凌乱草稿，想整理成可发布文章）。
- 正文四块：
  1. 三条红线（尊重草稿不编事实 / 补的必标 / 去 AI 味），AI 味黑名单精简列出
  2. 7 步工作流（拿草稿→判类型→补常识→一次性反问→成稿→预览→落盘）
  3. 边界处理（草稿太短/自相矛盾/分类模糊/细节没把握→反问或留 `【待确认】`）
  4. 详细规范指针：指向 3 个 references

**验证：**
- [ ] frontmatter 合法（`---` 包裹，name/description 齐全）
- [ ] description 写清「何时触发」，无歧义
- [ ] 正文精简（目标 < 70 行），详细规则下沉到 references

---

## Task 2: 创建 references/writing-style.md

**Files:**
- Create: `.claude/skills/blog-writer/references/writing-style.md`

**职责：** 详细写作风格，提炼自代表作 `content/Linux/sizeof vs strlen：从一次 fwrite 踩坑说起.md`。

**交付章节：**
1. 标题：正文从 `## 1` 开始（first-level 2，说明 Quartz 把 title 渲染成 h1 的原因），子节 `### 1.1`，最多 3 级
2. 代码：标语言、关键行加中文注释、错写法与对写法放一起对比（给 c 示例）
3. 表格：对比/参数/引脚/考点优先用表格
4. 引用块 `>`：放报错原文、重要提示、datasheet 摘录
5. 笔触：平实口语、短句收束、技术词用代码格式、正文中文、不啰嗦
6. 收尾：文末给「要点速查」表格
7. AI 味黑名单（强制）：禁用词/套路 + 正面范例

**验证：**
- [ ] 每个风格点有可执行的具体说明（非空话）
- [ ] AI 味黑名单含正反例

---

## Task 3: 创建 references/article-structures.md

**Files:**
- Create: `.claude/skills/blog-writer/references/article-structures.md`

**职责：** 5 类文章骨架，skill 按草稿内容自动选。开篇说明：骨架是参考不是模板，内容不够的段直接省。

**交付内容（5 类，每类含：适用场景 + 真实范例路径 + 骨架步骤）：**
1. 踩坑记录（范例：`content/Linux/sizeof vs strlen：从一次 fwrite 踩坑说起.md`）— 起因→问题代码(错vs对)→根因分析→要点速查→参考代码
2. 外设/知识讲解（范例：`content/项目/PN532驱动库移植_基于SMT32L4/PN532模块基本介绍和接线说明.md`）— 概述/特点→关键参数→接线/配置→用法
3. 项目连载（范例：`content/项目/PN532驱动库移植_基于SMT32L4/` 系列）— 背景目标→本篇子题→问题与解决→成果/代码；放项目子目录
4. 随笔/方法论（范例：`content/随笔/红蓝标记法：二分查找万能解法.md`）— 痛点→思路→步骤/实例→要点
5. 工具/环境教程（范例：`content/工具与方法/STM32开发环境搭建-VSCode+CMake+GCC.md`）— 为什么用→装什么→怎么配→验证

**验证：**
- [ ] 5 类齐全，每类骨架步骤清晰
- [ ] 范例路径指向真实存在的文章

---

## Task 4: 创建 references/frontmatter-spec.md

**Files:**
- Create: `.claude/skills/blog-writer/references/frontmatter-spec.md`

**职责：** 统一 frontmatter 模板 + 字段说明 + 分类目录 + 文件命名。

**交付内容：**
1. 标准 frontmatter 模板（可直接复制）：title / date / description / tags / draft / number headings
2. 字段说明：每个字段必需/可选/作用
3. 不用的字段：`categories`（Quartz 只认 tags）、`created`/`updated`（让 Quartz 自动取 modified）
4. 分类目录（8 个）+ 项目连载放子目录
5. 文件命名：简洁中文描述名 + 范例

**验证：**
- [ ] 模板 YAML 合法、可复制即用
- [ ] 字段说明覆盖所有字段
- [ ] 分类目录与 `content/` 实际结构一致

---

## Task 5: 验证 skill 注册与一致性

**Files:**
- 检查：`.claude/skills/blog-writer/` 整个目录

**步骤：**
- [ ] 目录结构：SKILL.md + references/ 下 3 个文件齐全
- [ ] SKILL.md frontmatter 的 name/description 合法
- [ ] 交叉引用一致：SKILL.md 指向的 3 个 references 文件名与实际一致
- [ ] frontmatter-spec.md 分类目录与 `content/` 实际一致
- [ ] article-structures.md 范例路径真实存在
- [ ] （可选）用户拿一段真实草稿试跑

**验证：** 所有引用路径存在；skill 结构符合 Claude Code skill 规范。

---

## Self-Review

- **Spec 覆盖**：spec §2 三条红线→T1；§4 工作流→T1；§5 风格→T2；§6 骨架→T3；§7 frontmatter→T4；§8 分类命名→T4；§9 边界处理→T1；§10 成功标准→T5。全覆盖。
- **占位符**：无 TODO/TBD，每个 task 有具体交付内容与验证。
- **一致性**：SKILL.md 引用的 3 个 references 文件名与 Task 2/3/4 创建的一致。

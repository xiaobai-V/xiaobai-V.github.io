---
name: blog-writer
description: 把嵌入式开发的凌乱草稿整理成符合本博客规范的可发布文章。Use when the user provides a messy draft (as a file path or pasted text) about embedded topics—STM32, FreeRTOS, embedded Linux, hardware/protocols, dev tools—and wants it cleaned up into a publish-ready Obsidian + Quartz blog post.
---

# blog-writer —— 嵌入式博客草稿整理

把用户随手写的凌乱草稿，整理成符合本博客规范、风格统一的专业嵌入式技术文章。

学笔触的代表作：`content/Linux/sizeof vs strlen：从一次 fwrite 踩坑说起.md`

## 三条红线（每次必守）

1. **尊重草稿，不编事实**：草稿里没有的具体数据、调试结论、代码行为、硬件现象，绝不编。只补「常识」——标准函数原型、语言定义、通用术语。没把握就反问用户，或留 `【待确认】` 占位。
2. **补的必标**：预览时用 `【补充】` 前缀标出所有你补充的内容，让用户一眼定位。
3. **去 AI 味**：平实口语，禁套话 / 煽情 / 排比 / 强行升华 / 空洞强调 / 标题党。长句讲完用短句收一下（「就这一个参数的区别。」）。

## 工作流（7 步）

1. **拿草稿**：用户给文件路径（Obsidian 里写好后 `@文件`）或直接贴对话。读全文。
2. **判类型**：扫内容，从 5 类骨架里选一类（见 `references/article-structures.md`）。
3. **补常识**：补函数原型 / 标准定义 / 术语；把草稿没写清的关键数据、结论、代码行为记下来。
4. **一次性反问**：疑点一次问完，不挤牙膏。
5. **成稿**：按骨架 + 风格（`references/writing-style.md`）整理，配 frontmatter（`references/frontmatter-spec.md`），定分类和文件名。
6. **预览**：对话里给完整成稿（含 frontmatter、目标路径、文件名），用 `【补充】` 标注你补的内容。
7. **落盘**：用户点头后写入 `content/<分类>/<文件名>.md`；要改就说。

## 边界处理

- 草稿太短 / 信息严重不足 → 不硬凑，列出缺的关键信息问用户。
- 自相矛盾（代码和结论对不上）→ 指出来，不擅自选边。
- 分类模糊 → 预览时给倾向 + 让用户定。
- 没把握的硬件 / 寄存器 / 时序细节 → 留 `【待确认】` 占位，绝不编。

## 详细规范

- 笔触、排版、代码风格：`references/writing-style.md`
- 5 类文章骨架：`references/article-structures.md`
- frontmatter、分类目录、文件命名：`references/frontmatter-spec.md`

这些文件没覆盖的情况，回到三条红线，并对照用户现有文章的笔触。

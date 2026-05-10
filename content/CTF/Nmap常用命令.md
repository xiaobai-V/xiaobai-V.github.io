---
title: Nmap常用命令
description: 记录 Nmap 在 CTF 和靶机测试中的常用扫描命令。
date: 2026-04-29
tags:
  - CTF
  - Nmap
  - 信息收集
---

# Nmap常用命令

Nmap 是常用的端口扫描和服务识别工具。

## 常用扫描

```bash
nmap -sC -sV 192.168.56.105
```

## UDP 扫描

```bash
nmap -sU 192.168.56.105
```

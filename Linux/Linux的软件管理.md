---
title: Linux的软件管理
date: 2025-12-14 21:29:20
tags:
---


<!-- more -->

# 前言
本机使用的是RHEL-9.6未订阅版本，所以为了屏蔽订阅管理器的信息影响，我们可以先把订阅管理器插件禁用。
```bash
echo "plusins=0" | tee -a /etc/dnf/dnf.cong
```
---
layout: post
title:  "5.8 启用和禁用中断"
date:   2020-01-09 14:18:01 +0800
categories: Assembly
tag: IA-32 Intel Architecture Software Developer's Manual, Volume 3
---

* content
{:toc}

> 翻译

处理器禁止某些中断的产生，具体取决于处理器的状态以及EFLAGS寄存器中IF和RF标志的状态，如以下各节所述。
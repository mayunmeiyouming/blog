---
layout: post
title:  "Multitasking"
date:   2019-12-29 13:45:01 +0800
categories: Assembly
tag: 80386
---

* content
{:toc}

>本文为原创

为了提供有效的，受保护的多任务处理，80386采用了几种特殊的数据结构。但是，它没有使用特殊的指令来控制多任务。相反，当它们引用特殊数据结构时，它会以不同的方式去解释普通控制传递指令。支持多任务的寄存器和数据结构有：

+ 任务状态段（Task state segment）

+ 任务状态段描述符（Task state segment descriptor）

+ 任务寄存器（Task register）

+ 任务门描述符（Task gate descriptor）

使用这些结构，80386可以快速的从一个任务切换到另一个任务，并且保存原始任务的上下文，以便以后可以重新启动该任务。除了简单的任务切换之外，80386还提供了另外两个任务管理功能：

1. 中断和异常可能导致任务切换（如果系统设计需要）。处理器不仅需要自动切换到处理中断或异常的任务，而且在处理中断或异常后也自动切换回被中断的任务。Interrupt tasks may interrupt lower-priority interrupt tasks to any depth.

2. 每次切换到另一个任务时，80386也可以切换到另一个LDT和另一个页目录。因此，每个任务可以具有逻辑到线性地址的不同映射以及线性到物理地址的不同映射。这是另一种保护功能，因为可以隔离任务并防止它们相互干扰。

# 任务状态段（Task state segment）

处理器管理任务所需的所有信息都存储在特殊类型的段中，即任务状态段（TSS）。[Figure 7-1](#01)显示了用于执行80386任务的TSS格式。

The fields of a TSS belong to two classes:

A dynamic set that the processor updates with each switch from the task. This set includes the fields that store:
The general registers (EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI).
The segment registers (ES, CS, SS, DS, FS, GS).
The flags register (EFLAGS).
The instruction pointer (EIP).
The selector of the TSS of the previously executing task (updated only when a return is expected).
A static set that the processor reads but does not change. This set includes the fields that store:
The selector of the task's LDT.
The register (PDBR) that contains the base address of the task's page directory (read only when paging is enabled).
Pointers to the stacks for privilege levels 0-2.
The T-bit (debug trap bit) which causes the processor to raise a debug exception when a task switch occurs . (Refer to Chapter 12 for more information on debugging.)
The I/O map base (refer to Chapter 8 for more information on the use of the I/O map).
Task state segments may reside anywhere in the linear space. The only case that requires caution is when the TSS spans a page boundary and the higher-addressed page is not present. In this case, the processor raises an exception if it encounters the not-present page while reading the TSS during a task switch. Such an exception can be avoided by either of two strategies:
By allocating the TSS so that it does not cross a page boundary.
By ensuring that both pages are either both present or both not-present at the time of a task switch. If both pages are not-present, then the page-fault handler must make both pages present before restarting the instruction that caused the task switch.

<span id="01">
![]({{ '/styles/images/2019-04-06-hello-github/01.png' | prepend: site.baseurl }})
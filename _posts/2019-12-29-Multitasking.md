---
layout: post
title:  "Chapter 7 Multitasking"
date:   2019-12-29 13:45:01 +0800
categories: [Tech]
tag: 
  - Intel 80386 Reference Programmer's Manual Table of Contents
  - 汇编
---

>本文为翻译

为了提供有效的，受保护的多任务处理，80386采用了几种特殊的数据结构。但是，它没有使用特殊的指令来控制多任务。相反，当它们引用特殊数据结构时，它会以不同的方式去解释普通控制传递指令。支持多任务的寄存器和数据结构有：

* 任务状态段（Task state segment）

* 任务状态段描述符（Task state segment descriptor）

* 任务寄存器（Task register）

* 任务门描述符（Task gate descriptor）

使用这些结构，80386可以快速的从一个任务切换到另一个任务，并且保存原始任务的上下文，以便以后可以重新启动该任务。除了简单的任务切换之外，80386还提供了另外两个任务管理功能：

1. 中断和异常可能导致任务切换（如果系统设计需要）。处理器不仅需要自动切换到处理中断或异常的任务，而且在处理中断或异常后也自动切换回被中断的任务。Interrupt tasks may interrupt lower-priority interrupt tasks to any depth.

2. 每次切换到另一个任务时，80386也可以切换到另一个LDT和另一个页目录。因此，每个任务可以具有逻辑到线性地址的不同映射以及线性到物理地址的不同映射。这是另一种保护功能，因为可以隔离任务并防止它们相互干扰。

## 任务状态段（Task state segment）

处理器管理任务所需的所有信息都存储在特殊类型的段中，即任务状态段（TSS）。[Figure 7-1](#01)显示了用于执行80386任务的TSS格式。

TSS的字段分为两类：

1. 处理器随着任务的每一次切换而更新动态集。该集合包括存储以下内容的字段：

* 通用寄存器（EAX，ECX，EDX，EBX，ESP，EBP，ESI，EDI）
* 段寄存器（ES，CS，SS，DS，FS，GS）
* 标志寄存器（EFLAGS）
* 指令指针（EIP）
* 先前执行的任务的TSS的selector（仅在可能返回时更新）

2. 处理器读取但不会更改的静态集。该集合包括存储以下内容的字段：

* 任务的LDT的selector
* 包含任务页目录基址的寄存器（PDBR）（仅在启用分页时才只读）
* 指向特权级别0-2的堆栈的指针。
* T位（调试陷阱位），当任务切换发生时，导致处理器触发调试异常。
* The I/O map base

任务状态段可以位于线性空间中的任何位置。唯一需要注意的情况是TSS跨越页面边界以及高地址页面不存在。在这种情况下，如果在任务切换期间读取TSS时处理器遇到不存在的页面，则会引发异常。可以通过以下两种策略之一避免此类异常：

1. 通过分配TSS，使其不跨越页面边界。

2. 通过确保在任务切换时两个页面都存在或不存在。如果两个页面都不存在，那么缺页异常处理程序必须在重新启动导致任务切换的指令之前使两个页面都存在。

<span id="01">
![]({{ '/assets/images/posts/2019-12-29-Multitasking/01.gif' | prepend: site.baseurl }})

## TSS描述符（TSS Descriptor）

与其他所有段一样，任务状态段由描述符定义。 TSS描述符的格式如[图7-2](#02)所示。

类型字段中的B位指示任务是否忙。类型代码9表示忙碌(B位为0)的任务；类型代码11表示任务繁忙(B位为1)。任务是不可重入的。 B位允许处理器检测意图切换到已经繁忙的任务的行为。BASE，LIMIT和DPL字段以及G位和P位的功能类似于数据段描述符中的对应功能。但是，LIMIT字段的值必须大于或等于103。尝试切换到其TSS描述符的限制小于103的任务会导致异常。更大的限制是可以的，如果存在I / O权限映射，则需要更大的限制。如果将其他数据存储在与TSS相同的段中，则较大的限制可能也便于系统软件的使用。

有权访问TSS描述符的程序可能引发任务切换。在大多数系统中，TSS描述符的DPL字段应设置为零，以便只有受信任的软件才有权执行任务切换。

程序拥有对TSS描述符的访问权，但是没有读取或修改TSS的权利。只能使用能将TSS重新定义为数据段的描述符来完成读取和修改。尝试将TSS描述符加载到任何段寄存器（CS，SS，DS，ES，FS，GS）中都会导致异常。

TSS描述符只能存放在GDT中。尝试使用TI = 1（指当前LDT）的selector去识别TSS会导致异常。

<span id="02">
![]({{ '/assets/images/posts/2019-12-29-Multitasking/02.gif' | prepend: site.baseurl }})

## Task Register

任务寄存器（TR）通过指向TSS来标识当前正在执行的任务。[图7-3](#03)显示了处理器访问当前TSS的方法。

任务寄存器具有`可见`部分（即，可以由指令读取和改变）和`不可见`部分（由处理器维护以对应于可见部分；不能由任何指令读取）。可见部分中的`selector`选择GDT中的TSS描述符。处理器使用不可见部分来缓存TSS描述符中的base和limit值。将base和limit保存在寄存器中可以使任务的执行效率更高，因为处理器在引用当前任务的TSS时不需要从内存中重复获取这些值。

LTR和STR指令用于修改和读取任务寄存器的可见部分。两条指令都使用一个操作数，即位于内存或通用寄存器中的16位selector。

LTR（Load task register）使用selector作为操作数存储到任务寄存器的可见部分，这个selector一定会指向GDT中的一个TSS描述符。LTR还会将操作数指向的TSS描述符中的信息存储在任务寄存器中的不可见部分。 LTR是特权指令；它只有在CPL为零时才能执行。LTR通常用于在系统初始化期间为任务寄存器提供初始值。此后，TR的内容通过任务切换操作进行更改。

STR（Store task register）读取任务寄存器的可见部分，并存储在通用寄存器或内存中。 STR不需要特权。

<span id="03">
![]({{ '/assets/images/posts/2019-12-29-Multitasking/03.gif' | prepend: site.baseurl }})

## Task Gate Descriptor

Task Gate Descriptor对TSS提供了间接的，受保护的引用。[图7-4](#04)说明了Task Gate的格式。

Task Gate的SELECTOR字段必须引用TSS描述符。处理器未使用selector中的RPL值。

Task Gate的DPL字段控制使用描述符进行任务切换的权限。除非selector的RPL和程序的CPL的最大值在数值上小于或等于描述符的DPL，否则程序可能不会选择Task Gate Descriptor。此约束可防止不受信任的程序导致任务切换。（请注意，使用task gate时，目标TSS描述符的DPL不用于特权检查。）

可以访问task gate的程序有权执行任务切换，就像可以访问TSS描述符的程序一样。80386除TSS描述符外还具有task gate，可以满足以下三个需求：

1. 一个任务需要有一个busy bit。因为busy bit存储在TSS描述符中，所以每个任务应该只有一个这样的描述符。但是，可能会有多个task gate选择同一个TSS描述符。

2. 提供对任务的选择性访问的需求。任务门可以满足此需求，因为它们可以驻留在LDT中，并且拥有与TSS描述符的DPL不同的DPL。如果程序没有访问GDT中的TSS描述符（通常DPL为0）的特权，但该程序可以访问该任务在LDT中的task gate，则该程序仍可以切换到另一个任务。使用task gate，系统软件拥有限制任务切换到特定任务的权利。

3. 需要中断或异常触发任务切换。task gate也可以驻留在IDT中，从而使中断和异常可能触发任务切换。当中断或异常引导到包含task gate的IDT条目时，80386将切换到指定的任务。因此，系统中的所有任务都可以从中断任务隔离提供的保护中受益。

[图7-5](#05)说明了LDT中的task gate和IDT中的task gate是如何识别同一任务。

<span id="04">
![]({{ '/assets/images/posts/2019-12-29-Multitasking/04.gif' | prepend: site.baseurl }})

<span id="05">
![]({{ '/assets/images/posts/2019-12-29-Multitasking/05.gif' | prepend: site.baseurl }})

## 任务切换（Task Switching）

在以下四种情况中的任何一种情况下，80386都会切换到另一个任务：

1. 当前任务将执行引用TSS描述符的`JMP`或`CALL`。

2. 当前任务执行引用task gate的`JMP`或`CALL`。

3. 中断或异常触发了在IDT中的task gate。

4. 设置NT标志时，当前任务将执行`IRET`。

JMP，CALL，IRET，中断和异常是80386的所有普通机制，可以在不需要任务切换的情况下使用。引用的描述符类型或标志字中的NT（嵌套任务）位都可以区分标准机制和导致任务切换的变量。

要引起任务切换，JMP或CALL指令可以引用TSS描述符或任务门。两种情况下的效果都相同：80386切换到指定的任务。

当异常或中断引导至IDT中的任务门时，将导致任务切换。如果它引导到IDT中的中断或陷阱门，则不会发生任务切换。

无论是作为任务被调用还是作为中断任务的程序被调用，中断处理程序始终将控制权返回给被中断的程序。但是，如果设置了NT标志，则处理程序为中断任务，IRET切换回中断的任务。

任务切换操作涉及以下步骤：

1. 检查是否允许当前任务切换到指定任务。数据访问特权规则适用于JMP或CALL指令。 TSS描述符或任务门的DPL必须在数值上大于（例如，较低的特权级别）大于或等于`selector`的CPL和RPL的最大值。无论目标任务门的DPL或TSS描述符如何，都可以通过异常，中断和IRET来切换任务。

2. 检查新任务的TSS描述符是否被标记为存在并具有有效限制。到此为止的任何错误都将在传给任务的上下文。错误是可重新启动的，并且可以以一种对应用程序透明的方式进行处理。

3. 保存当前任务的状态。处理器找到任务寄存器中缓存的当前TSS的基地址。它将寄存器复制到当前的TSS（EAX，ECX，EDX，EBX，ESP，EBP，ESI，EDI，ES，CS，SS，DS，FS，GS和标志寄存器）。TSS的EIP字段指向导致任务切换的指令之后的指令。

4. 使用任务寄存器的`selector`加载任务的TSS描述符，并将任务的TSS描述符标记为忙的状态，然后将MSW的TS位（任务切换）置1。`selector`可以是控制传递指令的操作数，也可以是从任务门获取的。

5. 从其TSS加载传入任务的状态并恢复执行。加载的寄存器为LDT寄存器；标志寄存器；通用寄存器EIP，EAX，ECX，EDX，EBX，ESP，EBP，ESI，EDI;段寄存器ES，CS，SS，DS，FS和GS；和PDBR。在此步骤中检测到的任何错误都在传入任务的上下文中发生。

请注意，发生任务切换时，正在进行的任务的状态始终会保存。如果恢复执行该任务，它将在引起任务切换的指令之后开始。当任务停止执行时，寄存器将恢复为其所保存的值。

每个任务切换都会将MSW（机器状态字）中的TS位（任务切换）置1。如果存在协处理器（例如数字协处理器），则TS标志对于系统软件很有用。TS位发出信号，表明协处理器的上下文可能不同于当前的80386任务。

在传入任务中记录任务切换异常的异常处理程序（表7-1的Test 4 - 16导致的异常）应谨慎对待可能加载导致异常的选择器的任何操作。除非异常处理程序首先检查`selector`并解决任何潜在问题，否则此类操作可能会导致另一个异常。

新任务的恢复执行的特权级别不受要旧任务执行时的特权级别的限制或影响。由于任务被它们各自的地址空间和TSS隔离，并且可以使用特权规则来防止对TSS的不正确访问，因此不需要特权规则来约束任务CPL之间的关系。新任务以TSS中的CS的selectord1值的RPL指示的特权级别开始执行。

```bash
Table 7-1. Checks Made during a Task Switch

NP = Segment-not-present exception
GP = General protection fault
TS = Invalid TSS
SF = Stack fault

Validity tests of a selector check that the selector is in the proper
table (e.g., the LDT selector refers to the GDT), lies within the bounds of
the table, and refers to the proper type of descriptor (e.g., the LDT
selector refers to an LDT descriptor).

Test     Test Description                   Exception    Error Code Selects

  1      Incoming TSS descriptor is
         present                            NP           Incoming TSS
  2      Incoming TSS descriptor is
         marked not-busy                    GP           Incoming TSS
         marked not-busy
  3      Limit of incoming TSS is
         greater than or equal to 103       TS           Incoming TSS

             -- All register and selector values are loaded --

  4      LDT selector of incoming
         task is valid                      TS           Incoming TSS
  5      LDT of incoming task is  
         present                            TS           Incoming TSS
  6      CS selector is valid               TS           Code segment
  7      Code segment is present            NP           Code segment
  8      Code segment DPL matches  
         CS RPL                             TS           Code segment
  9      Stack segment is valid             GP           Stack segment
 10      Stack segment is present           SF           Stack segment
 11      Stack segment DPL = CPL            SF           Stack segment
 12      Stack-selector RPL = CPL           GP           Stack segment
 13      DS, ES, FS, GS selectors are
         valid                              GP           Segment
 14      DS, ES, FS, GS segments
         are readable                       GP           Segment
 15      DS, ES, FS, GS segments
         are present                        NP           Segment
 16      DS, ES, FS, GS segment DPL  
         >= CPL (unless these are
         conforming segments)               GP           Segment
```

## Task Linking

TSS的back-link字段和标志字的NT位（嵌套任务）允许80386自动返回调用了另一个任务或被另一个任务中断的任务。当CALL指令，中断指令，外部中断或异常导致切换到新任务时，80386自动使用旧任务的TSS的选择器填充新TSS的back-link，同时，将新任务的标志寄存器中的NT位置1。NT标志指示back-link字段是否有效。新任务通过执行IRET指令释放控制权。解释IRET时，80386将检查NT标志。如果设置了NT，则80386将切换回由back-link字段选择的任务。表7-2总结了这些字段的用法。

```bash
Table 7-2. Effect of Task Switch on BUSY, NT, and Back-Link

Affected Field      Effect of JMP      Effect of            Effect of
                    Instruction        CALL Instruction     IRET Instruction

Busy bit of         Set, must be       Set, must be 0       Unchanged,
incoming task       0 before           before               must be set

Busy bit of         Cleared            Unchanged            Cleared
outgoing task                          (already set)

NT bit of           Cleared            Set                  Unchanged
incoming task

NT bit of           Unchanged          Unchanged            Cleared
outgoing task

Back-link of        Unchanged          Set to outgoing      Unchanged
incoming task                          TSS selector

Back-link of        Unchanged          Unchanged            Unchanged
outgoing task
```

### Busy Bit Prevents Loops

TSS描述符的B位（busy bit）确保反向链接的完整性。随着中断任务中断其他中断任务或被调用任务调用其他任务，反向链接链的长度可能会增加。busy bit确保CPU可以检测到任何尝试创建循环的行为。循环将表明尝试重新进入已经在运行的任务。但是，TSS不是可重入的资源。

处理器按以下方式使用busy bit：

1. 切换到任务时，处理器会自动设置新任务的busy bit。

2. 从任务切换时，如果不将旧任务的busy bit放置在反向链接链上（即引起任务切换的指令是JMP或IRET），则处理器会自动清除该任务的busy bit。如果将任务放置在反向链接链上，则其busy bit将保持忙的状态。

3. 切换到任务时，如果新任务的busy bit已设置，则处理器会发出异常信号。

通过这些操作，处理器可以防止任务切换到自身或切换到反向链接链上的任何任务，从而防止任务的无效的重入。

即使在多处理器配置中busy bit也有效，因为处理器在设置或清除busy位时会自动对总线加锁。此操作可确保两个处理器不会同时调用同一任务。

### Modifying Task Linkages

任务链顺序的任何修改都应仅通过可信任的软件来完成，以便正确的更新back-link和the busy-bit。在中断任务之前，可能需要进行此类更改才能恢复中断的任务。受信任的软件从反向链接链中删除任务必须遵循以下策略之一：

1. 首先更改中断任务的TSS中的反向链接字段，然后将列表中被删除的任务的TSS描述符中的busy位清除。

2. 确保在更新back-link和busy-bit的时候没有中断发生。

## 任务地址空间（Task Address Space）

TSS的LDT selector和PDBR字段使软件系统设计人员可以灵活地利用80386的段和页面映射功能。通过为每个任务选择合适的段和页面映射，任务可以共享地址空间，也可以拥有彼此大不相同的地址空间，或者可以在这两个程序之间具有任何程度的共享。

任务具有不同地址空间的能力是80386保护模式的重要方面。如果模块无法访问相同的地址空间，则一个任务中的模块不能干扰另一任务中的模块。80386灵活的内存管理功能使系统设计人员可以将共享地址空间的区域分配给需要相互协作的不同任务的一些模块。

### Task Linear-to-Physical Space Mapping

任务的线性地址到物理地址的映射的有两种选择：

1. 所有任务之间共享一个线性地址到物理地址的映射。未启用分页时，这是唯一的可能性。没有页表，所有线性地址都映射到相同的物理地址。启用分页时，这种线性地址到物理地址的映射方式是所有任务使用一个页目录。如果操作系统还实现页面级虚拟内存，则利用的线性空间可能会超过可用的物理空间。

2. 线性地址到物理地址的映射有几个部分重叠。通过为每个任务使用不同的页目录来实现这种形式。由于PDBR（页目录基址寄存器）是在任务切换时从TSS中加载的，因此每个任务可能具有不同的页目录。

从理论上讲，不同任务的线性地址空间可以映射到完全不同的物理地址。如果不同页目录的条目指向不同的页表，并且页表指向物理内存的不同页面，则任务不共享任何物理地址。

实际上，所有任务的线性地址空间的某些部分必须映​​射到相同的物理地址。任务状态段必须位于公共空间中，以便处理器在任务切换期间读取和更新TSS的时候，TSS地址的映射不会被修改。GDT的线性空间也应被映射到公共物理空间；否则，GDT的目的就无法实现。[图7-6](#06)显示了两个任务的线性空间是怎样通过共享页表重叠物理空间的。

### Task Logical Address Space

就其本身而言，常见的线性地址到物理地址空间的映射无法实现任务之间的数据共享。要共享数据，任务还必须具有通用的逻辑地址到线性地址空间的映射；即，它们还必须有权访问指向共享线性地址空间的描述符。有三种创建常见的逻辑地址到物理地址空间映射的方法：

1. 通过GDT。所有任务都可以访问GDT中的描述符。如果这些描述符指向线性地址空间，该线性地址空间映射到所有任务的公共物理地址空间，则任务可以共享数据和指令。

2. 通过共享LDT。如果其TSS中的LDT selector选择相同的LDT段，则两个或多个任务可以使用相同的LDT。那些LDT驻留描述符指向被映射到公共物理空间的线性空间，并且允许任务共享物理内存。这种共享方法比GDT共享更具选择性。共享可以限于特定任务。系统中的其他任务可能具有不同的LDT，这些LDT无法使它们访问共享区域。

3. LDT中的描述符别名。不同LDT的某些描述符可能指向相同的线性地址空间。如果通过涉及的任务的页面映射将该线性地址空间映射到相同的物理空间，则这些描述符允许任务共享公共空间。这样的描述符通常称为`aliases`。这种共享方法比前两种方法更具选择性。 LDT中的其他描述符可能指向不同的线性地址或未共享的线性地址。

<span id="06">
![]({{ '/assets/images/posts/2019-12-29-Multitasking/06.gif' | prepend: site.baseurl }})

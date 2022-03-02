---
layout: post
title:  "6.828 Lab 4: Preemptive Multitasking : Part B + C"
date:   2020-01-08 21:31:01 +0800
categories: [Tech]
tag: 
  - xv6
  - OS
---

>本文为原创

## Part B: Copy-on-Write Fork

如前所述，Unix提供`fork()`系统调用作为其主要的进程创建原语。 `fork()`系统调用复制调用进程（父进程）的地址空间以创建一个新进程（子进程）。

xv6 Unix通过将父页面的所有数据复制到为孩子分配的新页面中来实现`fork()`。这本质上与`dumbfork()`所采用的方法相同。将父进程地址空间复制到子进程是`fork()`操作中最昂贵的部分。

但是，在`fork()`的调用产生的子进程中，通常会紧跟着`exec()`，这会用新程序替换了子进程的内存。在这种情况下，复制父进程地址空间所花费的时间大大浪费了，因为子进程在调用`exec()`之前只使用了很少的内存。

因此，更高版本的Unix会利用虚拟内存硬件实现父进程和子进程共享映射到其各自地址空间的内存，直到其中一个进程实际对其进行修改为止。这种技术称为写时复制。为此，内核将在`fork()`上将地址空间映射从父进程复制到子进程，而不是映射页面的内容，同时将当前共享的页面标记为只读。当两个进程之一尝试写入这些共享页面时，该进程将出现`page fault`。此时，Unix内核知道该页面实际上是`虚拟`副本或`写时复制`副本，因此它会出现该错误的进程创建一个这个页面的新的，私有的，可写的副本。这样，直到它们被实际写入为止，各个页面的内容实际上都不会被复制。这种优化使在子进程中的`fork()`之后调用`exec()`要轻便得多：子进程在调用`exec()`之前可能只需要复制一页（其堆栈的当前页）。

在本实验的下一部分中，你将实现一个`适当的`类Unix的`fork()`，该类具有写时复制功能，并且作为用户空间库例程。在用户空间中实现`fork()`和写时复制的好处是内核保持简单。它还允许单个用户模式程序为`fork()`定义自己的语义。可以轻松为自己的程序提供一个稍有不同的fork实现（例如，全复制版本`dumbfork()`，或者父进程和子进程之后共享内存的版本）。

### 用户级page fault处理

用户级别的写时复制`fork()`需要了解写保护页上的页面错误，因此这是你首先要实现的。写时复制只是用户级页面错误处理的多种可能用途之一。

设置地址空间是很常见的，以便页面错误指示何时需要执行某些操作。例如，大多数Unix内核最初只在新进程的堆栈区域中映射单个页面，并在以后`按需`分配和映射其他堆栈页面，因为该进程的堆栈消耗增加，并导致尚未映射的堆栈地址出现页面错误。在进程空间的每个区域中发生页面错误时，典型的Unix内核必须跟踪采取的措施。例如，堆栈区域中的故障通常会分配并映射新的物理内存页面。程序的BSS区域中的故障通常会分配一个新页面，并用零填充并映射它。在具有按需分页的可执行文件的系统中，文本区域中的错误将从磁盘读取二进制文件的相应页面，然后将其映射。

这是内核要跟踪的很多信息。你无需采取传统的Unix方法，而是可以决定如何处理用户空间中每个页面错误，而这些错误对破坏性的影响较小。这种设计的附加好处是允许程序在定义其内存区域时具有极大的灵活性。你稍后将使用用户级页面错误处理来映射和访问基于磁盘的文件系统上的文件。

#### 设置页面错误处理程序

为了处理自己的页面错误，用户环境将需要在JOS内核中注册页面错误处理程序入口点。用户环境通过新的系统调用`sys_env_set_pgfault_upcall`注册其页面错误程序入口点。我们在Env结构中添加了一个新成员`env_pgfault_upcall`，以记录此信息。

##### Exercise 8

实现系统调用`sys_env_set_pgfault_upcall`。查找目标环境的环境ID时，请确保启用权限检查，因为这是`危险的`系统调用。

sys_env_set_pgfault_upcall的代码如下：

```c
static int sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
    // LAB 4: Your code here.
    // panic("sys_env_set_pgfault_upcall not implemented");
    struct Env* e;
    if(envid2env(envid, &e, 1) < 0)
        return -E_BAD_ENV;
    e->env_pgfault_upcall = func;

    return 0;
}
```

`kern/syscall.c syscall`增加代码：

```c
case SYS_env_set_pgfault_upcall:
    ret = sys_env_set_pgfault_upcall(a1, (void*)a2);
    break;
```

#### 用户环境中的正常堆栈和异常堆栈

在正常执行期间，JOS中的用户环境将在普通用户堆栈上运行：其`ESP`寄存器开始指向`USTACKTOP`，而其推送的堆栈数据位于`USTACKTOP-PGSIZE`和`USTACKTOP-1`（含）之间。但是，当在用户模式下发生页面错误时，内核将在另一个堆栈（即用户异常堆栈）上运行指定的用户级页面错误处理程序。本质上，我们将使JOS内核代表用户环境实现自动`堆栈切换`，这与x86处理器已经在从用户模式转换到内核模式时已经代表JOS实现堆栈切换的方式几乎相同！

JOS用户异常堆栈的大小也为一页，并且其顶部定义为虚拟地址`UXSTACKTOP`，因此用户异常堆栈的有效字节从`UXSTACKTOP-PGSIZE`到`UXSTACKTOP-1`（包括`UXSTACKTOP-1`）。在此异常堆栈上运行时，用户级页面错误处理程序可以使用JOS的常规系统调用来映射新页面或调整映射，以修复最初导致页面错误的任何问题。然后，用户级页面错误处理程序通过汇编语言`stub`返回给原始堆栈一个错误代码。

希望支持用户级页面错误处理的每个用户环境都将需要使用A部分中引入的系统调用`sys_page_alloc()`为其自身的异常堆栈分配内存。

#### 调用用户页面故障处理程序

现在，你将需要更改`kern/trap.c`中的页面错误处理代码，以从用户模式处理页面错误，如下所示。我们将发生故障时的用户环境状态称为`trap-time`状态。

如果没有注册页面错误处理程序，则JOS内核会像以前一样通过一条消息销毁用户环境。否则，内核将在异常堆栈上设置一个陷阱帧，该陷阱帧看起来像来自`inc/trap.h`的`struct UTrapframe`：

```c
                    <-- UXSTACKTOP
trap-time esp
trap-time eflags
trap-time eip
trap-time eax       start of struct PushRegs
trap-time ecx
trap-time edx
trap-time ebx
trap-time esp
trap-time ebp
trap-time esi
trap-time edi       end of struct PushRegs
tf_err (error code)
fault_va            <-- %esp when handler is run
```

然后内核安排用户环境在具有此堆栈帧的异常堆栈上运行页面错误处理程序，并恢复执行。您必须弄清楚如何做到这一点。 `fault_va`是导致页面错误的虚拟地址。

如果在发生异常时用户环境已经在用户异常堆栈上运行，则页面错误处理程序本身已发生故障。在这种情况下，您应该在当前`tf-> tf_esp`下而不是在`UXSTACKTOP`下开始新的堆栈帧。您应该首先推送一个空的32位字，然后再推送一个`struct UTrapframe`。

要测试`tf->tf_esp`是否指向用户异常堆栈上，请检查它是否在`UXSTACKTOP-PGSIZE`和`UXSTACKTOP-1`之间（包括两端）。

##### Exercise 9

在`kern/trap.c`中的`page_fault_handler`的实现要求将页面错误分派到用户模式处理程序。写入异常堆栈时，请确保采取适当的预防措施。（如果用户环境在异常堆栈上的空间不足，会发生什么？）

page_fault_handler的代码如下：

```c
void
page_fault_handler(struct Trapframe *tf)
{
    uint32_t fault_va;

    // Read processor's CR2 register to find the faulting address
    fault_va = rcr2();

    // Handle kernel-mode page faults.

    // LAB 3: Your code here.
    if ((tf->tf_cs & 3) == 0)
        panic("Page Fault in Kernel-Mode at %08x.\n", fault_va);

    // LAB 4: Your code here.
    if (curenv->env_pgfault_upcall != NULL)
    {
        uintptr_t va;

        if (tf->tf_esp > UXSTACKTOP - PGSIZE && tf->tf_esp < UXSTACKTOP) {
            va = tf->tf_esp - 4 - sizeof(struct UTrapframe);   //每个栈帧之间留空
        } else {
            va = UXSTACKTOP - sizeof(struct UTrapframe);
        }

        user_mem_assert(curenv, (void *) va, sizeof(struct UTrapframe), PTE_W | PTE_U | PTE_P);

        struct UTrapframe *utf = (struct UTrapframe *) (va);
        utf->utf_fault_va = fault_va;
        utf->utf_err = tf->tf_err;
        utf->utf_regs = tf->tf_regs;
        utf->utf_eip = tf->tf_eip;
        utf->utf_eflags = tf->tf_eflags;
        utf->utf_esp = tf->tf_esp;

        tf->tf_esp = va;
        tf->tf_eip = (uintptr_t) curenv->env_pgfault_upcall;
        env_run(curenv);
    }


    // Destroy the environment that caused the fault.
    cprintf("[%08x] user fault va %08x ip %08x\n",
        curenv->env_id, fault_va, tf->tf_eip);
    print_trapframe(tf);
    env_destroy(curenv);
}
```

#### 用户模式的页面错误处理程序入口点

接下来，你需要实现汇编例程，该例程将负责调用C语言编写的页面错误处理程序并按照原始错误指令恢复执行。该汇编例程是在内核中使用函数`sys_env_set_pgfault_upcall()`注册的处理程序。

用户程序调用`sys_env_set_pgfault_upcall()`来设置`_pgfault_upcall`处理程序和`handler`程序。当出现pgfault的时候，会产生中断信号`14`，进入的处理程序，然后处理程序运行`_pgfault_upcall`程序，`_pgfault_upcall`会调用handler程序。

##### Exercise 10

在`lib/pfentry.S`中实现`_pgfault_upcall`例程。有趣的是返回到导致页面错误的用户代码中的原始点。你将直接返回那里，而无需返回内核。困难的部分是同时切换堆栈并重新加载EIP。

```c
    // LAB 4: Your code here.
    // 这里将异常栈存储的trap time eip存入发生异常的栈的esp-4处。
    // 本来打算使用jmp命令直接跳转到trap time eip处的，结果测试不能通过
    // 有点迷
    movl 48(%esp), %eax
    subl $4, %eax
    movl %eax, 48(%esp)
    movl 40(%esp), %ebx
    movl %ebx, (%eax)

    // Restore the trap-time registers.  After you do this, you
    // can no longer modify any general-purpose registers.
    // LAB 4: Your code here.
    addl $8, %esp
    popal
    // Restore eflags from the stack.  After you do this, you can
    // no longer use arithmetic operations or anything else that
    // modifies eflags.
    // LAB 4: Your code here.
    addl $4, %esp
    popfl

    // Switch back to the adjusted trap-time stack.
    // LAB 4: Your code here.
    popl %esp

    // Return to re-execute the instruction that faulted.
    // LAB 4: Your code here.
    ret
```

##### Exercise 11

Finish set_pgfault_handler() in lib/pgfault.c.

set_pgfault_handler的代码如下：

```c
void set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
{
    int r;

    if (_pgfault_handler == 0) {
        // First time through!
        // LAB 4: Your code here.
        if ((r = sys_page_alloc(thisenv->env_id, (void *) (UXSTACKTOP - PGSIZE), PTE_W | PTE_U | PTE_P)) < 0)
            panic("sys_page_alloc: %e", r);

        if((r = sys_env_set_pgfault_upcall(thisenv->env_id, _pgfault_upcall)) < 0)
            panic("sys_env_set_pgfault_upcall: %e", r);
    }

    // Save handler pointer for assembly to call.
    _pgfault_handler = handler;
}
```

### Implementing Copy-on-Write Fork

现在，你有了内核工具，可以完全在用户空间中实现写时复制`fork()`。

我们在`lib/fork.c`中为你的`fork()`提供了一个框架。像`dumbfork()`一样，`fork()`应该创建一个新环境，然后扫描父环境的整个地址空间，并在子环境中设置相应的页面映射。关键区别在于，尽管`dumbfork()`复制了页面，而`fork()`最初仅复制页面映射。仅当其中一种环境尝试写入页面时，`fork()`才会复制每个页面。

`fork()`的基本控制流程如下：  

1. 父进程使用上面实现的`set_pgfault_handler()`函数将`pgfault()`安装为C级页面错误处理程序。

2. 父进程调用`sys_exofork()`创建一个子进程环境。

3. 对于`UTOP`下地址空间中的每个可写或写时复制页面，父进程调用`duppage`，后者应将写时复制页面映射到子地址空间，然后重新映射到写时复制页面。它自己的地址空间。 [注意：此处的顺序（即在父进程中将页面标记为子进程，然后在父进程中标记）实际上很重要！你知道为什么吗？尝试考虑一种特殊情况，在这种情况下，颠倒顺序可能会引起麻烦。 ] `duppage`设置两个`PTE`，以使该页面不可写，并在`avail`字段中包含`PTE_COW`，以区分写时复制页面和真正的只读页面。  

    但是，异常堆栈不会以这种方式重新映射。相反，你需要在子进程中为异常堆栈分配一个新页面。由于页面错误处理程序将进行实际的复制，并且页面错误处理程序在异常堆栈上运行，因此无法将异常堆栈写时复制：谁可以复制它？  

    `fork()`还需要处理已经存在的页面，而不是可写或写时复制的页面。

4. 父进程为子进程设置用户页面错误入口点，使其看起来像自己一样。

5. 子进程现在可以运行了，因此父进程将其标记为可运行。

每次环境向写时复制页面写入数据，都会发生页面错误。这是用户页面错误处理程序的控制流：  

1. 内核将页面错误传到`_pgfault_upcall`，后者调用`fork()`的`pgfault()`处理。

2. `pgfault()`检查错误是否是写操作（检查错误代码中的`FEC_WR`），以及页面的PTE是否标记为`PTE_COW`。如果没有,则调用panic()。

3. `pgfault()`分配一个映射到临时位置的新页面，并将故障页面的内容复制到其中。然后，故障处理程序将新页面映射到具有读/写权限的适当地址，以代替旧的只读映射。

用户级别的`lib/fork.c`代码必须查阅环境的页表以进行上述一些操作（例如，将页面的PTE标记为`PTE_COW`）。为此，内核正是在`UVPT`上映射环境的页表。它使用巧妙的映射技巧使其变得易于查找用户代码的PTE。 `lib/entry.S`设置`uvpt`和`uvpd`，以便你可以轻松地在`lib/fork.c`中查找页表信息。

#### Exercise 12

在`lib/fork.c`中实现`fork`，`duppage`和`pgfault`。

使用`forktree`程序测试你的代码。它应该产生以下消息，以及散布的'new env'， 'free env'和'exiting gracefully'消息。消息可能不会按此顺序出现，并且环境ID可能不同。

```bash
    1000: I am ''
    1001: I am '0'
    2000: I am '00'
    2001: I am '000'
    1002: I am '1'
    3000: I am '11'
    3001: I am '10'
    4000: I am '100'
    1003: I am '01'
    5000: I am '010'
    4001: I am '011'
    2002: I am '110'
    1004: I am '001'
    1005: I am '111'
    1006: I am '101'
```

pgfault代码如下：

```c
static void pgfault(struct UTrapframe *utf)
{
    void *addr = (void *) utf->utf_fault_va;
    uint32_t err = utf->utf_err;
    int r;

    // LAB 4: Your code here.
    //判断pte是否是可写或写时复制页面，如果不是则中断
    pte_t pte = uvpt[PGNUM((uintptr_t)addr)];
    if(!(err & FEC_WR) || !(pte & PTE_COW))
    {
        cprintf("[%08x] user fault va %08x ip %08x\n", sys_getenvid(), addr, utf->utf_eip);
        panic("Page Fault!");
    }

    // LAB 4: Your code here.
    // 将addr所在的页表映射到新页表，并复制内容
    uintptr_t start_addr = ROUNDDOWN((uintptr_t)addr, PGSIZE);
    if((r = sys_page_alloc(0, PFTEMP, PTE_W | PTE_U | PTE_P)) < 0)
        panic("Page Alloc Failed: %e", r);
    memmove((void*)PFTEMP, (void*)start_addr, PGSIZE);
    if((r = sys_page_map(0, (void*)PFTEMP, 0, (void*)start_addr, PTE_W | PTE_U | PTE_P)) < 0)
        panic("Page Map Failed: %e", r);

    if ((r = sys_page_unmap(0, PFTEMP)) != 0) {
        panic("pgfault: %e", r);
    }

}
```

注意取消注释

duppage代码如下：

```c
static int duppage(envid_t envid, unsigned pn)
{
    int r;

    // LAB 4: Your code here.
    // panic("duppage not implemented");
    void* va = (void *) (pn * PGSIZE);
    pte_t pte = uvpt[PGNUM(va)];

    if((pte & PTE_W) || (pte & PTE_COW))
    {
        // 因为envid2env根据envid返回环境，而envid是0时，则返回当前环境，所以使用0代表当前环境
        // 将当前环境的va地址所在的页复制到envid环境的地址空间va所在的页
        if((r = sys_page_map(0, va, envid, va, PTE_U | PTE_COW | PTE_P)) < 0)
            panic("Page Map Failed: %e", r);
        // 把当前环境的页表权限改变为写时复制
        if((r = sys_page_map(0, va, 0, va, PTE_U | PTE_COW | PTE_P)) < 0)
            panic("Page Map Failed: %e", r);
    } else { // 当va所在的页的没有写权限时，只需要简单复制
        if((r = sys_page_map(0, va, envid, va, PTE_U | PTE_P)) < 0)
            panic("Page Map Failed: %e", r);
    }
    return 0;
}
```

fork代码如下：

```c
envid_t
fork(void)
{
    // LAB 4: Your code here.
    // panic("fork not implemented");
    // Set up our page fault handler appropriately.
    set_pgfault_handler(pgfault);
    // Create a child.
    envid_t envid = sys_exofork();
    // Copy our address space and page fault handler setup to the child.
    if (envid == 0) {
        // child
        thisenv = &envs[ENVX(sys_getenvid())];
        return 0;
    } else {
        // parent
        int r;
        for (uintptr_t va = 0; va < USTACKTOP; va += PGSIZE) {
            if ((uvpd[PDX(va)] & PTE_P) && (uvpt[PGNUM(va)] & PTE_P)) {
                // uvpd指向页目录，uvpt指向页表，页目录是一页，页表有1024页
                duppage(envid, PGNUM(va));
            }
        }
        // 映射异常堆栈
        if ((r = sys_page_alloc(envid, (void *) (UXSTACKTOP - PGSIZE), PTE_U | PTE_W | PTE_P)) < 0) {
            return r;
        }
        extern void _pgfault_upcall(void);
        if ((r = sys_env_set_pgfault_upcall(envid, _pgfault_upcall)) < 0) {
            return r;
        }
        sys_env_set_status(envid, ENV_RUNNABLE);

        return envid;
    }

}
```

到这里可以通过Part B的测试了

## Part C: 抢占式多任务和进程间通信（IPC）

### 时钟中断和抢占

运行`user/spin`测试程序。该测试程序产生一个子环境中，该子环境一旦收到CPU的控制，就会一直执行循环。父进程和内核都不会重新获得CPU。就保护系统免受用户模式环境中的错误或恶意代码的影响而言，这显然不是理想的情况，因为任何用户模式环境都可以通过进入无限循环而永远不放弃CPU，从而使整个系统停机。为了允许内核抢占正在运行的环境，并从中强行夺回CPU的控制权，我们必须扩展JOS内核以支持时钟硬件的外部硬件中断。

#### Interrupt discipline

外部中断（即设备中断）称为`IRQ`。有16个可能的IRQ，编号为0到15。从IRQ编号到IDT条目的映射不是固定的。 `picirq.c`中的`pic_init`将IRQ 0-15映射到IDT条目`IRQ_OFFSET`至`IRQ_OFFSET + 15`。

在`inc/trap.h`中，`IRQ_OFFSET`定义为十进制32。因此IDT条目32-47对应于IRQ 0-15。例如，时钟中断为IRQ0。因此，IDT [IRQ_OFFSET + 0]（即IDT [32]）包含内核中时钟中断处理程序例程的地址。选择此`IRQ_OFFSET`的目的是使设备中断不会与处理器异常重叠，因为这样显然会引起混乱。（实际上，在运行MS-DOS的PC的早期，`IRQ_OFFSET`实际上为零，这确实在处理硬件中断和处理处理器异常之间引起了巨大的混乱！）

与xv6 Unix相比，在JOS中，我们进行了一些简化。当运行在内核中时，始终禁用外部设备中断（与xv6类似，运行在用户空间中时，则启用外部设备中断）。外部中断由`％eflags`寄存器的`FL_IF`标志位控制（请参见`inc/mmu.h`）。当该位置1时，使能外部中断。尽管可以通过多种方式修改该位，但是由于我们的简化，我们仅在进入和退出用户模式时通过保存和恢复％eflags寄存器的过程来处理该位。

你必须确保在用户环境运行时设置了FL_IF标志，以便在中断到达时将其传递给处理器并由你的中断代码进行处理。否则，将屏蔽或忽略中断，直到重新启用中断。我们使用引导加载程序的第一条指令屏蔽了中断，到目前为止，我们还没有解决过重新启用它们的问题。

##### Exercise 13

修改`kern/trapentry.S`和`kern/trap.c`来初始化IDT中的适当条目，并为IRQ 0到15提供处理程序。然后修改`kern/env.c`中`env_alloc()`中的代码以确保始终保持用户环境在启用中断的情况下运行。

还要取消注释`sched_halt()`中的`sti`指令，以便空闲的CPU接受中断。

调用硬件中断处理程序时，处理器从不推送错误代码。此时，你可能想重新阅读[80386 Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)的9.2节或[IA-32 Intel Architecture Software Developer's Manual, Volume 3](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf)的5.8节。

完成此练习后，如果你使用运行时间很短（例如，旋转）的任何测试程序来运行内核，则应该看到内核打印的陷阱帧。现在在处理器中启用了中断，但是JOS尚未处理它们，因此你应该看到它将每个中断错误地分配给了当前正在运行的用户环境并销毁了它。

trapentry.S代码如下：

```c
TRAPHANDLER_NOEC(IRQ_0, 32)
TRAPHANDLER_NOEC(IRQ_1, 33)
TRAPHANDLER_NOEC(IRQ_2, 34)
TRAPHANDLER_NOEC(IRQ_3, 35)
TRAPHANDLER_NOEC(IRQ_4, 36)
TRAPHANDLER_NOEC(IRQ_5, 37)
TRAPHANDLER_NOEC(IRQ_6, 38)
TRAPHANDLER_NOEC(IRQ_7, 39)
TRAPHANDLER_NOEC(IRQ_8, 40)
TRAPHANDLER_NOEC(IRQ_9, 41)
TRAPHANDLER_NOEC(IRQ_10, 42)
TRAPHANDLER_NOEC(IRQ_11, 43)
TRAPHANDLER_NOEC(IRQ_12, 44)
TRAPHANDLER_NOEC(IRQ_13, 45)
TRAPHANDLER_NOEC(IRQ_14, 46)
TRAPHANDLER_NOEC(IRQ_15, 47)
```

trap_init.c代码如下：

```c
    // IRQ
    extern int IRQ_0;
    extern int IRQ_1;
    extern int IRQ_2;
    extern int IRQ_3;
    extern int IRQ_4;
    extern int IRQ_5;
    extern int IRQ_6;
    extern int IRQ_7;
    extern int IRQ_8;
    extern int IRQ_9;
    extern int IRQ_10;
    extern int IRQ_11;
    extern int IRQ_12;
    extern int IRQ_13;
    extern int IRQ_14;
    extern int IRQ_15;

    SETGATE(idt[IRQ_OFFSET + 0], 0, GD_KT, &IRQ_0, 0);
    SETGATE(idt[IRQ_OFFSET + 1], 0, GD_KT, &IRQ_1, 0);
    SETGATE(idt[IRQ_OFFSET + 2], 0, GD_KT, &IRQ_2, 0);
    SETGATE(idt[IRQ_OFFSET + 3], 0, GD_KT, &IRQ_3, 0);
    SETGATE(idt[IRQ_OFFSET + 4], 0, GD_KT, &IRQ_4, 0);
    SETGATE(idt[IRQ_OFFSET + 5], 0, GD_KT, &IRQ_5, 0);
    SETGATE(idt[IRQ_OFFSET + 6], 0, GD_KT, &IRQ_6, 0);
    SETGATE(idt[IRQ_OFFSET + 7], 0, GD_KT, &IRQ_7, 0);
    SETGATE(idt[IRQ_OFFSET + 8], 0, GD_KT, &IRQ_8, 0);
    SETGATE(idt[IRQ_OFFSET + 9], 0, GD_KT, &IRQ_9, 0);
    SETGATE(idt[IRQ_OFFSET + 10], 0, GD_KT, &IRQ_10, 0);
    SETGATE(idt[IRQ_OFFSET + 11], 0, GD_KT, &IRQ_11, 0);
    SETGATE(idt[IRQ_OFFSET + 12], 0, GD_KT, &IRQ_12, 0);
    SETGATE(idt[IRQ_OFFSET + 13], 0, GD_KT, &IRQ_13, 0);
    SETGATE(idt[IRQ_OFFSET + 14], 0, GD_KT, &IRQ_14, 0);
    SETGATE(idt[IRQ_OFFSET + 15], 0, GD_KT, &IRQ_15, 0);
```

env_alloc代码如下：

```c
    // Enable interrupts while in user mode.
    // LAB 4: Your code here.
    e->env_tf.tf_eflags |= FL_IF;
```

取消在`kern/shed.c sched_halt()`中`sti`的注释，开启中断

如果出现下面的错误：

```bash
kernel panic on CPU 0 at kern/trap.c:310: assertion failed: !(read_eflags() & FL_IF)
```

是因为在注册中断的开启了trap，设置为0就行

```bash
SETGATE(idt[T_PGFLT], 0, GD_KT, &th_pgflt, 0);
```

注释掉`kern/shed.c shed_halt()`的下列部分

```c
    // Uncomment the following line after completing exercise 13
    "sti\n"
```

#### 处理时钟中断

在`user/spin`程序中，子进程首次运行后，它只是循环旋转，内核再也无法收回控制权。我们需要对硬件进行编程以定期生成时钟中断，这将迫使控制权回到内核，在内核中我们可以将控制权切换到其他用户环境。

在`init.c`的`i386_init`中调用`lapic_init`和`pic_init`，用来设置时钟和中断控制器产生中断。现在，你需要编写代码来处理这些中断。

##### Exercise 14

修改内核的`trap_dispatch()`函数，以便在收到时钟中断时调用`sched_yield()`来查找并运行不同的环境。

现在，你应该能够通过`user/spin`测试：父进程产生子进程，并调用`sys_yield()`几次，只有在经过一个时间片后，它们都将重新获得对CPU的控制权，最后终止子进程并优雅终止。

代码如下：

```c
    if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
        lapic_eoi();
        sched_yield();
        return;
    }
```

现在应该能得到`65/80`

### 进程间通信（IPC）

我们一直专注于操作系统的隔离方面，使它给人的感觉是每个程序都拥有一台机器。操作系统的另一个重要服务是允许程序在需要时相互通信。让程序与其他程序交互可能会更加强大。Unix管道模型是典型示例。

进程间通信的模型很多。即使在今天，仍然存在关于哪种模型最好的争论。我们不会参加这场辩论。相反，我们将尝试实现一个简单的IPC机制。

#### JOS中的IPC

你将为JOS内核实现一些其他的系统调用，这些调用共同提供了一种简单的进程间通信机制。你将实现两个系统调用`sys_ipc_recv`和`sys_ipc_try_send`。然后，你将实现两个库包装器`ipc_recv`和`ipc_send`。

用户环境可以使用JOS的IPC机制相互发送的`消息`，这个机制由两个组件组成：单个32位值和可选的单个页面映射。允许环境在消息中传递页面映射可以提供一种有效的方式来传输比单个32位整数所容纳的数据更多的数据，并且还允许环境轻松地设置共享内存。

#### 发送和接收消息

环境调用`sys_ipc_recv`接收消息。此系统调用将对当前环境进行调度，并且在消息被收到之前不会再次运行它。当环境正在等待接收消息时，任何其他环境都可以向其发送消息-不仅是特定环境，而且不仅仅是与接收环境具有父子关系的环境。换句话说，你在A部分中实现的权限检查将不适用于IPC，因为IPC系统调用经过了精心设计，因此是`安全的`：一个环境不能仅仅通过向其发送消息而导致它发生故障。

环境调用`sys_ipc_try_send`根据接收者的环境ID将要发送的值发送给接收者。如果指定的环境实际上正在接收信息（它已调用`sys_ipc_recv`但是还没有获得值），则发送消息给发送者，并返回0。否则，返回`-E_IPC_NOT_RECV`给发送者以指示目标环境当前不希望接收值。

用户空间中的库函数`ipc_recv`将负责调用`sys_ipc_recv`，然后在当前环境的结构Env中查找有关接收的信息。

同样，库函数`ipc_send`将负责重复调用`sys_ipc_try_send`，直到发送成功。

#### Transferring Pages

当环境使用有效的`dstva`参数（在`UTOP`之下）调用`sys_ipc_recv`时，则表明该环境愿意接收页面映射。如果发送方发送一个页面，则该页面应被映射到接收方地址空间中的`dstva`。如果接收者已经在`dstva`上映射了页面，则该先前映射的页面将被解除映射。

当环境使用有效参数`srcva`（位于`UTOP`下）调用`sys_ipc_try_send`时，这意味着发送方希望将当前映射到`srcva`的页面发送给接收方，并具有权限`perm`。成功完成IPC之后，发送方将其页面的原始映射保留在其地址空间中的`srcva`处，然后接收方将在自己的地址空间中的`dstva`处获得此同一物理页面的映射。最后，该页面在发送者和接收者之间共享。

如果发送方或接收方均未指示应传送页面，则不会传送任何页面。在完成任何IPC之后，内核会将接收者的Env结构中的新字段`env_ipc_perm`设置为接收到的页面的权限；如果未接收到任何页面，则将其设置为零。

#### Exercise 15

在`kern/syscall.c`中实现`sys_ipc_recv`和`sys_ipc_try_send`。在实现它们之前，请先阅读两者的注释，因为它们必须协同工作。在这些例程中调用`envid2env`时，应将`checkperm`标志设置为0，这意味着允许任何环境将IPC消息发送到任何其他环境，并且内核除了验证目标envid是否有效外，不执行任何特殊权限检查。

然后在`lib/ipc.c`中实现`ipc_recv`和`ipc_send`函数。

使用`user/pingpong`和`user/primes`函数来测试IPC机制。`user/primes`将为每个素数生成一个新环境，直到JOS用尽环境为止。你可以通过阅读`user/primes.c`以了解所有的分支和IPC的情况。

`kern/syscall.c syscall`添加如下代码：

```c
case SYS_ipc_try_send:
    return sys_ipc_try_send(a1, a2, (void *)a3, a4);
case SYS_ipc_recv:
    return sys_ipc_recv((void *)a1);
```

sys_ipc_recv代码如下：

```c
// 阻塞直到值准备就绪。
// 使用struct Env的env_ipc_recving和env_ipc_dstva字段记录要接收的内容，将自己标记为不可运行，然后放弃CPU。
//
// 如果'dstva'小于UTOP，则可以接收数据页面。 “dstva”是发送页面应映射到的虚拟地址。
//
// 该函数仅在出错时返回，但是系统调用最终将在成功时返回0。
// Return < 0 on error.  Errors are:
//  -E_INVAL if dstva < UTOP but dstva is not page-aligned.
static int sys_ipc_recv(void *dstva)
{
    // LAB 4: Your code here.
    if ((uintptr_t) dstva < UTOP && PGOFF(dstva) != 0) {
        return -E_INVAL;
    }

    curenv->env_ipc_dstva = dstva;
    curenv->env_ipc_recving = true;
    curenv->env_status = ENV_NOT_RUNNABLE;

    sys_yield();

    return 0;
}
```

```c
// 尝试将`value`发送到目标环境“envid”。
// 如果 srcva < UTOP，则将发送当前映射到“srcva”的页面，以便接收者共享同一页面。
//
// 如果目标未被阻塞，则发送失败，返回值为-E_IPC_NOT_RECV，等待IPC。
//
// 下面列出的其他原因，发送也会失败。
//
// 否则，发送将成功，并且目标的ipc字段将更新如下：
//    env_ipc_recving设置为0以阻塞接下来的发送；
//    env_ipc_from设置为发送对象的envid；
//    env_ipc_value设置为参数'value'；
//    如果已传输页面，则env_ipc_perm设置为“perm”，否则设置为0。
// 目标环境再次标记为可运行，从暂停的sys_ipc_recv系统调用中返回0。 （提示：sys_ipc_recv函数实际上会返回吗？）
//
// 如果发送方要发送一个页面，但接收方不要求发送页面，则不会传输任何页面映射，并且不会发生错误。
// ipc仅在没有错误发生时发生。
//
// Returns 0 on success, < 0 on error.
// Errors are:
//  -E_BAD_ENV if environment envid doesn't currently exist.
//      (No need to check permissions.)
//  -E_IPC_NOT_RECV if envid is not currently blocked in sys_ipc_recv,
//      or another environment managed to send first.
//  -E_INVAL if srcva < UTOP but srcva is not page-aligned.
//  -E_INVAL if srcva < UTOP and perm is inappropriate
//      (see sys_page_alloc).
//  -E_INVAL if srcva < UTOP but srcva is not mapped in the caller's
//      address space.
//  -E_INVAL if (perm & PTE_W), but srcva is read-only in the
//      current environment's address space.
//  -E_NO_MEM if there's not enough memory to map srcva in envid's
//      address space.
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
    // LAB 4: Your code here.
    envid_t src_envid = sys_getenvid();
    struct Env *dst_e;
    // -E_BAD_ENV if environment envid doesn't currently exist.
    //     (No need to check permissions.)
    if (envid2env(envid, &dst_e, 0) < 0) {
        return -E_BAD_ENV;
    }

    // -E_IPC_NOT_RECV if envid is not currently blocked in sys_ipc_recv,
    //     or another environment managed to send first.
    if (!(dst_e->env_ipc_recving)) {
        return -E_IPC_NOT_RECV;
    }

    dst_e->env_ipc_from = src_envid;
    dst_e->env_ipc_value = value;
    dst_e->env_ipc_perm = 0;

    if ((uintptr_t)srcva < UTOP && ((uintptr_t)dst_e->env_ipc_dstva) < UTOP) {
        // sys_page_map可以检查地址是否对齐，也能检查权限
        int r = sys_page_map(src_envid, srcva, envid, (void *)dst_e->env_ipc_dstva, perm);
        if (r < 0)
            return r;
        dst_e->env_ipc_perm = perm;
    }

    dst_e->env_status = ENV_RUNNABLE;
    // 系统调用的返回值，设置在%eax
    dst_e->env_tf.tf_regs.reg_eax = 0;
    dst_e->env_ipc_recving = false;
    return 0;
}
```

ipc_recv代码如下：

```c
// 通过IPC接收值并将其返回。
// 如果“pg”为非空，则发件人发送的任何页面都将映射到该地址。
// 如果“from_env_store”为非空，则将IPC发送方的envid存储在*from_env_store中。
// 如果'perm_store'为非null，则将IPC发件人的页面权限存储在*perm_store中
//（如果页面已成功传输到'pg'，则此值为非零）。
// 如果系统调用失败，则*fromenv和*perm中将存储0（如果它们为非null）并返回错误。
// 否则，返回发件人发送的值
//
// Hint:
//   使用“thisenv”发现值和发送者。
//   如果'pg'为null，则向sys_ipc_recv传递一个被理解为“no page”的值。
//  （0不是正确的值，因为这是映射页面的完全有效的位置。）
int32_t
ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
{
    // LAB 4: Your code here.
    int r;
    if (pg != NULL) {
        r = sys_ipc_recv(pg);
    } else {
        r = sys_ipc_recv((void *) UTOP);
    }
    if (r < 0) {
        // failed
        if (from_env_store != NULL)
            *from_env_store = 0;
        if (perm_store != NULL)
            *perm_store = 0;
        return r;
    } else {
        if (from_env_store != NULL)
            *from_env_store = thisenv->env_ipc_from;
        if (perm_store != NULL)
            *perm_store = thisenv->env_ipc_perm;
        return thisenv->env_ipc_value;
    }
}
```

ipc_send代码如下：

```c
//将“val”（如果“pg”为非空，则将“pg”与“perm”一起发送）到“toenv”。
//此函数一直尝试直到成功。
//除-E_IPC_NOT_RECV以外的任何错误均应调用panic()。
//
// Hint:
//   Use sys_yield() to be CPU-friendly.
//   If 'pg' is null, pass sys_ipc_try_send a value that it will understand
//   as meaning "no page".  (Zero is not the right value.)
void
ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
{
    // LAB 4: Your code here.
    int32_t retval = -1;
    while (retval != 0)
    {
        if(pg != NULL)
            retval = sys_ipc_try_send(to_env, val, pg, perm);
        else
            retval = sys_ipc_try_send(to_env, val, (void*)UTOP, perm);

        if(retval == 0)
            sys_yield();
        else if(retval != -E_IPC_NOT_RECV)
            panic("Receving wrong return value of sys_ipc_try_send");
    }
}
```

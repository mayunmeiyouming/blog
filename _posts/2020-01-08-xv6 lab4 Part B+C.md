---
layout: post
title:  "6.828 Lab 4: Preemptive Multitasking : Part B + C"
date:   2020-01-08 21:31:01 +0800
categories: OS
tag: xv6
---

* content
{:toc}

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

```
static int sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
	// LAB 4: Your code here.
	//panic("sys_env_set_pgfault_upcall not implemented");
	struct Env* e;
	if(envid2env(envid, &e, 1) < 0) 
		return -E_BAD_ENV;
	e->env_pgfault_upcall = func;
	
	return 0;
}
```

#### 用户环境中的正常堆栈和异常堆栈

在正常执行期间，JOS中的用户环境将在普通用户堆栈上运行：其`ESP`寄存器开始指向`USTACKTOP`，而其推送的堆栈数据位于`USTACKTOP-PGSIZE`和`USTACKTOP-1`（含）之间。但是，当在用户模式下发生页面错误时，内核将在另一个堆栈（即用户异常堆栈）上运行指定的用户级页面错误处理程序。本质上，我们将使JOS内核代表用户环境实现自动`堆栈切换`，这与x86处理器已经在从用户模式转换到内核模式时已经代表JOS实现堆栈切换的方式几乎相同！

JOS用户异常堆栈的大小也为一页，并且其顶部定义为虚拟地址`UXSTACKTOP`，因此用户异常堆栈的有效字节从`UXSTACKTOP-PGSIZE`到`UXSTACKTOP-1`（包括`UXSTACKTOP-1`）。在此异常堆栈上运行时，用户级页面错误处理程序可以使用JOS的常规系统调用来映射新页面或调整映射，以修复最初导致页面错误的任何问题。然后，用户级页面错误处理程序通过汇编语言`stub`返回给原始堆栈一个错误代码。

希望支持用户级页面错误处理的每个用户环境都将需要使用A部分中引入的系统调用`sys_page_alloc()`为其自身的异常堆栈分配内存。

#### 调用用户页面故障处理程序

现在，你将需要更改`kern/trap.c`中的页面错误处理代码，以从用户模式处理页面错误，如下所示。我们将发生故障时的用户环境状态称为`trap-time`状态。

如果没有注册页面错误处理程序，则JOS内核会像以前一样通过一条消息销毁用户环境。否则，内核将在异常堆栈上设置一个陷阱帧，该陷阱帧看起来像来自`inc/trap.h`的`struct UTrapframe`：

```
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

```
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


```
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

```
void set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
{
	int r;

	if (_pgfault_handler == 0) {
		// First time through!
		// LAB 4: Your code here.
		envid_t id = sys_getenvid();
		if ((r = sys_page_alloc(id, (void *) (UXSTACKTOP - PGSIZE), PTE_W | PTE_U | PTE_P)) < 0 ||
			(r = sys_env_set_pgfault_upcall(id, _pgfault_upcall)) < 0) {
			panic("sys_page_alloc: %e", r);
		}
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

3.	对于`UTOP`下地址空间中的每个可写或写时复制页面，父进程调用`duppage`，后者应将写时复制页面映射到子地址空间，然后重新映射到写时复制页面。它自己的地址空间。 [注意：此处的顺序（即在父进程中将页面标记为子进程，然后在父进程中标记）实际上很重要！你知道为什么吗？尝试考虑一种特殊情况，在这种情况下，颠倒顺序可能会引起麻烦。 ] `duppage`设置两个`PTE`，以使该页面不可写，并在`avail`字段中包含`PTE_COW`，以区分写时复制页面和真正的只读页面。  

	但是，异常堆栈不会以这种方式重新映射。相反，你需要在子进程中为异常堆栈分配一个新页面。由于页面错误处理程序将进行实际的复制，并且页面错误处理程序在异常堆栈上运行，因此无法将异常堆栈写时复制：谁可以复制它？  

	`fork()`还需要处理已经存在的页面，而不是可写或写时复制的页面。

4. 父进程为子进程设置用户页面错误入口点，使其看起来像自己一样。

5. 子进程现在可以运行了，因此父进程将其标记为可运行。

每次环境向写时复制页面写入数据，都会发生页面错误。这是用户页面错误处理程序的控制流：  

1. 内核将页面错误传到`_pgfault_upcall`，后者调用`fork()`的`pgfault()`处理。

2. `pgfault()`检查错误是否是写操作（检查错误代码中的`FEC_WR`），以及页面的PTE是否标记为`PTE_COW`。如果没有,则调用panic()。

3. `pgfault()`分配一个映射到临时位置的新页面，并将故障页面的内容复制到其中。然后，故障处理程序将新页面映射到具有读/写权限的适当地址，以代替旧的只读映射。

用户级别的`lib/fork.c`代码必须查阅环境的页表以进行上述一些操作（例如，将页面的PTE标记为`PTE_COW`）。为此，内核正是在`UVPT`上映射环境的页表。它使用巧妙的映射技巧使其变得易于查找用户代码的PTE。 `lib/entry.S`设置`uvpt`和`uvpd`，以便你可以轻松地在`lib/fork.c`中查找页表信息。

##### Exercise 12

在`lib/fork.c`中实现`fork`，`duppage`和`pgfault`。

使用`forktree`程序测试你的代码。它应该产生以下消息，以及散布的'new env'， 'free env'和'exiting gracefully'消息。消息可能不会按此顺序出现，并且环境ID可能不同。

```
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

```
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
	//将addr所在的页表映射到新页表，并复制内容
	uintptr_t start_addr = ROUNDDOWN((uintptr_t)addr, PGSIZE);
	if((r = sys_page_alloc(0, PFTEMP, PTE_W | PTE_U | PTE_P)) < 0)
		panic("Page Alloc Failed: %e", r);
	memmove((void*)PFTEMP, (void*)start_addr, PGSIZE);
	if((r = sys_page_map(0, (void*)PFTEMP, 0, (void*)start_addr, PTE_W | PTE_U | PTE_P)) < 0)
		panic("Page Map Failed: %e", r);

}
```

duppage代码如下：

```
static int duppage(envid_t envid, unsigned pn)
{
	int r;

	// LAB 4: Your code here.
	//panic("duppage not implemented");
	uintptr_t* va = (void *) (pn * PGSIZE);
	pte_t pte = uvpt[PGNUM(va)];
	if(!(pte & PTE_P))
		return -1;

	if((pte & PTE_W) || (pte & PTE_COW))
	{
		//因为envid2env根据envid返回环境，而envid是0时，则返回当前环境，所以使用0代表当前环境
		//将当前环境的va地址所在的页复制到envid环境的地址空间va所在的页
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

```
envid_t
fork(void)
{
	// LAB 4: Your code here.
	//panic("fork not implemented");
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
		//parent
		int r;
		for (uintptr_t va = 0; va < USTACKTOP; va += PGSIZE) {
			if ((uvpd[PDX(va)] & PTE_P) && (uvpt[PGNUM(va)] & PTE_P)) {
				//uvpd指向页目录，uvpt指向页表，页目录是一页，页表有1024页
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


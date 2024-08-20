---
layout: page
title: L08 Page Faults
permalink: /L08
nav_order: 8



---

# Lecture 8 - Page Faults

## Page Fault 与虚拟内存功能

今天的课程主要讨论 Page Fault 及其相关的虚拟内存功能。这些功能在现代操作系统中非常重要，典型的功能包括：

- **Lazy Allocation**（惰性分配）：下一个实验室的内容。
- **Copy-On-Write Fork**（写时复制的 Fork）。
- **Demand Paging**（按需分页）。
- **Memory Mapped Files**（内存映射文件）。

这些功能几乎在所有现代操作系统中都得到了实现，比如 Linux 就实现了所有这些功能。然而，在 XV6 中，出于简化的目的，这些功能并未实现。XV6 中的 Page Fault 处理方式非常保守：一旦用户空间进程触发 Page Fault，系统会直接杀掉该进程。

在今天的课程中，我们将探讨在 Page Fault 发生时，操作系统可以采取的一些更有趣的处理方式。虽然我们不会深入分析代码，但会重点讨论这些机制的设计理念，因为 XV6 并未实现这些功能。特别是，我们将看到如何利用 Page Fault 来实现一些复杂的虚拟内存管理功能，这也是后续实验的重点内容。下一个实验 Lazy Lab 将会发布，之后的实验中还会涉及 Copy-On-Write Fork 和 Memory Mapped Files 这些主题。它们都是操作系统中非常有趣和复杂的部分。

### 虚拟内存的基础

在深入探讨 Page Fault 的具体功能之前，我们首先回顾一下虚拟内存的基本概念。虚拟内存有两个主要的优点：

1. **隔离性（Isolation）**：虚拟内存使得操作系统可以为每个应用程序提供独立的地址空间。这样，一个应用程序无论是有意还是无意，都无法修改另一个应用程序的内存数据。此外，虚拟内存还实现了用户空间与内核空间的隔离。我们在前面的课程中已经讨论过很多关于虚拟内存隔离的内容，并且通过之前的 Page Table 实验，大家也应该对虚拟内存的隔离性有了更深的理解。

2. **抽象层（Level of Indirection）**：虚拟内存为处理器和所有的指令提供了一层抽象，使得这些指令只需处理虚拟地址，内核则负责将虚拟地址映射到物理地址。这种抽象使得许多高级的内存管理功能成为可能。在目前为止介绍的 XV6 中，内存地址映射基本上是静态的，一旦在初始化时设置好，无论是用户页表还是内核页表，通常都不会再发生变化。

### 动态内存映射与 Page Fault

Page Fault 的出现使得地址映射关系可以动态变化。通过 Page Fault，内核有机会在程序访问内存时动态更新 Page Table，从而实现灵活的内存管理。这一机制为操作系统提供了巨大的灵活性，并支持一些复杂且有趣的功能。

1. **内核映射的示例**：尽管 XV6 的内存映射通常是静态的，但它也包含了一些动态映射的例子。例如：
   - **Trampoline Page**：内核可以将同一个物理内存页映射到多个用户地址空间中，实现内核代码的高效调用。
   - **Guard Page**：在用户空间和内核空间中，Guard Page 保护栈免受溢出或错误访问的影响。

2. **Page Fault 的潜力**：在传统的静态映射中，内核只需在启动时设定好页表即可，而 Page Fault 机制让内核能够在运行时动态调整这些映射。例如，当程序访问尚未映射的虚拟地址时，触发 Page Fault，内核可以选择在此时分配物理内存并更新页表。这种动态调整的能力使得实现 Lazy Allocation、Copy-On-Write Fork、Demand Paging 等高级功能成为可能。

通过这节课的内容，大家将对 Page Fault 及其带来的动态内存管理功能有更深的理解，这些内容也将为接下来的实验和实际操作打下坚实的基础。

## Page Fault 处理所需的关键信息

在处理 Page Fault 之前，首先需要明确的是，当 Page Fault 发生时，操作系统需要掌握哪些关键信息才能做出正确的响应。

### 1. **出错的虚拟地址**

最明显的一个信息是导致 Page Fault 的虚拟地址，即触发错误的内存地址。当一个用户进程访问了一个未被映射的虚拟地址时，会触发 Page Fault。这个出错的地址会被自动存储在 RISC-V 架构的 `STVAL` 寄存器中。因此，操作系统可以从 `STVAL` 寄存器中读取这个出错的地址，以确定哪个内存访问导致了错误。

- **STVAL 寄存器**：这是一个专门用于存储导致 Page Fault 的虚拟地址的寄存器。当 Page Fault 发生时，系统会将出错的地址存储在 `STVAL` 中。这是处理 Page Fault 时首先需要获取的信息。

### 2. **出错的原因**

了解触发 Page Fault 的原因也非常重要。Page Fault 可能是由于不同类型的内存操作引起的，例如：

- **加载操作（Load）**：例如，程序尝试从一个未映射的地址读取数据。
- **存储操作（Store）**：例如，程序尝试向一个未映射的地址写入数据。
- **指令取值操作（Instruction Fetch）**：例如，程序试图执行存储在一个未映射地址的指令。

RISC-V 的 `SCAUSE` 寄存器可以存储触发异常的具体原因。不同的异常原因会在 `SCAUSE` 中有不同的编码，例如：

- **13**：表示是由于加载操作引起的 Page Fault。
- **15**：表示是由于存储操作引起的 Page Fault。
- **12**：表示是由于指令取值操作引起的 Page Fault。

- **SCAUSE 寄存器**：这个寄存器用于存储导致进入 supervisor mode 的原因。当出现 Page Fault 时，`SCAUSE` 寄存器会保存具体的错误类型，这对于操作系统决定如何处理这个 Page Fault 至关重要。

### 3. **触发 Page Fault 的指令地址**

除了出错的地址和原因外，还需要知道引起 Page Fault 的指令的地址。RISC-V 架构通过 `SEPC` 寄存器来存储触发异常的指令的地址。这一点非常重要，因为在修复 Page Fault 后，系统需要重新执行这条指令。这个指令地址也会保存在 XV6 的 `trapframe->epc` 中，以便稍后恢复执行。

- **SEPC 寄存器**：`SEPC` 寄存器保存了发生异常的指令地址，这使得操作系统能够在处理完 Page Fault 后，从引发错误的地方继续执行程序。

### Page Fault 处理的关键流程

通过以上三个关键信息，当发生 Page Fault 时，操作系统可以有效地处理错误，并在必要时更新 Page Table 以修复错误。

1. **获取虚拟地址**：从 `STVAL` 寄存器中读取出错的虚拟地址。这是最基础的操作，操作系统需要知道具体哪个地址导致了错误。

2. **确定出错原因**：检查 `SCAUSE` 寄存器，确定是由于加载、存储还是指令取值引发的 Page Fault。这一步有助于操作系统采取不同的策略进行处理。

3. **获取触发指令的地址**：通过 `SEPC` 寄存器获取引发错误的指令地址。这使得在修复 Page Table 后，能够重新执行这条指令，确保程序继续正常运行。

理想情况下，当操作系统处理完 Page Fault 并修复 Page Table 后，可以恢复被中断的指令继续执行。这样，程序不会因为 Page Fault 而崩溃，而是能继续按预期运行。接下来，我们将深入探讨如何利用 Page Fault Handler 来动态修复 Page Table 并实现一些复杂的内存管理功能，例如惰性分配、写时复制等。

## 内存分配与 `sbrk` 系统调用

在操作系统中，内存的分配是一个关键的操作。`sbrk` 是 XV6 提供的一个系统调用，用于扩展用户程序的堆（heap）。当一个应用程序启动时，堆的初始位置由 `p->sz` 表示，即堆的底端，同时也是堆的起始位置。

### 1. **`sbrk` 的工作机制**

`sbrk` 接受一个整数参数，表示要扩展的字节数。调用 `sbrk` 后，堆的上边界将被扩大，即 `p->sz` 的值增加。

- **堆扩展**：`sbrk` 通过调整 `p->sz` 的值来扩展堆。这意味着应用程序可以多次调用 `sbrk` 以获得更多的内存。
- **堆缩减**：相反，传入负数参数时，`sbrk` 将缩小堆的边界。不过，本次讨论主要集中在内存的扩展上。

在 XV6 的默认实现中，`sbrk` 使用的是 **eager allocation**，即一旦调用 `sbrk`，内核就会立即分配相应的物理内存，并将其映射到用户程序的地址空间中。这种方式虽然简单直接，但会导致过度的内存分配，因为应用程序通常无法精确预估所需的内存量，往往会申请比实际需要更多的内存。

### 2. **Lazy Allocation：更聪明的内存分配策略**

为了优化内存的使用，可以采用 **lazy allocation** 策略。其核心思想是在 `sbrk` 调用时，仅调整 `p->sz`，而不立即分配物理内存。实际的内存分配推迟到应用程序真正访问这部分内存时才进行。

- **内存分配时机**：当应用程序访问尚未实际分配的内存时（地址在 `p->sz` 范围内，同时大于`stack`的合法地址中），会触发一个 Page Fault。在 Page Fault 处理程序中，内核通过 `kalloc` 分配物理内存，将其映射到用户的页表中，并将内容初始化为 0，然后重新执行导致 Page Fault 的指令。这种方式可以显著减少不必要的内存占用，特别是在程序申请了大量未使用的内存时。

- **例外处理**：如果内核在分配物理内存时发现系统已经没有可用内存，则可以选择杀掉进程并返回错误，这种方式称为 OOM（Out Of Memory）处理。

在处理 Page Fault 时，内核需要根据触发 Page Fault 的虚拟地址判断其所属的内存区域，如果属于合法的堆区域，则分配物理内存，否则就认为是非法访问并杀掉进程。

### 3. **关于 OOM 处理的讨论**

在采用 Lazy Allocation 策略时，如果系统内存不足，内核可以选择杀掉进程。这种方法虽然简单，但不够灵活。实际操作系统可能会采取更为智能的措施，例如尝试释放缓存或交换空间以获取更多内存，但如果最终无法满足需求，仍然可能会杀掉占用过多资源的进程。

一些学生可能会问，为什么操作系统不能简单地返回一个错误，而不是直接杀掉进程？这涉及到操作系统的设计哲学和复杂性管理。在实际系统中，有时杀掉进程是最直接和有效的处理方式，特别是在资源耗尽的极端情况下。尽管如此，实际操作系统通常会有更多的机制来延迟或避免 OOM 情况的发生。

通过 Lazy Allocation，我们可以实现更加智能的内存管理，减少不必要的内存浪费。这一机制虽然在 XV6 中没有直接实现，但它展示了如何通过 Page Fault 机制动态管理内存的可能性。这为后续的实验，如 Lazy Allocation Lab、Copy-On-Write Fork 和 Memory Mapped Files，奠定了基础，帮助我们理解操作系统如何更高效地管理资源。

> > 以下是关于 `sbrk` 和 Lazy Allocation 的简单框图，描述了它们的工作流程：
> >
> > ```
> > +-----------------+    调用 sbrk(n)         +-----------------+
> > |   User Process  |  ------------------->  |    Kernel       |
> > +-----------------+                        +-----------------+
> >       |                                           |
> >       |                                           |
> >       V                                           V
> > +-----------------+      调整 p->sz         +-----------------+
> > |  p->sz += n     |  ------------------->  |  不分配物理内存    |
> > +-----------------+                        +-----------------+
> >       |                                           |
> >       |                                           |
> >       V                                           V
> > +-----------------+                        +-----------------+
> > | 继续执行代码     |                         | 等待实际访问内存   |
> > +-----------------+                        +-----------------+
> >       |                                           |
> >       |                                           |
> >       V                                           V
> > +-----------------+    访问未分配内存         +-----------------+
> > | Page Fault 发生  |  ------------------->  |分配物理内存，并映射 |
> > +-----------------+    触发Page Fault       +-----------------+
> >       |                                           |
> >       |                                           |
> >       V                                           V
> > +-----------------+                        +-----------------+
> > | 重新执行访问指令   |                        | 内存访问成功      |
> > +-----------------+                        +-----------------+
> > ```
> >
> > 在 Lazy Allocation 策略中，`sbrk` 调用时仅调整 `p->sz`，而不立即分配物理内存。这个过程可以解释为以下几个关键点：
> >
> > 1. **调整 `p->sz` 是什么操作？**
> >    - 调整 `p->sz` 的意思是修改进程控制块（PCB）中的一个字段 `p->sz`，使其值增加指定的字节数 `n`。`p->sz` 代表当前进程的虚拟地址空间的顶部（即堆的末端）。
> >    - 这个操作只是更新 PCB 中的一个变量，而不是立即在内存中执行任何物理操作。因此，内核并不会立即为这个新增加的地址空间分配实际的物理内存。
> >
> > 2. **内存分配的时机**
> >    - 实际的内存分配被推迟到应用程序访问新分配的地址空间时。这意味着当程序访问这些尚未分配物理内存的虚拟地址时，会触发一个 Page Fault。
> >    - 在 Page Fault 发生时，内核会捕捉到这个异常，识别出该地址属于已分配的虚拟空间但尚未映射的部分，然后在这个时候内核才会实际分配物理内存，并更新页表。
> >
> > 3. **`p->sz` 是否可以超过实际堆的最大大小？**
> >    - 理论上，`p->sz` 可以调整到超过当前分配的实际物理内存的大小。因为 Lazy Allocation 的设计初衷就是允许这种“超前”分配，以便在未来需要时才实际分配物理资源。
> >    - 但是，`p->sz` 的增加必须在操作系统允许的虚拟地址空间范围内。如果超过了进程的最大虚拟地址空间，内核将会拒绝这种分配请求。
> >    - 这种机制与立即分配物理内存的区别在于，Lazy Allocation 不会占用过多的物理内存资源，直到程序实际使用这些内存。它在内存管理上提供了更高的效率和灵活性。
> >
> > 4. **为什么 Lazy Allocation 更加高效？**
> >    - 在实际应用中，程序往往会预分配大量的内存以防万一，但这部分内存可能永远不会被实际使用。如果采用 eager allocation，这些内存就会被立即分配，从而浪费物理内存资源。
> >    - Lazy Allocation 则允许程序在需要时动态分配内存，这不仅节省了物理内存资源，还降低了由于过度分配导致的内存碎片问题。

## Lazy Allocation 实现探讨

接下来我们将通过修改 `sys_sbrk` 函数来实现 Lazy Allocation，并观察它在 `XV6` 中的效果。

### 原始代码分析

```c
// kernel/sysproc.c

uint64 sys_sbrk(void)
{
    int addr;
    int n;

    if(argint(0, &n) < 0)
        return -1;
    addr = myproc()->sz;
    if(growproc(n) < 0)
        return -1;
    return addr;
}
```

在原始代码中：

1. **参数解析**：`argint(0, &n)` 从系统调用的参数中获取一个整数 `n`，表示要扩展的内存大小（以字节为单位）。如果获取失败，则返回 `-1` 表示错误。
2. **获取当前内存大小**：`addr = myproc()->sz` 记录当前进程的内存大小 `sz`。`sz` 是进程的堆顶地址，表示进程当前已分配的内存大小。
3. **内存扩展**：调用 `growproc(n)` 函数来扩展当前进程的内存空间。`growproc(n)` 会实际分配物理内存，并将这部分内存映射到进程的地址空间。如果内存分配失败，返回 `-1` 表示错误。
4. **返回值**：如果内存扩展成功，则返回原始堆顶地址 `addr`，表示分配成功。

`sys_sbrk` 函数负责实际增加应用程序的地址空间，分配内存等相关操作。原本的实现中，它会立即分配所需的物理内存。但在 Lazy Allocation 的思想下，我们希望推迟这种内存分配，仅在进程真正访问这些内存时才进行分配。

### 修改后的代码分析

```c
uint64 sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz = myproc()->sz + n;
  // if(growproc(n) < 0)
  //   return -1;
  return addr;
}
```

在修改后的代码中：

1. **参数解析**：与原始代码相同，通过 `argint(0, &n)` 获取扩展的内存大小 `n`。
2. **获取当前内存大小**：与原始代码相同，通过 `addr = myproc()->sz` 获取当前进程的内存大小 `sz`。
3. **调整进程的内存大小**：`myproc()->sz = myproc()->sz + n` 直接修改 `sz`，增加 `n` 字节的空间。这一步只是在进程控制块（PCB）中记录了一个新的堆顶地址，但**并没有实际分配物理内存**。
4. **注释掉内存扩展**：原始代码中的 `growproc(n)` 被注释掉了，也就是说，这里并没有进行实际的内存分配。
5. **返回值**：依然返回原始的堆顶地址 `addr`，与原始代码一致。

### 运行 `XV6` 并观察 Page Fault

在启动 `XV6` 并执行命令 `echo hi` 后，会引发一个 `Page Fault`。这发生在 `Shell` 中 fork 了一个子进程，然后该子进程通过 `exec` 执行 `echo` 程序。在这个过程中，`Shell` 试图申请一些内存，因此调用了我们修改过的 `sys_sbrk` 函数。然而，由于我们没有实际分配物理内存，当 `Shell` 尝试访问这块新“分配”的内存时，触发了 `Page Fault`。

![image-20240820150051267]({{ site.baseurl }}/docs/assets/image-20240820150051267.png)

观察 `XV6` 输出的调试信息，可以帮助我们理解 `Page Fault` 的原因：

1. **`SCAUSE` 寄存器**：其值为 `15`，表示这是一个 Store Page Fault。
2. **进程的 PID**：显示 `3`，可能是 `Shell` 的 PID。
3. **`SEPC` 寄存器**：显示 `0x12a4`，指向引发 `Page Fault` 的指令地址。
4. **出错的虚拟地址**：`STVAL` 寄存器的值为 `0x4008`，这是引发 `Page Fault` 的具体地址。

### 分析引发 `Page Fault` 的代码

通过查看 `Shell` 的汇编代码，我们可以定位到触发 `Page Fault` 的具体指令，并进一步分析原因。在这个例子中，`Page Fault` 发生在 `malloc` 的实现中，这是因为 `malloc` 函数试图向未被分配的内存写入数据。

```c
void*
malloc(uint nbytes)
{
    11f8:	7139                	addi	sp,sp,-64
    11fa:	fc06                	sd	ra,56(sp)
    11fc:	f822                	sd	s0,48(sp)
    11fe:	f426                	sd	s1,40(sp)
...
    129e:	6398                	ld	a4,0(a5)
    12a0:	e118                	sd	a4,0(a0)
    12a2:	bff1                	j	127e <malloc+0x86>
  hp->s.size = nu;
    12a4:	01652423          	sw	s6,8(a0)
  free((void*)(hp + 1));
    12a8:	0541                	addi	a0,a0,16
...
```

当执行 `malloc` 时，程序会调用 `sbrk` 来扩展堆空间。然而，修改后的 `sys_sbrk` 仅调整了 `p->sz`，而没有实际分配物理内存。当 `malloc` 尝试访问这块地址时，发现该地址并没有映射到物理内存，于是引发了 `Page Fault`。

另一个可以证明内存尚未被分配的地方是，在 xv6 中 Shell 程序的典型布局是由 4 个页（page）组成的，包括代码段（text）和数据段（data）。根据汇编代码，`a0` 寄存器持有的是 `0x4000` 地址，而偏移量为 `8`。这意味着程序试图访问的是 `0x4000 + 0x8 = 0x4008` 这个地址，而这个地址位于 Shell 程序的前 4 个页之外。因为这块区域尚未被 `sbrk` 真正映射到物理内存上，所以尝试访问该地址时会触发 `page fault`。

引发了 `Page Fault` 后，操作系统可以在 `Page Fault` 处理程序中为该地址分配物理内存，并将其映射到进程的地址空间中，然后重新执行导致 `Page Fault` 的指令。这样就可以实现 Lazy Allocation 的机制——内存的实际分配推迟到真正需要的时候，从而有效地利用系统资源。

> > ###  `12a4: sw s6,8(a0)` 
> >
> > 这条指令的作用是将寄存器 `s6` 中的值存储到内存地址 `a0 + 8` 处。这正是 `malloc` 函数中尝试将新分配的内存的大小写入到 `Header` 结构的 `size` 字段中。
> >
> > 在此指令之前，`a0` 寄存器中存储的是 `p`，即由 `sbrk` 函数返回的地址。如果 `sbrk` 调用返回了一个尚未实际分配的内存地址（由于我们修改了 `sys_sbrk`，它不会立即分配物理内存），那么当 `malloc` 尝试在 `a0 + 8` 处写入数据时，就会导致 `Page Fault`，因为这部分内存实际上还没有映射到任何物理内存。
> >
> > > ### `Header` 
> > >
> > > `Header` 是在实现 `malloc`（内存分配函数）时，用来管理已分配内存块的一种常见的数据结构。它通常包含了与内存块相关的元数据，如内存块的大小、下一个内存块的指针等。在动态内存分配中，`Header` 结构体被用于链接已分配的内存块和空闲的内存块，形成一个链表，方便内存的管理和回收。
> > >
> > > ### `Header` 的作用
> > >
> > > - **表示内存块的元数据**：`Header` 结构体保存了内存块的大小信息，通常还包括一个指向下一个块的指针，以形成一个链表结构。
> > > - **管理空闲链表**：`malloc` 函数在分配内存时，使用 `Header` 结构体来管理可用的空闲内存块。通过检查 `Header` 中的 `size` 字段，`malloc` 可以找到合适大小的空闲块，进行分配。
> > > - **合并空闲块**：在 `free` 函数中，释放内存后，通过 `Header` 中的链表结构，可以尝试合并相邻的空闲内存块，以减少内存碎片。
> > >
> > > ###  `Header`的结构
> > >
> > > 虽然不同的实现细节可能会有所不同，但典型的 `Header` 结构体可能像这样：
> > >
> > > ```c
> > > typedef struct Header {
> > >     struct Header *next;  // 指向下一个空闲块的指针
> > >     unsigned int size;    // 当前块的大小（以字节或某种单位表示）
> > > } Header;
> > > ```
> > >
> > > - **`next`**：指向下一个空闲内存块的指针，形成一个链表。
> > > - **`size`**：当前内存块的大小，用于判断是否能够满足分配请求。
> > >
> > > 在 `malloc` 的实现中，`Header` 通常位于每个已分配或空闲的内存块的起始位置，这样 `malloc` 和 `free` 函数可以通过这个结构来管理内存。
> > >
> > > ### 具体到 `malloc` 的实现
> > >
> > > 在本节内容中：
> > >
> > > - 当 `malloc` 调用 `sbrk` 请求内存时，它会返回一个指向新分配内存块的指针，通常这个指针指向新块的起始地址，而这个地址紧跟着的是一个 `Header` 结构。
> > > - `Header` 的 `size` 字段记录了这个内存块的大小。在你的代码中，指令 `sw s6, 8(a0)` 是试图在 `Header` 结构体中的 `size` 字段写入内存块的大小信息。
> > >
> > > 通过使用 `Header` 结构，`malloc` 和 `free` 函数可以有效地管理堆内存，跟踪哪些内存已经分配，哪些内存可以再次分配。这也是为什么 `malloc` 中的 `12a4` 地址是 `sw` 指令的原因，因为它试图写入这个元数据。
> > >
> > > > ### xv6 中的`Header`
> > > >
> > > > `header` 是在 xv6 的内存分配器中使用的一个重要结构体。这个分配器的设计基于 Kernighan 和 Ritchie 在《The C Programming Language》第二版第 8.7 节中描述的内存分配器。
> > > >
> > > > ```c
> > > > // user/umalloc.c
> > > > ...
> > > > // Memory allocator by Kernighan and Ritchie,
> > > > // The C programming Language, 2nd ed.  Section 8.7.
> > > > 
> > > > typedef long Align;
> > > > 
> > > > union header {
> > > >   struct {
> > > >     union header *ptr;
> > > >     uint size;
> > > >   } s;
> > > >   Align x;
> > > > };
> > > > 
> > > > typedef union header Header;
> > > > 
> > > > static Header base;
> > > > static Header *freep;
> > > > 
> > > > void
> > > > free(void *ap)
> > > > ...
> > > >   
> > > > void*
> > > > malloc(uint nbytes)
> > > > ...
> > > > ```
> > > >
> > > > ### 代码解释
> > > >
> > > > ### 1. `Align` 和 `Header`
> > > >
> > > > ```c
> > > > typedef long Align;
> > > > ```
> > > >
> > > > `Align` 是用来强制 `header` 对齐的。内存块的开始位置通常需要对齐到某个特定的字节边界（比如 4 字节或 8 字节），以满足硬件架构对内存访问的要求。`Align` 的大小一般和系统的最大对齐要求一致。
> > > >
> > > > ### 2. `union header`
> > > >
> > > > ```c
> > > > union header {
> > > >   struct {
> > > >     union header *ptr;
> > > >     uint size;
> > > >   } s;
> > > >   Align x;
> > > > };
> > > > ```
> > > >
> > > > `union header` 是一个联合体，包含两个成员：
> > > >
> > > > - **`s`**：这是一个结构体，包含两个字段：
> > > >   - `ptr`：指向下一个空闲块的指针。这使得空闲块形成一个链表，便于内存管理。
> > > >   - `size`：当前内存块的大小，单位通常是 `Header` 的数量，而不是字节数。
> > > >
> > > > - **`x`**：用于确保 `header` 的大小和对齐满足 `Align` 的要求。这个字段没有实际的功能，仅用于保证 `header` 的内存对齐。
> > > >
> > > > ### 3. `typedef union header Header`
> > > >
> > > > ```c
> > > > typedef union header Header;
> > > > ```
> > > >
> > > > 将 `union header` 定义为 `Header` 类型，以便在代码中更简洁地引用它。
> > > >
> > > > ### 4. `static Header base` 和 `static Header *freep`
> > > >
> > > > ```c
> > > > static Header base;
> > > > static Header *freep;
> > > > ```
> > > >
> > > > - **`base`**：这是一个静态的 `Header` 变量，通常用于初始化空闲块链表。它充当了链表的一个哨兵节点或起始节点。这个节点不会被实际使用，但它为空闲块链表提供了一个固定的起点。
> > > >
> > > > - **`freep`**：这是指向空闲链表中第一个块的指针。当程序首次调用 `malloc` 或 `free` 函数时，`freep` 会指向 `base`，然后在分配和释放内存时动态更新。
> > > >
> > > > 这个内存分配器基于空闲链表管理内存。`Header` 结构体用来表示每个内存块的元数据（包括大小和下一个块的指针），通过将空闲块链接成链表，内存管理器可以高效地分配和释放内存。在释放内存时，通过合并相邻的空闲块，可以减少内存碎片，提高内存利用率。

## 智能处理 Page Fault

在操作系统的设计中，智能处理 Page Fault 可以显著提升内存管理的效率。下面我们简单分析如何通过 XV6 的代码来实现这一点，特别是在 `usertrap` 函数中进行的修改。在 lazy lab 中还需要完成更多的工作。

### 检查 Page Fault 的原因

```c
// kernel/trap.c
...
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
    printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
...
```

在 `usertrap` 函数中，我们首先需要根据 `SCAUSE` 寄存器的值来判断触发 Page Fault 的原因。在 L06 中，我们遇到的情况有以下几种：

- **普通系统调用**：`SCAUSE == 8` 时，表示这是一个系统调用触发的 Trap。

- **设备中断**：如果不是系统调用，我们会检查是否是设备中断，通过 `devintr()` 函数来处理。
- 两个条件都不满足，打印一些信息并杀掉进程。

现在我们需要增加一个情况：

- **Page Fault**：如果 `SCAUSE == 15`，表示这是由于 Store 操作引发的 Page Fault。

在处理 Page Fault 时，我们需要首先确认虚拟地址 `STVAL` 是否在进程的合法地址范围内。一种可能的处理方式是，我们可以检查 `p->sz` 是否大于 `STVAL` 中保存的虚拟地址。如果 `p->sz` 比虚拟地址大，说明这个虚拟地址是在进程的地址空间内，但尚未实际分配物理内存。

### Page Fault 的处理流程

这里只以演示为目的简单的处理一下，在 lazy lab 中需要完成更多的工作。

如果确定这是一个合法的 Page Fault，我们需要进行如下操作：

1. **调试信息输出**：打印当前 Page Fault 的虚拟地址，以便调试和验证。
2. **物理内存分配**：调用 `kalloc()` 为进程分配一个新的物理页面。如果分配失败（`ka == 0`），则说明系统内存耗尽，必须杀掉进程。
3. **内存初始化**：如果分配成功，将物理页面初始化为全零，以确保内存安全和一致性。
4. **映射虚拟地址**：将物理页面与引发 Page Fault 的虚拟地址关联。为了确保对齐，我们会使用 `PGROUNDDOWN(va)` 将虚拟地址向下对齐到页面边界，然后使用 `mappages()` 函数将虚拟地址和物理页面映射到进程的页表中，并设置权限标志位（`PTE_W | PTE_U | PTE_R`）。

```c
} else if (r_scause() == 15) {
    uint64 va = r_stval();
    printf("page fault %p\n", va);
    uint64 ka = (uint64) kalloc();
    if (ka == 0) {
        p->killed = 1;
    } else {
        memset((void *) ka, 0, PGSIZE);
        va = PGROUNDDOWN(va);
        if (mappages(p->pagetable, va, PGSIZE, ka, PTE_W|PTE_U|PTE_R) != 0) {
            kfree((void *)ka);
            p->killed = 1;
        }
    }
}
```

在这个片段中，如果 `r_scause()` 为 15，表示这是一个 Page Fault（通常是由于访问未映射的内存地址引发的）。

- `r_stval()` 返回引发 Page Fault 的虚拟地址，并将其存储在 `va` 变量中。
- 打印调试信息，输出引发 Page Fault 的虚拟地址。
- 尝试使用 `kalloc()` 函数分配一个新的物理页面。如果分配失败（返回值为 `0`），则标记当前进程为需要终止 (`p->killed = 1`)。
- 如果分配成功，将分配的页面初始化为全零 (`memset`)。
- 使用 `PGROUNDDOWN()` 函数将虚拟地址 `va` 向下取整到页面边界，确保其对齐。
- 尝试将新的物理页面映射到用户空间的虚拟地址。如果 `mappages()` 函数调用失败，释放刚才分配的物理页面，并标记当前进程为需要终止。

### 出现双重 Page Fault 的问题

![image-20240820203834580]({{ site.baseurl }}/docs/assets/image-20240820203834580.png)

再次执行`echo hi`并没有正常工作。在处理完一个 Page Fault 后，我们遇到另一个 Page Fault。在图中，第二个 Page Fault 地址为 `0x13f48`，`uvmunmap`在报错。这是因为未分配的虚拟地址空间尝试被释放，即使这些地址空间还没有对应的物理内存。通常，`uvmunmap` 函数会尝试解除这些映射，这时会检测到没有物理页面对应的情况，而产生错误信息。

解决这一问题需要在代码中进行进一步的检查和验证，确保未分配的内存空间不会被误操作。

## 处理 `uvmunmap` 中的 Page

在继续深入讨论 `usertrap` 的 Page Fault 处理逻辑之前，我们首先需要分析 `uvmunmap` 函数的 Page Fault 。

```c
// kernel/vm.c
...
// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
...
```

### `uvmunmap` 函数的修改分析

原始的 `uvmunmap` 函数在发现某个页表项（PTE）的 `V` 标志位（表示页面有效）为 0 时，会直接 panic，表示这是一个错误的状态。这样的设计在传统的 `XV6` 环境中是合理的，因为默认情况下，所有用户进程的内存都应该是映射过的。未映射的内存被访问到会引发错误，因此直接 panic 是一种防御性编程的策略。

```c
if((*pte & PTE_V) == 0)
  panic("uvmunmap: not mapped");
```

然而，随着我们引入 Lazy Allocation 的概念，情况发生了变化。

当启用了 Lazy Allocation 时，用户进程请求的某些内存区域可能已经被记录在进程的 `sz`（大小）字段中，但并没有实际分配物理内存。这就意味着，有些页表项在逻辑上是属于进程的地址空间的，但因为它们从未被访问过，所以它们没有映射到物理内存。此时，如果 `uvmunmap` 函数仍然坚持要求所有 PTE 都必须是有效的，那就会导致不必要的 panic。

解决方案是**直接跳过**这些未映射的页表项，因为这些页表项本来就没有物理内存与之关联，也就不存在实际的内存释放问题。在 Lazy Allocation 中，我们有意让一些内存页面在最初不进行映射，直到真正访问它们时才分配物理内存。因此，跳过这些未映射的页面是合乎逻辑的行为。这正是 `continue` 语句的作用。

```c
if((*pte & PTE_V) == 0)
  continue;
```

### 重编译和运行

在做了上述修改后，我们重新编译 XV6，并运行 `echo hi` 命令。尽管有两个 Page Fault 被触发，但由于我们已经在代码中处理了这些情况，最终程序成功执行了 `echo hi`，这标志着 Lazy Allocation 的基本框架已经搭建起来了。

![image-20240820205301370]({{ site.baseurl }}/docs/assets/image-20240820205301370.png)

随着 Lazy Allocation 的引入，XV6 的内存管理机制变得更加复杂，这对系统的其他部分也提出了更高的要求。特别是我们需要考虑如何在缩小用户进程的地址空间时，安全地处理这些未映射的页面，避免可能出现的内存泄露或其他问题。

通过这些改动，我们已经在 XV6 中实现了一个基础的 Lazy Allocation 机制。这不仅使得内存管理更加高效，而且为后续的实验打下了基础。

---
layout: page
title: L06 Isolation & System Call Entry/Exit
permalink: /L06
nav_order: 6





---

# Lecture 6 - Isolation & System Call Entry/Exit

**隔离&系统调用进入/退出**

## 用户空间与内核空间的切换

在操作系统中，程序运行时经常会在用户空间和内核空间之间进行切换。这种切换在以下几种情况下发生：

1. **程序执行系统调用**：例如，当程序需要访问硬件资源或操作文件时，必须通过系统调用进入内核空间。
2. **程序出现错误**：如页面错误（page fault）或除以零等异常，程序需要切换到内核空间进行处理。
3. **设备中断**：当外部设备（如键盘、硬盘等）触发中断时，当前程序必须切换到内核空间响应中断。

这种切换通常被称为**陷入（trap）**。陷入机制的设计对于系统的安全性和性能至关重要。因为系统调用和异常处理频繁发生，所以陷入机制必须设计得尽可能简单高效。

### 用户空间到内核空间的切换流程

当用户程序（例如Shell）在用户空间运行时，它可能会通过执行系统调用进入内核空间。一个典型的例子是Shell通过`write`系统调用输出信息到终端。在此过程中，程序的执行环境从用户模式切换到内核模式，这涉及到硬件状态的变化。

以下是 `trap` 过程中一些关键的硬件状态和寄存器：

1. **用户寄存器**：RISC-V架构中有32个通用寄存器（如`a0`，`a1`等），这些寄存器可以被用户程序使用，并且它们的使用效率最高。

2. **堆栈指针寄存器（Stack Pointer Register, SP）**：这是32个寄存器之一，它指向当前堆栈的顶部位置。

3. **程序计数器寄存器（Program Counter Register, PC）**：这个寄存器保存当前正在执行的指令地址。

4. **模式寄存器（Mode Register）**：这个寄存器标志着当前处理器的模式，是用户模式（User Mode）还是内核模式（Supervisor Mode）。

5. **控制寄存器**：

   - **SATP寄存器（Supervisor Address Translation and Protection Register）（地址翻译与保护寄存器）**：这个寄存器指向页表的物理地址，用于内存地址的转换和保护。陷入过程中，内核需要访问不同的内存区域，SATP寄存器提供了页表的物理地址，帮助进行内存地址转换。

   - **STVEC寄存器（Supervisor Trap Vector Base Address Register）（陷入向量基地址寄存器）**：这个寄存器保存了内核处理陷入的入口地址。当Shell在用户模式下运行并执行系统调用时，硬件会自动将控制权交给内核，查找STVEC寄存器，找到内核处理陷入的入口地址。

   - **SEPC寄存器（Supervisor Exception Program Counter Register）（异常程序计数器寄存器）**：在发生陷入时，这个寄存器保存了程序计数器的值，即导致陷入的指令地址。确保在内核处理完陷入后能够返回正确的位置继续执行。

   - **SSRATCH寄存器（Supervisor Scratch Register）（临时寄存器）**：这是一个临时存储寄存器，通常用于在陷入过程中保存一些关键数据。

## Trap机制中的状态转换与保护

在用户程序运行过程中，当发生 `trap`（系统调用、中断或异常）时，操作系统需要做出一系列复杂的操作来切换到内核模式下执行。这种切换不仅涉及硬件状态的改变，还必须确保系统的安全性和对用户代码的透明性。

### 需要在 Trap 中处理的状态

1. **保存32个用户寄存器**：当 `trap` 发生时，操作系统必须保存所有32个用户寄存器的内容。这是因为在 `trap` 处理完成后，操作系统需要恢复用户程序的执行。如果 `trap` 是由设备中断引发的，用户程序不应该察觉到它的执行被中断了。因此，操作系统会在进入内核之前将这些寄存器的内容保存起来，以便稍后恢复。

2. **保存程序计数器（PC）**：程序计数器记录了用户程序正在执行的指令地址。与用户寄存器一样，操作系统需要保存这个值，以便在 `trap` 处理完成后，能够继续执行被中断的用户程序。

3. **切换到 `supervisor mode`**：内核代码具有更高的权限，因此需要将 CPU 的模式切换到 `supervisor mode`。这允许内核使用特权指令，并访问更多的系统资源。

4. **更新 SATP 寄存器**：`SATP`（Supervisor Address Translation and Protection）寄存器指向当前的页表。用户页表通常只映射了用户程序需要的内存，而内核代码需要访问整个内存。因此，在 `trap` 发生后，操作系统需要将 `SATP` 指向内核页表，以便内核代码可以正确访问内存。

5. **更新堆栈指针（SP）**：当切换到内核模式时，操作系统需要将堆栈指针指向内核中的一个安全地址，以便内核可以使用它来管理函数调用栈。

6. **跳转到内核 C 代码**：一旦所有硬件状态被正确配置，操作系统会跳转到内核中的 C 代码，开始处理 `trap`。在内核中运行的代码与普通 C 代码无异，主要区别在于它运行在更高权限的 `supervisor mode` 下。

### 操作系统设计的高层目标

操作系统设计的一些高层目标直接影响了 `trap` 机制的实现方式，主要有两个核心目标：

1. **安全性和隔离性**：`trap` 机制必须确保用户代码不能干预 `trap` 处理过程，否则可能会导致系统安全性的漏洞。因此，在 `trap` 过程中，操作系统不依赖于用户寄存器的内容，这些寄存器仅被保存，而不被用于决定 `trap` 的行为。

2. **透明性**：对于用户程序来说，`trap` 机制应该是完全透明的。用户程序不应该察觉到它的执行被中断并进入内核处理系统调用或异常。操作系统会在处理完 `trap` 之后恢复所有寄存器和程序计数器，使得用户程序可以从中断的位置继续执行，而不感知 `trap` 的存在。

### Trap机制的实现步骤

1. **保存上下文**：保存用户程序的所有寄存器和程序计数器内容到内核栈中。
2. **切换到内核模式**：切换到 `supervisor mode`，以获得更高的权限。
3. **更新页表**：更新 `SATP` 寄存器，指向内核页表。
4. **设置内核栈**：将堆栈指针指向内核栈，准备处理内核中的函数调用。
5. **执行内核代码**：跳转到内核中的 `trap` 处理代码，执行系统调用或处理异常。
6. **恢复上下文**：在内核处理完成后，恢复用户寄存器和程序计数器的内容。
7. **返回用户模式**：切换回 `user mode`，继续执行用户程序。

`trap` 机制是操作系统中至关重要的一部分，它不仅要处理用户空间与内核空间的切换，还要在此过程中确保系统的安全性和对用户代码的透明性。通过设计一套高效、安全的 `trap` 机制，操作系统可以在保证稳定性和安全性的前提下，高效地处理系统调用、中断和异常。

## 安全切换到内核模式

在用户空间和内核空间之间的切换过程中，安全性是至关重要的，尤其是在操作系统内核中实现这一切换机制时。下面我们将详细探讨这些安全措施，以及在 `trap` 机制中涉及的关键寄存器和模式切换。

### 用户空间到内核空间的安全切换

虽然我们在讨论从用户空间切换到内核空间的过程时，关注了隔离和安全，但这仅仅是整个操作系统安全性的一个方面。事实上，内核中所有的代码都需要特别注意安全性。即使 `trap` 机制本身是安全的，整个内核的其他部分也必须经过严密设计，以防止用户代码尝试欺骗或利用内核漏洞。

### 模式标志位的重要性

在前面我们提到过一个特殊的寄存器——模式标志位（mode bit）。该标志位用于区分当前处理器是运行在 user mode 还是 supervisor mode 。以下是与模式标志位相关的关键点：

1. **模式切换的权限控制**：当模式从 user mode 切换到 supervisor mode 时，处理器会获得更多的权限。这些权限主要包括对一些控制寄存器的读写能力。虽然 supervisor mode 确实赋予了操作系统更多的控制权，但这些权限比很多人预期的要少。

2. **控制寄存器的访问权限**：在 supervisor mode 下，可以读写如下关键的控制寄存器：
   - **SATP（Supervisor Address Translation and Protection）寄存器**：它指向当前使用的页表，是内存地址转换的关键。
   - **STVEC（Supervisor Trap Vector Base Address Register）寄存器**：存储了处理 `trap` 的内核代码的起始地址。
   - **SEPC（Supervisor Exception Program Counter）寄存器**：当发生 `trap` 时，保存当时的程序计数器。
   - **SSCRATCH（Supervisor Scratch Register）寄存器**：这是一个通用寄存器，用于在 `trap` 过程中保存数据。

3. **PTE_U 标志位的控制**：在页表条目（PTE）中，`PTE_U` 标志位决定了该页表条目是否对用户代码可见。如果 `PTE_U=1`，那么用户代码可以访问该页表指向的内存；如果 `PTE_U=0`，那么只有 supervisor mode 下的代码可以访问该页表指向的内存。这一机制确保了用户代码不能访问不应访问的内存区域。

###  supervisor mode 的局限性

尽管 supervisor mode 提供了对系统资源的更多控制，但它并不能随意访问所有物理内存。 supervisor mode 的代码依然需要通过当前 `SATP` 指向的页表来访问内存。如果某个虚拟地址在当前页表中不存在，或者该地址的 `PTE_U` 标志位为1，那么 supervisor mode 下的代码也无法访问该地址。这一限制确保了即使在 supervisor mode 下，代码也受到严格的内存访问控制。

### 进入内核空间时的 Trap 执行流程

接下来，我们将详细探讨在进入内核空间时 `trap` 代码的执行流程。该流程涉及一系列关键步骤，包括保存用户上下文、切换模式、调整控制寄存器、设置内核栈指针等。这些步骤共同确保系统能够安全、高效地从用户模式切换到内核模式，并开始执行内核代码。

```
[User Space] ----(System Call / Trap)----> [Kernel Space]

              |                                 |
        Save User Context                 Load Kernel Context
              |                                 |
          Switch to                         Execute Kernel
       Supervisor Mode                           Code
              |                                 |
          Adjust SATP                       Return to User
          (Point to Kernel                   Space and
           Page Table)                     Restore Context
              |                                 |
          Set SP to Kernel                  Continue User
              Stack                            Execution
```

操作系统在用户空间和内核空间之间的切换过程中，通过合理的机制设计，确保了系统的安全性和稳定性。特别是在 `trap` 处理流程中，操作系统必须小心地保存和恢复上下文，控制硬件状态，并严格控制内存访问权限。这些机制的设计和实现对于确保操作系统的安全和性能至关重要。

##  `trap` 处理流程概述

在操作系统中，当用户程序需要访问系统资源或处理异常情况时，会通过 `trap` 机制切换到内核空间来执行相关操作。这个切换过程涉及一系列精密的步骤和函数调用，确保用户程序能够安全且有效地请求内核服务。下面我们详细讨论这个切换过程的具体实现，以及涉及的关键函数和汇编代码。

要理解从用户空间切换到内核空间的 `trap` 处理流程，首先需要了解操作系统如何响应用户程序的系统调用请求以及如何处理异常和中断。在这个过程中，涉及多个关键函数和汇编代码段，这些代码在内核中逐步处理系统调用、异常或中断，并最终恢复到用户空间继续执行。

1. **Shell 调用系统调用**：
   - 从用户程序（如 Shell）的角度来看，系统调用就像是一个普通的 C 函数调用。例如，当 Shell 调用 `write` 系统调用时，实际上会执行一条特定的机器指令（在 RISC-V 中是 `ECALL` 指令）来触发 `trap`，从而切换到内核空间执行请求。

2. **执行 `ECALL` 指令**：
   - 当用户程序执行 `ECALL` 指令时，处理器会从用户模式切换到超级用户模式（`supervisor mode`），并跳转到 `STVEC` 寄存器中存储的内核处理陷入的入口地址。`STVEC` 寄存器指向内核中的一个汇编代码位置，在 XV6 中，这个位置是 `trampoline.s` 文件中的 `uservec` 函数。

3. **汇编函数 `uservec`**：
   - `uservec` 函数是 `trap` 处理过程的第一个执行步骤。该函数的主要任务是保存当前用户空间的寄存器状态，以便稍后可以恢复。这一步确保了用户程序的执行上下文不会因为切换到内核空间而丢失。`uservec` 还会准备好进入内核空间处理系统调用的必要环境。

4. **C 函数 `usertrap`**：
   - 在 `uservec` 函数保存用户空间的寄存器状态后，控制权会跳转到由 C 语言实现的 `usertrap` 函数。`usertrap` 函数位于 `trap.c` 文件中，它负责识别 `trap` 的类型（是系统调用、异常还是中断），并执行相应的处理逻辑。对于系统调用，`usertrap` 将进一步调用 `syscall` 函数处理。

5. **执行 `syscall` 函数**：
   - 在 `usertrap` 中，如果检测到 `trap` 是由 `ECALL` 指令触发的系统调用，系统会调用 `syscall` 函数。`syscall` 函数通过检查系统调用号（通常存储在特定寄存器中），查找并执行相应的系统调用处理函数。例如，对于 `write` 系统调用，`syscall` 会调用 `sys_write` 函数来执行实际的写操作。

6. **内核功能执行**：
   - 在 `sys_write` 函数中，数据被实际输出到控制台或其他指定的输出设备。系统调用处理完成后，`syscall` 函数返回结果，并将控制权交还给 `usertrap`。

7. **恢复用户空间**：
   - 在内核完成必要的操作后，`usertrap` 函数会调用 `usertrapret` 函数。`usertrapret` 函数负责部分内核状态的恢复，并为返回用户空间做好准备。

8. **汇编函数 `userret`**：
   - 返回用户空间的最后步骤由 `trampoline.s` 文件中的 `userret` 函数完成。`userret` 恢复用户程序的寄存器状态，并执行相关指令以返回用户空间，使用户程序从 `ECALL` 指令之后继续执行。

9. **继续用户程序的执行**：
   - 用户程序恢复执行，继续从 `ECALL` 之后的代码运行。此时，用户程序已经完成了请求的系统调用，可能得到了某个结果或数据。

```plaintext
[User Space: Shell] 
     |
  ECALL (write)
     |
  [trap: STVEC -> uservec (assembly)]
     |
  Save user registers
     |
  [usertrap (C function)]
     |
  if syscall:
     |
  -> syscall -> sys_write
     |
  [usertrapret (C function)]
     |
  Restore kernel state
     |
  [userret (assembly)]
     |
  Restore user registers
     |
  Return to User Space
```

为了更好地理解 `trap` 的执行流程，可以使用调试工具如 `gdb` 来跟踪代码执行路径。通过设置断点、逐步执行代码、检查寄存器和内存状态，你可以深入分析系统调用的内部机制。例如在 `usertrap` 函数或 `syscall` 函数中设置断点，然后通过 `step` 和 `next` 命令逐步执行代码，观察系统调用的具体实现和执行结果。

在 XV6 中，`trap` 机制不仅实现了用户空间与内核空间的安全切换，还确保了内核对用户程序的严格控制和管理。这种机制使操作系统能够有效地管理系统资源，处理用户请求，并保证系统的稳定和安全。

## 使用 GDB 跟踪 XV6 系统调用的执行过程

在这一部分，我们将通过 GDB 进行实时调试，跟踪一个 XV6 系统调用的完整执行流程。我们具体分析的是 Shell 程序将提示信息写入控制台的 `write` 系统调用的实现过程。

### 系统调用的触发

我们从用户空间的 `sh.c` 文件中的 `write` 系统调用开始。在用户代码中，`write` 调用实际触发了与 Shell 关联的库函数。

![image-20240818145439228]({{ site.baseurl }}/docs/assets/image-20240818145439228.png)

示例代码是 Shell 通过文件描述符 2 执行 `write` 系统调用的地方，写入的数据是提示符“$ ”。接下来，我们通过 GDB 跟踪这次系统调用的具体过程。

### 进入 GDB 调试环境

我们启动 XV6 的 GDB 调试器，开始调试整个系统调用的过程。

当 Shell 代码调用 `write` 系统调用时，它实际上调用的是系统库中的一个函数，这个函数的实现可以在 `usys.s` 文件中找到。

```nasm
.global write
write:
 li a7, SYS_write
 ecall
 ret
```

这几行汇编代码展示了 `write` 系统调用的具体实现：

1. **加载系统调用编号**：
   - 首先，将系统调用编号 `SYS_write`（常量 16）加载到 `a7` 寄存器中。这个操作告诉内核接下来将执行编号为 16 的系统调用，即 `write`。

2. **执行 `ecall` 指令**：
   - 随后，执行 `ecall` 指令。这一指令会触发从用户模式到内核模式的切换，使得程序控制权从用户空间转移到内核空间。

3. **返回到用户空间**：
   - 当内核完成 `write` 操作后，程序会返回用户空间，继续执行 `ret` 指令，从 `write` 函数返回到 Shell 中。

### 在 GDB 中设置断点

![image-20240818150616190]({{ site.baseurl }}/docs/assets/image-20240818150616190.png)

为了更好地观察 `ecall` 指令的执行，我们在 `ecall` 指令处设置了一个断点。要找到 `ecall` 指令的具体地址，我们可以查阅 XV6 编译过程中生成的 `sh.asm` 文件，该文件记录了汇编代码和对应的指令地址。

```
(gdb) b *0xde6
```

在这个例子中，`ecall` 指令的地址是 `0xde6`，因此我们在该地址处设置了一个断点。

### 执行 XV6 并触发断点

我们开始执行 XV6，期望程序在 Shell 代码中的 `ecall` 指令处停住。

从 GDB 的输出中可以看出，程序确实停在了 `ecall` 指令之前。为了验证我们确实在预期的位置，我们可以打印程序计数器（Program Counter, PC）：

![image-20240818150845955]({{ site.baseurl }}/docs/assets/image-20240818150845955.png)

通过 GDB，我们可以确认当前程序计数器（PC）的值确实是 `0xde6`，这表明我们在预期的位置上成功设置了断点。

通过 GDB 逐步调试，我们能够清晰地看到从用户空间到内核空间的切换过程。这不仅帮助我们理解 `trap` 机制的底层工作原理，还展示了如何在 GDB 中有效地跟踪和调试 XV6 的系统调用执行过程。这个过程不仅展示了 XV6 内核的具体实现，也帮助我们理解了操作系统如何管理和调度系统资源以响应用户请求。

### 打印用户寄存器和系统调用的参数

在跟踪 `write` 系统调用的过程中，我们可以使用 GDB 的 `info reg` 命令打印出当前所有 32 个用户寄存器的状态。

```bash
(gdb) info reg
```

![image-20240818151342435]({{ site.baseurl }}/docs/assets/image-20240818151342435.png)

从上图可以看到，寄存器 `a0`、`a1`、`a2` 保存了 Shell 传递给 `write` 系统调用的参数：

- `a0`：文件描述符（2），表示标准错误输出（stderr）。
- `a1`：指向 Shell 想要写入字符串的指针。
- `a2`：要写入的字符数。

我们还可以通过打印 `a1` 寄存器所指向的内存地址来验证要写入的字符串内容。

```bash
(gdb) x/2c $a1
0x12e0: 36 '$'  32 ' '
```

从输出可以看出，字符串的内容确实是美元符号 `$` 和一个空格，这表明程序已经运行到我们预期的 `write` 系统调用位置。

### 用户空间地址的确认

从寄存器的内容来看，程序计数器 (`pc`) 和堆栈指针 (`sp`) 的地址都位于相对较小的内存区域。这进一步证明了当前代码在用户空间运行，因为用户空间的地址通常较小。一旦程序进入内核空间，内核将使用更大的内存地址。

### 检查当前的 Page Table

在执行系统调用时，系统状态会发生很多重要的变化，其中之一就是当前的 Page Table。我们可以通过查看 `SATP` 寄存器来确认当前使用的 Page Table。

```bash
(gdb) print/x $satp
$2 = 0x8000000000087f63
```

SATP` 寄存器保存了当前正在使用的 Page Table 的物理地址，但并没有直接展示 Page Table 的具体映射关系。

### 使用 QEMU 查看 Page Table

幸运的是，在 QEMU 中有一种方法可以打印当前的 Page Table。进入 QEMU 控制台（通过按下 `Ctrl + A` 然后 `C`），输入以下命令查看内存布局和 Page Table 信息：

```bash
(qemu) info mem
```

![image-20240818151937890]({{ site.baseurl }}/docs/assets/image-20240818151937890.png)

这个 Page Table 非常小，仅包含 6 条映射关系。它是用户程序 Shell 的 Page Table，Shell 作为一个小程序，Page Table 仅包含与 Shell 指令和数据相关的映射，以及一个无效的页，用作 Guard Page，防止 Shell 使用过多的堆栈空间。

- **虚拟地址**：Page Table 映射的虚拟地址。
- **物理地址**：对应的物理内存地址。
- **attr** 列：标志位，包括 `rwx`（可读、可写、可执行）、`u`（用户模式可访问）、`a`（是否访问过）、`d`（是否被修改过）等。

### Page Table 中的特定映射

我们注意到，Page Table 中的最后两条映射指向虚拟地址空间的顶端，分别对应 `trapframe` 和 `trampoline` 页表。用户代码不能访问这两条映射，因为它们的 `u` 标志位没有设置（即用户模式不可访问）。一旦程序进入 `supervisor mode`，这些映射将变得可访问。

此外，这个 Page Table 中没有包含任何与内核部分地址的映射。除了最后两条特定映射外，该 Page Table 完全为用户代码执行而设计，并没有直接包含内核数据或指令的映射。

> **提问**：PTE 中 `a` 标志位是什么意思

`a` 表示这条 PTE 是否被访问过，即是否有某个内存访问操作涉及到该 PTE 范围内的地址。`d` 标志位表示该 PTE 所映射的内存是否被写入过。这些标志位由硬件维护，供操作系统使用。在更复杂的操作系统中，`a` 标志位可以帮助操作系统决定哪些内存页可以被释放或交换到磁盘中。

## 准备进入内核空间

最后，我们可以通过 `x/i` 命令打印出当前即将执行的指令，并确认程序计数器指向 `ecall` 指令。

```bash
(gdb) x/i $pc
```

![image-20240818152719231]({{ site.baseurl }}/docs/assets/image-20240818152719231.png)

现在程序仍然在用户空间中，一旦执行 `ecall` 指令，程序将进入内核空间，开始处理 `write` 系统调用。

> > ### GDB 和 QEMU 命令解释及类似命令补充
> >
> > ### 1. `(gdb) x/i $pc`
> >
> > - **解释**: 这个命令用于查看当前程序计数器（`$pc`）处的机器指令。`x` 是 GDB 中的“examine”命令，`/i` 表示按照指令的格式来查看内容，而 `$pc` 是当前的程序计数器寄存器，指向下一条将要执行的指令。
> >
> >   **示例输出**:
> >
> >     ```
> >   => 0x08048484 <main+4>:    mov    %esp,%ebp
> >     ```
> >
> > - **类似命令**:
> >
> >  - `(gdb) x/x $pc`：以十六进制格式查看程序计数器处的内容。
> >
> >   - `(gdb) x/2i $pc`：查看从程序计数器开始的两条指令。
> >
> >   - `(gdb) disassemble`：反汇编当前函数的代码。
> >
> > ### 2. `(gdb) x/3i 0xde4`
> >
> > - **解释**: 这个命令查看从内存地址 `0xde4` 开始的三条机器指令。`x/3i` 表示以指令格式查看三条指令，后面的地址 `0xde4` 是指令的起始地址。
> >
> >   **示例输出**:
> >
> >     ```
> >   0x00000de4:    mov    %ebp,%esp
> >   0x00000de5:    pop    %ebp
> >   0x00000de6:    ret
> >     ```
> >
> > - **类似命令**:
> >
> >  - `(gdb) x/5i 0x1000`：查看从内存地址 `0x1000` 开始的五条指令。
> >
> >   - `(gdb) x/4i $sp`：查看从栈指针（`$sp`）指向的位置开始的四条指令。
> >
> >   - `(gdb) disassemble 0xde0,0xdf0`：反汇编从 `0xde0` 到 `0xdf0` 范围内的指令。
> >
> > ### 3. `(gdb) print/x $satp`
> >
> > - **解释**: 这个命令打印 RISC-V 中 `satp` 寄存器的值，格式为十六进制。`print/x` 表示以十六进制格式打印内容，`$satp` 是 `satp` 寄存器的名称，它通常用于管理虚拟内存的页表基址。
> >
> >   **示例输出**:
> >
> >     ```
> >   $1 = 0x8000000000083
> >     ```
> >
> > - **类似命令**:
> >
> >   - `(gdb) print $pc`：打印程序计数器寄存器的值。
> >   - `(gdb) info registers`：显示所有寄存器的当前值。
> >   - `(gdb) info registers satp`：仅显示 `satp` 寄存器的当前值。
> >
> > ### 4. `(gdb) x/2c $a1`
> >
> > - **解释**: 这个命令查看从寄存器 `a1` 指向的地址开始的两个字符。`x/2c` 表示以字符格式查看两个单位，`$a1` 是寄存器 `a1` 的地址。
> >
> >   **示例输出**:
> >
> >   ```
> >   0x7fffffffe3c8:    0x41 'A'    0x42 'B'
> >   ```
> >
> > - **类似命令**:
> >
> >   - `(gdb) x/4c $a2`：查看从寄存器 `a2` 指向的地址开始的四个字符。
> >   - `(gdb) x/s $a1`：查看从寄存器 `a1` 指向的地址开始的字符串。
> >   - `(gdb) x/8b $a1`：查看从寄存器 `a1` 指向的地址开始的8个字节内容。
> >
> > ### 5. `(qemu) info mem`
> >
> > - **解释**: 这个命令用于显示 QEMU 虚拟机中当前的内存布局和状态。它会列出已映射的内存区域及其属性，如起始地址、大小和读写权限等。
> >
> >   **示例输出**:
> >
> >   ![image-20240818151937890]({{ site.baseurl }}/docs/assets/image-20240818151937890.png)
> >
> >   - **类似命令**:
> >   - `(qemu) info registers`：显示虚拟机中 CPU 的寄存器状态。
> >   - `(qemu) info cpus`：显示虚拟机中的 CPU 状态。
> >   - `(qemu) info mtree`：显示内存和设备树。

## 执行 `ecall` 指令后的状态变化

### 执行 `ecall` 指令后的位置确认

现在，我们执行了 `ecall` 指令。执行完 `ecall` 指令后，首要的问题是：我们现在的执行位置在哪？可以通过打印程序计数器（Program Counter, PC）来查看当前执行位置。

```bash
(gdb) print $pc
```

![image-20240818155426961]({{ site.baseurl }}/docs/assets/image-20240818155426961.png)

从结果可以看出，程序计数器的值发生了变化。之前程序计数器指向一个较小的地址 `0xde6`，这是我们在用户空间执行 `ecall` 指令前的位置。而现在，程序计数器指向了一个较大的地址。

### 验证页表是否发生变化

为了确认页表的状态，我们可以在 QEMU 中执行 `info mem` 命令，查看当前的页表。

```bash
(qemu) info mem
```

![image-20240818160159187]({{ site.baseurl }}/docs/assets/image-20240818160159187.png)

结果显示，当前页表与之前的页表完全相同，这意味着执行 `ecall` 指令后，页表并没有改变。即便如此，程序计数器现在指向了一个很大的地址，这表明代码已经切换到内存中的 `trampoline` 页。

### 查看当前执行的指令

我们可以进一步查看程序计数器当前指向的指令。

![image-20240818155756461]({{ site.baseurl }}/docs/assets/image-20240818155756461.png)

这些指令是内核在 `supervisor` 模式下执行的最初几条指令，也是 `trap` 机制中最早执行的指令之一。由于 GDB 的一些行为，我们实际上已经执行了 `trampoline` 页中第一条指令（即 `csrrw` 指令），而当前正准备执行第二条指令。

### 寄存器状态与 `csrrw` 指令的作用

我们可以查看当前寄存器的状态。对比之前的状态可以发现，寄存器的内容没有发生变化，依然是用户程序的数据。

![image-20240818160136579]({{ site.baseurl }}/docs/assets/image-20240818160136579.png)

由于这些寄存器中还保存着用户程序的关键数据，在将这些数据妥善保存到内核空间的某处之前，我们必须非常谨慎，不能轻易使用这些寄存器。否则，一旦内核在此时使用了任何寄存器，就会覆盖掉其中的用户数据，导致后续用户程序恢复时数据错误，程序执行失败。

### `csrrw` 指令的作用

关于 `csrrw` 指令，这条指令是如何帮助内核在不使用任何用户寄存器的情况下执行操作的关键。`csrrw` 指令的作用是交换寄存器 `a0` 和 `sscratch` 的内容：

```
csrrw a0, sscratch, a0
```

这一操作非常重要，因为它允许内核在保存用户寄存器数据之前，首先将关键的 `a0` 寄存器内容存储到 `sscratch` 寄存器中。同时，它也将 `sscratch` 寄存器的内容加载到 `a0` 中，这样内核可以在不破坏用户数据的前提下使用 `a0` 寄存器进行其他操作。

这一步确保了 `trap` 机制的安全性，使得内核可以安全地过渡到使用内核的寄存器和堆栈进行操作，而不会干扰用户空间的数据。



## Trampoline Page 和 ecall 指令的工作机制

### 进入 Trampoline Page 并开始执行 Trap 处理代码

我们目前位于地址 `0x3ffffff000`，这是我们之前在 page table 输出中看到的最后一个 page，即 Trampoline Page。此时我们正在 Trampoline Page 中执行程序，这个 page 包含了内核的 trap 处理代码。`ecall` 指令不会切换 page table，这是 `ecall` 的一个重要特点。这意味着，trap 处理代码必须存在于每个用户 page table 中，以便在用户空间的 page table 仍在使用时，能够在一个受控的地方执行内核代码的最初部分。

### STVEC 寄存器和 Trampoline Page 的映射

内核通过设置 STVEC 寄存器来控制代码的执行位置，STVEC 是一个只能在 supervisor mode 下读写的特权寄存器。在从内核空间进入用户空间之前，内核会预先将 STVEC 寄存器的内容设置为指向内核希望 trap 代码运行的位置。

```bash
(gdb) print/x $stvec
$4 = 0x3ffffff000
```

在我们的示例中，STVEC 寄存器被设置为 `0x3ffffff000`，即 Trampoline Page 的起始位置。因此，当 `ecall` 指令被执行时，程序计数器将指向这个地址并开始执行。

尽管 Trampoline Page 被映射在用户空间的 page table 中，用户代码是无法写入它的。这是因为这些 page 对应的 PTE 并没有设置 PTE\_u 标志位，这保证了 trap 机制的安全性。

我们可以通过代码执行情况推测确认当前运行模式。因为程序计数器现在指向 Trampoline Page 并且代码没有崩溃，这意味着我们必然处于 supervisor mode。我们是通过ecall来到 Trampoline Page 的。

### ecall 指令的作用

执行 `ecall` 指令会导致以下三件事情发生：

1. **切换到 Supervisor Mode**: `ecall` 将代码从 user mode 切换到 supervisor mode。

2. **保存程序计数器**: `ecall` 会将程序计数器的值保存在 SEPC 寄存器中。我们可以通过打印 SEPC 寄存器来确认这一点。

   ```bash
   (gdb) print/x $sepc
   $6 = 0xde6
   ```

   SEPC 寄存器中保存的地址 `0xde6` 对应的是 `ecall` 指令在用户空间的位置。

3. **跳转到 STVEC 指向的地址**: `ecall` 将程序计数器设置为 STVEC 寄存器指向的地址，即 Trampoline Page 的起始位置，并开始执行内核 trap 代码。

### 下一步需要完成的任务

虽然 `ecall` 指令帮我们完成了部分工作，但要进入内核并执行 C 代码，还有以下几件事需要完成：

1. **保存 32 个用户寄存器的内容**: 这样才能在将来恢复用户程序的执行时，还原寄存器的状态。
2. **切换到 Kernel Page Table**: 当前我们仍在使用 user page table，需要切换到 kernel page table 才能访问内核内存。
3. **设置 Kernel Stack**: 需要找到或创建一个 kernel stack，并将 Stack Pointer 寄存器指向这个 stack，以便 C 代码有栈空间可用。
4. **跳转到内核 C 代码的起始位置**: 完成上述步骤后，需要跳转到内核 C 代码的合理起始位置，继续执行。

`ecall` 指令并不会为我们自动完成这些工作，所以接下来我们需要手动完成这些步骤，以便顺利过渡到内核的 C 代码执行。

## RISC-V 架构中 ecall 的设计理念及其灵活性

### 为什么 ecall 指令仅完成最少量的工作？

虽然我们可以通过修改硬件来让 `ecall` 指令完成更多的工作，例如保存用户寄存器、切换 page table 指针、设置 Stack Pointer 并跳转到内核的 C 代码，但在 RISC-V 架构中，`ecall` 指令仅完成最少的必要操作。这样设计的原因是 RISC-V 的设计者希望为操作系统的开发者提供最大的灵活性，让他们能够根据需要定制系统调用的实现方式。

### 硬件和软件职责的分配

在 RISC-V 架构中，`ecall` 指令仅执行以下三个基本操作：

1. **切换到 Supervisor Mode**：`ecall` 将 CPU 的模式从 User Mode 切换到 Supervisor Mode，这是系统调用进入内核的第一步。

2. **保存程序计数器**：`ecall` 会将当前的程序计数器（Program Counter）的值保存到 SEPC 寄存器中，这样当系统调用处理完毕后，可以恢复用户程序的执行。

3. **跳转到 STVEC 指向的地址**：`ecall` 会将程序计数器的值更新为 STVEC 寄存器中的地址（通常是 trampoline page 的起始地址），并开始执行内核的 trap 处理代码。

### 设计灵活性的好处

这种简化的设计带来了很多灵活性，使得操作系统开发者可以根据不同的需求优化系统调用的性能：

1. **延迟或避免 page table 切换**：切换 page table 的开销较高，因此某些操作系统可能希望在不切换 page table 的前提下执行某些系统调用。RISC-V 中的 `ecall` 不会自动切换 page table，从而为这类优化提供了可能。

2. **保持 user 和 kernel 的映射一致**：一些操作系统将用户空间和内核空间的虚拟地址映射到同一个 page table 中，这样在用户空间和内核空间之间切换时不需要更改 page table。如果 `ecall` 强制切换 page table，这种优化就无法实现。

3. **选择性保存寄存器**：根据具体的系统调用，有时不需要保存所有的 32 个寄存器。例如，在某些系统调用中，可能只有少数寄存器会被修改。通过不保存不必要的寄存器，可以提高性能。

4. **避免不必要的 stack 切换**：对于某些简单的系统调用，可能不需要使用内核栈。`ecall` 不会自动切换 stack pointer，这样的设计为对性能要求极高的操作系统提供了优化空间。

### 关于 gdb 中 `ecall` 的显示

`ecall` 指令的功能主要是切换 CPU 模式，并将程序计数器更新为 STVEC 寄存器的值。在 gdb 中，我们无法直接看到 `ecall` 的具体实现细节，因为 `ecall` 是 CPU 的一条指令，它只负责更新 CPU 的模式标志位并调整程序计数器。`ecall` 完成这些操作后，程序计数器会指向 trampoline page 的起始地址，然后开始执行 trampoline.s 中的汇编代码。

通过这样的设计，RISC-V 架构最大限度地简化了 `ecall` 指令的功能，同时为操作系统开发者提供了更大的灵活性，以便他们可以根据需求优化系统调用的处理过程。

## 保存用户寄存器的方法

当我们进行trap处理时，第一步是保存所有的用户寄存器，以便在完成内核操作后能够正确地恢复用户程序的执行。RISC-V架构的一个关键点是，supervisor mode（内核模式）下的代码并不能直接访问物理内存，只能通过当前的page table来访问内存。这意味着我们需要一种有效的方法来保存这些寄存器内容，而又不能直接使用内存操作。

### 保存用户寄存器的挑战：

1. **无法直接访问物理内存**：在RISC-V架构中，supervisor mode下的代码不能直接访问物理内存，这样就不能简单地将寄存器内容直接写入物理地址中的某个位置。

2. **需要切换到Kernel Page Table**：理论上，我们可以将SATP寄存器指向内核的page table，这样我们就可以利用内核的映射关系来存储寄存器。然而，在trap的初始阶段，我们并不知道内核page table的地址。此外，切换SATP寄存器本身需要将地址加载到寄存器中，而在这个时候，寄存器已经被用户程序占用。

### 解决方案：

为了应对这些挑战，XV6采取了一个特别的方案：**每个用户进程都在其page table中映射了一个trapframe page**。这个page专门用于在trap时保存该进程的所有用户寄存器。这个trapframe page在用户的虚拟地址空间中有一个固定的虚拟地址（通常是`0x3ffffffe000`），这样在每次发生trap时，内核都可以方便地访问这个page并保存寄存器内容。

`trapframe`结构体定义在`proc.h`中，它包含了用户进程在陷入内核模式时需要保存的所有寄存器的槽位。具体来说，`trapframe`的结构体中预留了32个寄存器的位置，其中包括了a0-a7、s0-s11、t0-t6等寄存器。此外，`trapframe`结构体中还包含了一些额外的数据，例如保存了`kernel page table`的地址，这对于后续内核操作非常重要。

```c
// kernel/proc.h

...
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
};
```

### 保存寄存器的流程

当`ecall`指令触发系统调用陷入内核时，程序计数器会被更新到`STVEC`寄存器指向的地址，即 trampoline 页的起始地址。接着，`uservec`函数开始执行，它的首要任务就是将所有32个用户寄存器的内容保存到在用户page table中预先映射的 trapframe page 中对应的槽位。这样做的好处是，当内核需要切换回用户模式并恢复用户进程的执行时，可以轻松地将寄存器的值恢复到之前的状态。

在RISC-V架构上，虽然`ecall`指令本身并不会自动切换`page table`或保存寄存器，但通过巧妙的系统设计，如`trapframe`的使用，能够让系统在保持高度灵活性的同时，确保用户进程与内核代码之间的隔离与安全性。

## 利用SSCRATCH寄存器保存用户寄存器的策略

为了在进入内核模式时保存用户寄存器的内容，RISC-V架构提供了一个非常有用的寄存器：**SSCRATCH**。这个寄存器的存在就是为了在处理trap时能够灵活使用。内核会提前将某个关键的值——在本例中是`trapframe`页的地址——保存在`SSCRATCH`寄存器中。当陷入trap后，内核代码能够使用这个值并结合RISC-V的特殊指令，安全地交换寄存器的内容，从而达到保存和恢复用户寄存器的目的。

```c
...
# trap.c sets stvec to point here, so
# traps from user space start here,
# in supervisor mode, but with a
# user page table.
#
# sscratch points to where the process's p->trapframe is
# mapped into user space, at TRAPFRAME.
#

 # swap a0 and sscratch
# so that a0 is TRAPFRAME
csrrw a0, sscratch, a0

# save the user registers in TRAPFRAME
sd ra, 40(a0)
sd sp, 48(a0)
sd gp, 56(a0)
sd tp, 64(a0)
sd t0, 72(a0)
sd t1, 80(a0)
sd t2, 88(a0)
sd s0, 96(a0)
sd s1, 104(a0)
sd a1, 120(a0)
sd a2, 128(a0)
sd a3, 136(a0)
sd a4, 144(a0)
sd a5, 152(a0)
sd a6, 160(a0)
sd a7, 168(a0)
sd s2, 176(a0)
sd s3, 184(a0)
sd s4, 192(a0)
sd s5, 200(a0)
sd s6, 208(a0)
sd s7, 216(a0)
sd s8, 224(a0)
sd s9, 232(a0)
sd s10, 240(a0)
sd s11, 248(a0)
sd t3, 256(a0)
sd t4, 264(a0)
sd t5, 272(a0)
sd t6, 280(a0)
...
```

### `csrrw` 指令的作用

在`trampoline.S`代码中，程序执行的第一条指令是`csrrw`，它的作用是交换`a0`寄存器和`SSCRATCH`寄存器的内容。这个指令非常关键，因为它在不破坏现有寄存器值的情况下，把用户寄存器的内容（如`a0`寄存器的值）保存到`SSCRATCH`中，同时将`trapframe`页的地址从`SSCRATCH`加载到`a0`中。

我们可以通过实际的gdb调试，来观察这条指令的执行效果：

```bash
(gdb) print/x $a0
$1 = 0x3fffffe000
```

这表示`a0`寄存器现在包含了`trapframe`页的地址。这个地址原本保存在`SSCRATCH`寄存器中，现在通过`csrrw`指令，交换到了`a0`寄存器。我们再来看一下`SSCRATCH`寄存器的值：

```bash
(gdb) print/x $sscratch
$2 = 0x2
```

可以看到，`SSCRATCH`寄存器现在包含了之前`a0`中的值，即`write`系统调用的第一个参数（文件描述符2）。通过这个过程，`trapframe`页的地址安全地保存在了`a0`中，为接下来的寄存器保存操作做好了准备。

### `trapframe`页的具体操作

`trapframe`页在这个阶段发挥了重要的作用，它为保存32个用户寄存器提供了存储空间。在`trampoline.S`代码的后续部分，将执行一系列的`sd`（Store Doubleword）指令，这些指令会将各个寄存器的值存储到`trapframe`的不同偏移位置。

这些`sd`指令依次将用户寄存器的值保存到`trapframe`中的相应位置，这个过程虽然机械但至关重要。它确保在处理系统调用或中断后，能够正确恢复用户进程的执行状态。

### `SSCRATCH`寄存器的设置

在内核模式下，`a0` 寄存器会被用于各种不同的操作。在处理系统调用或中断时，内核需要使用多个寄存器，包括 `a0`，来完成各项工作。因此，当内核完成对 `trap` 的处理并准备返回用户态时，`a0` 寄存器的内容已经不再是原始的 `trapframe` 地址，而可能被其他操作修改过。

当内核即将从内核模式返回到用户模式时，内核会再次设置`SSCRATCH`寄存器，以便在下次陷入trap时，它能提供正确的`trapframe`页地址。这个设置过程通常发生在内核即将完成系统调用或中断处理，并准备切换回用户空间的时刻。例如，内核返回用户空间之前会调用如下代码：

```c
// trap.c
...
// tell trampoline.S the user page table to switch to.
uint64 satp = MAKE_SATP(p->pagetable);

// jump to trampoline.S at the top of memory, which 
// switches to the user page table, restores user registers,
// and switches to user mode with sret.
uint64 fn = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
...
```

> > ```c
> > // jump to trampoline.S at the top of memory, which 
> > // switches to the user page table, restores user registers,
> > // and switches to user mode with sret.
> > uint64 fn = TRAMPOLINE + (userret - trampoline);
> > ```
> >
> > - 这一行计算了一个地址 `fn`，它代表一个函数地址，函数位于内存的 `trampoline` 代码区域。
> > - `TRAMPOLINE` 是一个定义在内核中的常量，它指向内存顶部的一个固定位置，那里存放着 `trampoline.S` 文件中的代码。`trampoline.S` 是一个汇编代码文件，它负责在上下文切换时进行必要的设置。
> > - `userret` 和 `trampoline` 是两个符号，分别代表 `trampoline.S` 中不同函数的地址。`userret` 是一个函数，处理器将跳转到这个函数执行，这个函数负责从内核模式返回用户模式。
> > - 通过计算 `userret - trampoline`，代码得到了 `userret` 相对于 `trampoline` 的偏移量，将这个偏移量加到 `TRAMPOLINE` 的基地址上，得到完整的函数地址 `fn`。
> >
> > ```c
> > ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
> > ```
> >
> > - 这一行将前面计算的地址 `fn` 转换为一个函数指针，并调用它。
> > - `(void (*)(uint64,uint64))fn` 是一种类型转换，将 `fn` 解释为一个接受两个 `uint64` 类型参数并返回 `void` 的函数。
> > - `TRAPFRAME` 和 `satp` 是传递给这个函数的两个参数。
> >   - `TRAPFRAME` 是一个内核中的数据结构，它保存了用户程序在陷入内核时的寄存器状态。传递这个值是为了在返回用户模式时恢复用户寄存器。
> >   - `satp` 是我们之前设置的那个值，用于指示处理器切换到用户模式时应该使用哪个页表。
> > - 这个函数调用实际上会跳转到 `trampoline.S` 中的代码，这段代码负责完成以下任务：
> >   1. 切换到用户页表（通过设置 `satp` 寄存器）。
> >   2. 恢复用户程序的寄存器状态。
> >   3. 使用 `sret` 指令从内核模式切换到用户模式，继续执行用户程序。

在这个C函数中，内核将`trapframe`页地址作为参数传递给汇编代码`userret`，该函数将设置`a0`寄存器为`trapframe`地址。在`trampoline.S`中，`csrrw`指令再次交换`a0`和`SSCRATCH`的内容，将`trapframe`地址存入`SSCRATCH`，从而在下次发生trap时能够正确使用。

```c
# trampoline.S
...
ld t5, 272(a0)
ld t6, 280(a0)

	# restore user a0, and save TRAPFRAME in sscratch
csrrw a0, sscratch, a0

# return to user mode and user pc.
# usertrapret() set up sstatus and sepc.
sret
```

通过`SSCRATCH`寄存器与`csrrw`指令的巧妙结合，RISC-V和XV6实现了在进入内核模式时保存用户寄存器状态的功能。这种设计为内核提供了极大的灵活性，同时也确保了用户进程和内核之间的隔离和安全性。

> > 这里后续有改动。在最新版本的代码中，`uservec` 函数删除了 `csrrw a0, sscratch, a0` 这条指令，而是直接使用 `csrw sscratch, a0` 将用户态的 `a0` 寄存器的值存储到 `sscratch` 中。
> >
> > ```c
> > # trampoline.S
> > 
> > # save user a0 in sscratch so
> > # a0 can be used to get at TRAPFRAME.
> > csrw sscratch, a0
> > 
> > # each process has a separate p->trapframe memory area,
> > # but it's mapped to the same virtual address
> > # (TRAPFRAME) in every process's user page table.
> > li a0, TRAPFRAME
> > 
> > # save the user registers in TRAPFRAME
> > sd ra, 40(a0)
> > sd sp, 48(a0)
> > sd gp, 56(a0)
> > sd tp, 64(a0)
> > sd t0, 72(a0)
> > sd t1, 80(a0)
> > sd t2, 88(a0)
> > sd s0, 96(a0)
> > sd s1, 104(a0)
> > sd a1, 120(a0)
> > sd a2, 128(a0)
> > sd a3, 136(a0)
> > sd a4, 144(a0)
> > sd a5, 152(a0)
> > sd a6, 160(a0)
> > sd a7, 168(a0)
> > sd s2, 176(a0)
> > sd s3, 184(a0)
> > sd s4, 192(a0)
> > sd s5, 200(a0)
> > sd s6, 208(a0)
> > sd s7, 216(a0)
> > sd s8, 224(a0)
> > sd s9, 232(a0)
> > sd s10, 240(a0)
> > sd s11, 248(a0)
> > sd t3, 256(a0)
> > sd t4, 264(a0)
> > sd t5, 272(a0)
> > sd t6, 280(a0)
> > 
> > ```
> >
> > 在新的实现中，不再使用 `sscratch` 寄存器来存储 `TRAPFRAME` 的地址，而是直接在 `uservec` 函数中通过固定的虚拟地址读取 `TRAPFRAME`。具体来说，`TRAPFRAME` 的地址被硬编码在汇编代码中，这样在 `trap` 处理过程中，内核可以直接通过该地址访问 `trapframe` 页。
> >
> > ### 如何读取 `TRAPFRAME`
> >
> > 1. **固定地址读取**：
> >    在 `uservec` 函数中，通过以下指令获取 `TRAPFRAME` 页的地址：
> >
> >    ```nasm
> >    li a0, TRAPFRAME
> >    ```
> >
> >    这条指令将固定的虚拟地址 `TRAPFRAME` 加载到 `a0` 寄存器中。`TRAPFRAME` 是一个在每个进程的页表中都映射到**相同虚拟地址的页面**。这意味着，尽管每个进程的 `trapframe` 实际上位于不同的物理内存地址，但在虚拟地址空间中，它们都映射到相同的 `TRAPFRAME` 虚拟地址。
> >
> >
> > > ```
> > > // kernel/memlayout.h
> > > ...
> > > // map the trampoline page to the highest address,
> > > // in both user and kernel space.
> > > #define TRAMPOLINE (MAXVA - PGSIZE)
> > > 
> > > ...
> > > // User memory layout.
> > > // Address zero first:
> > > //   text
> > > //   original data and bss
> > > //   fixed-size stack
> > > //   expandable heap
> > > //   ...
> > > //   TRAPFRAME (p->trapframe, used by the trampoline)
> > > //   TRAMPOLINE (the same page as in the kernel)
> > > #define TRAPFRAME (TRAMPOLINE - PGSIZE)
> > > ```
> > >
> > > 
> >
> > 1. **通过 `a0` 寄存器访问**：
> >    内核代码使用 `a0` 寄存器中的 `TRAPFRAME` 地址，直接访问内存中与 `trapframe` 相关的数据。例如：
> >
> >    ```nasm
> >    sd ra, 40(a0)
> >    ```
> >
> >    这条指令将 `ra` 寄存器的值保存到 `trapframe` 中相应的偏移地址。这一切都是通过 `a0` 指向的固定 `TRAPFRAME` 虚拟地址来实现的。
> >
> > 2. **不再使用 `sscratch` 的原因**：
> >    通过直接使用固定的虚拟地址，简化了寄存器的使用，减少了不必要的寄存器切换。这种方法使得代码更加简单和直接，不再依赖 `sscratch` 寄存器来临时存储 `TRAPFRAME` 地址，而是直接通过访问固定的内存地址来完成 `trap` 的处理。
> >
> >    这种设计减少了复杂性，不再需要在内核和用户态之间传递 `sscratch` 寄存器的内容。代码变得更加简洁，容易维护。
> >
> > ### 回到用户态时：
> >
> > 1. 内核通过一系列的 `ld` 指令，从 `trapframe` 中恢复用户态下的所有寄存器值。
> > 2. 其中，`ld a0, 112(a0)` 恢复了用户态下的 `a0` 寄存器内容，这个值是在进入内核态时保存的。
> > 3. 完成所有寄存器的恢复后，使用 `sret` 指令从内核态返回到用户态，并恢复 `sepc` 寄存器中的程序计数器，继续执行用户态下的代码。
> > 4. `TRAPFRAME` 的地址不再需要通过 `sscratch` 寄存器来保存和恢复。
> >
> > ---
> >
> > ### `csrrw` 指令
> >
> > `csrrw` 是 "CSR Read and Write" 的缩写，表示对 CSR 寄存器进行读写操作。具体来说，它会将一个通用寄存器的值写入到指定的 CSR 寄存器中，同时将 CSR 寄存器的旧值存入另一个指定的通用寄存器。其语法为：
> >
> > ```
> > csrrw rd, csr, rs1
> > ```
> >
> > - `rd`：存放 CSR 寄存器旧值的通用寄存器。
> > - `csr`：要访问的 CSR 寄存器。
> > - `rs1`：提供新值的通用寄存器。
> >
> > **例子**
> >
> > ```
> > csrrw x5, sscratch, x10
> > ```
> >
> > - `x10` 的值会被写入到 `sscratch` CSR 寄存器。
> > - `sscratch` CSR 寄存器的旧值会被写入到 `x5` 中。
> >
> > ### `csrw` 指令
> >
> > `csrw` 是 "CSR Write" 的缩写，它只对 CSR 寄存器进行写操作，而不保留旧值。其语法为：
> >
> > ```
> > csrw csr, rs1
> > ```
> >
> > - `csr`：要写入的 CSR 寄存器。
> > - `rs1`：提供新值的通用寄存器。
> >
> > **例子**
> >
> > ```
> > csrw sscratch, x10
> > ```
> >
> > - 仅将 `x10` 的值写入到 `sscratch` CSR 寄存器，`sscratch` 的旧值不会被保留或存储。
> >
> > **区别**
> >
> > - **`csrrw`**：同时进行读和写操作，写入CSR新值的同时保存其旧值。
> > - **`csrw`**：仅进行写操作，将新值写入CSR，旧值被覆盖且不会保留。
> >
> > **使用场景**
> >
> > - `csrrw` 适用于需要在修改CSR寄存器时，还需要保留其旧值以供后续使用的场景。
> > - `csrw` 适用于只需简单更新CSR寄存器，而不关心其旧值的场景。



## 关于进程启动和`ecall`指令的执行

在操作系统的执行过程中，`ecall`指令起到了非常关键的作用，它将用户模式切换到内核模式，使得内核能够处理系统调用、异常或中断。在理解这一切之前，我们必须清楚进程在启动时的行为，以及如何通过`ecall`指令触发内核的介入。

### 进程启动与初始设置

在一台机器启动时，最先执行的代码总是在内核模式下运行的。当机器启动时，操作系统的内核会先初始化所有必要的硬件和软件环境，确保整个系统处于一个可控的状态下。这包括了设置各种控制寄存器、初始化内存、配置中断向量表等。

在一个新进程被启动时，内核会为这个进程分配必要的资源，如内存空间、页表、栈、文件描述符等。当这些初始化步骤完成后，内核会使用`sret`指令将控制权交给用户进程，从而开始在用户模式下执行。

### `ecall`指令与`trampoline`代码的执行

一旦进程在用户模式下运行，它可能在某个时间点需要通过`ecall`指令请求内核服务。执行`ecall`指令时，会发生以下几件事情：

1. **切换模式**：`ecall`指令首先会将CPU从用户模式切换到 supervisor mode。
2. **保存程序计数器**：`ecall`指令会将当前的程序计数器（PC）保存到`SEPC`寄存器中，以便在内核处理完请求后能够返回到用户程序的正确位置。
3. **设置程序计数器**：`ecall`指令还会将`STVEC`寄存器中的值加载到程序计数器中。这意味着，接下来的代码将从`STVEC`寄存器指向的地址开始执行。

在XV6操作系统中，`STVEC`寄存器被设置为指向`trampoline`页的起始地址。因此，当`ecall`指令执行时，程序计数器将跳转到`trampoline`页的开始，并开始执行其中的汇编代码。

当`ecall`指令触发时，内核需要保存所有用户寄存器的内容，以便在完成系统调用或中断处理后能够准确地恢复用户进程的状态。

### 为什么需要保存寄存器？

在内核处理系统调用或中断时，内核代码会使用到寄存器。如果不保存用户寄存器的状态，那么在内核执行完毕返回用户进程时，用户寄存器的内容可能已经被覆盖，导致用户程序无法继续正确执行。因此，内核在进入系统调用处理之前，必须保存所有用户寄存器的状态。

### 为什么寄存器保存到`trapframe`而不是用户栈？

选择将寄存器保存到`trapframe`而不是用户栈，有几个重要原因：

1. **用户栈的不确定性**：内核无法保证用户程序的栈总是存在且结构合理。某些编程语言可能不使用传统的栈结构，或者栈可能被分配在堆上。内核无法理解或预测这些用户栈的具体实现，因此无法依赖用户栈来保存关键寄存器。

2. **安全性与隔离**：内核需要在处理用户请求时保持对用户程序的完全控制。通过将寄存器保存到内核管理的`trapframe`，可以确保寄存器数据的安全和完整性，避免用户程序篡改寄存器数据，保证系统的稳定性和安全性。

3. **内核对用户内存的独立性**：`trapframe`页是内核专门为保存寄存器而设立的，内核可以完全控制这个页的内容和访问权限。这与用户内存空间独立，使得内核在处理系统调用时不依赖于用户内存的任何部分，避免了潜在的安全隐患。



## 从用户页表切换到内核页表

在当前程序执行的这一阶段，我们位于 `trampoline` 代码的起点，即 `uservec` 函数的开头。在这一阶段，程序将逐步从使用 `user page table` 切换到 `kernel page table`，这一切换对于操作系统至关重要，因为它允许内核访问整个内存和资源，而不再受到用户空间的限制。

```c
# trampoline.S
...
sd t5, 272(a0)
sd t6, 280(a0)

 # save the user a0 in p->trapframe->a0
csrr t0, sscratch
sd t0, 112(a0)

# restore kernel stack pointer from p->trapframe->kernel_sp
ld sp, 8(a0)

# make tp hold the current hartid, from p->trapframe->kernel_hartid
ld tp, 32(a0)

# load the address of usertrap(), p->trapframe->kernel_trap
ld t0, 16(a0)

# restore kernel page table from p->trapframe->kernel_satp
ld t1, 0(a0)
csrw satp, t1
sfence.vma zero, zero

# a0 is no longer valid, since the kernel page
# table does not specially map p->tf.

# jump to usertrap(), which does not return
jr t0
```

### 0. **保存 `a0` 寄存器的值到 `trapframe`**

```c
 # save the user a0 in p->trapframe->a0
csrr t0, sscratch
sd t0, 112(a0)
```

`csrr t0, sscratch` 指令将 `sscratch` 寄存器的内容读取到 `t0` 寄存器中。接着，`sd t0, 112(a0)` 将 `t0` 的值保存到 `trapframe` 中 `a0` 寄存器对应的槽位（偏移量为 `112`）中。这一步的目的是保存 `a0` 的值。

> > 由于在进入 `trap` 处理时，`a0` 需要暂时被用来存储 `TRAPFRAME` 地址，而原本 `a0` 中的值（例如系统调用参数）不能丢失，所以系统将 `a0` 的值暂时保存到 `SSCRATCH` 中。
> >
> > 一旦 `TRAPFRAME` 地址被加载并使用完毕后，`a0` 的值就可以被安全地保存到 `trapframe` 中的适当位置。这样做的目的是确保系统在 `trap` 处理过程结束后，能够准确地恢复 `a0` 的原始值，同时也能使用 `TRAPFRAME` 地址完成必要的操作。

### 1. 初始化内核栈指针

```c
ld sp, 8(a0)
```

代码通过 `ld` 指令将 `a0` 寄存器中存储的 `trapframe` 页地址的第8个字节加载到 `Stack Pointer`（`sp`）寄存器中。这一操作的目的是将 `Stack Pointer` 设置为内核栈的顶端。内核栈在内核进入用户空间之前已经为当前进程专门分配并设置好。

```c
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
```

通过设置 `Stack Pointer`，内核现在可以安全地在栈上进行操作，而不会干扰用户空间中的栈数据。

### 2. 设置 Hart ID（核 ID）

接下来，代码将当前 CPU 核的编号（`hartid`）存储在 `tp` 寄存器中。在 RISC-V 中，没有直接方法可以查询当前运行在哪个 CPU 核上，因此 `tp` 寄存器用于手动保存核 ID。这有助于内核识别当前进程正在使用哪个 CPU 核，多核处理器系统中尤其重要。

```c
ld tp, 32(a0)
```

在 QEMU 模拟中，我们看到只有一个 CPU 核在运行，因此 `tp` 的值为 0，这意味着程序运行在第一个（也是唯一的）CPU 核上。

### 3. 准备进入 `usertrap` 函数

接下来的操作是将 `usertrap` 函数的地址加载到 `t0` 寄存器中。`usertrap` 是一个 C 函数，负责处理来自用户空间的各种 trap 事件（如系统调用或异常）。这一步为稍后跳转到内核代码做好了准备。

```c
ld t0, 16(a0)
```

### 4. 切换到内核页表

最后一条指令是将 `kernel page table` 的地址加载到 `t1` 寄存器中，并使用 `csrw` 指令将其写入 `SATP` 寄存器。`SATP` 寄存器负责管理当前使用的页表。当 `SATP` 的值发生变化时，系统便切换到了新的页表，在这里即为 `kernel page table`。

```c
ld t1, 0(a0)
sfence.vma zero, zero
csrw satp, t1
sfence.vma zero, zero
```

切换完成后，内核现在可以访问更广泛的内存范围，包括所有的内核数据结构、内核代码以及其他系统资源。

> > ```
> > sfence.vma zero, zero
> > ```
> >
> > **`sfence.vma` 指令**：`sfence.vma` 全称是 "Supervisor Fence for Virtual Memory Addressing"。它是 RISC-V 中的特权指令，用来协调软件对页表的修改与硬件对这些页表的使用。特别是在操作系统中切换页表时，使用 `sfence.vma` 可以确保在页表更新后，硬件不再使用旧的或错误的映射。
> >
> > **`zero, zero` 参数**：`sfence.vma` 指令可以接受两个参数，第一个参数指定要刷新的虚拟地址，第二个参数指定要刷新的 ASID（地址空间标识符，Address Space Identifier）。然而，常用的 `sfence.vma zero, zero` 表示刷新整个 TLB，而不仅仅是某个特定的虚拟地址或 ASID。因此，这个指令表示**刷新所有的虚拟地址映射**，使得所有的 TLB 缓存失效。
> >
> > 在修改页表（如 `csrw satp, t1` 更改了当前使用的页表）后，执行 `sfence.vma zero, zero` 是为了确保接下来所有的内存访问都使用最新的地址转换规则。如果不执行这条指令，系统可能会继续使用缓存中的旧地址转换结果，从而导致内存访问错误。
> >
> >  `sfence.vma` 指令的另一个重要作用，即作为一种**内存屏障（memory barrier）**，确保内存操作按预期顺序执行，防止内存访问的乱序执行。
> >
> > 在现代处理器中，为了提高性能，处理器通常会进行**指令乱序执行**（out-of-order execution）。这意味着处理器可能会根据情况调整指令执行的顺序，以充分利用可用的资源，例如执行单元和缓存。这种优化通常是透明的，不影响程序的正确性。然而，当涉及到内存访问、页表切换等操作时，乱序执行可能会导致问题。
> >
> > 在操作系统中，当我们切换页表或进行其他与内存管理相关的操作时，我们需要确保所有先前的内存操作已经完成，并且不会与接下来的内存访问产生冲突。否则，可能会出现旧的内存访问与新的页表设置不一致的情况，导致内存访问错误。
> >
> > `sfence.vma` 指令在此起到了两方面的作用：
> >
> > 1. **完成前面的内存操作**：执行 `sfence.vma` 指令之前，处理器必须确保之前的所有内存操作（如读写操作）已经完成。这意味着，在页表切换之前，所有的内存读写操作都必须按照旧的页表设置完成。
> > 2. **防止指令乱序**：`sfence.vma` 作为一种内存屏障，确保前面的指令（如页表切换之前的内存操作）在执行完毕后，再执行后续指令（如 `csrw satp` 之后的内存访问）。这样可以防止由于乱序执行而导致的内存访问错误。

### 5. 验证页表切换

在页表切换完成后，您可以使用 QEMU 提供的 `info mem` 命令来查看当前的页表内容。此时，页表内容应该与之前的用户页表完全不同，包含了更多的内核地址映射。这验证了页表已成功切换到内核页表。

![image-20240819141641698]({{ site.baseurl }}/docs/assets/image-20240819141641698.png)

## 跳转到 `usertrap` 函数并执行

在完成页表切换、栈指针初始化后，程序即将从 `trampoline` 代码跳入内核的 C 代码，这一过程对于操作系统的正常运行至关重要。

### trampoline page 的特殊作用

在用户空间和内核空间中，trampoline page 的地址和内容在页表中是完全一致的。这确保了在切换 `page table` 期间，程序计数器（PC）指向的地址保持有效，避免内存访问错误。

- **trampoline page 的双重映射**：在 `user page table` 和 `kernel page table` 中，trampoline page 的地址一致，无论是用户模式还是超级模式，程序在该页面上执行的代码都相同。
- **防止崩溃的设计**：由于虚拟地址在不同 `page table` 中指向的物理地址相同，切换 `page table` 后，程序能够继续执行，而不会因为地址空间变化导致崩溃。

### 跳转到 `usertrap` 函数

在设置好内核栈和切换到内核 `page table` 后，程序将跳转到内核的 `usertrap` 函数。这个跳转通过 `jr t0` 指令完成，`t0` 寄存器中保存的是 `usertrap` 函数的地址。

- **`jr t0` 指令**：这条指令负责跳转到 `t0` 寄存器中存储的地址，将控制权交给内核中的 `usertrap` 函数。
- **`usertrap` 的作用**：`usertrap` 是处理从用户空间进入内核空间的所有 trap 事件的核心函数。它负责处理系统调用、异常（如非法内存访问）和中断。当 `usertrap` 函数被调用时，系统已完全切换到内核模式，可以访问所有内存和硬件资源。

### 进入 `usertrap` 函数后的执行流程

一旦跳转到 `usertrap` 函数，内核将开始处理具体的 trap 事件，这通常涉及以下几个步骤：

1. **识别 trap 类型**：内核检查 trap 的类型，确定这是一次系统调用、异常还是中断。
2. **处理 trap**：根据 trap 的类型，内核执行相应的处理逻辑，例如对系统调用，内核会找到相应的处理函数并传递参数；对异常，内核可能会终止进程并生成错误信息；对中断，内核可能会与硬件设备交互。
3. **准备返回用户空间**：一旦 trap 处理完成，内核将准备返回用户空间，包括恢复用户寄存器、设置程序计数器（PC）指向下一条用户指令，并最终调用 `sret` 指令返回到用户模式。

通过 `trampoline page` 的设计，XV6 实现了从用户空间到内核空间的安全过渡，并通过 `usertrap` 函数处理各种用户空间发起的 trap 事件。这个过程中的每一步都至关重要，确保了操作系统能够正确管理用户进程的执行，并在需要时进行适当的干预。

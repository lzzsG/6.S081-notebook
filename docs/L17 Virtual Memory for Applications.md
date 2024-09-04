---
layout: page
title: L17 Virtual Memory for Applications
permalink: /L17
description: "Lecture 17 - Virtual Memory for Applications，"
nav_order: 17



---

# Lecture 17 - Virtual Memory for Applications

## 应用程序使用的虚拟内存

本节内容主要是讨论用户应用程序使用虚拟内存所需的特性，并以1991年的一篇[论文](https://pdos.csail.mit.edu/6.828/2020/readings/appel-li.pdf)为背景，讨论了这些特性对应用程序的重要性。

### 1. **用户应用程序对虚拟内存特性的需求**

操作系统内核通过页表（Page Table）管理虚拟内存，并使用多种机制（如页面置换、懒分配、写时复制等）灵活地处理内存操作。论文的核心观点是，用户应用程序不应只被动地使用虚拟内存，还应主动利用虚拟内存的特性来提升效率。具体来说，用户应用程序可以像操作系统内核一样，通过生成和响应 **Page Fault（页面错误）** 来实现一些复杂的内存操作和优化。

例如，用户空间应用程序应该能够像内核一样，通过修改页表条目（Page Table Entry, PTE）的保护位和权限级别，来控制内存页面的访问权限。这种灵活的控制可以极大地提高某些类型应用的性能和灵活性。

论文列举了几类可以从虚拟内存特性中获益的应用程序，说明了用户应用程序使用虚拟内存的必要性。以下是一些示例应用程序：

**Garbage Collector（垃圾回收器）**：

垃圾回收器可以通过虚拟内存机制更高效地管理内存。例如，它可以利用Page Fault机制来跟踪哪些内存页面被访问，确定哪些页面需要回收或保持。此外，虚拟内存允许垃圾回收器监控特定页面的修改，判断是否需要复制或重新分配内存。

**Data Compression Application（数据压缩应用）**：

数据压缩应用程序通常需要高效地管理大块内存区域。通过使用虚拟内存，应用程序可以更精细地控制内存页面的读写权限，避免不必要的内存拷贝。此外，通过在用户空间处理Page Fault，可以实现更高效的压缩和解压缩操作。

**Shared Virtual Memory（共享虚拟内存）**：

在分布式系统中，多个节点之间共享内存区域是非常常见的需求。虚拟内存可以让多个进程共享相同的物理内存，同时通过控制不同的虚拟内存映射来实现对共享内存的保护。例如，一个进程可以以只读的方式访问某个内存区域，而另一个进程则可以写入同一个区域。

为了使用户应用程序能够有效地利用虚拟内存，这些应用程序需要操作系统提供以下特性：

- **Page Fault Trap**：当发生Page Fault时，操作系统将控制权交给用户空间的处理程序，这样用户程序就可以通过Page Fault事件动态调整内存的访问策略。
- **修改PTE的保护位（Protection Bit）**：用户程序需要修改内存页面的访问权限（如可读、可写、不可访问），以实现精细化的内存管理。
- **权限级别控制（Privileged Level）**：用户程序可以在不同权限级别之间切换，允许或限制对特定内存区域的访问，以保护敏感数据或控制共享内存的使用。

### 2. **虚拟内存特性详解**

1. **Trap机制**：
   - **功能**：当用户空间的应用程序访问受限内存时，操作系统会触发一个页错误（Page Fault），并将控制权转移到用户空间的处理程序（handler），而不是直接在内核中处理。这种机制允许用户程序在发生Page Fault时自定义响应行为。
   - **重要性**：这是用户空间应用程序能够利用虚拟内存特性的基础。没有这个机制，应用程序就无法通过Page Fault实现自定义的内存管理策略。

2. **Prot1**（保护单个内存页面）：
   - **功能**：降低单个内存页面的访问权限，例如将一个可读写的页面变为只读，或者将只读页面变为不可访问。
   - **用途**：Prot1可以用于防止对关键数据的未授权写入，也可以用于实现写时复制（Copy-on-Write）等内存优化策略。

3. **ProtN**（保护多个内存页面）：
   - **功能**：一次性降低多个内存页面的访问权限，相当于对N个页面执行Prot1的操作。
   - **优势**：ProtN通过一次性修改多个页面的访问权限并只刷新一次TLB，提高了性能，特别是在需要同时保护多个页面的情况下。

4. **Unprot**（解除保护）：
   - **功能**：增加页面的访问权限，例如将一个只读页面恢复为可读写。
   - **用途**：Unprot可以用于在应用程序检测到需要写入时动态提升页面的权限。

5. **Dirty Bit**（脏页标记）：
   - **功能**：检测一个页面是否被修改过。操作系统可以通过这个位标识哪些页面需要写回磁盘或者保存到其他存储中。
   - **用途**：脏页标记在内存管理和垃圾回收中非常有用，帮助系统确定哪些页面需要被刷新或者写回，以避免数据丢失。

6. **Map2**（双重映射）：
   - **功能**：允许同一物理内存地址映射到两个不同的虚拟地址空间中，且这两个虚拟地址可以有不同的访问权限。
   - **用途**：Map2可以用于实现共享内存和多进程通信，允许不同的进程或线程以不同的方式访问同一段内存。

### 3. **XV6与现代操作系统的支持情况**

- **XV6的局限性**：在XV6操作系统中，虽然内核具备许多虚拟内存机制，但这些机制并未通过系统调用暴露给用户空间。XV6的设计相对简单，没有实现像Prot1、ProtN、Unprot、Dirty Bit和Map2这些高级特性。
- **现代操作系统（如Linux）**：相比之下，现代的Unix系统如Linux已经实现了许多这类特性，尽管可能形式不同。例如，Linux支持Page Fault处理、内存映射控制（如`mprotect`系统调用），并且可以通过`mmap`实现共享内存。

这些虚拟内存特性对于现代应用程序的高效运行至关重要，并且在现代操作系统中已经得到了广泛支持。通过灵活使用这些特性，应用程序可以实现更高级的内存管理策略，提升性能和可靠性。在接下来将继续探讨如何实现这些特性，以及它们在实际应用中的作用。

在用户应用程序使用虚拟内存的过程中，有几个非常重要的系统调用，它们为应用程序提供了强大的内存映射和权限管理功能。

## 1. **`mmap` 系统调用**

`mmap` （**Memory Map**（内存映射））是操作系统提供的一种将文件或其他对象映射到内存地址空间的机制。它允许应用程序通过内存地址直接访问文件内容，避免频繁使用传统的 `read`/`write` 系统调用。`mmap` 有许多用途，包括文件映射、内存分配等。让我们详细了解其参数和工作方式：

### `mmap` 参数解析

- **第一个参数：`addr`**
  - 这是用户想要映射到的特定虚拟内存地址。如果传入 `NULL`，系统将自动选择合适的地址进行映射。

- **第二个参数：`len`**
  - 这是映射的长度。用户需要指定映射对象（文件、内存）的大小。

- **第三个参数：`prot`**
  - 用于设置内存区域的访问权限，例如可读（`PROT_READ`）、可写（`PROT_WRITE`）等。

- **第四个参数：`flags`**
  - 这个参数控制映射的行为。常见值之一是 `MAP_PRIVATE`，表示对映射区域的修改不会影响底层文件，只会在内存中反映修改。

- **第五个参数：`fd`**
  - 这是被映射的对象的文件描述符。常用于映射文件内容。

- **第六个参数：`offset`**
  - 指定从文件的哪个偏移量开始映射。可以从文件的某个位置开始映射一段区域。

### `mmap` 的功能和用途

1. **内存映射文件（Memory Mapped File）**：通过将文件内容映射到虚拟内存空间，应用程序可以使用指针直接访问文件内容，避免反复调用 `read`/`write` 系统调用。

2. **匿名内存映射（Anonymous Memory）**：`mmap` 也可以用于申请物理内存，而不涉及文件。这可以用来替代 `sbrk` 系统调用，通过匿名内存映射为应用程序分配内存。

3. **共享内存**：不同进程可以通过 `mmap` 映射相同的文件，来共享内存区域，实现高效的跨进程通信。

`mmap` 是虚拟内存管理中一个强大且灵活的工具，它提供了应用程序与文件或物理内存之间的桥梁，使应用程序可以灵活管理内存。

> #### **`sbrk` 系统调用**
>
> `brk` 和 `sbrk` 是早期用于管理进程堆内存的两个系统调用，堆内存是进程内存空间中动态增长的区域，常用于动态内存分配。
>
> - **`sbrk` 全称**：**Set Break**（设置断点）
>
>   - `sbrk` 通过移动堆的“断点”来分配或释放进程的堆空间。
>
> - **功能**：
>
>   - `sbrk` 通过增加或减少进程的堆空间来管理内存。
>   - 它接受一个参数 `increment`，表示要增长或减少的堆大小。正值会增加堆空间，负值会减少堆空间。
>
> - **实现**：
>
>   - 进程的堆起始于数据段的结束位置，称为“断点”（break）。通过调用 `sbrk`，程序可以动态地增长或缩小堆空间。增大时，新的内存从操作系统申请，缩小时，堆空间被释放。
>
> - **限制**：
>
>   - `sbrk` 的局限性在于它只能调整堆的大小，无法对特定地址范围进行精细控制，也无法灵活设置页面的权限。这促使开发者转向使用更灵活的内存管理方式，如 `mmap`。
>
> - **示例**：
>
>   ```c
>   void* new_break = sbrk(4096);  // 增加4KB的堆空间
>   ```
>
> #### **匿名内存区域**
>
> 匿名内存区域是通过 `mmap` 系统调用创建的一种内存映射，和 `sbrk` 类似，它用于为应用程序分配内存。但与 `sbrk` 不同，`mmap` 提供了更灵活的内存管理方式。
>
> - **定义**：
>
>   - 匿名内存区域是指通过 `mmap` 创建的没有与具体文件或设备关联的内存区域。它不依赖于文件描述符，仅用于在内存中分配匿名页。
>
> - **使用 `mmap` 分配匿名内存**：
>
>   - 在调用 `mmap` 时，通过指定 `MAP_ANONYMOUS` 标志，程序可以向操作系统申请一块匿名内存区域。这些内存不与任何文件关联，通常用于分配临时或动态内存。
>
> - **区别于 `sbrk`**：
>
>   - 与 `sbrk` 不同，匿名内存区域可以分配到任何虚拟内存地址，而不是局限于堆的增长。
>   - `mmap` 允许用户设置具体的页面权限（如只读、可写等），并且可以按页精确控制。
>
> - **匿名内存的用途**：
>
>   - 动态分配内存：与 `malloc` 类似，`mmap` 可以用于动态分配内存。
>   - 大型数据结构：可以用于映射大型数据结构，特别是在需要精细控制内存的场合。
>
> - **示例**：
>
>   ```c
>   void* addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
>   ```
>
>   在此例中，`mmap` 分配了一个大小为 4KB 的匿名内存区域，且该区域为可读写的。

## 2. **`mprotect` 系统调用**

`mprotect` （**Memory Protection**（内存保护））用于修改已经映射到虚拟内存地址空间中的页面的权限。通过 `mprotect`，应用程序可以控制哪些内存页面是可读、可写、可执行，或者完全不可访问。这对于内存保护、调试、优化等场景非常有用。

### `mprotect` 参数解析

- **第一个参数：`addr`**
  - 指定要更改权限的虚拟内存地址起始位置。

- **第二个参数：`len`**
  - 需要修改权限的内存区域的长度。

- **第三个参数：`prot`**
  - 新的权限，可能的值包括：
    - `PROT_READ`：页面可读
    - `PROT_WRITE`：页面可写
    - `PROT_EXEC`：页面可执行
    - `PROT_NONE`：页面不可访问

### `mprotect` 的功能和用途

1. **控制页面的访问权限**：`mprotect` 可以改变已经映射的内存页面的访问权限。比如，可以将一段内存设置为只读，从而防止误写操作。

2. **实现写时复制（Copy-on-Write）**：结合 `mmap` 和 `mprotect`，可以实现写时复制。初始时，页面被设置为只读，当用户试图写入时，会触发页面错误（Page Fault），系统可以动态为该用户分配新的内存空间。

3. **内存保护**：通过限制某些关键数据的访问权限，可以防止内存泄露或者未授权的写操作。

## 3. **`munmap` 系统调用**

`munmap` （**Memory Unmap**（解除内存映射））是 `mmap` 的反操作，主要用于解除某段虚拟内存的映射关系。它可以释放已经映射的内存资源，让系统回收这些资源。

### `munmap` 参数解析

- **第一个参数：`addr`**
  - 要解除映射的内存区域的起始地址。

- **第二个参数：`len`**
  - 要解除映射的内存区域的长度。

通过 `munmap`，应用程序可以释放不再需要的内存映射区域，避免资源浪费。

## 4. **`sigaction` 系统调用**

`sigaction` （**Signal Action**（信号操作））是一个用于处理信号（signals）的系统调用。信号是操作系统向进程发送的一种通知机制，通常用于告知进程发生了某些异常或事件。例如，当程序试图访问非法内存地址时，会产生一个 **segfault** 信号。

- **功能**：`sigaction` 允许应用程序为特定的信号（如 `SIGSEGV`，**Segmentation Fault**（段错误））指定一个信号处理程序（handler）。当操作系统检测到这个信号时，会调用应用程序注册的handler来处理该信号，而不是直接终止进程。

- **与Page Fault的关系**：
  - **Page Fault** 是一种特定类型的异常，通常由程序访问受保护的内存页面引发。发生Page Fault时，操作系统会生成一个信号，如 `SIGSEGV`。
  - 默认情况下，收到 `SIGSEGV` 信号的程序会崩溃并终止。但是通过 `sigaction`，程序可以为 `SIGSEGV` 信号注册一个handler，这样当发生segfault时，程序可以通过handler来处理这个信号，而不必崩溃。

- **使用场景**：
  - 应用程序可以使用 `sigaction` 在发生 Page Fault 时动态修复内存问题。例如，可以在handler中调用 `mprotect` 修改内存页面的权限，然后恢复程序的正常执行。
  - 通过这种方式，应用程序能够灵活处理复杂的内存管理任务，例如写时复制（Copy-on-Write）或内存的动态分配。

### **与`sigalarm`的比较**

`sigaction` 具有通用性，适用于处理多种类型的信号。而 `sigalarm` （**Signal Alarm**（信号定时器））是一个专门用于定时触发信号的系统调用，可以设置定期触发的信号。然而，`sigaction` 也可以实现相同的功能，只是使用更广泛。

### **结合`mprotect`进行权限管理**

在虚拟内存管理中，`mprotect` 可以修改某些内存区域的访问权限。结合 `sigaction` 处理 Page Fault 的机制，应用程序可以动态响应内存访问异常，调整页面权限。

例如：

- 将某段内存设置为只读，触发写操作时引发 Page Fault。
- 使用 `sigaction` 捕捉 Page Fault，在handler中调用 `mprotect` 修改内存权限（如改为可写），然后继续执行程序。

## 对应的虚拟内存特性和Unix接口

将虚拟内存的特性与对应的 Unix 接口进行对应和总结：

- **Trap** 对应 `sigaction`：当发生 Page Fault 时，信号处理机制通过 `sigaction` 实现。程序可以注册一个信号处理程序，在发生Page Fault时进行处理，类似于操作系统内核对Page Fault的响应方式。

- **Prot1、ProtN 和 Unprot** 对应 `mprotect`：
  - `mprotect` 可以灵活地修改单个或多个页面的访问权限，这相当于实现了论文中提到的 Prot1（保护单个页面）和 ProtN（保护多个页面）的功能。
  - 当 `mprotect` 修改多个页面时，能够减少TLB的刷新次数，从而提高性能。

- **查看页面的Dirty位**：
  - 操作系统没有直接暴露一个用于检测页面是否被修改（Dirty）的系统调用。但通过某些技巧，程序可以间接实现类似功能。例如，通过映射内存为只读并捕捉写操作，可以判断页面是否被修改。

- **map2**：
  - 没有直接的系统调用支持 `map2`（将同一个物理页面映射为两个不同的虚拟页面，且具有不同的访问权限），但通过多次调用 `mmap`，程序可以实现类似效果。即，将同一文件或匿名内存区域分别映射到不同的虚拟地址，并设置不同的权限。

现代操作系统已经实现了这些功能。Linux等操作系统通过系统调用（如 `mmap`、`mprotect` 和 `sigaction`）为应用程序提供了灵活的虚拟内存管理能力。尽管这些特性可能并不是完全按照论文中的描述实现的，但它们为应用程序提供了强大的控制内存使用和管理的能力。

接下来讨论这些系统调用在操作系统中的底层实现，并探讨具体应用场景中的使用方式。



## 虚拟内存系统如何支持用户应用程序

虚拟内存系统为用户应用程序提供了内存保护、地址翻译等基础功能，通过这些功能，操作系统能够有效管理内存资源并确保进程隔离。然而，为了支持用户应用程序灵活使用虚拟内存（如通过`mmap`或响应Page Fault），操作系统还需要在硬件页表之外引入一些软件层面的结构和机制。

### **虚拟内存区域（VMAs）**

**Virtual Memory Areas (VMAs)** 是现代操作系统中的一项重要概念，用于管理进程的虚拟地址空间。虽然地址翻译主要由硬件中的页表（Page Table）完成，但VMA是一个软件层面的数据结构，记录了每个虚拟内存段的信息。

#### **VMA的作用**

- **记录地址段信息**：每个VMA对象对应一段连续的虚拟内存地址空间。在一个进程中，多个VMA可能对应不同的虚拟地址段，如代码段、数据段、栈段等。
- **管理权限**：每个VMA记录该段内存的权限（如可读、可写、可执行），这些权限最终会反映在页表项（PTE）中。
- **文件映射信息**：如果进程通过`mmap`将文件映射到内存中，VMA对象会记录文件的相关信息，如文件描述符、偏移量等。

#### **VMA的使用场景**

- **Memory Mapped File（内存映射文件）**：当一个进程使用`mmap`将文件映射到内存时，操作系统会为该文件在虚拟地址空间中分配一个VMA对象。VMA记录该段内存的映射信息，例如文件的权限、偏移量等。
- **匿名内存映射**：VMA也可以用于匿名内存区域的管理，这些区域不与文件关联，常用于动态分配内存。

### **用户级别trap的实现**

现代操作系统还允许用户级别的程序捕获Page Fault等异常，并通过注册的handler处理这些异常。这种机制称为**用户级trap**，它使得用户程序可以灵活处理内存问题，例如页面缺失或权限不正确的访问。

#### **Page Fault触发过程**

1. **发生Page Fault**：当进程试图访问的内存页面未被映射或权限不匹配时，硬件会触发Page Fault，CPU跳转到内核执行相关的trap处理代码。
2. **进入内核处理**：内核会保存进程状态，查询虚拟内存系统，并检查是否设置了对应的handler。
3. **通过VMA处理Page Fault**：内核可以根据VMA信息来决定如何处理Page Fault。如果该地址映射了文件，内核可以根据文件描述符和偏移量重新加载页面到内存中。
4. **传递到用户空间的handler**：如果应用程序为Page Fault注册了handler（例如通过`sigaction`），则内核会通过**upcall**将控制权传递到用户空间，执行handler中的逻辑。
5. **处理结束，恢复进程执行**：handler执行完成后，控制权返回内核，内核根据处理结果恢复进程的执行。

#### **安全性问题**

学生提出了一个关于安全性的关键问题：用户级别的trap处理会不会引入安全漏洞？教授的回答是：

- **隔离性仍然有效**：即使应用程序在用户空间设置了handler来处理Page Fault，它只能影响自己的地址空间，不能访问或影响其他进程的内存或Page Table。因此，内核与用户空间的隔离仍然有效。
- **自我限制**：应用程序只能修改自己的内存状态，无法破坏其他进程的安全性。如果handler行为异常，最终内核仍会杀掉该进程。

### **实现用户应用程序虚拟内存的挑战**

现代虚拟内存系统在支持用户应用程序灵活使用虚拟内存时，涉及到多方面的复杂实现：

- **VMA的管理**：为了支持`mmap`等系统调用，操作系统需要管理多个VMA对象，这些对象保存着虚拟内存段的信息。
- **用户级trap处理**：允许用户应用程序处理异常，如Page Fault，提供了灵活的内存管理机制，但同时要求内核能够安全地将异常传递给用户空间并确保进程隔离。



## 利用虚拟内存特性构建高效缓存表

接下来通过一个简单的应用程序示例看一下如何利用虚拟内存的特性来构建一个高效的缓存表。这个示例演示了虚拟内存如何在处理大数据结构（如大型表单）时提供帮助，特别是在内存资源有限的情况下。

### 缓存表应用的基本概念

**缓存表（Cache Table）** 是一种用于存储计算结果的表单，以便在需要时可以快速查找结果，而无需重新计算。这对于那些计算复杂且需要多次重复执行的函数特别有用。

- **表单的结构**：假设有一个函数 `f(i)`，其计算非常耗时。通过创建一个缓存表，可以将 `f(i)` 的结果预先计算并存储在表单中。表单的索引 `i` 对应 `f(i)` 的结果，这样当再次需要 `f(i)` 的时候，只需直接查找表单的 `i` 槽位即可。
- **查找效率**：通过这种方式，可以将原本费时的计算转换为快速的表单查找，从而大幅提高程序的执行效率。

## 虚拟内存在缓存表中的应用

**挑战**：表单可能非常大，甚至超过了物理内存的容量。在这种情况下，传统的分配内存和保存计算结果的方法可能无法奏效，因为它会消耗过多的物理内存。

**解决方案**：通过利用虚拟内存的特性，可以有效管理这类大型数据结构：

### 1. **分配大块虚拟地址空间**

- **虚拟地址段**：首先，程序会分配一个大的虚拟地址段，用于存储缓存表。此时，并没有实际分配物理内存，只是预留了虚拟地址空间。

### 2. **延迟分配物理内存**

- **按需分配内存（Lazy Allocation）**：当程序首次访问缓存表中的某个槽位时（如访问 `tb[i]`），由于该地址没有实际分配物理内存，会触发一个 **Page Fault**。
- **Page Fault 处理**：在 Page Fault 处理程序中，操作系统会为触发访问的虚拟地址段分配实际的物理内存，并计算 `f(i)` 的结果，将其存储到表单中。

### 3. **优化内存使用**

- **重复访问的优化**：当同一页面的其他槽位（如 `tb[i+1]`）被访问时，由于这段虚拟地址已经映射了物理内存，访问不会再次触发 Page Fault，从而避免了重复的物理内存分配和计算。
- **内存回收机制**：如果整个缓存表的大小超出了物理内存容量，Page Fault 处理程序还需要能够回收一些不再使用的物理内存页面。通过修改这些页面的页表项（PTE），降低它们的可访问性（使用 `Prot1` 或 `ProtN`），使得当这些页面再次被访问时，会再次触发 Page Fault，从而动态管理内存。

> 在分配物理内存页面时，操作系统会将这些页面映射到特定的虚拟地址空间中，还是随机的地址？

**地址分配的灵活性**：操作系统负责将物理内存页面映射到虚拟地址空间中。操作系统会选择合适的虚拟地址进行映射，并通知程序使用哪个地址。这意味着在内存分配时，地址空间的具体选择是由操作系统决定的，而不是程序本身。

这个示例展示了如何利用虚拟内存的特性来高效管理大型数据结构，特别是在内存资源有限的情况下。通过按需分配物理内存和动态回收未使用的内存页面，可以构建出既高效又灵活的内存管理机制。这种方法不仅提升了程序的执行效率，还减少了物理内存的消耗，使得处理大型数据结构变得更加可行。

这种利用虚拟内存的方式也为更复杂的应用（如垃圾回收器）提供了基础，教授在后续讲解中将进一步探讨这些更复杂的应用场景。

## 利用虚拟内存构建高效缓存表的实现

接下来通过一个小的实现讨论如何利用Unix的虚拟内存机制（包括 `mmap` 和 `sigaction` 等系统调用）来实现一个延迟分配内存的缓存表。通过该方法，即使缓存表很大，系统也只会在需要时分配物理内存，有效管理内存资源。

### 1. `main` 函数

```c
int main(int argc, char *argv[])
{
    page_size = sysconf(_SC_PAGESIZE);  // 获取系统页面大小
    printf("page_size is %ld\n", page_size);  // 输出页面大小
    setup_sqrt_region();  // 设置平方根表的虚拟内存区域
    test_sqrt_region();  // 测试平方根表的正确性
    return 0;
}
```

- `main` 函数的主要任务是获取系统页面大小，设置平方根表的虚拟内存区域，然后测试该区域是否正确工作。

### 2. `test_sqrt_region` 函数

```c
static void test_sqrt_region(void)
{
    int i, pos = rand() % (MAX_SQRTS - 1);
    double correct_sqrt;

    printf("Validating square root table contents...\n");
    srand(0xDEADBEEF);  // 设置随机数种子以保证结果的可重复性

    for (i = 0; i < 500000; i++) {
        if (i % 2 == 0)
            pos = rand() % (MAX_SQRTS - 1);  // 随机选择一个位置
        else
            pos += 1;  // 线性增长位置

        calculate_sqrts(&correct_sqrt, pos, 1);  // 计算pos位置的平方根
        if (sqrts[pos] != correct_sqrt) {  // 检查计算结果是否正确
            fprintf(stderr, "Square root is incorrect. Expected %f, got %f.\n",
                    correct_sqrt, sqrts[pos]);  // 如果不正确则输出错误并退出
            exit(EXIT_FAILURE);
        }
    }

    printf("All tests passed!\n");  // 所有测试通过
}
```

- `test_sqrt_region` 函数随机遍历表单，并检查每个位置的平方根值是否正确。
- 在实际运行过程中，若访问未分配物理页面的位置，会触发Page Fault，导致内核调用相应的信号处理程序。

### 3. `setup_sqrt_region` 函数

```c
static void setup_sqrt_region(void)
{
    struct sigaction act;

    // 仅仅映射内存以找到表单的安全存放位置
    sqrts = mmap(NULL, MAX_SQRTS * sizeof(double) + AS_LIMIT, PROT_NONE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (sqrts == MAP_FAILED) {  // 如果映射失败则输出错误信息并退出
        fprintf(stderr, "Couldn't mmap() region for sqrt table; %s\n", 
                strerror(errno));
        exit(EXIT_FAILURE);
    }

    // 现在释放这段虚拟内存
    if (munmap(sqrts, MAX_SQRTS * sizeof(double) + AS_LIMIT) == -1) {  // 解除映射
        fprintf(stderr, "Couldn't munmap() region for sqrt table; %s\n", 
                strerror(errno));
        exit(EXIT_FAILURE);
    }

    // 注册信号处理程序以捕获 SIGSEGV
    act.sa_sigaction = handle_sigsegv;  // 设置信号处理函数
    act.sa_flags = SA_SIGINFO;  // 使用 `sa_sigaction` 而不是 `sa_handler`
    sigemptyset(&act.sa_mask);  // 清空信号屏蔽集
    if (sigaction(SIGSEGV, &act, NULL) == -1) {  // 注册 SIGSEGV 信号处理程序
        fprintf(stderr, "Couldn't set up SIGSEGV handler; %s\n", 
                strerror(errno));
        exit(EXIT_FAILURE);
    }
}
```

- **`setup_sqrt_region`** 函数：
  - **内存映射**：首先，通过 `mmap` 创建一个虚拟地址段，但不实际分配物理内存（设置 `PROT_NONE`）。
  - **解除映射**：然后，通过 `munmap` 释放该虚拟内存段。这是为了确保即使之前已经占用了该区域，也能重新设置。
  - **注册信号处理程序**：通过 `sigaction` 注册 `handle_sigsegv` 作为 `SIGSEGV` 信号的处理程序，这样当发生Page Fault时，会调用该函数。

### 4. `handle_sigsegv` 函数

```c
static void handle_sigsegv(int sig, siginfo_t *si, void *ctx)
{
    uintptr_t fault_addr = (uintptr_t)si->si_addr;  // 获取发生Page Fault的地址
    double *page_base = (double *)align_down(fault_addr, page_size);  // 对齐到页面的基地址
    static double *last_page_base = NULL;  // 记录上一个页面基地址

    // 分配触发Page Fault的页面
    if (mmap(page_base, page_size, PROT_READ | PROT_WRITE,
             MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED, -1, 0) == MAP_FAILED) {
        fprintf(stderr, "Couldn't mmap(); %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }

    // 计算平方根并存入表单
    calculate_sqrts(page_base, page_base - sqrts, page_size / sizeof(double));

    // 释放前一个页面的物理内存以节省内存资源
    if (last_page_base && munmap(last_page_base, page_size) == -1) {
        fprintf(stderr, "Couldn't munmap(); %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }

    last_page_base = page_base;  // 更新记录的最后一个页面基地址
}
```

- **`handle_sigsegv`** 函数：
  - 当程序试图访问未分配的页面时，会触发 `SIGSEGV` 信号，此函数被调用来处理Page Fault。
  - **获取Page Fault地址**：函数首先获取触发Page Fault的内存地址。
  - **分配页面**：通过 `mmap` 分配相应的物理内存，并将其映射到引发Page Fault的虚拟地址。
  - **计算并存储结果**：调用 `calculate_sqrts` 计算并存储平方根值。
  - **释放旧页面**：为了节约内存，函数会在分配新页面后释放前一个页面的物理内存。

这个程序展示了如何使用Unix的虚拟内存管理机制来高效地管理大型数据结构（如缓存表）。通过 `mmap` 进行内存映射，并通过 `sigaction` 捕捉 `SIGSEGV` 信号，程序能够在需要时动态分配物理内存，并且在物理内存不足时有效地释放未使用的页面。这种按需分配内存的方式非常适用于处理大规模数据计算场景。

在这段代码中展示了一个极端情况下的内存管理示例，演示如何使用虚拟内存特性，通过仅使用一个物理内存页面来管理一个巨大的虚拟表单。这种方式虽然在实际应用中很少直接使用，但它有效展示了操作系统虚拟内存机制的灵活性和强大功能。

### 程序运行分析

运行该程序，并通过 `test_sqrt_region` 函数随机访问表单项。在这个过程中，由于表单项在虚拟地址空间中并未实际映射到物理内存，因此会触发多个 `Page Fault`。每当 `Page Fault` 发生时，`handle_sigsegv` 函数被调用，并为相应的表单项分配和计算一个页面的值。

**极端内存使用场景**：

- **单一物理页面的使用**：尽管表单非常大，占用了很大的虚拟地址空间，但实际内存中始终只有一个物理页面被使用。每次访问新页面时，程序会释放上一次分配的页面，并将新的数据填充到当前页面中。
- **展示虚拟内存特性的能力**：这种方式显示了虚拟内存系统的灵活性——它可以将一个大表单的表示映射到一个小得多的物理内存空间中，且通过 `Page Fault` 机制动态管理物理内存的分配和释放。

> 为什么一个物理页面可以工作？这和惰性分配（Lazy Allocation）有什么区别？

- **初始状态**：在程序开始时，`setup_sqrt_region` 函数分配了一个虚拟地址段，但并未分配任何物理内存页面。这是因为分配后立即通过 `munmap` 释放了该虚拟内存段的物理内存。
- **第一次 `Page Fault` 处理**：当第一次 `Page Fault` 发生时，程序没有任何物理页面被映射。因此，`handle_sigsegv` 函数会分配第一个页面，并在其中存储平方根表的部分值。
- **后续 `Page Fault` 处理**：当访问更多的表单项时，如果访问的项不在已经分配的页面中，会再次触发 `Page Fault`。`handle_sigsegv` 会分配新页面并释放上一个页面，从而实现只使用一个物理页面来处理整个表单。

**惰性分配的区别**：

- 惰性分配是指内存只有在实际需要时才被分配，而不是在程序初始化时分配。在这个示例中，惰性分配被进一步极端化——不仅内存分配是按需的，而且在任何时候都只保持一个物理页面的使用。
- 这种方法展示了极端情况下，如何利用虚拟内存机制来节省物理内存，同时处理大规模数据。

### 实际应用的考虑

尽管该示例展示了只使用一个物理页面管理大表单的可能性，但在实际应用中，这种方法并不常见。原因包括：

- **性能问题**：频繁的页面分配和释放会导致性能下降。
- **实际内存管理需求**：通常情况下，应用程序会保留多个物理页面以避免频繁的 `Page Fault` 处理，从而提高运行效率。

然而，这个极端示例很好地展示了虚拟内存系统的灵活性，并帮助理解了操作系统如何通过 `Page Fault` 机制动态管理内存。

## 使用虚拟内存特性实现 Garbage Collector（GC）

接下来讨论 `Garbage Collector`（垃圾回收器，简称 GC）的基本思想和一种特定的实现方式——**copying GC**。GC 是许多现代编程语言中的核心机制，它通过自动内存管理，解除了程序员手动释放内存的负担。教授还展示了如何利用虚拟内存特性来优化 GC 的实现。

### 1. **GC 的基本概念**

GC 是一种自动管理内存的机制，帮助程序员避免手动调用 `free` 来释放不再使用的内存。在拥有 GC 的编程语言中，程序员只需要申请内存，无需关心如何释放它，GC 会自动跟踪哪些对象仍在使用，哪些对象不再需要，并释放不再使用的内存。

**带有 GC 的语言**：

- **带有 GC 的语言**：Java、Python、Golang 等几乎所有现代语言（除了 C 和 Rust）都包含 GC。
- **没有 GC 的语言**：C 和 Rust 中，内存管理需要由程序员手动完成。

### 2. **Copying GC 的基本思想**

Copying GC 是一种垃圾回收算法，通过将还在使用的对象从一个空间（from 空间）复制到另一个空间（to 空间），并丢弃未使用的对象来完成内存释放。

### **步骤：**

1. **内存分区**：将内存分成两个空间——`from` 空间和 `to` 空间。
   - 应用程序最初从 `from` 空间中分配内存。
   - 当 `from` 空间快被用尽时，GC 开始执行。

2. **根节点扫描**：GC 从程序中存储的根节点开始遍历，这些根节点存储在寄存器或栈中。根节点指向仍在使用的对象。
   - 假设内存中存在一些指针相互引用的对象（如一个树结构）。

3. **对象拷贝**：GC 会将仍然使用的对象从 `from` 空间复制到 `to` 空间。
   - 开始时将根节点指向的对象复制到 `to` 空间，并更新根节点中的指针，指向 `to` 空间中的新对象。

4. **递归跟踪对象**：对于每个对象，GC 继续检查对象中的指针，并将这些指针指向的对象也复制到 `to` 空间。
   - 拷贝过程中，会在 `from` 空间的对象留下 **forwarding 指针**（转发指针），用来记录已经拷贝的对象及其在 `to` 空间中的地址。

5. **避免重复拷贝**：当 GC 发现一个指针指向的对象已经被拷贝过，它会根据 `from` 空间中留下的 forwarding 指针，更新其他指针指向 `to` 空间中的对象，而不是重新拷贝对象。

6. **释放 `from` 空间**：一旦所有仍然使用的对象都被复制到 `to` 空间，并且指针全部更新完毕，GC 可以丢弃 `from` 空间中的所有对象并将其标记为空闲空间。

### **关键点：**

- GC 只会复制还在使用的对象，未使用的对象不会被复制，自动释放内存。
- 在垃圾回收完成后，`from` 和 `to` 空间角色对换，新的对象会继续分配到现在作为 `from` 空间的区域中。

<img src="{{ site.baseurl }}/docs/assets/image-20240904113557548.png" alt="image-20240904113557548"  />

1. **空白的 to 空间**：一开始，`to` 空间是空的，GC 从 `from` 空间开始扫描根节点。根节点指针存储在寄存器或栈中，并且指向仍在使用的对象。

2. **复制根节点到 to 空间**：GC 将根节点 `obj 1` 从 `from` 空间复制到 `to` 空间，更新根节点指针指向 `to` 空间中的对象。

3. **复制第二个对象到 to 空间**：GC 继续扫描，发现 `obj 1` 的指针指向 `obj 2`，于是将 `obj 2` 也从 `from` 空间复制到 `to` 空间，更新 `obj 1` 中的指针。

4. **复制第三个对象并通过 forwarding 指针更新指针**：GC 扫描到 `obj 2` 的指针指向 `obj 3`，并发现 `obj 3` 已经被转发（forwarding）。GC 直接将 `obj 2` 中的指针更新为指向 `to` 空间中的 `obj 3`。

### **GC 与虚拟内存特性的结合**

GC 可以利用操作系统提供的虚拟内存特性（如页面保护、`Page Fault` 处理等）来优化对象的管理和拷贝过程。虚拟内存使得 GC 能够在需要时按需分配物理内存，并动态管理内存的使用，避免内存碎片等问题。

**虚拟内存的优势**：

- **惰性分配（Lazy Allocation）**：只有在程序实际访问内存时，才会通过 `Page Fault` 分配物理内存，这使得内存分配更加高效。
- **内存保护**：虚拟内存可以通过设置页面的读写权限，控制哪些对象可以被访问，哪些对象应当被复制或丢弃。

## Baker算法的实时垃圾回收机制

论文中讨论的是一种更为复杂的GC算法——Baker算法。Baker算法是一种 **incremental GC**（增量垃圾回收）算法，它的主要特点是能够在 **程序运行的同时** 进行垃圾回收，而不需要暂停整个程序。这使得程序运行可以保持实时性，不会因为垃圾回收而导致长时间的停顿。

### 1. **Baker算法的基本思想**

Baker算法使用了两个空间：**from 空间** 和 **to 空间**，类似于之前介绍的 Copying GC，但是它是渐进式地完成垃圾回收过程的。具体来说，垃圾回收不再需要一次性地完成，而是随着程序的运行逐步进行。

**步骤概述：**

1. **拷贝根节点**：当垃圾回收（GC）开始时，唯一必要的操作是将根节点（即仍在使用的内存的起点）从 `from` 空间复制到 `to` 空间。这时，根节点的指针仍然指向 `from` 空间的对象，尚未更新。

2. **分批扫描对象**：在应用程序调用 `new` 申请新的内存时，GC 逐渐扫描并将一些对象从 `from` 空间复制到 `to` 空间。这意味着垃圾回收不再一次性处理整个 `heap`，而是随着每次内存分配逐步完成回收。
   - 每次调用 `new`，GC 会进一步扫描并处理更多对象。
   - 这样，GC 的工作被拆分成了多个小步骤，使得程序和垃圾回收可以同时进行。

3. **处理指针的dereference**：在程序运行时，当应用程序尝试使用一个指针时（即 dereference 操作），如果这个指针仍然指向 `from` 空间的对象，GC 会将该对象从 `from` 空间复制到 `to` 空间，并更新指针指向新的位置。

   - **关键点**：每次指针访问时，程序需要检查该指针是否指向 `from` 空间。如果是，则立即触发复制操作（forwarding）。

4. **最终回收**：当所有仍在使用的对象都从 `from` 空间复制到 `to` 空间后，`from` 空间可以被完全清空并重新用于分配内存。

   - 在这种方式下，GC 与应用程序可以并行工作，每次新对象分配和指针访问都会推动垃圾回收的进展。

### 2. **Baker算法的优势**

Baker算法的主要优势在于它的 **实时性** 和 **渐进性**：

- **实时性**：垃圾回收不需要停止整个程序，而是随着应用程序的内存分配和使用逐步进行。这样可以减少程序的停顿时间（即 GC pause），特别适用于对延迟敏感的应用。

- **渐进性**：垃圾回收是分步进行的，每次内存分配或对象访问时，GC 只处理一部分对象。这使得 GC 的工作量可以被分散，而不是集中在某个时刻进行大量的对象复制和内存清理。

### 3. **Baker算法的挑战**

尽管 Baker 算法有其优点，但它也面临着一些挑战，主要包括 **性能开销** 和 **并行处理的复杂性**。

**性能开销**

在 Baker 算法中，每次指针访问（dereference）都需要额外的检查和处理：

- **额外的 dereference 操作**：每次应用程序访问一个指针时，必须检查该指针是否指向 `from` 空间。如果是，就必须将指针指向的对象从 `from` 空间复制到 `to` 空间。这增加了程序的执行开销。

  - 在传统内存管理中，dereference 是一个非常快速的操作，只需要一次内存读取即可。
  - 在 Baker 算法中，dereference 变成了一个较为复杂的操作，可能需要多次读取内存并进行对象复制和指针更新，降低了程序的运行效率。

**并行处理的复杂性**

如果程序运行在多核 CPU 环境下，并且 GC 也在并行运行，可能会出现 **race condition**（竞争条件）的问题：

- **竞争条件**：应用程序和 GC 都可能同时访问和修改对象。如果两者同时对同一个对象执行 dereference 检查或对象复制，可能会导致错误。
  - 应用程序可能在 dereference 一个指针时，GC 也在复制该对象。这样可能会导致对象被复制两次，或者指针被更新到错误的位置。

- **难以并行执行**：为了防止竞争条件，需要额外的同步机制或锁，来确保 GC 和应用程序之间的正确性。这增加了算法的实现复杂性，并可能降低并行执行的效率。

## 使用虚拟内存特性优化 Garbage Collector（GC）

论文提出了一种更为复杂的增量式（incremental）垃圾回收算法，利用了虚拟内存的特性来减少指针检查的开销，并能够实现几乎零成本的并行垃圾回收。该方案基于虚拟内存的页面保护机制，通过自动触发 `Page Fault` 来完成垃圾回收操作，避免了每次访问对象时手动检查指针的负担。

### **空间划分与内存保护**

在标准的复制式 GC 中，堆内存被分为 `from` 和 `to` 两个空间。在论文提出的优化方案中， `to` 空间进一步划分为 **scanned** 和 **unscanned** 区域。每次垃圾回收启动时，GC 首先从 `from` 空间中找到根节点对象，将它们复制到 `to` 空间。这些根节点对象被放置在 `to` 空间的 `unscanned` 区域，且其内存页面的权限被设置为 `None`（即不可访问）。

- **scanned 区域**：已经扫描并完成指针更新的对象，GC 已确保其指向 `to` 空间中的新地址。
- **unscanned 区域**：尚未扫描的对象，内存访问会触发 `Page Fault`，进而由 GC 进行处理。

### **GC 的根节点拷贝与 Page Fault Handler**

**根节点拷贝**：在 GC 开始时，根节点对象（即仍在使用的对象）会首先从 `from` 空间复制到 `to` 空间的 `unscanned` 区域。根节点中的指针仍然指向 `from` 空间中的其他对象，这些对象尚未被复制。

**页面保护**：当程序首次访问根节点对象时，由于 `to` 空间的 `unscanned` 区域内存权限为 `None`，该访问会触发 `Page Fault`。此时，操作系统将调用 `Page Fault Handler` 来处理该错误，并让 GC 继续执行。

**扫描对象**：在 `Page Fault Handler` 中，GC 会扫描当前发生 Page Fault 的整个内存页面，处理页面中所有对象的指针。对于每个指针，GC 会检查其是否指向 `from` 空间中的对象，如果是，则会将指针所指的对象从 `from` 空间复制到 `to` 空间的 `unscanned` 区域。

**标记已扫描**：当某个页面中的所有对象被扫描并更新指针后，GC 会将该页面的权限恢复（通过 `Unprot` 系统调用），并将页面标记为 `scanned`。此时，这些对象可以安全地被应用程序访问，因为它们的指针已经指向了 `to` 空间中的其他有效对象。

**递归 Page Fault**：随着程序继续运行，若应用程序访问尚未被扫描的对象（即 `to` 空间中 `unscanned` 区域的对象），同样会再次触发 `Page Fault`。GC 会在 Page Fault Handler 中扫描这些对象，更新它们的指针并将其指向 `to` 空间中合适的地址。

这种递归的方式确保了在程序运行期间，GC 可以逐步完成对象的复制和指针更新，而无需一次性处理整个堆内存。

### **GC 完成后的空间切换**

当所有对象都被扫描并复制到 `to` 空间后，`from` 空间将不再包含任何仍在使用的对象，因此可以完全清空并释放。这时，GC 可以切换 `from` 和 `to` 空间的角色：现在 `to` 空间成为新的 `from` 空间，用于新的内存分配，而空闲的 `from` 空间则变为下一个垃圾回收周期中的 `to` 空间。

### **方案的优势**

该方案有多个显著优势：

- **减少指针检查的开销**：传统的复制式 GC 需要在每次访问指针时手动检查指针是否指向 `from` 空间中的对象，而这个优化方案通过虚拟内存的页面保护机制，将指针检查交由硬件完成。当程序访问未经扫描的对象时，会自动触发 `Page Fault`，GC 通过处理 `Page Fault` 来完成对象转发。因此，指针检查的开销大大降低，转而由硬件高效地处理。

- **增量式 GC**：该方案仍然保持了增量式垃圾回收的特性。每次 `Page Fault` 处理仅需扫描少量对象，GC 的工作被拆解为小块，可以与程序的正常执行交替进行，避免了一次性停顿，特别适合对实时性要求较高的系统。

- **并行运行 GC**：由于指针检查通过硬件完成，GC 的执行与应用程序的正常运行能够在后台并行进行。对于多核系统，可以利用空闲的 CPU 资源来并行处理垃圾回收任务，而不会对程序的执行造成显著影响。

该优化方案通过利用虚拟内存的页面保护机制和 `Page Fault` 处理，成功减少了指针检查的开销，并实现了增量式、几乎无停顿的垃圾回收。该方案将垃圾回收的过程自动化，且通过操作系统的内存管理功能，使得 GC 与应用程序可以并行运行，在不影响性能的前提下，有效地回收内存。这种设计在现代系统中具有很强的实用性，尤其适用于实时性强或多线程并行执行的应用场景。

# 代码示例

为了更清晰地说明论文的内容，教授提供了一个针对论文中方法的简单实现，代码的结构展示了如何通过一个简单的 GC (垃圾回收) 框架来管理动态分配的内存。应用程序会频繁地创建和销毁链表，造成大量的垃圾对象，GC 则通过检查指针并在需要时复制对象来管理内存。下面将逐步讲解代码的核心部分。

### **API Functions**

应用程序使用的API

```c
//
// API to collector
//

struct elem *readptr(struct elem **ptr);  // 读取指针，如果指向的是from空间，执行拷贝
struct elem *new();  // 分配新的元素（分配内存）

```

- **`readptr`**：负责检查传入的指针 `ptr` 是否指向 `from` 空间。如果是，它会将指向的对象拷贝到 `to` 空间，然后返回指向 `to` 空间的地址。对于虚拟内存的实现，如果指针已经指向 `to` 空间，该函数的操作成本会非常低，只需要简单地返回参数。

- **`new`**：用于分配新的元素，也就是在 `to` 空间中分配内存，以支持新的链表元素。

## **应用程序逻辑**

应用程序线程通过循环反复创建并检查链表。在创建新链表时，之前的链表被垃圾回收机制标记为垃圾并被处理。

### **`app_thread` Function**

```c
void *app_thread(void *x) {
  for (int i = 0; i < 1000; i++) {
    make_clist();           // 创建一个新的循环链表
    check_clist(LISTSZ);     // 检查链表是否正确
  }
}
```

- **`app_thread`**：这是应用程序的主循环，它会循环 1000 次，每次调用 `make_clist` 来创建新的循环链表，然后调用 `check_clist` 来检查链表的正确性。

  每次创建新的链表后，之前创建的链表就变成了垃圾对象，GC 需要清理这些不再使用的内存。

### **`make_clist` Function**

```c
// 创建一个循环链表
void make_clist(void) {
  struct elem *e;

  root_head = new();  // 创建链表的头节点
  readptr(&root_head)->val = 0;  // 初始化头节点的值为0
  readptr(&root_head)->next = readptr(&root_head);  // 设置头节点的next指针指向自己，形成单节点循环链表
  root_last = readptr(&root_head);  // 设置链表的尾节点等于头节点

  // 创建链表的其余节点
  for (int i = 1; i < LISTSZ; i++) {
    e = new();  // 分配一个新的元素
    readptr(&e)->next = readptr(&root_head);  // 新节点的next指针指向链表的当前头节点
    readptr(&e)->val = i;  // 初始化新节点的值
    root_head = readptr(&e);  // 更新链表的头节点，指向新创建的节点
    readptr(&root_last)->next = readptr(&root_head);  // 更新尾节点的next指针，指向新的头节点，保持循环结构
    check_clist(i+1);  // 每次添加一个新节点后，检查链表的完整性
  }
}
```

- **创建链表**：该函数构建一个大小为 `LISTSZ` 的循环链表。每次创建链表时，链表头节点 `root_head` 和尾节点 `root_last` 会被动态更新。函数执行的步骤如下：

  1. 使用 `new` 创建一个新的节点并将其作为 `root_head`，然后初始化其值和 `next` 指针。
  2. 在循环中，每次分配一个新节点，并将其添加到链表的开头。新节点的 `next` 指向链表的旧头节点，链表尾节点的 `next` 指针更新为新创建的节点，形成循环链表结构。

  由于 `readptr` 包裹了每个指针操作，`readptr` 会检查指针是否位于 `from` 空间中，并在必要时复制对象。

**设计意图**

- **大量垃圾产生**：由于链表不断被创建，每次新的链表创建后，之前的链表就变成了垃圾对象。这会导致 GC 需要频繁地处理未使用的内存。

- **指针检查与复制**：`readptr` 的作用是在程序访问指针时，自动检查该指针是否指向 `from` 空间，并在需要时将对象复制到 `to` 空间。这种方式模仿了现代 GC 系统在运行时对指针的自动管理。

## `new` 和 `readptr`

GC 的核心部分就是如何在 `new` 和 `readptr` 这两个 API 中处理对象的分配和指针的转发（forwarding）。通过这些步骤，GC 可以有效地将仍在使用的对象从 `from` 空间复制到 `to` 空间，同时回收未使用的内存。

### 1. **`new()` 函数的实现**

```c
struct elem *
new(void) {
  struct elem *n;
  
  pthread_mutex_lock(&lock);  // 加锁，确保多线程环境下的同步
  if (collecting && scanned < to_free_start) {  // 如果正在进行垃圾回收，且还未扫描完所有对象
    scan(scanned);  // 扫描对象，处理垃圾回收的对象转移
    if (scanned >= to_free_start) {
      end_collecting();  // 如果扫描完成，结束回收过程
    }
  }
  // 检查to空间是否有足够的内存来分配新对象
  if (to_free_start + sizeof(struct obj) >= to + SPACESZ) {
    flip();  // 如果没有足够空间，则触发GC的flip操作
  }
  // 分配新的对象
  n = (struct elem *) alloc();
  pthread_mutex_unlock(&lock);  // 解锁
  
  return n;  // 返回新分配的对象
}
```

`new()` 函数的主要工作是为新对象分配内存，同时检查当前是否正在进行垃圾回收（`collecting`）。如果正在进行 GC，那么会调用 `scan()` 函数继续扫描对象，并将对象从 `from` 空间转移到 `to` 空间。当 `to` 空间不够时，调用 `flip()` 触发垃圾回收。

### 2. **`flip()` 函数的实现**

```c
void
flip() {
  char *tmp = to;

  printf("flip spaces\n");

  assert(!collecting);
  to = from;  // 交换from和to空间
  to_free_start = from;
  from = tmp;

  collecting = 1;  // 标记开始垃圾回收
  scanned = to;  // 设置scanned为to空间的开始地址

#ifdef VM
  // 如果启用了虚拟内存，解除对to空间的映射，防止直接访问
  if (mprotect(to, SPACESZ, PROT_NONE) < 0) {
    fprintf(stderr, "Couldn't unmap to space; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }
#endif

  // 将根节点从from空间转移到to空间
  root_head = (struct elem *) forward((struct obj *)root_head);
  root_last = (struct elem *) forward((struct obj *)root_last);

  pthread_cond_broadcast(&cond);  // 通知所有等待的线程
}
```

`flip()` 函数的作用是启动垃圾回收过程。它通过交换 `from` 和 `to` 空间指针来开始新一轮的 GC，并将根节点从 `from` 空间转发（forward）到 `to` 空间。此时，`from` 空间即将被清空，而 `to` 空间开始接收新的对象。如果启用了虚拟内存，`flip()` 还会解除对 `to` 空间的映射，防止对未转移的对象的直接访问。

### 3. **`forward()` 函数的实现**

```c
struct obj *
forward(struct obj *o) {
  struct obj *n = o;
  if (in_from(o)) {  // 检查对象是否在from空间
    if (o->newaddr == 0) {   // 如果对象还未被拷贝
      n = copy(o);  // 将对象拷贝到to空间
      printf("forward: copy %p (%d) to %p\n", o, o->e.val, n);
    } else {
      n = o->newaddr;  // 对象已经被拷贝，返回to空间的地址
      printf("forward: already copied %p (%d) to %p\n", o, o->e.val, n);
    }
  }
  return n;  // 返回对象的新地址
}
```

`forward()` 函数用于将对象从 `from` 空间转移到 `to` 空间。它首先检查对象是否在 `from` 空间，如果是且尚未被复制，则将对象复制到 `to` 空间；否则，直接返回对象在 `to` 空间的地址。如果对象已经被转移，它会使用 `newaddr` 字段记录新地址，避免重复拷贝。

### 4. **`readptr()` 函数的实现**

```c
struct elem *
__attribute__ ((noinline))
readptr(struct elem **p) {
#ifndef VM
  struct obj *o = forward((struct obj *) (*p));  // 如果没有使用虚拟内存，执行forward操作
  *p = (struct elem *) o;  // 更新指针，指向to空间的地址
#endif
  return *p;  // 返回指针
}
```

`readptr()` 函数用于读取并处理指针。如果不使用虚拟内存（即未定义 `VM` 宏），它会通过 `forward()` 函数检查指针是否指向 `from` 空间，如果是，则将对象复制到 `to` 空间并返回新地址。这样可以确保当程序访问指针时，对象已位于 `to` 空间，从而避免访问未复制的对象。

### **没有虚拟内存时的实现：**

- **每次指针访问都需要检查和转发**：在没有虚拟内存的情况下，`readptr` 和 `forward` 函数会对每个指针执行额外的检查，以确保指向的是正确的内存空间。这增加了程序的开销，特别是在大量指针操作时，因为每次访问都需要判断并可能复制对象。

### **使用虚拟内存的实现优化：**

- 如果启用了虚拟内存（通过定义 `VM` 宏），`readptr` 中的 `forward` 操作会被虚拟内存机制取代。通过页面保护（`mprotect`）将未扫描的对象所在页面设置为不可访问，程序在访问这些页面时会触发 `Page Fault`，系统会在处理 `Page Fault` 时自动完成对象的转发。这种方式减少了每次访问时的指针检查开销。

## 利用虚拟内存技术实现垃圾回收（GC）

接下来看一下这里如何使用虚拟内存。下面代码片段展示了如何利用虚拟内存技术实现垃圾回收（GC），并且在应用程序运行时避免复杂的指针检查。这种方法使用了共享内存对象（shared memory object）和 `mmap` 系统调用来映射内存区域，同时依赖 `SIGSEGV` 信号和 `mprotect` 操作来处理未扫描的内存页面。以下是对关键部分的详细解释：

### 1. **设置虚拟内存区域**

```c
static void setup_spaces(void) {
  struct sigaction act;
  int shm;

  // 创建共享内存对象
  if ((shm = shm_open("baker", O_CREAT|O_RDWR|O_TRUNC, S_IRWXU)) < 0) {
    fprintf(stderr, "Couldn't shm_open: %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }

  // 删除共享内存对象的链接
  if (shm_unlink("baker") != 0) {
    fprintf(stderr, "shm_unlink failed: %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }

  // 调整共享内存对象的大小，使其适合 from 和 to 空间
  if (ftruncate(shm, TOFROM) != 0) {
    fprintf(stderr, "ftruncate failed: %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }

  // 为 mutator (应用程序) 映射 from 和 to 空间
  mutator = mmap(NULL, TOFROM, PROT_READ|PROT_WRITE, MAP_SHARED, shm, 0);
  if (mutator == MAP_FAILED) {
    fprintf(stderr, "Couldn't mmap() from space; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }

  // 为 GC (回收器) 再次映射 from 和 to 空间
  collector = mmap(NULL, TOFROM, PROT_READ|PROT_WRITE, MAP_SHARED, shm, 0);
  if (collector == MAP_FAILED) {
    fprintf(stderr, "Couldn't mmap() from space; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }

  // 初始化from和to指针
  from = mutator;
  to = mutator + SPACESZ;
  to_free_start = to;

  // 注册SIGSEGV信号的处理器，捕捉Page Fault
  act.sa_sigaction = handle_sigsegv;
  act.sa_flags = SA_SIGINFO;
  sigemptyset(&act.sa_mask);
  if (sigaction(SIGSEGV, &act, NULL) == -1) {
    fprintf(stderr, "Couldn't set up SIGSEGV handler: %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }
}
```

- **共享内存对象 (`shm_open`)**：`shm_open` 用于创建一个共享内存对象，该对象将被映射到 `from` 和 `to` 空间。共享内存对象类似于一个位于内存中的文件，它没有与磁盘文件关联的持久性。
- **`ftruncate`**：用于将共享内存对象的大小调整为 `from` 和 `to` 空间的总大小。
- **`mmap`**：`mmap` 系统调用两次映射了共享内存对象，一次为 mutator（应用程序）使用，另一次为 GC 使用。两者映射相同的内存区域，意味着 GC 和应用程序可以共享对这片内存的访问。
- **信号处理 (`sigaction`)**：设置 `SIGSEGV` 信号处理程序，当访问未被映射的内存时（产生 `Page Fault`），会触发处理器 `handle_sigsegv`。

### 2. **虚拟内存机制中的 `readptr` 实现**

```c
struct elem *
__attribute__ ((noinline))
readptr(struct elem **p) {
#ifndef VM
  struct obj *o = forward((struct obj *) (*p));
  *p = (struct elem *) o;
#endif
  return *p;
}
```

- **使用虚拟内存时的 `readptr`**：当启用了虚拟内存时（`VM` 被定义），`readptr` 不再需要进行复杂的指针检查操作。如果指针指向未扫描区域，将产生 `Page Fault`，由 `handle_sigsegv` 信号处理器进行处理。相比没有虚拟内存的情况，这大大简化了 `readptr` 的操作。

### 3. **Page Fault 处理器 `handle_sigsegv`**

```c
#define align_down(v, a) ((v) & ~((typeof(v))(a) - 1))

static void
handle_sigsegv(int sig, siginfo_t *si, void *ctx) {
  uintptr_t fault_addr = (uintptr_t)si->si_addr;
  double *page_base = (double *)align_down(fault_addr, PGSIZE);

  printf("fault at adddr %p (page %p)\n", fault_addr, page_base);

  pthread_mutex_lock(&lock);
  scan(page_base);  // 扫描并转移Page中的对象
  pthread_mutex_unlock(&lock);

  // 恢复Page的读写权限
  if (mprotect(page_base, PGSIZE, PROT_READ|PROT_WRITE) < 0) {
    fprintf(stderr, "Couldn't mprotect to-space page; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }
}
```

- **`handle_sigsegv`**：当应用程序尝试访问未扫描的内存页面时，会触发 `Page Fault`，随后系统通过 `SIGSEGV` 信号捕捉到该错误。
  - **地址对齐 (`align_down`)**：获取发生错误的页面基地址。
  - **扫描 (`scan`)**：在 `scan` 函数中，GC 会扫描该页面中的对象，并将其从 `from` 空间复制到 `to` 空间。
  - **恢复页面权限 (`mprotect`)**：在完成对象转移和扫描后，`mprotect` 恢复该页面的读写权限，使得应用程序可以继续访问它。

### 4. **关键概念**

- **共享内存 (`shm_open`)**：通过 `shm_open` 系统调用创建一个共享内存对象，两个 `mmap` 操作使 GC 和应用程序共享这片内存。

- **Page Fault**：当应用程序访问未扫描的页面时，触发 `Page Fault` 信号，通过 `handle_sigsegv` 信号处理器来处理页面中的对象转移。

- **`mprotect` 和 `虚拟内存`**：通过 `mprotect` 调整页面的访问权限（例如将 `unscanned` 区域设置为不可访问），在适当的时间段恢复页面访问权限，避免不必要的检查。

通过虚拟内存，GC 可以简化对指针的处理。`readptr` 不再需要复杂的指针检查，直接返回指针。当应用程序尝试访问未扫描区域时，GC 使用 `SIGSEGV` 信号捕捉 `Page Fault`，通过 `scan` 函数转移页面内的对象并更新其指针。

之前的代码中，`readptr` 函数在没有虚拟内存时通过 `forward` 函数来检查并转发指针，而当启用了虚拟内存（VM）时，这段代码直接跳过这部分逻辑。

```c
struct elem *
__attribute__ ((noinline))
readptr(struct elem **p) {
#ifndef VM
  struct obj *o = forward((struct obj *) (*p));
  *p = (struct elem *) o;
#endif
  return *p;
}

```

其行为可以总结如下：

- **没有 VM**：通过垃圾回收器中的 `forward` 函数来检查对象是否已经转发，并确保指针指向最新的对象。
- **有 VM**：不做任何操作。因为虚拟内存可以处理 Page Fault，当对象未扫描时会触发 Page Fault，通过 Page Fault handler 来处理。

## **`handle_sigsegv` 函数**

当程序试图访问未扫描的内存时，虚拟内存机制会产生 `Page Fault`，操作系统捕获此异常，并调用 `handle_sigsegv` 作为信号处理程序。处理程序的主要任务是对触发 `Page Fault` 的页面进行扫描，并在扫描后通过 `mprotect` 修改页面权限，使应用程序可以访问该页面。

具体步骤如下：

- **计算页面基地址**：首先，将 `fault_addr`（发生 `Page Fault` 的地址）对齐到页面大小的倍数，获取页面的基地址。这样可以确保我们锁定并操作整个页面，而不是仅仅一个地址。
- **加锁并调用垃圾回收器的扫描函数**：为了确保线程安全，处理程序在扫描页面前获取互斥锁。`scan` 函数负责检查页面内容，更新对象引用，确保内存中的数据被正确管理。
- **修改页面权限**：在扫描完成后，处理程序调用 `mprotect` 将页面权限恢复为可读写，使应用程序能够继续访问该页面中的数据。
- **释放锁**：完成所有操作后，处理程序释放锁，允许其他线程继续执行。

```c
static void
handle_sigsegv(int sig, siginfo_t *si, void *ctx)
{
  uintptr_t fault_addr = (uintptr_t)si->si_addr;
  double *page_base = (double *)align_down(fault_addr, PGSIZE);

  printf("fault at adddr %p (page %p)\n", fault_addr, page_base);

  pthread_mutex_lock(&lock);
  scan(page_base);
  pthread_mutex_unlock(&lock);
  
  if (mprotect(page_base, PGSIZE, PROT_READ|PROT_WRITE) < 0) {
    fprintf(stderr, "Couldn't mprotect to-space page; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }
}
```

**虚拟内存与垃圾回收结合的优点：**

- **减少指针转发开销**：在传统 GC 实现中，每次读取指针都需要检查对象是否已转发，这会导致频繁的检查操作。而通过利用虚拟内存，`Page Fault` 处理程序可以在需要访问内存时自动处理这些操作，大大减少了不必要的开销。
- **延迟处理**：只有在对象真正被访问时，才触发 `Page Fault` 和 `scan`，使得垃圾回收更加按需进行，避免提前扫描未使用的对象。
- **内存保护管理**：通过 `mprotect` 动态调整页面的读写权限，系统能够更加灵活地控制内存访问，同时确保垃圾回收和应用程序之间不会产生冲突。

## `flip` 函数

在 `flip` 函数中，垃圾回收（GC）通过从 `from` 和 `to` 空间的切换来管理内存。为了确保应用程序在切换过程中不会访问未处理的内存，虚拟内存技术与 `mprotect` 一起用于控制对 `to` 空间的访问。

```c
void
flip() {
  char *tmp = to;

  printf("flip spaces\n");

  assert(!collecting);
  to = from;
  to_free_start = from;
  from = tmp;

  collecting = 1;
  scanned = to;

#ifdef VM
  if (mprotect(to, SPACESZ, PROT_NONE) < 0) {
    fprintf(stderr, "Couldn't unmap to space; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }
#endif

  // move root_head and root_last to to-space
  root_head = (struct elem *) forward((struct obj *)root_head);
  root_last = (struct elem *) forward((struct obj *)root_last);

  pthread_cond_broadcast(&cond);
}
```

### 1. **切换 `from` 和 `to` 空间**

```c
to = from;
to_free_start = from;
from = tmp;
```

- `to` 和 `from` 这两个空间分别用于存放活跃对象（to）和垃圾回收前的旧对象（from）。
- 在 `flip` 中，首先将 `from` 空间中的数据拷贝到 `to` 空间，并将指针更新为新的 `from` 和 `to` 空间位置。这样，`to` 空间开始存储新的对象，而 `from` 空间中的对象将逐步被垃圾回收。

### 2. **通过 `mprotect` 保护 `to` 空间**

```c
if (mprotect(to, SPACESZ, PROT_NONE) < 0) {
    fprintf(stderr, "Couldn't unmap to space; %s\n", strerror(errno));
    exit(EXIT_FAILURE);
}
```

- 这里，使用 `mprotect` 将整个 `to` 空间标记为不可访问 (`PROT_NONE`)。这意味着应用程序在未经过处理前无法访问 `to` 空间的任何部分。
- 当应用程序尝试访问 `to` 空间时，会触发 `Page Fault`，进入 `Page Fault handler` 进行进一步处理。

### 3. **移动 `root_head` 和 `root_last` 到 `to` 空间**

```c
root_head = (struct elem *) forward((struct obj *)root_head);
root_last = (struct elem *) forward((struct obj *)root_last);
```

- 这一步将根节点（`root_head` 和 `root_last`）对象从 `from` 空间转移到 `to` 空间，标记这些重要对象不再受垃圾回收的影响。
- 此时，应用程序若试图访问这些对象，仍会触发 `Page Fault`。

### 4. **处理 Page Fault**

当应用程序尝试访问被 `mprotect` 标记为不可访问的 `to` 空间对象时，会触发 `Page Fault`。在 `Page Fault handler` 中，GC 处理这些请求并将对象从 `from` 空间转移到 `to` 空间。具体流程如下：

- **扫描未处理的Page**：`Page Fault handler` 中首先会扫描触发 `Page Fault` 的页面，检查和处理 `from` 空间中需要迁移到 `to` 空间的对象。

- **恢复页面的可访问性**：在扫描和对象拷贝完成后，调用 `mprotect` 恢复当前页面的读写权限，使得应用程序可以继续安全访问这些对象。

### 5. **为什么顺序重要？**

在 `Page Fault handler` 中，必须先扫描并处理页面中的对象，然后再恢复页面的访问权限。以下是原因：

- **防止访问未处理的对象**：如果提前解除页面的访问限制，应用程序可能会在 `GC` 处理完成前访问未扫描的对象，导致数据不一致或访问到未更新的指针。
- **确保 GC 和应用程序的安全并发**：只有在 GC 完成对页面的扫描并更新对象的地址后，应用程序才可以访问这些数据。这避免了应用程序和 GC 之间的资源竞争和数据不一致问题。

通过这个顺序，GC 可以在保证正确性和安全性的前提下，并发运行，逐步清理和转移对象，同时保证应用程序可以继续运行并访问经过处理的对象。



## 总结这节课的内容

总结这节课的内容，主要讨论了虚拟内存在垃圾回收（GC）中的应用，以及虚拟内存特性是否值得使用。以下是几个关键点：

### 1. **是否应该使用虚拟内存？**

虚拟内存在GC中的应用并不是唯一的选择。许多GC实现并不依赖虚拟内存，而是通过编译器生成的代码来完成内存管理。例如，编译器可以在程序运行时插入检查代码，确保内存的正确管理。这种方案可以避免对虚拟内存的依赖，且在性能上有更直接的控制。

- **编译器优化**：大部分现代编程语言的GC依赖编译器进行优化，编译器会生成检查和跟踪指针的代码，并通过增量式GC或并行GC等机制来减小性能开销。编译器生成的代码足够高效，可以在大多数情况下替代虚拟内存管理的复杂性。

### 2. **虚拟内存的价值**

然而，虚拟内存仍然是非常有价值的工具，尤其是在没有编译器或程序运行时参与的情况下。在以下几类场景中，虚拟内存特性变得非常有用，甚至是不可或缺的：

- **没有编译器或运行时支持的应用**：在一些系统中，比如 `checkpointing`（系统快照）或 `shared-virtual memory`（共享虚拟内存）应用中，编译器不会主动介入，这时使用虚拟内存来管理内存和捕获 `Page Fault` 可以极大简化系统设计。虚拟内存机制能够自动捕获内存访问异常，并触发相应的处理流程，减少了编写手动内存管理代码的复杂性。

- **灵活性**：虚拟内存允许系统通过 `mprotect` 和 `Page Fault handler` 进行内存权限管理，并且可以实现诸如延迟加载、内存映射文件、垃圾回收等功能，而不需要应用程序编写额外的复杂代码。这为某些特定的应用场景提供了灵活而高效的解决方案。

### 3. **现实中的广泛支持**

实际中，虚拟内存特性已被广泛应用，足够多的开发人员发现了它们的价值，现代操作系统普遍支持这些功能。虚拟内存不仅在垃圾回收中发挥作用，还广泛用于内存映射、共享内存、文件系统缓存等方面。尤其在现代的操作系统中，虚拟内存系统的优化和改进，使得它在处理大规模应用程序和复杂系统时，表现得更加高效和稳定。

## 虚拟内存系统变化

自1991年虚拟内存相关论文发表以来，虚拟内存系统发生了许多重要变化和改进，尤其是在Unix系统和Linux内核中。这些变化使得虚拟内存系统更加高效和安全。以下是其中一些重要的改变：

### 1. **5级Page Table**

现代的虚拟内存系统已经采用了5级Page Table。传统的系统一般使用3级或4级Page Table，但随着地址空间的增长，特别是在64位系统中，为了支持更大的地址空间，操作系统引入了5级Page Table。这允许系统管理更大的物理内存和更广泛的虚拟地址空间，特别是在需要处理超大数据集的应用程序中。

### 2. **TLB Flush 和地址空间标识符（ASID）**

TLB（Translation Lookaside Buffer）是缓存地址转换的硬件组件，用于加速虚拟地址到物理地址的转换。传统的TLB在切换进程时需要清空，但随着ASID（Address Space Identifier）技术的发展，系统可以在切换进程时避免频繁清空TLB。ASID允许不同进程的地址空间并存于TLB中，从而减少了切换开销并提高了性能。

### 3. **KPTI（Kernel Page Table Isolation）**

KPTI 是一个针对 Meltdown 攻击的安全补丁，旨在防止内核内存泄露。Meltdown 是一种硬件漏洞，利用CPU的投机执行机制绕过内存保护机制，读取不应访问的内存内容。KPTI通过将用户态和内核态的Page Table隔离，避免了进程直接访问内核的Page Table。这个补丁于2018年引入，成为操作系统增强安全性的一个里程碑。

### 4. **内核持续更新与改进**

Linux内核的虚拟内存子系统是一个持续开发和改进的领域。随着技术的发展和新的应用需求，内核中的虚拟内存子系统不断演进。每隔几个月，Linux内核的各个方面都会有大量的更新，其中包括对虚拟内存管理的优化、功能扩展和性能提升。举例来说，内核经常会对Page Table的管理、内存分配机制、TLB优化以及不同硬件平台的支持进行改进。

### 5. **虚拟内存与并发垃圾回收（GC）**

基于虚拟内存的垃圾回收机制也受到了广泛的关注。论文中介绍的基于虚拟内存的GC机制是一个增量式的并发回收方案，通过利用虚拟内存特性，如 `Page Fault` 处理和 `mprotect`，GC可以在不阻塞应用程序运行的情况下，渐进式地回收内存。虚拟内存系统为并发GC的实现提供了更高的效率，使GC能够持续运行，而不会显著影响应用程序的性能。

在基于虚拟内存的GC中，扫描对象图时，系统从根节点开始，逐步遍历和拷贝对象到 `to` 空间。当 `unscanned` 区域停止增长时，GC就知道所有的对象都已被处理完毕。这种方案允许GC并发运行，不必像传统GC一样暂停应用程序。此外，虚拟内存系统还减少了指针检查的复杂度，通过捕获 `Page Fault`，GC可以智能地处理指针指向的对象，确保高效地进行垃圾回收。


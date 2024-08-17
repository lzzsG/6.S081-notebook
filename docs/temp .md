## XV6内核地址空间的初始化与映射

接下来，我们将通过代码的具体实现，进一步加深对内核地址空间初始化和物理设备映射的理解。在这个过程中，我们会发现前面所讨论的概念如何通过代码来实现。

### 启动XV6并进入内核初始化

在这部分课程中，我们通过启动QEMU模拟的主板，并打开gdb进行调试，来一步步观察内核初始化的过程。

首先，启动XV6并进入到`main`函数，这是内核启动的主要入口。我们会在这里跟踪`kvminit`函数的执行，`kvminit`函数是用来初始化内核地址空间的。

```c
/*
 * the kernel's page table.
 */
pagetable_t kernel_pagetable;

extern char etext[];  // kernel.ld sets this to end of kernel code.

extern char trampoline[]; // trampoline.S

// Make a direct-map page table for the kernel.
pagetable_t
kvmmake(void)
{
  pagetable_t kpgtbl;

  kpgtbl = (pagetable_t) kalloc();
  memset(kpgtbl, 0, PGSIZE);

  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // PLIC
  kvmmap(kpgtbl, PLIC, PLIC, 0x4000000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  // allocate and map a kernel stack for each process.
  proc_mapstacks(kpgtbl);
  
  return kpgtbl;
}

// Initialize the one kernel_pagetable
void
kvminit(void)
{
  kernel_pagetable = kvmmake();
}
```

### `kvminit`(`kvmmake`)函数的代码解析

`kvminit`(`kvmmake`)函数是设置内核地址空间的关键代码。这个函数的主要任务是：

1. **分配最高级的Page Directory**：
   
   - 通过调用`kalloc()`函数为最高级的Page Directory分配物理页。这一页将用作内核页表的根，表示最高级的页目录。
   - 通过调用`memset()`初始化这段内存，将其内容清零，确保后续使用时的正确性。
   
   在gdb中，我们可以设置断点并查看`kvminit`函数的执行情况：
   
   ```gdb
   (gdb) b kvminit
   (gdb) c
   ```
   
   通过`layout split`命令，可以清楚地看到代码的执行过程，特别是在分配Page Directory时的操作。
   
   ![image-20240817213014981]({{ site.baseurl }}/docs/assets/image-20240817213014981.png)
   
   
   
2. **映射I/O设备**：
   
   - `kvminit`(`kvmmake`)函数的另一项重要任务是通过调用`kvmmap`函数，将I/O设备映射到内核的地址空间。这些映射主要是将物理地址映射到相同的虚拟地址，以便内核可以直接访问这些I/O设备。
   
   例如，UART0设备被映射到内核地址空间中的0x10000000。
   
   - **代码片段**：
     ```c
     kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);
     
     // void kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
     ```
   
   ![image-20240817213242660]({{ site.baseurl }}/docs/assets/image-20240817213242660.png)

> > `kvmmap`通常用于在内核的虚拟地址空间中创建一个映射，将虚拟地址映射到物理地址。这个函数的具体实现可能会根据操作系统的设计有所不同，但它通常用于设置页表项，以便将虚拟地址与物理地址关联起来。
> >
> > 在修改后的版本中，`kvmmap` 函数增加了一个新的参数 `pagetable_t kpgtbl`。我们来逐一解释每个参数及其意义，并分析这个新增参数的作用。
> >
> > ### 函数签名及其参数
> >
> > ```c
> > void kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
> > ```
> >
> > - **`pagetable_t kpgtbl`**：新增的参数，表示内核的页表（kernel page table）。在之前的版本中，可能隐含地使用了某个全局或默认的页表。现在这个参数明确传递了一个页表指针，使得调用者可以指定要操作的页表。这为页表管理提供了更多的灵活性，尤其是在初始化或处理多个页表时，调用者可以控制不同的页表。
> > - **`uint64 va`**：虚拟地址（virtual address），表示在虚拟地址空间中的起始地址。在此函数中，`va` 是需要映射到物理地址空间的虚拟地址。
> >   - 在这个上下文中，`UART0` 通常是一个宏或常量，定义了串口（UART）设备的物理地址。在许多操作系统（例如XV6）中，`UART0` 指的是第一个串口设备的基地址。
> > - **`uint64 pa`**：物理地址（physical address），表示要映射到的物理地址。该参数决定了虚拟地址 `va` 将指向的实际物理内存地址 `pa`。
> >   - 在这个上下文中，`UART0` 是指要映射到的物理地址。在这个例子中，虚拟地址 `UART0` 被映射到同一个物理地址 `UART0`。这种映射通常用于设备内存或I/O内存的访问，其中内核需要直接访问硬件设备。
> > - **`uint64 sz`**：映射的大小（size），表示需要映射的内存区域的大小。通常 `sz` 会以页大小为单位进行分配，`kvmmap` 会根据 `sz` 的值设置从虚拟地址 `va` 到物理地址 `pa` 的映射。
> >   - 在这个例子中，`PGSIZE` 通常是一个宏，定义了页的大小（通常是 4096 字节，也就是 4KB）。在这个调用中，它表示将从 `UART0` 开始的内存区域（大小为 `PGSIZE` 字节）映射到虚拟地址空间中。
> > - **`int perm`**：权限（permissions），控制页表项的读、写、执行权限。这通常包括读取权限（`PTE_R`）、写入权限（`PTE_W`）和执行权限（`PTE_X`）。这些标志控制了映射区域的访问权限。
> >   - 在这个调用中，`PTE_R | PTE_W` 的组合表示这个映射的内存区域是可读和可写的。这意味着内核在访问这个虚拟地址时，可以读写对应的物理地址。
> >
> > ### 函数行为
> >
> > ```c
> > if(mappages(kpgtbl, va, sz, pa, perm) != 0)
> >   panic("kvmmap");
> > ```
> >
> > `kvmmap` 函数调用了 `mappages` 函数，它负责实际在页表中设置从虚拟地址 `va` 到物理地址 `pa` 的映射，大小为 `sz`，并应用权限 `perm`。
> >
> > - **`mappages(kpgtbl, va, sz, pa, perm)`**：`mappages` 函数执行将 `va` 到 `pa` 的映射，并将其插入到指定的页表 `kpgtbl` 中。如果映射失败，它返回非零值。
> >
> > - **错误处理**：如果 `mappages` 返回非零值，意味着映射失败，`kvmmap` 会触发 `panic("kvmmap")`，导致系统崩溃。`panic` 通常用于操作系统开发中的严重错误处理，它表示程序遇到了无法继续的错误。
> >
> > > ### 新增参数 `pagetable_t kpgtbl` 的意义
> > >
> > > 这个参数是对内核页表的一个显式引用。之前的版本可能是默认操作某个全局的或静态的页表，但这种方式缺少灵活性。在新的版本中，传递 `kpgtbl` 作为参数，允许：
> > >
> > > 1. **指定不同的页表**：调用者可以传入不同的页表，以便将虚拟地址映射到不同的物理地址空间。这样可以支持多个页表的管理，比如在多进程、多核处理的操作系统中，可能需要为不同的内核或虚拟机创建单独的页表。
> > >
> > > 2. **更灵活的页表管理**：通过指定 `kpgtbl`，可以在不同场景下（如系统启动、进程切换）映射虚拟地址，而不局限于操作某个固定的内核页表。这在需要动态管理多个页表时非常有用。
> >
> > `  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);` 这个调用在内核中创建了一个虚拟地址到物理地址的映射，将 `UART0` 地址映射到自身，并且设置为大小为 `PGSIZE` 字节的内存区域，并且该区域具有读写权限。这通常用于让内核能够访问串口设备或其他I/O设备。
> >
> > 

#### 查看`memlayout.h`文件

为了更好地理解`kvminit`中的`kvmmap`函数调用，我们需要查看`memlayout.h`文件。这是一个重要的头文件，定义了内核所需的内存布局和常量值。

```c
// Physical memory layout

// qemu -machine virt is set up like this,
// based on qemu's hw/riscv/virt.c:
//
// 00001000 -- boot ROM, provided by qemu
// 02000000 -- CLINT
// 0C000000 -- PLIC
// 10000000 -- uart0 
// 10001000 -- virtio disk 
// 80000000 -- boot ROM jumps here in machine mode
//             -kernel loads the kernel here
// unused RAM after 80000000.

// the kernel uses physical memory thus:
// 80000000 -- entry.S, then kernel text and data
// end -- start of kernel page allocation area
// PHYSTOP -- end RAM used by the kernel

// qemu puts UART registers here in physical memory.
#define UART0 0x10000000L
#define UART0_IRQ 10

// virtio mmio interface
#define VIRTIO0 0x10001000
#define VIRTIO0_IRQ 1

// qemu puts platform-level interrupt controller (PLIC) here.
#define PLIC 0x0c000000L
#define PLIC_PRIORITY (PLIC + 0x0)
#define PLIC_PENDING (PLIC + 0x1000)
#define PLIC_SENABLE(hart) (PLIC + 0x2080 + (hart)*0x100)
#define PLIC_SPRIORITY(hart) (PLIC + 0x201000 + (hart)*0x2000)
#define PLIC_SCLAIM(hart) (PLIC + 0x201004 + (hart)*0x2000)

// the kernel expects there to be RAM
// for use by the kernel and user pages
// from physical address 0x80000000 to PHYSTOP.
#define KERNBASE 0x80000000L
#define PHYSTOP (KERNBASE + 128*1024*1024)

...
```

在`memlayout.h`中，我们可以看到将文档中的物理地址（如UART0的地址0x10000000）翻译成了代码中的常量。

```c
#define UART0 0x10000000
```

这个文件将所有关键的物理地址都定义为常量，这样在内核代码中，我们可以方便地引用这些设备的地址。

通过这部分的分析，我们清楚地了解了`kvminit`函数是如何初始化内核地址空间的，以及如何通过`kvmmap`函数将物理地址映射到虚拟地址。这些操作对于内核的正常运行至关重要，因为它们确保了内核能够正确地访问物理内存和I/O设备。

这种代码与硬件的紧密结合展示了操作系统设计的核心理念之一：操作系统需要在软件层面上对硬件资源进行有效的管理和抽象，而这些管理和抽象的实现往往通过如上所述的页表设置与内存映射等方式得以实现。

## Page Table 实验与 Kernel Page Table 的验证

在XV6的Page Table实验中，第一个练习是实现`vmprint`函数，该函数的作用是打印当前的kernel page table。接下来，我们将跳过`vmprint`的具体实现，直接查看在执行完第一个`kvmmap`后的kernel page table的状态。

### Kernel Page Table 的输出与验证

通过在代码中调用`kvmmap`函数，XV6会逐步完成对设备和内存的映射。此时，可以通过插入`vmprint()`打印kernel page table来观察内核地址空间的配置情况。

![image-20240817213937666]({{ site.baseurl }}/docs/assets/image-20240817213937666.png)

```c
  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  vmprint(kpgtbl);
```

当我们查看kernel page table的输出时，可以看到以下信息：

1. **Page Directory 层次结构**：
   - **第一行**：打印的是最高一级page directory的物理地址，该地址会存储在SATP寄存器中，代表内核页表的根。
   - **第二行**：显示最高一级page directory的第一个PTE（序号为0），它指向了中间级page directory的物理地址。
   - **第三行**：显示中间级page directory中的一个PTE（序号为128），指向最低级page directory的物理地址。
   - **第四行**：最低级page directory中的PTE指向实际的物理内存地址，例如UART0的物理地址`0x10000000`。

   这段输出中的PTE结构验证了之前的解释，即三级页表结构如何分层管理虚拟地址到物理地址的映射。

2. **虚拟地址到物理地址的验证**：
   - 我们可以通过位移操作将虚拟地址转换成用于索引page directory的索引值。具体而言，将虚拟地址`0x10000000`右移12位，可以得到高27位的index部分，再将这部分右移9位，得到128，这对应于中间级page directory中的序号。


> > 在位移操作中，首先将虚拟地址右移12位，这样可以去掉页内偏移部分，剩下的就是用于索引页表的高27位（即3个9位）。
> >
> > ```gdb
> > (gdb) p /x (0x10000000 >> 12)
> > $1 = 0x10000
> > ```
> >
> > 上面命令的结果`0x10000`是27位的高位部分，对应的是`L2 + L1 + L0`的组合。
> >
> > 接下来，再右移9位，将L0部分去掉，得到的就是`L2 + L1`索引部分。
> >
> > ```gdb
> > (gdb) p /x (0x10000 >> 9)
> > $2 = 0x80
> > ```
> >
> > 在这个例子中，由于虚拟地址中的最高9位（即L2的索引）是0，所以当我们将地址右移9位后，L2部分就被移除了，剩下的就是L1的索引部分。因此，虽然我们在位移操作中处理的是L2和L1的组合，但因为L2是0，实际得到的就是L1的索引值，也就是128。
> >
> > 这种方式让我们能够准确地索引到中间级的页表条目 (PTE)，进而找到下一步所需的物理页表地址。
> >
> > 如果L2索引不为0，那么我们在右移9位后，还会保留部分L2的值，因此需要进一步操作来提取L1的值。所以在你的例子中，因为L2索引为0，所以处理起来相对简单，直接得到了L1的索引。

从这些验证中，我们可以看到kernel page table的设置是符合预期的。

### 标志位的解析

在最低级page directory中的PTE标志位（`fl`部分）包含了读写标志位，并且Valid标志位也被设置。这表明此PTE可以被有效使用来翻译虚拟地址到物理地址。

![image-20240817215625782]({{ site.baseurl }}/docs/assets/image-20240817215625782.png)

- **Valid 位**：表示此条PTE有效，可以用来进行地址翻译。
- **读写标志位**：表明该内存区域可以进行读写操作。

### 内核地址空间的进一步设置

内核会持续调用`kvmmap`函数来设置整个内核地址空间的映射。这包括了对多个关键设备和内存区域的映射，例如：

- **VIRTIO0**：用于磁盘访问。
- **CLINT**：用于定时器和软件中断。
- **PLIC**：用于外部中断。
- **Kernel Text**：只读的内核代码区域。
- **Kernel Data**：可读写的内核数据区域。
- **TRAMPOLINE**：用于在用户空间和内核空间之间切换。

最后，通过调用`vmprint`函数，我们可以看到完整的kernel page directory。此时，多个PTE已经设置好，它们构成了内核地址空间的映射关系。

![vmprint 输出示意](https://user-images.githubusercontent.com/70982267/124379737-f491c480-dcc8-11eb-85be-2a7a59f4059c.png)

映射的正确性与安全性。这一过程不仅展示了XV6中的内存管理机制，也揭示了操作系统如何有效利用硬件提供的资源来确保系统的稳定与高效运行。

---

这个代码段展示了XV6内核如何设置和初始化内核页表 (`kernel_pagetable`)。它包括两个主要函数：`kvmmake` 和 `kvminit`。下面是对每一部分代码的详细讲解：

### 代码概述

- **`kernel_pagetable`**：这是一个全局变量，表示内核的页表。所有内核空间的虚拟地址翻译都会使用这个页表。
  
- **`etext[]`** 和 **`trampoline[]`**：这两个是外部声明的符号，分别表示内核代码的结束位置（由链接脚本`kernel.ld`设置）和用于处理陷阱（trap）的汇编代码段的起始位置。

### `kvmmake` 函数

**作用**：`kvmmake` 函数用于创建并初始化一个直接映射（Direct-map）的内核页表。

#### 1. 分配和清零页表
```c
kpgtbl = (pagetable_t) kalloc();
memset(kpgtbl, 0, PGSIZE);
```
- **`kalloc()`**：这是一个内存分配函数，返回一个物理页面的地址。这里它分配了一个新的页表。
- **`memset`**：将分配的内存清零，为后续的映射初始化一个空的页表。

#### 2. 映射硬件设备的物理地址
```c
// uart registers
kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

// virtio mmio disk interface
kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

// PLIC
kvmmap(kpgtbl, PLIC, PLIC, 0x4000000, PTE_R | PTE_W);
```
- **`kvmmap`**：这个函数将物理地址映射到虚拟地址空间。  
  - **UART0**: 映射UART0的寄存器，大小为一个页面，权限为读写。
  - **VIRTIO0**: 映射VirtIO磁盘的MMIO接口，大小为一个页面，权限为读写。
  - **PLIC**: 映射PLIC（平台级中断控制器），大小为`0x4000000`字节，权限为读写。

#### 3. 映射内核代码和数据
```c
// map kernel text executable and read-only.
kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

// map kernel data and the physical RAM we'll make use of.
kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
```
- **内核代码（text）**：
  - 映射从`KERNBASE`（内核基地址）开始，到`etext`结束的地址区间。这个区域存储的是内核代码，设置为只读且可执行（`PTE_R | PTE_X`）。
- **内核数据**：
  - 映射`etext`之后的内存区域到`PHYSTOP`，这是内核数据和可用物理内存的部分，设置为可读写（`PTE_R | PTE_W`）。

#### 4. 映射陷阱处理代码（Trampoline）
```c
// map the trampoline for trap entry/exit to the highest virtual address in the kernel.
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
```
- **Trampoline**: 映射一个用于处理陷阱（trap entry/exit）的小段代码，设置为只读和可执行。这个区域映射到了内核的最高虚拟地址。

#### 5. 为每个进程分配和映射内核栈
```c
proc_mapstacks(kpgtbl);
```
- **`proc_mapstacks`**：这个函数为每个进程分配并映射一个内核栈。栈的映射对于每个进程来说是独立的，以防止进程之间的干扰。

#### 6. 返回内核页表
```c
return kpgtbl;
```
- **`kvmmake`** 函数最终返回构建完成的内核页表。

### `kvminit` 函数

```c
void
kvminit(void)
{
  kernel_pagetable = kvmmake();
}
```
- **`kvminit`** 是一个简单的初始化函数，用来创建内核页表。它调用了`kvmmake`函数来生成内核页表，并将结果存储在全局变量`kernel_pagetable`中，供内核后续使用。

### 总结

这个代码段通过`kvmmake`函数构建了一个完整的内核页表，确保了内核能够正确映射设备和内存地址，同时也为每个进程创建了独立的内核栈。`kvminit` 函数只是简单地调用`kvmmake`来初始化内核页表。
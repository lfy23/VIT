# 一.NPUCore学习笔记

视频：Desktop 2024.12.25 - 21.04.34.05.mp4
链接: https://pan.baidu.com/s/1_f-3UYwBw9VGkorpgbtwdg?pwd=fmpq 提取码: fmpq

## 本地运行 NPUcore

### 环境配置

- **Rust 语言**：使用 Rust 编写，需安装 Rust 编译器。
- **RISC-V 编译器**：需要 RISC-V 目标的编译器。
- **Make 工具**：使用 `make` 辅助编译。
- **QEMU**：在 QEMU 上模拟运行，需安装 QEMU 虚拟机软件。

### 安装依赖

```bash
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

### 安装 Rust 工具

```bash
curl -sSf https://sh.rustup.rs | sh
sudo apt-get install curl
```

### 配置 Rust 环境

```bash
export RUSTUP_DIST_SERVER=https://mirrors.tuna.edu.cn/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.tuna.edu.cn/rustup/rustup
```

### 安装 Rust 相关软件包

```bash
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
rustup component add llvm-tools-preview
rustup component add rust-src
```

### 编译和运行

- 在 `os` 目录下执行 `make run` 运行 NPUcore。

## 基于 GDB 内核调试

### GDB 启动

```bash
sudo apt-get install gdb-multiarch
gdb-multiarch
```

### 调试命令

- `break <function_name `：设置断点。
- `continue`：继续执行直到下一个断点。
- `next`：单步执行，不进入函数内部。
- `step`：单步执行，进入函数内部。
- `finish`：执行完当前函数。
- `backtrace`：显示函数调用栈。
- `print <variable `：打印变量值。

### 自定义 GDB 命令

- 创建 `mycommands.gdb` 文件，添加自定义命令。
- 使用 `source mycommands.gdb` 加载自定义命令。

## 编写系统调用

### fork 系统调用

#### 功能

`fork` 系统调用用于创建一个新的进程，这个新进程是当前进程的副本。它是 Linux 系统中创建进程的基础。

#### 实现

- `fork` 调用通过执行 `ecall` 指令从用户态陷入内核态。
- 在内核中，`fork` 通过调用 `clone` 系统调用来创建一个新的进程。
- `clone` 系统调用需要一些参数，这些参数通过 `syscall` 函数传递。

#### 代码分析

- `fork` 函数在用户库中定义，其原型允许用户程序直接调用。
- `sys_fork` 函数封装了 `clone` 系统调用，传递 `SIGCHLD` 标志和其他参数。

#### 进程创建过程

1. 父进程调用 `fork`。
2. 内核创建一个新的进程，复制父进程的资源。
3. 新进程（子进程）从 `fork` 返回 0。
4. 父进程从 `fork` 返回子进程的 PID。

#### 并发和协作

- `fork` 创建的每个进程都独立并发执行。
- 使用 `wait` 函数等待子进程结束，防止僵尸进程产生。

### exec 系统调用

#### 功能

`exec` 系统调用用于在当前进程的上下文中执行一个新的程序。

#### 实现

- `exec` 调用替换当前进程的地址空间和资源，而不是创建一个新的进程。

#### 代码分析

- `exec` 函数使用路径和环境变量作为参数。
- 如果 `fork` 返回 0（子进程），则调用 `exec` 来加载新的程序。
- 如果 `fork` 返回非 0（父进程），则进入循环等待子进程结束。

#### 进程替换过程

1. 子进程调用 `exec` 替换其代码和数据。
2. 父进程使用 `wait` 监控子进程状态。

### sbrk 系统调用

#### 功能

`sbrk` 系统调用用于改变进程的堆大小，可以扩大或缩小。

#### 实现

- `sbrk` 通过移动堆顶指针来调整堆空间。

#### 代码分析

- `sbrk` 函数测试了堆空间的扩大和缩小。
- 如果请求的堆空间过大或过小，`sbrk` 不改变原来的堆空间。

#### 堆空间调整过程

1. 调用 `sbrk` 增加或减少堆空间。
2. `sbrk` 根据请求调整堆顶指针。

### 系统调用机制

#### 中断与系统调用

- **中断**：异步事件，可打断程序执行，分为硬件中断和软件中断（系统调用）。
- **系统调用**：应用程序请求操作系统服务的机制，通过陷入内核态实现。

#### RISC-V 系统调用接口

- RISC-V 使用 **SBI (Supervisor Binary Interface)** 作为系统调用接口，提供标准系统调用函数，如控制台输出、内存分配等。

#### 中断处理机制

- 使用 **TVEC (Trap Vector Base Address)** 寄存器指定中断向量表基地址。
- 中断向量表存储中断处理程序入口地址，CPU自动跳转至相应处理程序。

### NPUcore 系统调用实现

#### Trap 及中断使能

- **Trap 类型**：包括系统调用、外设中断处理、运行时意外。
- **控制寄存器**：内核通过写入控制寄存器来处理陷阱，读取寄存器明确陷阱原因。

#### 系统调用与 `ecall` 指令

- **`ecall` 指令**：触发系统调用，跳转至 STVEC 寄存器指定的处理程序。
- **参数传递**：系统调用时，用户态上下文保存，内核态执行操作后返回结果。

#### 设置 `stvec` 寄存器与 `trap_handle` 函数

- **`stvec` 寄存器**：存储中断向量表基地址，指定中断模式。
- **`trap_handle` 函数**：处理 Trap，分发系统调用，恢复上下文。

#### 上下文保存与恢复

- **汇编实现**：通过汇编代码保存和恢复 Trap 上下文。
- **`__alltraps` 与 `__restore`**：标记 Trap 上下文保存和恢复的汇编代码。

## 内存管理

### 内存管理架构

#### 内存管理代码树解析

NPUcore 的内存管理模块代码结构如下：

- **memory_distribution**：模块包含内存分布和管理的逻辑图，用于展示内存布局。
- **address.rs**：负责处理地址转换逻辑，包括虚拟地址到物理地址的转换。
- **frame_allocator.rs**：内存帧分配器，管理内存页框的分配和回收。
- **heap_allocator.rs**：堆内存分配器，负责用户态程序的动态内存分配。
- **memory_set.rs**：管理内存页集合，处理内存页的分配和回收逻辑。
- **mod.rs**：内存管理模块的入口文件，组织和协调其他模块。
- **page_table.rs**：页表管理模块，负责维护虚拟地址到物理地址的映射。
- **zram.rs**：实现压缩内存（ZRAM）功能，用于内存的压缩存储。

#### RISC-V 页表机制

- 虚拟地址与物理地址：
  - NPUcore 采用4KiB大小的页。
  - 虚拟地址分为页内偏移（低12位）和虚拟页号（VPN，高27位）。
  - 物理地址分为页内偏移（低12位）和物理页号（PPN，高44位）。
- 地址数据结构抽象：
  - 定义了`PhysAddr`、`VirtAddr`、`PhysPageNum`、`VirtPageNum`类型，封装`usize`类型，提供类型安全的地址转换。
  - 实现了`From`和`Into` trait，方便地址类型之间的转换。

#### 页表项（Page Table Entry, PTE）

- 页表项格式及含义：
  - 页表项是一个8字节的结构，包含物理页号（44位）和标志位（10位）。
  - 标志位包括有效位（V）、读写执行权限（R/W/X）、用户模式访问权限（U）等。
- 页表项的数据结构抽象：
  - 使用`bitflags!`宏定义`PTEFlags`，封装标志位。
  - 定义`PageTableEntry`结构体，包含页表项的位字段。

#### 页表结构

- SV39 三级页表结构：
  - 页表以三级树结构组织，减少页表本身占用的空间。
  - 每个页表存储512个页表项，对应512个虚拟页号。
- 页表的数据结构抽象：
  - 定义`PageTable`结构体，包含根页表的物理页号和关联的物理帧跟踪器。

#### 虚实地址映射过程

- satp 寄存器：
  - 控制分页模式，存储当前地址空间的根页表物理页号。
  - MODE字段控制分页模式，PPN字段存储根页表的物理页号。
- 三级页表的使用流程：
  - 通过虚拟页号的三个9位分段（VPN1, VPN2, VPN3）在三级页表中查询对应的页表项。
  - 最终从叶子节点的页表项中获取物理页号，并拼接虚拟地址的页内偏移，形成物理地址。

#### TLB（Translation Lookaside Buffer）

- TLB 概念：
  - TLB是页表的缓存，存储最近使用的页表项，减少虚实地址转换的内存访问次数。
  - TLB利用时间局部性，加速地址转换过程，提高系统执行效率。

### 内核地址空间

#### 内核虚拟地址空间分布

内核空间被划分为多个不同的区域，包括代码段、数据段、堆栈段、全局变量段、中断向量表和内核引导段。NPUcore 的内核虚拟地址空间分布图包括跳板页面、用户栈区域、保护页面、内核程序区域和临时存储区域。

- **跳板页面 (_trampoline)**：用于内核 panic。
- **用户栈区域 (user stack)**：每个栈大小为1 page。
- **保护页面 (guard page)**：防止内核栈溢出。
- **内核程序区域 (kernel program)**：包括.text、.rodata、.data、.bss段。
- **临时存储区域 (temporary storage area)**：用于exec系统调用。

#### 内核物理地址空间分布

内核物理地址空间分布涉及内核程序区域和内存映射空间（MMIO）。内核程序区域使用恒等映射，使得内核可以访问其数据段之外的物理页帧。MMIO 通过将外围设备映射到内存空间，便于CPU访问。

#### 地址空间实现

地址空间由页表和内存映射区域构成，其中页表可以直接创建为 PageTable 对象，内存映射区域需要使用一个 MapArea 的向量来表示。MemorySet 结构包含页表和映射区域。MapArea 结构表示内存映射区域，包含映射范围和权限。

- **MemorySet**：表示地址空间，由页表和内存映射区域构成。
- **MapArea**：表示内存映射区域，包含映射范围和权限。

地址空间创建和管理涉及创建空地址空间、创建内核地址空间、映射跳板区域、将MapArea添加到MemorySet、映射单个页面和无检查地映射页面等操作。内存映射宏 anonymous_identical_map 用于恒等映射内存区域。

### 物理内存分配

#### 物理内存分配的重要性

物理内存分配器负责动态地分配和释放物理内存资源，确保程序运行时能够获得足够的物理内存资源，并提高系统的稳定性和性能。

#### 物理内存空间

在 NPUcore 中，不同平台上的物理内存空间范围是预定义的，以确保内存分配器清楚待分配的物理空间范围。

- **MEMORY_START**：物理内存的起始地址，所有平台均为 `0x8000_0000`。

- MEMORY_END

  ：

  - k210：`0x8080_0000`，可用内存大小为 8MiB。
  - fu740：`0x9000_0000`，可用内存大小为 256MiB。
  - qemu：`0x809e_0000`，可用内存大小将近 10MiB。

- **PAGE_SIZE**：页面大小，所有平台均为 `0x1000`（4096 字节）。

#### 物理内存分配器初始化

物理内存分配器的初始化是内存管理中的重要步骤，确保内存分配器了解待分配的物理空间范围。

- `init_frame_allocator` 函数使用物理内存空间初始化内存分配器，确保从 `MEMORY_START` 到 `MEMORY_END` 的空间可以被分配使用。

#### 物理内存分配器接口

物理内存分配器提供以下接口：

- `new()`：创建一个新的分配器实例。
- `alloc()`：分配一个初始化的物理页面。
- `alloc_uninit()`：分配一个未初始化的物理页面，提高性能。
- `dealloc()`：释放一个物理页面。

#### RAII 与 FrameTracker

`FrameTracker` 结构体表示单个物理内存帧的状态，包括物理页号和操作方法。它实现了 RAII 特性，确保物理页面在申请时分配，在进程结束时释放。

#### 全局物理内存分配器

全局物理内存分配器是内核其他模块申请物理内存的统一方法，它通过 `FRAME_ALLOCATOR` 全局实例实现。

- `FRAME_ALLOCATOR`：使用 `RwLock` 包裹的 `StackFrameAllocator` 实例，提供线程安全的访问。
- `frame_alloc()` 和 `frame_alloc_uninit()`：分配物理页面的接口，返回 `Arc<FrameTracker ` 实例。
- `frame_dealloc()`：释放物理页面的接口。

#### 内存分配机制

物理内存分配器内部使用栈式内存管理策略，遵循后进先出的原则，实现物理内存的分配和回收。

- `StackFrameAllocator`：记录空闲的物理页面，并实现 `FrameAllocator` 接口。
- `alloc()` 和 `dealloc()`：实现物理页帧的分配和回收。
- `recycled`：存储可供重新分配的物理页面。

通过全局物理内存分配器的包装，内核其他模块申请物理页面时返回 `FrameTracker`，当 `FrameTracker` 生命周期结束，物理页面自动回收，体现 RAII 思想。

## 进程管理

### 进程管理代码树

NPUcore 的进程管理模块代码结构如下：

- **context.rs**：存储上下文信息。
- **elf.rs**：ELF文件加载和解析。
- **manager.rs**：任务管理器的结构存储。
- **mod.rs**：操作系统任务管理和调度的实现。
- **pid.rs**：实现内核栈、进程分配器、内核栈分配器等。
- **processor.rs**：进程轮询的主体和processor的声明与实现。
- **signal.rs**：进程的信号量信息。
- **switch.rs**：调用switch汇编函数的接口，实现任务上下文切换。
- **switch.S**：任务上下文切换的汇编实现。
- **threads.rs**：实现用户空间多线程快速互斥锁（Futex）。
- **task.rs**：TCB模块声明以及方法实现，rusage信号量声明。

#### 进程生命周期

进程生命周期包括创建、就绪、阻塞、运行和退出。进程创建后进入就绪队列，被调度后运行，时间片耗尽或主动让出CPU时回到就绪队列，执行等待操作如wait时进入阻塞队列，执行结束后退出并释放资源。

NPUcore 支持阻塞式进程调度模式，以减少IO操作导致的CPU挂起性能损失。阻塞和非阻塞IO是访问设备的两种模式，阻塞操作会导致进程挂起直到条件满足，非阻塞操作则不断查询直到可以进行操作。

#### 资源复用

为实现多进程同时运行，操作系统需要对CPU、内存和外设等资源进行复用。NPUcore 实现了进程的创建、资源分配和调度机制，避免了进程饥饿和资源永久占用。

- **上下文切换**：使用共享跳板代码和进程私有的保存现场帧实现。
- **透明调度**：通过内置计时器中断调用schedule函数实现。
- **资源回收**：进程退出后释放部分资源，剩余资源由父进程回收。
- **进程唤醒**：单核操作系统保证CPU不会调度其他进程，确保不错过进程唤醒。

### TCB

NPUcore 使用TCB（Task Control Block）数据结构管理进程，不同于传统的PCB（Process Control Block），TCB将线程视为共享栈的进程。TCB包含进程号、线程号、线程组号、内核栈、用户栈、退出信号和共享资源等信息。

进程控制块（PCB）是操作系统中用于描述和管理进程的数据结构。在NPUcore中，TCB被用作PCB，记录了操作系统所需的全部信息，包括描述进程的当前状态以及管理进程运行的全部信息。

TCB由以下几部分组成：

- 不可变部分：包括进程号（pid）、内核栈（kstack）、用户栈基地址（ustack_base）和退出信号（exit_signal）。
- 可变部分：包括信号掩码（sigmask）、待处理信号（sigpending）、Trap上下文物理页号（trap_cx_ppn）和任务状态（task_status）等。
- 共享且可变部分：包括执行文件描述符（exe）、线程号分配器（tid_allocator）、文件描述符表（files）、文件系统状态（fs）、虚拟内存集（vm）、信号处理（sighand）和Futex。

TCB的主要作用包括：

1. 作为独立运行基本单位的标志。
2. 实现间断性运行方式。
3. 提供进程管理所需的信息。
4. 提供进程调度所需的信息。
5. 实现与其他进程的同步和通信。

进程生命周期包括创建、就绪、阻塞、运行和退出。NPUcore通过调度器支持阻塞式进程调度模式，以减少IO操作导致的CPU挂起性能损失。

NPUcore实现多进程同时运行，对CPU、内存和外设等资源进行复用。这包括上下文切换、透明调度、进程资源回收和并发下的进程唤醒。

NPUcore中的进程创建涉及从系统文件中加载ELF文件，获取数据和内容，然后调用TCB中的new()方法创建内核的第一个进程initproc。其余进程均由initproc通过fork系统调用创建。

NPUcore采用时间片轮转调度策略，支持阻塞式的进程调度模式。进程调度涉及进程状态管理、任务切换和调度器的选择。

进程完成工作后，通过调用exit系统调用来结束生命周期。子进程退出后变为僵尸进程，等待父进程通过waitpid系统调用回收资源。

NPUcore支持kill系统调用，允许进程发送信号给其他进程，实现进程间的通信和控制。

## 文件系统管理

NPUcore 文件系统的代码结构如下，涉及文件系统的各个核心组件：

- `cache.rs`：缓存管理，负责文件系统的缓存机制，提高文件访问效率。
- `dev`：设备驱动模块，包含文件系统所需的底层设备驱动。
- `directory_tree.rs`：目录树管理，维护文件系统的目录结构。
- `fat32`：FAT32文件系统的具体实现。
- `file_trait.rs`：文件操作接口定义，规定了文件系统操作的标准接口。
- `filesystem.rs`：文件系统抽象层，提供文件系统的通用接口。
- `layout.rs`：定义文件系统中数据的组织结构。
- `mod.rs`：模块主入口，组织文件系统模块的加载和初始化。
- `poll.rs`：实现了文件描述符的轮询机制，用于文件I/O的事件通知。
- `swap.rs`：页面交换空间管理，用于虚拟内存管理中的页面交换。

### 文件系统概述

文件系统的核心目标是让应用能够方便地将数据持久保存。文件系统通过抽象持久存储设备的细节，为应用程序提供简单、高效的数据读写接口。

1. **实现文件访问相关的应用**：遵循Linux系统调用接口，实现简化版的文件系统模型。
2. **实现`easyfs`文件系统**：在用户态实现文件系统，并进行基本测试，验证其正确性后嵌入到操作系统内核中。
3. **将`easyfs`文件系统加入内核中**：实现块设备驱动程序，并扩展进程的管理范围，将文件纳入进程管理之中。

#### 块缓存层（Buffer Cache）

块缓存层是文件系统中提高性能和效率的关键组件，通过在内存中缓存磁盘块来减少对磁盘的直接访问，加速数据访问，并合并写操作。

NPUcore实现了块缓存管理，包括`Cache`和`CacheManager` trait，定义了缓存的基本操作，如`read`、`modify`和`sync`。此外，实现了`BlockCacheManager`和`PageCacheManager`来分别管理块缓存和页缓存。

#### 索引节点层

索引节点层负责文件的管理和读写控制逻辑，通过索引节点（Inode）来完成。Inode存放文件元数据信息，并指向文件内容。

##### Inode的数据结构

NPUcore中的Inode结构体包括inode锁、文件内容、文件缓存管理器、文件类型、父目录指针、文件系统指针和时间属性等。Inode提供了文件系统常用操作，如创建目录项、删除目录项、创建硬链接、删除硬链接和列出目录内容等。

##### NPUCore中的目录树数据结构及实现

NPUcore通过目录树结构实现文件的定位，使用`DirectoryTreeNode`和`OSInode`结构体来维护目录树和文件的索引节点。内核全局维护了一个目录节点向量`DIRECTORY_VEC`记录当前系统的目录。

#### 文件描述符层

文件描述符层管理进程访问的多个文件，每个进程都有一个文件描述符表，记录该进程请求内核打开并读写的文件集合。

##### 文件描述符表（Fdtable）

NPUcore使用文件描述符表`FdTable`来管理文件描述符，内部维护一个向量组，通过文件描述符（非负整数）来访问对应的文件记录信息。`FileDescriptor`结构体表示单个文件描述符，包含文件的读写属性和文件对象指针。

### 文件共享与 PIPE 机制

#### 软链接与硬链接

##### 硬链接实现

硬链接是文件系统中多个文件名指向同一个inode的情况，它们共享相同的内容和属性。在NPUcore中，硬链接的创建通过`sys_linkat`函数实现，该函数处理路径转换、文件描述符获取、以及链接创建等步骤。硬链接的删除则通过`sys_unlinkat`函数实现，涉及路径解析、inode节点管理和链接删除操作。

##### 软链接特性

软链接（符号链接）包含另一个文件或目录的路径信息，与原文件不共享inode。软链接提供了跨文件系统共享文件的灵活方式。由于NPUcore基于FAT32文件系统，不支持软链接，因此没有具体的实现代码。

#### 管道机制

##### 管道创建与使用

管道是进程间通信的基本机制，允许进程间直接传递数据。在NPUcore中，`sys_pipe2`函数用于创建管道，返回两个文件描述符，分别对应管道的读端和写端。管道的数据结构是一个环形队列，实现数据的流入和流出。

##### 管道操作流程

1. 父进程通过`pipe`系统调用创建管道，获取读端和写端文件描述符。
2. 父进程fork子进程，子进程继承相同的文件描述符。
3. 父进程关闭读端，子进程关闭写端，实现数据的单向传输。

##### 管道特殊情况处理

- 写端关闭后读端继续读取：管道中数据读完后，再次read返回0。
- 写端未关闭读端继续读取：管道中数据读完后，read操作阻塞直到有数据可读。
- 读端关闭写端写入：写入操作导致SIGPIPE信号，可被捕获避免进程终止。
- 读端未关闭写端写入：管道满时write操作阻塞直到有空位。

##### 管道关闭

管道关闭通过`sys_close`系统调用实现，关闭管道两端文件描述符，释放相关资源。当管道的读端和写端都被关闭后，管道的环形队列缓冲区被销毁，管道关闭。

# 二.适配评测

基于2022版本npucore进行修改

## 配置rust工具链

比赛环境采用的rustc版本为2024-02-03，与2022版npucore设置的指定版本不同，需要先将oskernel2022-npucore目录下的`rust-toolchain`中的内容修改为

  ```
  nightly-2024-02-03
  ```

以下是2024-02-03版本的rust的安装，如已经安装，请忽略

将以下配置添加到~/.bashrc中

  ```
  export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
  export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
  ```

假设你已经安装过其他版本的rust

在bash中输入

  ```bash
  rustup install nightly-2024-02-03
  ```

  安装完成之后可以使用

  ```bash
  rustup show
  ```

  查看已经安装的环境

  并可以通过以下命令将2024-02-03版本设置为默认使用版本

  ```bash
  rustup default nightly-2024-02-03
  ```

  安装完2024-02-03版本的rust之后需要进入到`oskernel2022-npucore/os/src`目录下修改`main.rs`文件

  将第5、6行的代码进行注释

  ```rust
  #![no_std]
  #![no_main]
  #![feature(panic_info_message)]
  #![feature(alloc_error_handler)]
  // #![feature(btree_drain_filter)]
  // #![feature(drain_filter)]
  #![feature(int_roundings)]
  #![feature(string_remove_matches)]xxxxxxxxxx #![no_std]#![no_main]#![feature(panic_info_message)]#![feature(alloc_error_handler)]// #![feature(btree_drain_filter)]// #![feature(drain_filter)]#![feature(int_roundings)]#![feature(string_remove_matches)]
  ```

## 加载bash与initproc

### 预加载initproc

首先在`oskernel2022-npucore/os/src`目录下新建一个`preload_app.S`的文件,并输入以下代码

```
# os\src\preload_app.S
.section .data
.global sinitproc
.global einitproc
.align 12
sinitproc:
.incbin "../user/target/riscv64gc-unknown-none-elf/release/initproc"
einitproc:
.align 12

.section .data
.global sbash
.global ebash
.align 12
sbash:
.incbin "../bash-5.1.16/bash"
ebash:
.align 12

```

然后在`oskernel2022-npucore/os/src/fs/mod.rs`文件中添加flush_preload()函数

```rust
pub fn flush_preload() {
extern "C" {
  fn sinitproc();
  fn einitproc();
  fn sbash();
  fn ebash();
}
println!(
  "sinitproc: {:X}, einitproc: {:X}, sbash: {:X}, ebash: {:X}",
  sinitproc as usize,
  einitproc as usize,
  sbash as usize,
  ebash as usize,
);
let initproc = ROOT_FD.open("initproc", OpenFlags::O_CREAT, false).unwrap();
initproc.write(None, unsafe {
  core::slice::from_raw_parts(
      sinitproc as *const u8,
      einitproc as usize - sinitproc as usize,
  )
});
for ppn in crate::mm::PPNRange::new(
  crate::mm::PhysAddr::from(sinitproc as usize).floor(),
  crate::mm::PhysAddr::from(einitproc as usize).floor(),
) {
  crate::mm::frame_dealloc(ppn);
}
let bash = ROOT_FD.open("bash", OpenFlags::O_CREAT, false).unwrap();
bash.write(None, unsafe {
  core::slice::from_raw_parts(sbash as *const u8, ebash as usize - sbash as usize)
});
for ppn in crate::mm::PPNRange::new(
  crate::mm::PhysAddr::from(sbash as usize).floor(),
  crate::mm::PhysAddr::from(ebash as usize).floor(),
) {
  crate::mm::frame_dealloc(ppn);
}
}
```

在`oskernel2022-npucore/os/src/main.rs`中，将其加入内核并在主函数中调用

```rust
core::arch::global_asm!(include_str!("preload_app.S"));
fs::flush_preload();
```

在`oskernel2022-npucore/os/src/mm/address.rs`的末尾添加

```rust
pub type PPNRange = SimpleRange<PhysPageNum>;
```

将`oskernel2022-npucore/os/src/mm/mod.rs`中的12行替换为

```rust
pub use address::{PhysAddr, PhysPageNum, StepByOne, VirtAddr, VirtPageNum,PPNRange};
```

### 修改调度表

进入到`oskernel2022-npucore/user/src/bin`目录下对`initproc.rs`文件进修改，将整个文件修改为以下代码:
通过`schedule`调度表，执行里面的内容，使用命令`run-all.sh`实现跑所有测例

```rust
#![no_std]
#![no_main]
use user_lib::{exec, exit, fork, wait, waitpid, yield_,shutdown};

#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
 exit(main());
}

#[no_mangle]
fn main() -> i32 {
 let bash_path = "/bash\0";
 let environ = [
     "SHELL=/bash\0".as_ptr(),
     "PWD=/\0".as_ptr(),
     "LOGNAME=root\0".as_ptr(),
     "MOTD_SHOWN=pam\0".as_ptr(),
     "HOME=/root\0".as_ptr(),
     "LANG=C.UTF-8\0".as_ptr(),
     "TERM=vt220\0".as_ptr(),
     "USER=root\0".as_ptr(),
     "SHLVL=0\0".as_ptr(),
     "OLDPWD=/root\0".as_ptr(),
     "PS1=\x1b[1m\x1b[32mNPUCore\x1b[0m:\x1b[1m\x1b[34m\\w\x1b[0m\\$ \0".as_ptr(),
     "_=/bin/bash\0".as_ptr(),
     "PATH=/:/bin\0".as_ptr(),
     "LD_LIBRARY_PATH=/\0".as_ptr(),
     core::ptr::null(),
 ];
 let schedule = [
     // [
     //     bash_path.as_ptr(),
     //     "-c\0".as_ptr(),
     //     "echo aaa > lat_sig\0".as_ptr(),
     //     core::ptr::null(),
     // ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./openat\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./dup2\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./pipe\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./getppid\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "brk\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./chdir\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "clone\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./close\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./dup\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./execve\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./exit\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./fork\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./fstat\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./getcwd\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./getdents\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./getpid\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./gettimeofday\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./mkdir_\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./mmap\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./mount\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./munmap\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./open\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./read\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./times\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./umount\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./uname\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./unlink\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./wait\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./waitpid\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./write\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./yield\0".as_ptr(),
         core::ptr::null(),
     ],
     [
         bash_path.as_ptr(),
         "-c\0".as_ptr(),
         "./sleep\0".as_ptr(),
         core::ptr::null(),
     ],
 ];
 let mut exit_code: i32 = 0;
 for argv in schedule {
     let pid = fork();
     if pid == 0 {
         exec(bash_path, &argv, &environ);
     } else {
         waitpid(pid as usize, &mut exit_code);
     }
 }
 // loop {
 //     wait(&mut exit_code);
 // }
 shutdown();
 0
}
```

### 添加shutdown系统调用

在`oskernel2022-npucore/user/src`目录下的`user_call.rs`的最后添加

```rust
pub fn shutdown() -> isize {
 sys_shutdown()
}
```

在`oskernel2022-npucore/user/src`中的`syscall.rs`的最后添加

```rust
pub fn sys_shutdown() -> isize {
 syscall(SYSCALL_SHUTDOWN, [0, 0, 0])
}
```

修改完成后在`oskernel2022-npucore/user`目录下执行命令

```bash
make clean
make build
```

若编译成功，说明操作正确

在`oskernel2022-npucore/os/src/syscall/`中的`mod.rs`中的第440行添加

```rust
     SYSCALL_SHUTDOWN => sys_shutdown(),
```

在同目录下的`process.rs`中添加

```rust
pub fn sys_shutdown() -> ! {
 shutdown()
}
```

修改`oskernel2022-npucore/os/`目录下的`Makefiles`文件的第15行为

```makefile
	U_FAT32 := ../sdcard.img
```

在该目录下执行以下命令即可

```bash
make run
```



###  对未通过的测例进一步修改

![image-20241225202025695](C:\Users\lfy\AppData\Roaming\Typora\typora-user-images\image-20241225202025695.png)

![image-20241225202035208](C:\Users\lfy\AppData\Roaming\Typora\typora-user-images\image-20241225202035208.png)

![image-20241225202045487](C:\Users\lfy\AppData\Roaming\Typora\typora-user-images\image-20241225202045487.png)

![image-20241225202056787](C:\Users\lfy\AppData\Roaming\Typora\typora-user-images\image-20241225202056787.png)

![image-20241225202105906](C:\Users\lfy\AppData\Roaming\Typora\typora-user-images\image-20241225202105906.png)

---
sort: 0
title: PMem相关专业术语
tag: pmem glossary
---

**PMem相关专业术语**

资料来源：[https://pmem.io/glossary/](https://pmem.io/glossary/)

您可以在此处找到与持久内存(PMem)相关的术语列表。其中许多术语具有广泛的含义，但以下定义特别关注它们与PMem的关系。

[TOC]

### 1LM

(One-level Memory)

​		**1LM**指的是内存没有使用任何内存端缓存来连接到系统的形式。**DRAM**或**system main memory**介绍时会提到，另外会使用**1LM**来明确说明相关内存未配置为[2LM](#2lm)。



### 2LM

(Two-level Memory)

​		持久内存有时配置为**2LM**，其中硬件管理两个级别（层）的内存。它是[Intel傲腾Optane PMem产品](#optane)的一个功能 ，该功能配置称为 [内存模式](#memory-mode)）。



### 3D XPoint

​		3D XPoint（读作*three dee cross point*）是 [Intel傲腾Optane](#optane)产品线中使用的介质，该介质分为固态磁盘(SSD) 产品和持久性内存(PMem) 产品。更多细节查看[英特尔的网站](http://intel.com/optane)、[维基百科](https://en.wikipedia.org/wiki/3D_XPoint)

<img src="./3dxpoint.jpeg" height="200px"/>



### ADR

(Asychronous DRAM Refresh)

​		**ADR**是一种硬件功能，可在断电时将存储数据从__内存控制器写入挂起队列 (WPQ) __刷到其目的地。ADR 也可以选择性地刷I/O控制器里pending的 DMA 存储数据。

<img src="./adr.jpeg" height="300px"/>

​		如上图所示，在通往持久内存 DIMM 的途中，存储数据可以驻留在多个位置。下面的红色虚线框显示了 ADR 域——到达该域的存储受到 ADR 的断电保护，ADR 会flushes内存控制器中的队列，如图中的梯形所示。所有支持持久内存的 Intel 系统都需要 ADR，这意味着必须在平台级别（包括 CPU、主板、电源、DIMM 和 BIOS）支持该功能。所有 [NVDIMM-N](#nvdimm)产品以及 Intel 的[傲腾Optane](#optane) PMem 都需要系统支持 ADR 。

​		ADR 使用储存的电能在断电后执行flushes。存储的电能通常来自电源中的电容器，但也可以通过其他方式实现，例如电池或 UPS。

​		图中较大的红色虚线框说明了一个可选功能 [eADR](#eadr)，其中 CPU 缓存也会在断电时flushes。



### App Direct

(Application Direct)

​		[Intel 傲腾Optane PMem](#optane)产品可以配置为两种模式：**App Direct**和[Memory Mode](#memory-mode)。有关产品详细信息，请参阅 [英特尔傲腾网站](http://intel.com/optane)。App Direct 模式提供 [持久内存编程模型](#programming-model)。

<img src="./app_direct.jpeg" height="300px"/>

​		以这种方式配置时，操作系统可以对持久化内存感知应用程序提供直接访问或[DAX](#dax)。这种方式允许应用程序像加载/存储内存一样访问持久化数据。



### ARS

(Address Range Scrub)

​		[NVDIMM](#nvdimm)产品可能提供一个接口，允许操作系统发现设备上的已知损坏的位置。 提前发现这些[bad blocks ](#bad-blocks)可以让应用程序避免使用损坏的位置，从而避免会杀死应用程序的相关异常。

​		ARS 接口通过[DSM](#dsm)提供。用于地址范围清理的 DSM 由 ACPI 规范描述，可在 [UEFI 网站上获得](https://uefi.org/)。

> 原文poison个人译为损坏，欢迎指正



### Bad Blocks

​		操作系统可以跟踪持久存储设备上的已知**Bad Blocks**列表。这些块可能会使用某些 NVDIMM 支持的地址范围扫描 ( [ARS](#ars) ) 操作来发现，或者当软件尝试使用某个位置但返回已损坏时被即时发现。

​		对于服务器上的常见的易失性内存 normal volatile memory （即 DRAM），不可纠正的错误将导致向使用它的应用程序返回一个*损坏*值。在 Intel 平台上，使用损坏值会导致*机器检查*，进而导致内核向使用的进程发送异常。在 Linux 上，异常采用 SIGBUS 信号的形式，大多数应用程序将因此挂掉。重新启动时，问题会消失，因为应用程序从头开始分配易失性内存，操作系统将隔离 sequestered 包含损坏的页面。

​		持久内存增加了这种情况的复杂性。同上面的 Linux 示例，应用使用了持久内存的损坏值导致相同的 SIGBUS，但是如果应用程序挂掉并重新启动，它很可能会读取相同的位置，因为程序可能存在持久性的场景——操作系统不能像易失性内存那样用新页面替换旧页面，因为它需要保留损坏来表示数据丢失。

​		ECC 信息用于检测不可纠正的错误，通常以缓存行形式维护，缓存行在 Intel 系统上为 64 字节。但是根据[blast-radius](#blast-radius)效应，这个小范围错误可以进制rounded up到更大的范围。

​		为了防止__感知PMem的应用程序__重复启动、使用损坏位置甚至挂掉的不当行为，操作系统为应用程序提供了一种访问已知**Bad Blocks**列表的方法。在 Linux 上， 也可以使用**ndctl**命令查看此信息：

```
# ndctl list --media-errors
```

​		**libndctl**提供的 API允许应用程序直接访问此信息，并且 PMDK 库使用该 API 来防止在 PMem 池包含已知**Bad Blocks**时被使用。在这种情况下，应用程序采取的常见操作是拒绝继续使用，迫使系统管理员从备份或冗余副本中恢复池数据。当然，应用程序可以尝试直接修复池，但这会导致应用程序逻辑更复杂。

​		[Intel 的 PMem RAS 页面](https://software.intel.com/content/www/us/en/develop/articles/pmem-RAS.html) 包含有关此主题的更多信息，重点介绍 傲腾Optane PMem 产品。



### Blast Radius

​		当[persistent-memory](#persistent-memory)某个位置遇到无法纠正的错误时，该数据就会丢失。Intel 平台上的内存以 64 字节缓存行的形式访问，但在某些情况下，丢失单个位置可能会导致应用程序丢失更大的数据块。这被称为**Blast Radius**效应。

<img src="./blast_radius.jpeg" height="300px"/>

​		如上图所示，包含64 字节损坏位置（例如不可纠正的错误）可能会使其损坏大小进制为该设备用作其*ECC 块大小的任何值*。Linux 操作系统可以通过[ARS](https://pmem.io/glossary/#ars)机制发现这些损坏位置。Linux 内核中使用 以512字节块的数据结构来追踪这些区域的[Bad Blocks](#bad-blocks)，因此损坏大小将进制到 512 字节。如果应用程序内存映射该文件，操作系统将在该位置映射一个无效page，再次将大小进制到更大的 4096 字节（Intel系统上的page大小）。



### Block Storage

(Storage, Disk)

​		将传统存储与持久内存进行比较时，主要区别如下：

- 块存储的接口是基于块的。软件只能读一个块或写一个块。文件系统通常根据需要通过将块[paging](#paging) 来存储或提取数据。

- PMem 的接口是字节寻址的。软件可以读取或写入任何大小的数据而无需分页。持久数据就地访问，减少了 DRAM 保存缓冲数据的需要。
- 移动块（例如 4k 数据）时，NVMe SSD 等块存储设备会启动 DMA 到内存。这允许 CPU 在等待数据传输时执行其他工作。
- PMem 作为[NVDIMM](#nvdimm)实现，连接到内存总线，无法启动 DMA。移动 4k 的数据，通常是 CPU 移动数据，这比存储设备的延迟低，但 CPU 利用率更高。对此的一种潜在解决方案是使用 DMA 引擎（如果平台提供）。
- PMem 可以模拟块存储设备，事实上这是 [PMem 编程模型的一部分](#programming-model)。但块存储无法模拟 PMem，因为它从根本上是不可字节寻址的。分页可以接近模拟 PMem，尤其是对于快速 SSD，但将修改数据刷新持久化仍需要执行具有块存储的内核代码，其中 PMem 可以直接从用户空间刷新持化（有关详细信息，请参阅[ADR](https://pmem.io/glossary/#adr) 和[eADR](#eadr)）。

​		另请参阅[BTT](#btt)以了解有关 PMem 如何模拟块存储的详细信息。



### BTT

(Block Translation Table)

​		**BTT**算法提供了断电时单块原子性写入持久化存储器顶部的能力。这允许 PMem 模拟存储并提供与 NVMe 类似的语义。NVMe 存储要求断电时至少以块形式原子性写入，这意味着在断电期间写入的块将被完全写入或根本不写入。由于软件可能依赖于这个存储属性，BTT 算法被设计为在软件中实现相同的语义。该算法被标准化为[UEFI 的](https://uefi.org/)一部分。

​		有关 BTT 工作原理的介绍，可简单参考[此博文](https://pmem.io/2014/09/23/btt.html)，此文是由实现 BTT 的 Linux 维护者 Vishal Verma 撰写。



### CLFLUSHOPT

(Instruction: Cache Line Flush, Optimized)

​		Intel 指令集长期以来一直包含一个缓存刷新指令 CLFLUSH，它将从 CPU 缓存中失效特定的缓存行。CLFLUSH 的定义早于 [persistent-memory](#persistent-memory)，其包含[fence](#fence)作为指令一部分。这意味着由于每次刷新之间的fence，用于刷新一系列缓存行的 一组 CLFLUSH 指令将被串行化。随着持久内存的出现，Intel 引入了**CLFLUSHOPT**指令，该指令通过去除嵌入式fence操作进行了优化。因此，一组 CLFLUSHOPT 指令启动刷新，将允许一定的并行度。这样的一组命令执行后要执行[fence](#fence)作为结束，确认在软件继续之前该范围内的存储数据已持久化。

​		CLFLUSHOPT 指令总是失效缓存行，这意味着对该地址的下一次访问将是 CPU 缓存未命中，即使它在刚刷新后不久发生。将此与[CLWB](#clwb) 指令进行比较，CLWB指令允许该缓存行保持有效。

<img src="./flush_isa.jpeg" height="300px"/>

> 原文evict个人译为失效，欢迎指正



### CLWB

(Instruction: Cache Line Write Back)

​		当平台需要时，**CLWB**指令是将 PMem 存储刷新持久化的首选方法。仅支持[ADR ](#adr)的平台就是这种情况。支持[eADR](#eadr)平台允许软件跳过 CLWB 指令以获得更好的性能。与 CLFLUSH 和[CLFLUSHOPT](#clflushopt)指令不同，CLWB 告诉 CPU 在写入任何脏数据后，希望在 CPU 缓存中保持缓存行有效。这为应用程序在刷新后不久再次访问该行的情况提供了更好的性能。

​		与[CLFLUSHOPT](https://pmem.io/glossary/#clflushopt)一样，CLWB 指令不影响包含的[fence](https://pmem.io/glossary/#fence)，因此在使用此指令刷新范围后，通常会发出 SFENCE 指令。

<img src="./flush_isa.jpeg" height="300px"/>



### CXL

(Compute Express Link)

​		引自[CXL 网站](https://www.computeexpresslink.org/)，Compute Express Link™ (CXL™) 是业界支持的用于处理器、内存扩展和加速器的缓存一致性互连。

​		对于[persistent-memory](https://pmem.io/glossary/#persistent-memory)，CXL 提供了一个新的附加点。2020 年 11 月发布的 CXL 2.0 规范包括对 PMem 的必要支持，包括对管理、配置、命名空间和区域标签的更改，以及类似于[eADR](#eadr)（在 CXL 术语中称为全局持久刷新的 GPF）的刷新失败机制 。

​		对于具有感知PMem的应用程序的程序员，CXL 最重要的方面是[PMem 编程模型](https://pmem.io/glossary/#programming-model)保持不变。为基于 NVDIMM 的 PMem 产品编写的程序无需修改即可在基于 CXL 的 PMem 产品上运行。

​		在*2021 PM + CS峰会*（2021年4月22日）包含这方面的[演讲](https://www.youtube.com/watch?v=x-OSKow9NM8)，简要概述了为支持PMEM，CXL所做的更改的。



### DAX

(Direct Access)

​		[Pmem编程模型](#programming-model)里规定，应用程序可以利用操作系统提供的标准内存映射文件API，直接映射永久存储器。这个功能，在系统中常被称为[mmap](#mmap)和[MapViewOfFile](#mapviewoffile)，其绕过page cache，在 Linux 和 Windows 中都被命名为**DAX**。DAX 是*direct access*缩写 ，是添加到操作系统以支持持久内存的关键功能。

​		<img src="./dax.jpeg" height="300px"/>

​		如上图所示，应用程序使用标准 API 打开文件，然后进行内存映射。 感知PMem的文件系统的工作是提供 DAX 映射，以便在应用程序访问内存范围时不会发生[paging](#paging)。

​		为了使 PMem 的存储持久化，可以使用刷新的标准 API（Linux 上的[msync](#msync)，或Windows上的[FlushViewOfFile](#flushviewoffile)）。编程模型还允许使用[CLWB ](#clwb)等指令直接将用户空间刷新持久化，但在 Linux 上仅在`mmap`使用`MAP_SYNC` 标志成功调用的情况下才允许。在 Windows 上，所有 DAX 映射都允许从用户空间刷新。

​		更多介绍查看[Linux doc on DAX](https://www.kernel.org/doc/Documentation/filesystems/dax.txt)。



### DDR

(Double Data Rate)

​		该**DDR**术语是各种版本的 DDR 协议的概括。[Wikipedia](https://en.wikipedia.org/wiki/DDR4_SDRAM)上讲解了DDR4 。

​		在谈论持久内存时，该术语用于谈论将哪些类型的内存插入到哪些类型的sockets中。例如，NVDIMM-N 产品通常像普通 DRAM 一样插入 DDR 插槽。英特尔的傲腾Optane PMem 可插入 DDR 插槽，但在 DDR 电子设备上运行[DDR-T](#ddrt)协议。



### DDR-T

​		**DDR-T**是在 Intel 平台上使用[傲腾Optane](#optane) PMem 产品插入系统的[DDR](#ddr)插槽的协议。



### Device DAX

(devdax)

​		Linux 支持[DAX](#dax)，它允许感知PMem文件系统在内存映射文件时让应用程序 *直接访问direct access* PMem。Linux 还通过称为**Device DAX**的配置在不使用文件系统的情况下支持**DAX**。Linux `ndctl`命令使用术语 `fsdax`和`devdax`在这两种类型的 DAX 之间进行选择，如 [ndctl-create-namespace](https://pmem.io/ndctl/ndctl-create-namespace.html) 手册页中所述。

​		在大多数情况下，可以像管理文件一样 PMem（包括命名、权限和 POSIX 文件 API），所以优先选择`fsdax`类型。`devdax`不遵循[PMem 编程模型](#programming-model)，因此普通文件API不能操作它。以下是 Linux 上` fsdax` 和`devdax` 之间的主要区别：

   - 在 fsdax 和 devdax 两种情况下，正常的 I/O 操作是加载load和存储store指令，它们允许直接从用户空间访问，数据操作不需要内核代码。

   - devdax 将整个命名空间 Namespace 作为单个设备公开，因为不使用文件系统，不能通过命名文件切分空间。

   - 缺少文件系统也意味着缺少文件权限，因此 devdax 需要以 root 身份运行应用程序，或者更改设备本身的权限。

   - devdax 提供对 PMem 的更多原始访问，因此应用程序更容易保证大页面的对齐。这是应用程序使用 devdax 的最常见原因，如 RedHat 在[此文所述](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/configuring-persistent-memory-for-use-in-device-dax-mode)。

   - 当系统知道PMem 中的[Bad Blocks](#bad-blocks)时，fsdax 会在这些区域映射无效页面，以防止应用程序使用损坏位置。在英特尔服务器上使用损坏位置会导致难以处理的*机器检查machine checks*。devdax 访问更加原始，允许映射包含损坏的页，因此应用程序在接触这些区域时会触发机器检查，即使系统已经知道该损坏。

   - devdax 允许 long-lived RDMA 内存注册，就像其它一些基于 RDMA 的库所需要的那样。这是应用程序使用Device DAX 的另一个最常见原因。使用 fsdax，不允许 long-lived RDMA 内存注册，只有支持*按需分页On-Demand Paging (ODP)* 的RDMA 卡才支持。

   - devdax 没有实现所有的 POSIX。例如，[msync](#msync) 不起作用，必须改用[CLWB ](#clwb)等指令刷新用户空间。查询PMEM的大小比起只调用`stat(2)`复杂更多，但[PMDK](#pmdk)库消除了这些差异，可以按照预想方式使用devdax。

   - 由于 devdax 不包含文件系统，因此文件系统提供的常见安全措施（譬如清零分配的块）是没有的。这意味着应用程序可以看到历史遗留的旧数据，因此应用程序设计人员必须考虑到这一点。

​		[PMDK](https://pmem.io/pmdk/manpages/linux/v1.10/daxio/daxio.1.html)提供了一个实用程序[daxio](https://pmem.io/pmdk/manpages/linux/v1.10/daxio/daxio.1.html)用于保存/恢复/清零 devdax 设备。

> I/O path个人译为I/O操作，data path译为数据操作



### Dirty Shutdown Count

(DSC, Unsafe Shutdown Count)

​		失败刷新Flush-on-fail机制（譬如[ADR](#adr)和[eADR](#eADR)）为[PMem](#adr)提供了一个[编程模型](#programming-model)，确保即使在突然断电的情况下，存储也能持久化。当失败刷新机制本身失败时，则持久化承诺不再有效，未刷新的存储数据会丢失。这种情况很少发生，通常是硬件故障的结果，但显然必须将其报告给软件以避免静默数据损坏。**Dirty Shutdown Count**就是这类故障报告给软件的方式。

​		当感知PMem应用程序开始使用 PMem 文件时，它会查找当前的 Dirty Shutdown Count 并将其存储在该文件的头信息中。每次打开文件时，都会根据存储的计数检查当前计数，以查看是否发生了Dirty Shutdown。如果发生了这种情况，应用程序应该认为文件处于未知状态。应用程序可能会尝试修复文件，但最常见处理是当作文件已丢失，并从冗余数据（例如备份副本）中恢复它。

​		[PMDK](https://pmem.io/glossary/#pmdk)库存储和检查 Dirty Shutdown Count 的操作如上文所述，将拒绝打开没有通过检查的任何资源。



### DRAM

(Dynamic Random Access Memory)

​		**DRAM**是当今几乎所有计算机采用的传统主内存存储器。

​		[持久性内存](persistent-memory)可以由DRAM制成，这正是市场上的NVDIMM-N产品所做的。使用 NVDIMM-N，PMem 以 DRAM 速度运行，因为它实际上是 DRAM，当断电时，NVDIMM-N 将数据持久保存在 NAND 闪存芯片上。

​		[持久性内存](https://pmem.io/glossary/#persistent-memory)可以由非 DRAM 的其他介质制成。例如，英特尔的 [傲腾Optane](#optane) PMem 使用[3D XPoint](#3d-xpoint)作为其介质。

​		通过比较NVDIMM-N 和 Intel 的 傲腾Optane ，得出一个产品是否被认为是 PMem，更多的是它提供的[编程模型](#programming-model)，而不是实现该模型使用的介质类型。



### DSM

(Device Specific Method, _DSM)

​		ACPI 定义了Device Specific Method 的概念，通常命名如**_DSM**，它允许预引导环境消除一些硬件细节，并为操作系统提供统一的调用接口。NVDIMM 的标准 DSM 由 ACPI 规范描述，可在 [UEFI 网站上获得](https://uefi.org/)。此外，英特尔发布了[DSM Interface for Optane](https://pmem.io/documents/IntelOptanePMem_DSM_Interface-V2.0.pdf) (pdf)。



### eADR

(Extended ADR)

​		**eADR**是一种硬件功能，可在**断电时**将存储从*CPU 缓存*和*内存控制器写入挂起队列 (WPQ)*刷新到目的地。

<img src="./adr.jpeg" height="300px"/>

​		如上图所示，在通往持久内存 DIMM 的途中，存储可以驻留在多个位置。下面的红色虚线框显示了 ADR 域——到达该区域的存储数据受到[ADR ](#adr)断电保护，它会刷新内存控制器中的队列，如图中的梯形所示。所有支持持久内存的英特尔系统都需要 ADR。图中较大的红色虚线框说明了一个可选的平台功能**eADR**，用于 CPU 缓存刷新。

​		eADR 使用储存的电量在断电后执行刷新。

​		BIOS 使用[NFIT](#nfit)表中的字段通知操作系统 CPU 缓存需要断电持久化保护。反过来，操作系统通常会为应用程序提供一个接口，以发现它们是否需要使用像[CLWB](#clwb)这样的缓存刷新指令，或者它们是否可以依赖 eADR 提供的断电自动刷新。使用 [PMDK](#pmdk)将允许应用程序自动检查 eADR 并根据需要跳过缓存刷新指令。



### Fence

(SFENCE)

​		**Fence**是程序员常用来排序内存操作的排序指令。对于 Intel 架构，[软件开发手册](http://intel.com/sdm)(SDM)中描述了 fence 指令的复杂细节。对于[持久内存](https://pmem.io/glossary/#persistent-memory)编程，*SFENCE* 指令特别有趣。SFENCE 的排序属性由 Intel 的 SDM 描述如下：

> 处理器确保在 SFENCE 之后的任何存储变为全局可见之前，SFENCE 之前的每个存储都是全局可见的

​		对于具有[eADR](#eadr)系统，全局可见性意味着持久性，SFENCE 指令具有双重用途，即存储数据对其他线程可见和持久化。

​		对于需要缓存刷新以使存储持久化的[ADR](https://pmem.io/glossary/#adr)系统，必须确保已刷新的存储数据已被内存子系统接受，以便将它们视为持久存储。像[CLWB](https://pmem.io/glossary/#clwb)这样的缓存刷新指令是异步*启动*和运行的。在假定刷新范围是持久的情况下，软件可以继续之前，它必须发出 SFENCE 指令。

​		[PMDK](#pmdk)库旨在消除 SFENCE 等复杂指令的细节，以便应用程序开发人员更轻松地进行 PMem 编程。



### FlushViewOfFile

(Windows Flush System Call)

​		[PMem 编程模型](#programming-model)的一项原则是标准文件API同样能在 PMem 文件上操作。对于 Windows，用于内存映射文件的标准 API 是 [MapViewOfFile](#mapviewoffile)，将任何存储刷新到该范围以使其持久化的标准方法是**FlushViewOfFile**。

​		在 Windows 上，当 PMem 文件被[DAX](#dax)映射时，还可以使用[CLWB](#clwb)等刷新指令直接从用户空间刷新存储。这通常比使用系统调用进行刷新要快得多，但两者都可以。

​		需要注意的是，根据[微软的文档](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-flushviewoffile)，FlushViewOfFile可以在刷新完成前返回，因此通常调用FlushViewOfFile后调用*FlushFileBuffers*。

​		Windows 上的 FlushViewOfFile 大致相当于POSIX 系统上的[msync](#msync)。



### Interleave Set

​		维基百科上的[Interleaved Memory](https://en.wikipedia.org/wiki/Interleaved_memory)条目描述了如何使用交错来提高性能，类似于跨存储阵列中的磁盘进行条带化。对于[持久内存](#persistent-memory)，还有另一个考虑。由于 PMem 是持久化的，因此系统每次配置 PMem 时都必须以相同的方式构造交错集，否则持久化数据会在软件中出现乱码。

​		PMem 交错集是如何每次以相同的方式创建的，这是[NVDIMM](#nvdimm)的产品特定细节。对于[CXL](#cxl)上的持久内存 ，已经定义了一个标准机制：*区域标签 region labels*。这些标签永久存储在 CXL 规范定义的 [标签存储区域中](#label-storage-area)。

​		Linux 和 CXL 规范也将交错集称为[区域region](#region)。



### KMEM DAX

​		**KMEM DAX**是[内存模式](#memory-mode)的半透明semi-transparent替代方案，用于使用易失性的 PMem。 [Device DAX](#device-dax) 可以配置为[system-ram mode](https://pmem.io/ndctl/daxctl-reconfigure-device.html). 此模式将 PMem 公开为热插拔内存区域。以这种方式配置，持久性内存采用单独的仅内存 NUMA 节点的形式。与**内存模式Memory Mode** 不同，PMem 表示为由操作系统明确管理的独立内存资源。有关 KMEM DAX 的更多信息，请参阅[KMEM DAX 博客文章](https://pmem.io/2020/01/20/memkind-dax-kmem.html)。



### Label Storage Area

(LSA)

​		持久内存设备（例如[NVDIMM](#nvdimm)）可以划分出逻辑分区（如[命名空间namespace](#namespace)）。在 [ACPI规范](https://uefi.org/)描述了管理命名空间的标准方式，通过创建*命名空间标签*并将其存储在**标签存储区**（LSA）。

​		该[CXL规范](https://pmem.io/glossary/#cxl)扩展了LSA的想法，同时存储命名空间标签和[区域region](https://pmem.io/glossary/#region)标签。

​		通常，LSA 是一个相当小的保留区域，大小为数十 KB。它是持久的，读写它的特定方式有助于检测错误，例如交错集中丢失的设备。



### libmemkind

​		**memkind**库主要是使用的Pmem的易失性功能，其中软件忽略Pmem的持久性功能，使用它是出于容量和价格的考虑。 **memkind**在内部使用流行的**jemalloc**库为多个独立堆提供灵活的分配器。更多信息可以在[memkind网站](http://memkind.github.io/memkind/)。



### libpmem

​		**libpmem**库提供低级持久内存的操作支持。像更高级别[libpmemobj](#libpmemobj)库在新的[libpmem2](#libpmem2)出现前，是基于**libpmem**实现的。[PMDK网站](https://pmem.io/pmdk/)包含了所有PMDK的 [GitHub](https://pmem.io/repoindex)上开源的库的文档。



### libpmem2

​		**libpmem2**库提供低级持久内存的操作支持，是原来的[libpmem](#libpmem)库的替代者。 **libpmem2**提供了更通用且与平台无关的接口。希望推出自己的持久内存算法的开发人员会发现这个库很有用，但大多数开发人员可能会直接使用*提供内存分配*和*事务支持*的[libpmemobj](#libpmemobj)高级库。libpmemobj 是基于**libpmem2**实现的。[PMDK网站](https://pmem.io/pmdk/)包含了所有PMDK的 [GitHub](https://pmem.io/repoindex)上开源的库的文档。



### libpmemblk

​		**libpmemblk** 库支持大小相同的pmem-resident blocks常驻块的数组的原子更新。例如，在 pmem 中保存固定大小对象缓存的程序可能会发现这个库很有用。**libpmemblk**使用的算法与[UEFI 规范](https://uefi.org/) 标准化的[BTT](#btt)算法相同。[PMDK网站](https://pmem.io/pmdk/)包含了所有PMDK的 [GitHub](https://pmem.io/repoindex)上开源的库的文档。



### libpmemkv

​		**libpmemkv**库提供本地/嵌入式的键值数据存储在持久内存上的优化。**pmemkv**不局限于单一语言或实现支持，而是为语言绑定和存储引擎提供了不同的选项。[PMDK网站](https://pmem.io/pmdk/)包含了所有PMDK的 [GitHub](https://pmem.io/repoindex)上开源的库的文档。



### libpmemlog

​		**libpmemlog**库提供一个Pmem常驻日志文件。这对于频繁追加日志文件的数据库等程序很有用。[PMDK网站](https://pmem.io/pmdk/)包含了所有PMDK的 [GitHub](https://pmem.io/repoindex)上开源的库的文档。



### libpmemobj

​		**libpmemobj**库是[PMDK](#pmdk)集合中最流行和最强大的类库。它提供了一个事务对象存储，为持久内存编程提供内存分配、事务和通用服务。不熟悉持久内存的开发人员可以想从这个库开始入手。[PMDK网站](https://pmem.io/pmdk/)包含了所有PMDK的 [GitHub](https://pmem.io/repoindex)上开源的库的文档。



### librpma

​		**librpma**提供用于*远程访问持久内存*的API。该[PMDK](#pmdk)库旨在帮助应用程序使用 RDMA 访问远程 PMem。

​		详细信息请参阅[librpma 手册页](https://pmem.io/rpma/manpages/master/librpma.7.html)。



### LLPL

(Java Low Level Persistence Library)

​		**LLPL**是 Java 低级持久内存库，提供了对持久内存分配持久堆上的块的访问。可以在[LLPL GitHub](https://github.com/pmem/llpl)上找到更多信息。



### MapViewOfFile

(Windows Memory Map System Call)

​		Windows 文件 API 包括使用系统调用**MapViewOfFile**对文件进行内存映射的能力。

​		内存映射文件在应用程序的地址空间中显示为一系列虚拟内存，允许应用程序像加载和存储内存一样访问它。对于存储中的文件，这是有效的，因为当应用程序访问该page时，操作系统使用[paging](https://pmem.io/glossary/#paging)将页面的内容拉到[DRAM](#dram)中。对于[持久内存](https://pmem.io/glossary/#persistent-memory)，感知PMem文件系统允许内存映射文件直接访问 PMem，该功能称为[DAX](#dax)。

​		在[Windows文档](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile)中包含关于如何使用这个系统调用的细节。

​		基于 POSIX 的系统（如 Linux）上的等效系统调用是[mmap](https://pmem.io/glossary/#mmap)。



### Memory Mode

(2LM)

​		[Intel 的 Optane PMem](https://pmem.io/glossary/#optane)产品可以配置为两种模式：[App Direct](#app-direct)和**Memory Mode**。有关产品详细信息，请参阅 [英特尔傲腾](http://intel.com/optane)。下面我们对**内存模式**进行了描述，理解它与持久内存的关系。

​		**内存模式**结合了两层内存，DRAM 和 PMem，有时也称为**2LM**或**two-level memory**。

​		<img src="./memory_mode.jpeg" height="320px"/>

​		以这种方式配置时，系统 DRAM 充当内存端缓存。当访问缓存命中时，数据以 DRAM 性能从 DRAM（*近内存*）返回。当访问缓存未命中时，以 PMem 性能从 PMem（*远内存*）获取数据。Intel 的内存模式在内存端缓存中使用 64 字节大小的缓存行。

​		**内存模式**是持久内存的易失性用途，软件不需要内存的持久功能，事实上，傲腾Optane PMem 设备会在每次启动时以加密方式扰乱数据，以确保预期的易失性volatile语义。

​		尽管这种**2LM**配置在任何两层内存之间在技术上是可行的，但其主要流行是提供高容量的系统主内存，而不需要花费相同容量的 DRAM 成本。


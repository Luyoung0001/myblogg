---
title: "SSD 为什么会卡：从 NAND、控制器到固件 Bug"
date: 2026-07-03 17:25:24
tags:
  - "SSD"
  - "NAND"
  - "NVMe"
  - "storage"
categories:
  - "Computer Architecture"
---

## 前言

最近讨论 Linux 的 swap、SSD、NAND flash、控制器和固件时，我发现一个很容易误解的地方：操作系统把 SSD 看成一个很普通的块设备，但 SSD 内部其实是一个很复杂的小系统。

从 Linux 看，读写 SSD 只是对 `/dev/nvme0n1` 或 `/dev/sda` 发起读写请求；从 SSD 内部看，它要做地址翻译、错误纠正、坏块管理、磨损均衡、垃圾回收、温控、掉电保护和统计上报。也就是说，SSD 对操作系统保持透明，但它自己并不简单。

<!--more-->

## 先把缩写说清楚

| 缩写 | 全称 | 大概意思 |
| --- | --- | --- |
| SSD | Solid State Drive | 固态硬盘，没有机械磁头和盘片，主要用 NAND flash 保存数据 |
| HDD | Hard Disk Drive | 机械硬盘，靠磁性盘片和磁头保存数据 |
| NAND | NAND Flash Memory | 非易失闪存，断电后数据仍然保留 |
| Flash | Flash Memory | 闪存，NAND 是最常见的一类大容量闪存 |
| SLC | Single-Level Cell | 一个存储单元保存 1 bit，快、耐用、贵 |
| MLC | Multi-Level Cell | 一个存储单元保存 2 bit |
| TLC | Triple-Level Cell | 一个存储单元保存 3 bit，消费级 SSD 常见 |
| QLC | Quad-Level Cell | 一个存储单元保存 4 bit，容量高，但写入耐久和性能压力更大 |
| PCIe | Peripheral Component Interconnect Express | 高速总线，NVMe SSD 通常走 PCIe |
| NVMe | Non-Volatile Memory Express | 面向非易失存储的高速协议，常见于 M.2 NVMe SSD |
| SATA | Serial ATA | 老一些的存储接口协议，SATA SSD 也很常见 |
| AHCI | Advanced Host Controller Interface | SATA 时代常见的主机控制器接口 |
| LBA | Logical Block Address | 逻辑块地址，操作系统通过它描述“我要读第几个块” |
| FTL | Flash Translation Layer | 闪存转换层，把主机的逻辑地址映射到 NAND 的物理位置 |
| GC | Garbage Collection | 垃圾回收，SSD 内部搬运有效数据、擦除无效块 |
| WL | Wear Leveling | 磨损均衡，让 NAND 块尽量均匀消耗寿命 |
| ECC | Error Correcting Code | 错误纠正码，用来发现和纠正 NAND bit 错误 |
| LDPC | Low-Density Parity-Check | 现代 SSD 常用的强纠错码之一 |
| SMART | Self-Monitoring, Analysis and Reporting Technology | 设备健康状态和错误计数上报机制 |
| TRIM | ATA TRIM / NVMe Deallocate | 操作系统告诉 SSD：这些逻辑块已经不用了 |
| OP | Over-Provisioning | 预留空间，不给用户直接用，给 SSD 内部调度、GC 和磨损均衡用 |
| DRAM | Dynamic Random Access Memory | 有些 SSD 板上的独立缓存内存，常用来放映射表缓存 |
| HMB | Host Memory Buffer | 无 DRAM NVMe SSD 借用主机内存缓存部分元数据 |

这些词中最关键的是 `LBA` 和 `FTL`。操作系统永远在和逻辑块地址说话，而 NAND 芯片真正的数据位置由 SSD 控制器自己决定。

## 操作系统眼里的 SSD

CPU 不能像访问内存那样直接访问 SSD 上的某一个字节。程序读取文件时，大致会经过这条路径：

```text
应用程序
  -> VFS / 文件系统
  -> Linux block layer
  -> NVMe / SATA 驱动
  -> PCIe / SATA 总线
  -> SSD 控制器
  -> NAND flash
```

读 SSD 时，CPU 不是直接去 NAND 里拿数据，而是驱动向设备提交读命令，SSD 控制器把数据通过 DMA 搬到内存里，然后 CPU 再从内存读。

所以“CPU 读取 SSD 的单位是什么”这个问题要分层回答：

| 层次 | 常见单位 | 说明 |
| --- | --- | --- |
| 应用层 | 字节、文件偏移 | `read(fd, buf, len)` 看起来可以读任意长度 |
| 文件系统 | block，一般 4 KiB 常见 | 文件系统管理空间的基本单位之一 |
| 块设备层 | sector / logical block | Linux 和设备驱动提交块 I/O |
| NVMe 协议 | LBA + length | NVMe namespace 是一组主机可访问的 LBA |
| SSD 内部 | NAND page / block / plane / die | 真实读写和擦除单位，操作系统通常看不到 |

NVMe 规范里，namespace 可以理解为一组主机软件能访问的 LBA。一个 LBA 的数据大小可能是 512B，也可能是 4KiB 或其它 2 的幂大小。对 Linux 来说，`/dev/nvme0n1` 这样的设备就是一个 namespace；对 SSD 控制器来说，这只是它暴露给主机的一层逻辑地址空间。

## NAND flash 的物理限制

SSD 的复杂性主要来自 NAND flash 本身。

NAND 有几个很重要的特点：

1. 读和写通常以 page 为单位。
2. 擦除通常以 block 为单位，一个 block 里包含很多 page。
3. NAND 不能像 DRAM 那样原地任意改写一个字节。
4. 写入前通常需要先擦除，而擦除单位比写入单位大很多。
5. 每个 NAND block 的擦写次数有限。
6. 随着制程、层数和每个 cell 保存 bit 数增加，错误率、读扰动、保持时间等问题会更难处理。

这就导致一个后果：操作系统说“把 LBA 1000 的 4KiB 改掉”，SSD 内部并不一定真的在原地把某个 NAND page 改掉。它更可能是找一个新的空闲 page 写入新数据，然后把旧 page 标记为无效。等无效 page 积累多了，再由 GC 把还有效的数据搬走，擦除整个 block，重新变成可写空间。

这就是 SSD 写入可能突然变慢的根源之一：前台写入看似是用户的一个写请求，后台可能还要搬运旧数据、擦除 block、更新映射表、做 ECC、更新元数据。SSD 空间越满，空闲 block 越少，GC 越难做，写放大越明显。

## SSD 控制器不是简单转接头

SSD 控制器可以理解为 SSD 的“大脑”。它一般包括这些部分：

```text
主机接口：PCIe/NVMe 或 SATA/AHCI
  |
控制器 SoC
  |-- 嵌入式 CPU 核心：ARM、RISC-V 或厂商自研核心都有可能
  |-- SRAM：控制器内部高速小内存
  |-- DRAM 控制器：连接板载 DRAM，有些低端盘没有
  |-- DMA 引擎：在主机内存和 SSD 内部缓冲区之间搬数据
  |-- ECC / LDPC 引擎：纠错
  |-- 加密引擎：有些盘支持硬件加密
  |-- NAND 通道控制器：并行访问多个 NAND die / package
  |-- 温度、电源、时钟和复位管理
  |
NAND flash 阵列
```

所以“SSD 里是不是有处理器和操作系统”这个问题，答案是：有处理器，有固件，有运行时状态；但不一定是 Linux 这种通用操作系统。消费级 SSD 里通常跑的是厂商写的嵌入式固件，可能是裸机程序，也可能有小型 RTOS。它要实时处理主机命令和 NAND 后台任务。

SSD 运行时也需要内存。控制器内部会有 SRAM；中高端 SSD 往往还有一颗外置 DRAM，用来缓存 FTL 映射表、队列、元数据和热数据。无 DRAM 的 NVMe SSD 可能使用 HMB，从主机内存里借一小块空间辅助缓存。

固件放在哪里也分情况。控制器芯片里通常会有 ROM 或小容量片上存储，用来完成最早期启动；完整固件可能放在 NAND 的保留区域，也可能放在专门的 flash 中。启动时控制器先跑 boot ROM，再加载正式固件，然后才开始响应主机命令。

## FTL：SSD 内部最关键的软件层

FTL 是 Flash Translation Layer，字面意思是闪存转换层。它不是一个单独的硬件芯片，而是控制器固件中的核心逻辑。

FTL 最重要的工作是维护映射关系：

```text
主机看到的逻辑地址：
LBA 1000
LBA 1001
LBA 1002

SSD 内部真实位置：
channel 2 / die 0 / block 318 / page 42
channel 5 / die 1 / block 102 / page 9
channel 0 / die 3 / block 884 / page 77
```

操作系统以为 LBA 是连续的，但 SSD 内部的物理位置可能完全不连续。控制器故意这样做，因为它要并行利用多个 NAND 通道，要绕开坏块，要把写入均匀分布到不同 block，还要给 GC 留出空间。

FTL 还会处理这些事情：

1. 地址映射：把 LBA 翻译成 NAND 物理页。
2. 写入调度：决定新数据写到哪个 block、哪个 die、哪个 channel。
3. 垃圾回收：搬走有效页，擦除旧 block。
4. 磨损均衡：避免某些 block 被过度擦写。
5. 坏块管理：出厂坏块和运行中新增坏块都要隔离。
6. 元数据持久化：映射表和状态要安全保存，掉电后能恢复。
7. 读干扰处理：读太多次可能影响相邻单元，控制器要监控和搬迁。
8. SLC Cache：把 TLC/QLC 的一部分临时当 SLC 用，提高短时间写入性能。

因此，SSD 的性能不是只由 PCIe 带宽决定。控制器算力、固件算法、NAND 类型、并行通道数、DRAM/HMB、空闲空间、温度、写入模式都会影响实际体验。

## SSD 会对操作系统的请求做“校验”吗

更准确地说，SSD 控制器会做多层检查和保护，但它不只是简单地“检查一下请求对不对”。

第一层是协议检查。NVMe 或 SATA 命令里会带 namespace、起始 LBA、长度、队列信息、权限状态等字段。控制器要判断请求是否合法，比如 LBA 是否越界、命令是否支持、namespace 是否存在。

第二层是传输保护。PCIe、SATA、NVMe 协议本身有自己的链路和命令完整性机制。企业级场景还可能启用 PI/DIF 这类端到端保护，让主机和设备共同检查数据是否在路径中损坏。

第三层是 NAND 数据保护。NAND 读出来的原始数据并不总是完美的，控制器必须用 ECC/LDPC 去纠错。如果纠错失败，就可能上报介质错误，并更新 SMART / NVMe health log。

第四层是元数据一致性。FTL 映射表、GC 状态、坏块表、写入日志等内部元数据如果坏了，整块盘可能直接不可用。所以 SSD 固件会非常小心地维护这些元数据，有些高端盘还会用掉电保护电容把缓存中的关键数据落盘。

所以 SSD 对系统是透明的，但它内部做了很多系统软件才会做的事情。

## 为什么 SSD 会“卡”

很多人说“SSD 卡了”，背后可能不是一个原因。

第一种是 SLC Cache 用完。很多 TLC/QLC SSD 会用一部分空间模拟 SLC，提高短时间写入速度。你复制一个几十 GB 的大文件时，前面速度很高，过一会儿突然掉速，就是 cache 写满后开始直接写 TLC/QLC 或边写边回收。

第二种是垃圾回收压力大。盘越满，可用空闲块越少，GC 越难做。此时一个小写入也可能触发很多内部搬迁，写放大上升，延迟变差。

第三种是温度过高。NVMe SSD 很容易热，尤其是高性能盘。温度到阈值后控制器会降频限速，这叫 thermal throttling。

第四种是后台任务抢资源。SSD 可能在做 GC、磨损均衡、坏块扫描、SLC cache 折叠、元数据整理。这些任务不一定完全停止前台 I/O，延迟就可能抖动。

第五种是系统内存压力导致 swap。swap 确实在硬盘上，Linux 把一部分内存页换出到 swap file 或 swap partition。对于 SSD 来说，swap 写入也是普通块写入，先经过文件系统或块层，再变成 LBA 请求。然后 SSD 控制器再把这些 LBA 映射到 NAND 物理位置。也就是说，swap 的“物理地址”不会等于 NAND 的真实地址，它仍然要经过 SSD 的 FTL 映射。

第六种是固件 bug 或兼容性问题。有些问题不是你文件系统坏了，也不是 Linux 用错了，而是 SSD 固件本身的某个状态机、计数器、恢复流程、SMART 统计、掉电处理逻辑有 bug。

## 固件升级真的能修 SSD 问题吗

可以，真实世界中有很多例子。SSD 固件不是摆设，它控制 FTL、GC、ECC、温控、SMART 统计、掉电恢复和协议兼容性。固件升级可以修复可靠性、性能、兼容性甚至健康状态统计错误。

但固件升级也有几个基本原则：

1. 升级前先备份重要数据。
2. 确认型号、容量、固件版本，不要刷错。
3. 笔记本接电源，台式机避免升级过程中断电。
4. 系统盘升级前最好关闭重负载任务。
5. 已经发生介质损坏或映射表损坏时，固件升级不一定能救回数据。

三星官方 FAQ 也建议使用最新 firmware 和 Magician 软件，因为更新可能包含稳定性、性能和用户体验改进；Samsung Magician 页面还明确提醒，虽然升级通常不影响 SSD 中的数据，但升级前仍强烈建议完整备份。

## 几个真实固件案例

### Crucial m4：通电 5000 多小时后的异常

Crucial m4 是早期很典型的 SSD 固件案例。公开资料中提到，部分 m4 SSD 在累计通电约 5000 小时后会出现蓝屏或系统重启，之后每增加约 1 小时又可能再次触发。Crucial 后续提供了 0309 等固件更新来修复这个问题。

这个案例说明一个事实：SSD 固件里可能有计时器、状态计数器和后台任务调度逻辑。它们跑几千小时后才暴露 bug，并不奇怪。

### Intel SSD 320：8MB / BAD_CTX 问题

Intel SSD 320 系列曾出现过著名的 “8MB bug”。一些盘在特定掉电条件后，容量可能变成 8MB，序列号表现为 `BAD_CTX 0000013x`。后续固件 `4PC10362` 被用来处理这个 BAD_CTX 13x 问题。

这个案例很适合理解 FTL 元数据的重要性。SSD 不是只要 NAND 数据还在就一定能读出来；如果控制器启动后无法正确恢复映射表或上下文，主机看到的整块盘就可能异常。

### Samsung 980 PRO：建议从问题固件升级

2023 年，Puget Systems 发布过关于 Samsung 980 PRO 的关键固件更新说明。报告中提到，Samsung 确认 `3B2QGXA7` 固件存在问题，并建议 980 PRO 用户升级到 `5B2QGXA7`，以防止问题发生。Samsung 官方工具下载页也列出了 980 PRO 的 `5B2QGXA7` 固件 ISO。

这类问题最麻烦的地方是：等盘已经开始只读、SMART 错误累计或数据不可访问时，升级可能已经晚了。固件更新更像“预防针”，不是万能数据恢复工具。

### Samsung 990 PRO：健康度异常下降

Samsung 990 PRO 也出现过健康度异常下降的公开讨论。Puget Systems 记录过 `0B2QJXD7` 到 `1B2QJXD7` 的修复经过，并指出新固件用于阻止异常健康损耗继续发生，但不会恢复已经损失的健康度统计。Samsung 官方工具页后续还列出了 990 PRO 更新固件，例如 `4B2QJXD7`、`7B2QJXD7`、`8B2QJXD7`，对应温度记录、识别稳定性、读操作稳定性等改进。

这个案例说明，SMART / health 这类状态也不是“纯硬件事实”，它们同样依赖固件如何统计、解释和上报。

## 回到最初的问题

SSD 对操作系统保持透明，操作系统发的是逻辑块读写请求；SSD 内部自己维护一整套映射、调度、纠错和寿命管理系统。

所以可以把 SSD 理解成下面这样：

```text
操作系统眼里：
  一个线性的块数组
  LBA 0, LBA 1, LBA 2, ...

SSD 实际内部：
  控制器 + 固件 + SRAM/DRAM + 多通道 NAND
  FTL 映射表
  ECC/LDPC
  GC
  wear leveling
  bad block table
  SMART / health log
  thermal throttling
```

这也是为什么 SSD 看起来像一个“硬盘”，但它更像一个带专用处理器、专用内存、专用固件和复杂后台算法的嵌入式存储系统。

理解这一点后，很多现象就不神秘了：

1. swap 在 SSD 上，最终也是 LBA 写入，再被 FTL 映射到 NAND。
2. 删除文件后，TRIM/Deallocate 能告诉 SSD 哪些 LBA 不再有用，帮助后续 GC。
3. SSD 写满后容易变慢，因为 GC 和写放大压力变大。
4. 低端无 DRAM SSD 在随机写、长时间写、接近满盘时更容易掉速。
5. 固件升级确实能修 bug，但不能替代备份。

一句话总结：SSD 不是“没有机械结构的硬盘”这么简单，它是把大量复杂性藏进控制器和固件里的存储计算机。

## 参考资料

- [NVM Express: NVMe Namespaces](https://nvmexpress.org/resource/nvme-namespaces/)
- [NVM Express NVM Command Set Specification](https://nvmexpress.org/specifications/)
- [NVM Express: Zoned Namespaces and FTL background](https://nvmexpress.org/new-nvmetm-specification-defines-zoned-namespaces-zns-as-go-to-industry-technology/)
- [Red Hat Enterprise Linux: Solid-State Disk Deployment Guidelines](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-ssd)
- [Samsung Semiconductor: Tool & Software Download](https://semiconductor.samsung.com/consumer-storage/support/tools/)
- [Samsung Semiconductor: Internal SSD Firmware and Software FAQ](https://semiconductor.samsung.com/consumer-storage/support/faqs/internalssd-fw-sw/)
- [Samsung Magician Software FAQ](https://semiconductor.samsung.com/consumer-storage/magician/)
- [Puget Systems: Critical Samsung SSD Firmware Update](https://www.pugetsystems.com/support/guides/critical-samsung-ssd-firmware-update/)
- [Puget Systems: Samsung 990 Pro Critical Firmware Update](https://www.pugetsystems.com/blog/2023/03/14/samsung-990-pro-critical-firmware-update/)
- [Tom's Hardware: Crucial Offers Firmware Update For Crucial m4 SSD BSOD](https://www.tomshardware.com/news/Crucial-m4-Firmware-BSOD%2C14544.html)
- [Stone Group KB: Crucial M4 SSD Unresponsive or Not Reliably Detected](https://kb.stonegroup.co.uk/crucial-m4-solid-state-drive-ssd-unresponsive-or-not-reliably-detected_426.html)
- [Thomas-Krenn Wiki: Intel 320 Series SSDs Information](https://www.thomas-krenn.com/en/wiki/Intel_320_Series_SSDs_Information)

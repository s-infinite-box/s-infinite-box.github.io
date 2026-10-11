---
title: "从电源键到内核主函数：对照 Linux 0.11、aa_rust_kernel 与 Linux 6.18"
description: "沿着固件、镜像装载、CPU 模式、栈和页表的交接过程，对照 Linux 0.11、Rust 教学内核 aa_rust_kernel 与 Linux 6.18 的启动链。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - Linux内核
tags:
  - Linux
  - Rust
  - 内核启动
  - x86_64
  - Multiboot2
series:
  - aa-kernel-study
series_order: 2
draft: true
---

学习内核时，我最先面临的问题是：按下电源键以后，CPU 怎样找到内核，又怎样开始执行第一段 C 或 Rust 代码？

装载器把映像放入内存，启动代码再准备执行模式、栈、地址映射和参数，最后交给主初始化函数。下面用 Linux 0.11、正在仿写的 Rust 内核 aa_rust_kernel，以及 Linux 6.18 对照这条交接链。

## 先明确三个终点

| 内核 | 本文讨论的入口 | 当前情况 |
| --- | --- | --- |
| Linux 0.11 | `init/main.c:main` | 32 位通用初始化入口 |
| aa_rust_kernel | `src/main.rs:kernel_main` | 已定义，启动汇编尚未调用 |
| Linux 6.18.54 | `init/main.c:start_kernel` | 架构准备后的通用初始化入口 |

现代源码固定为 **6.18.54**，提交 `1b357ecb321392158d507b04672ffee57bfa071d`。它在 `arch/x86/boot/main.c` 中还有一个 setup 阶段的 `main`，两者不能混淆。

## 1. CPU 先进入固件

普通 x86 PC 上电、时钟和复位就绪后，CPU 从复位入口执行固件。这里讨论的 Intel 处理器首条指令物理地址为 `0xfffffff0`：CS 可见值 `0xf000`，隐藏基址 `0xffff0000`，EIP 为 `0xfff0`。这是特殊复位状态，不能直接用普通实模式的“段值乘 16”计算。[Intel 系统编程手册 §9.1.4](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf)

固件初始化必要硬件并选择启动目标。传统 BIOS 常把引导扇区读到 `0x7c00`；UEFI 通常装载 EFI 程序。QEMU 简化了物理上电过程，但仍有固件与装载器交接；GDB 在复位处显示的 `0xfff0` 只是指令偏移。[QEMU 启动过程](https://www.qemu.org/2020/07/03/anatomy-of-a-boot/)

## 2. Linux 0.11：自己读盘，再准备 C 环境

```text
BIOS → bootsect → setup → head:startup_32 → setup_paging → main
```

| 阶段 | 关键工作 |
| --- | --- |
| `bootsect.s` | 从 `0x7c00` 搬到 `0x90000`，调用 BIOS 读盘；setup 放到 `0x90200`，system 从 `0x10000` 开始装入 |
| `setup.s` | 保存 BIOS 参数，将 system 搬到地址 0；准备 GDT、IDT、A20 和 PIC，设置 CR0.PE 后远跳转 |
| `head.s` | 建立 32 位栈和描述符，准备分页；以 4 KiB 页恒等映射低 16 MiB |
| `main` | 继续初始化内存管理、设备与调度等 |

开启分页时，当前代码和栈仍须映射有效。0.11 提前把 `_main` 压栈，最后用 `ret` 进入 C；页目录覆盖已经执行完的早期代码，体现了启动内存的生命周期。

这里按项目的 0.11 学习副本阅读。它在倒序填写页表后没有清 DF，进入 `main` 时 DF 仍为 1；为新内核准备现代 ABI 时，应主动清 DF，不能照搬历史细节。

## 3. aa_rust_kernel：GRUB 装载，内核补齐入口条件

```text
QEMU BIOS → GRUB2 → Multiboot2 装载 ELF → _start
  → 保存 EAX/EBX → 建立 16 KiB 栈 → 检查 magic → hlt

后续计划：自有 GDT → 页表 → 64 位模式 → kernel_main
```

普通 Multiboot2 BIOS 路径交付的是 **32 位保护模式**：PE=1、PG=0、A20 已打开、IF=0；EAX 是 `0x36d76289`，EBX 是信息结构物理地址，ESP 未定义。[Multiboot2 机器状态](https://www.gnu.org/software/grub/manual/multiboot2/html_node/Machine-state.html)

当前 ELF 为 ELF64，入口仍用 `.code32`。ELF 格式和汇编编码都不能代替 CPU 模式切换。当前代码先 `cli`、`cld`，保存参数并建栈，magic 两条分支最后都停机；Rust 函数还没有调用者。

进入 Rust 前还有三个关键步骤：

1. 建立自有 GDT 和页表，映射代码、栈、页表及启动信息。候选为四级分页、2 MiB 页，在 PD 层结束映射，无需末级 PT。
2. 设置 CR4.PAE、CR3、EFER.LME、CR0.PG，再远转移到 `L=1、D=0` 的代码段。`.code64` 本身不会切模式。
3. 按实际函数签名准备参数、清 DF 和对齐栈。当前 x86_64 C ABI 下，`call` 前 RSP 模 16 为 0，函数入口为 8。[x86-64 psABI](https://gitlab.com/x86-psABIs/x86-64-ABI)

保存槽和栈都在 `.bss`；以后补 BSS 清零时，要避开已经保存的参数和正在使用的栈。当前仍停在 32 位汇编阶段，magic 正反分支的动态检查也待收尾。

## 4. Linux 6.18：先准备解压环境，再准备正式内核

以下选择 16 位 setup 路径；EFI 和其他协议入口可能跳过其中一些阶段。[Linux x86 启动协议](https://docs.kernel.org/6.18/arch/x86/boot.html)

```text
装载器准备 bzImage 与 boot_params
  → boot/header.S → boot/main.c:main → protected_mode_jump
  → boot/compressed/head_64.S:startup_32 → startup_64
  → extract_kernel：解压、装载、重定位
  → kernel/head_64.S:startup_64
  → __pi___startup_64：修正早期页表
  → x86_64_start_kernel → x86_64_start_reservations → start_kernel
```

只需抓住三次交接：

- **setup → 压缩镜像**：收集启动参数并切到保护模式，将 `boot_params` 地址传给下一阶段。
- **压缩镜像 → 解压后的内核**：准备 64 位执行环境和映射；6.18.54 的 `extract_kernel(rmode, output)` 完成解压、ELF 段搬移及重定位，返回入口地址。
- **架构入口 → 通用入口**：正式内核修正页表、建立高地址映射，准备栈和早期异常处理，再调用 `start_kernel`。

两个 `startup_64` 分别属于压缩镜像与正式内核，要连同路径定位。此版本的 `__startup_64` 位于 `arch/x86/boot/startup/map_kernel.c`，构建后带 `__pi_` 前缀；不要套用旧教程的文件位置或解压函数参数。

当前 QEMU 实验使用 `-kernel bzImage`、`-initrd`、`-append`，便于省去完整启动盘；它仍经过所用 x86 固件和 loader，不是 CPU 直接调用 `start_kernel`。调试配置关闭 KASLR，便于观察固定地址。[QEMU Direct Linux Boot](https://www.qemu.org/docs/master/system/linuxboot.html)

## 对照时只问四件事

| 检查点 | 要确认的状态 |
| --- | --- |
| 模式 | CS、CR0、CR4、EFER 与目标指令一致 |
| 映射 | 当前取指、栈和参数可访问；CR3 指向正确页表物理地址 |
| 调用 | 栈、DF、参数和返回路径满足实际 ABI |
| 生命周期 | 启动信息没有被搬移、解压或清零覆盖 |

进入主函数只是启动环境交付完成。调度、驱动和用户态还在后面；`start_kernel` 也不是 systemd 的入口。aa_rust_kernel 的实际检查方法见[下一篇 readelf 与 GDB](/posts/kernel-entry-readelf-gdb/)。

## 源码与参考

本文当前阅读副本：aa 的提交为 `0b54bdd83d69f574230bcad245219e8d922f5255`，0.11 学习副本为 `2ae7bd90246b33879eb1e9893c82564b70d2b023`。可下载[启动源码摘录](kernel-startup-source.zip)与[校验文件](kernel-startup-source.zip.sha256)，包内列出逐文件校验值；它用于阅读对照，不是完整构建仓库。

- Linux 0.11：项目学习副本的 `boot/*.s`、`tools/build.c`、`init/main.c`；原始版本见 [kernel.org 历史目录](https://www.kernel.org/pub/linux/kernel/Historic/old-versions/)。
- aa_rust_kernel：[实际仓库 aa](https://github.com/s-infinite-box/aa)的 `boot/entry.S`、`boot/linker.ld`、`src/main.rs`；协议见 [Multiboot2 规范](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)。
- Linux 6.18.54：[固定源码树](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/?id=1b357ecb321392158d507b04672ffee57bfa071d)，重点读 `boot/compressed/head_64.S`、`boot/compressed/misc.c`、`boot/startup/map_kernel.c`、`kernel/head_64.S`、`kernel/head64.c` 和 `init/main.c`。

本文由 AI 辅助整理与润色，依据个人学习副本和验证记录编写；待验证事项已在正文标明。

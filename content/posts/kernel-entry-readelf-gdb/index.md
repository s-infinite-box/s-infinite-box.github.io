---
title: "用 readelf 和 GDB 验证自己的内核入口"
description: "从 ELF 布局、Multiboot2 协议到 QEMU 指令单步，验证 GRUB 是否真正进入内核，以及启动参数和栈是否按约定交接。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - Linux内核
tags:
  - Rust
  - readelf
  - GDB
  - Multiboot2
  - QEMU
series:
  - aa-kernel-study
series_order: 3
draft: true
---

aa_rust_kernel 当前入口保存启动参数、建立栈后停在 `hlt`，尚未输出日志。先用 readelf 核对 ELF，再用 GDB 验证 GRUB 是否到达 `_start`，就能区分装载问题与预期停机。

本文延续[启动链对照](/posts/power-on-to-kernel-main/)，检查终点仍是 **32 位入口**。GDT、分页、long mode 和调用 Rust `kernel_main` 尚未完成。

## 1. 检查 ELF 布局与协议

在 aa_rust_kernel 仓库根目录执行，实际项目和产物名仍为 `aa`。环境需 Rust 裸机目标、GNU binutils、GRUB2、xorriso、QEMU 和 GDB：

```bash
make check
readelf -hW target/x86_64-unknown-none/debug/aa
readelf -SW target/x86_64-unknown-none/debug/aa
readelf -lW target/x86_64-unknown-none/debug/aa
readelf -sW target/x86_64-unknown-none/debug/aa
```

`-h` 看入口，`-S` 看 section，`-l` 看装载段，`-s` 看符号。**装载以 `LOAD` 段为依据**，不能只看到 `.bss` 就认为运行时内存已经准备好。[readelf 手册](https://sourceware.org/binutils/docs/binutils/readelf.html)

| 检查项 | 应确认什么 |
| --- | --- |
| Multiboot2 header | 8 字节对齐，完整位于文件前 32768 字节；当前长度 24 字节 |
| `_start` | ELF 入口与链接脚本符号一致 |
| `.boot` | 位于可执行 `LOAD` 段，文件偏移和内存地址分开看 |
| `.bss` | 位于可写 `LOAD` 段；`MemSiz` 覆盖保存槽和栈，允许大于 `FileSiz` |

2026 年 8 月 31 日产物中，`_start=0x100018`，`.boot` 文件偏移 `0x1000`，`.bss` 的装载内存大小 `0x5000`、文件大小为 0。后者包含保存槽、对齐填充和 16 KiB 栈；改代码后重新查符号，不能固定这些数值。

曾经未匹配的 `.eh_frame_hdr` 把入口布局推后；补齐 `.eh_frame*` 丢弃规则并确保链接脚本触发重建后恢复。检查最终 ELF 才能发现这类偏差。

ELF64 描述文件格式，`.code32` 描述指令编码。普通 Multiboot2 BIOS 交接仍是 PE=1、PG=0、IF=0，EAX 为 `0x36d76289`、EBX 为信息结构物理地址，ESP 未定义。文件 header 的 `0xe85250d6` 与运行时 EAX 的 magic 用途不同。[Multiboot2 规范](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)

## 2. 在第一条指令前停住

终端一在同一仓库运行：

```bash
make debug_server
```

它生成 GRUB ISO，以 Q35、TCG、qemu64、128 MiB、单核启动。`-S` 暂停虚拟 CPU，`-gdb tcp:127.0.0.1:1234` 开放回环调试入口。[QEMU GDB stub](https://www.qemu.org/docs/master/system/gdb.html)

终端二运行 `make gdb`，在 GDB 提示符中输入：

```gdb
set pagination off
set language c
file target/x86_64-unknown-none/debug/aa
target remote 127.0.0.1:1234
hbreak *_start
continue
info registers eax ebx esp ebp eflags rip
set $entry_info = $ebx
```

`file` 只加载宿主侧符号，内核仍由 GRUB 装载；`continue` 让 BIOS 和 GRUB 继续。这里用硬件断点，避免软件断点字节被之后的装载覆盖。[GDB 文件命令](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Files.html)、[断点](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Set-Breaks.html)

10 月 2 日手动记录中，GDB 从固件偏移 `0xfff0` 继续后命中 `_start=0x100018`，EAX 正确。EBX 和入口 ESP 是当次观察值，不作为固定地址。

## 3. 单步检查参数保存与建栈

先在 shell 用正确的 32 位解码核对机器码：

```bash
objdump -d -m i386 -M att --disassemble=_start \
  target/x86_64-unknown-none/debug/aa
```

本机 QEMU／GDB 将目标报告为 x86-64，GDB 可能错误解码 `.code32`；改变反汇编风格不改变位数。保留远端架构，用 objdump 核对指令、GDB 观察执行效果。[objdump 手册](https://sourceware.org/binutils/docs/binutils/objdump.html)

当前前六条指令依次为 `cli`、`cld`、保存 EAX、保存 EBX、设置 ESP、清 EBP。从 `_start` 执行：

```gdb
si 6
x/wx &multiboot_magic
x/wx &multiboot_info
p *(unsigned int *)&multiboot_info == $entry_info
p $esp == (unsigned long)&boot_stack_top
p/x $ebp
p ($eflags >> 9) & 1
p ($eflags >> 10) & 1
p $pc == (unsigned long)&boot_environment_ready
```

保存的 magic 应为 `0x36d76289`，信息地址等于入口 EBX，ESP 等于栈顶；EBP、IF、DF 均为 0，PC 到达比较前的检查点。需逐条观察时改用六次 `si`；`si` 按机器指令，`step` 按源码行。[GDB 单步](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Continuing-and-Stepping.html)

2026 年 9 月 15 日六次单步记录满足这些状态。另起一轮可用 `make gdb_script` 自动核对参数保存、栈顶和 DF；8 月 31 日四项检查通过。脚本的 `[PASS]` 来自宿主 GDB，范围止于 magic 比较之前。

## 4. 补测 magic 正反分支

当前已有比较与两条停机路径，动态验收仍待完成。停在 `boot_environment_ready` 后，正常路径检查：

```gdb
si
p ($eflags >> 6) & 1
si
p $pc == (unsigned long)&multiboot_magic_ok
```

ZF 应为 1，分支落到成功标签。错误路径**重新启动一轮 QEMU**，在比较前修改本轮内存中的保存槽：

```gdb
set {unsigned int}&multiboot_magic = 0
si
p ($eflags >> 6) & 1
si
p $pc == (unsigned long)&boot_bad_multiboot
```

ZF 应为 0，分支跳到失败标签。两者都停机，要按符号位置区分；magic 正确也不证明信息结构内容有效。

实验后退出 GDB，在自己启动的 QEMU 终端按 `Ctrl-C`。重新连接不会自动回到入口，从头观察必须重新启动。

## 下一步

先收尾分支验收，再建立自有 GDT、页表和 64 位环境。里程碑是 GDB 真正命中 `kernel_main`，同时核对模式、映射、栈和参数。当前没有调用者的 Rust 函数可能被链接垃圾回收移除，属于预期。

当前源码固定到 [aa 仓库](https://github.com/s-infinite-box/aa)提交 `0b54bdd83d69f574230bcad245219e8d922f5255`，重点文件为 `boot/entry.S`、`boot/linker.ld`、`Makefile` 与 `debug/entry.gdb`。也可下载[当前启动源码摘录](/posts/power-on-to-kernel-main/kernel-startup-source.zip)及[校验文件](/posts/power-on-to-kernel-main/kernel-startup-source.zip.sha256)。历史实验输出保留各自日期，不作为当前快照的完整验收。

本文由 AI 辅助整理与润色，依据个人学习副本和验证记录编写；待验证事项已在正文标明。

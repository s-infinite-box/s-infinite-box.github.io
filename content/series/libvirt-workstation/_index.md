---
title: "用 Libvirt 打造 Linux 与 Windows 工作台"
description: "从 CPU 与内存分配到双显卡直通，再到显示器和键鼠共享，记录一套 Linux 宿主机与 Windows 虚拟机的桌面实践。"
image: img/covers/linux_windows.jpg
draft: true
---

Linux 是我的开发主环境，Windows 负责一些办公和桌面应用。我把两者放到同一台工作站上：Linux 使用 AMD 显卡，Libvirt 中的 Windows 使用直通的 RTX 3080，再通过显示器输入切换和键鼠共享完成日常操作。

按下面的顺序配置资源、显卡和显示器，再选择适合自己的键鼠方案。各篇保留关键步骤、验收方法与待验证事项。

| 篇次 | 文章 | 要回答的问题 |
| --- | --- | --- |
| 1 | [Libvirt 虚拟机的 CPU 固定、NUMA 与大页配置](/posts/libvirt-cpu-numa-hugepages/) | vCPU 跑在哪里，内存从哪里来，大页是否实际生效？ |
| 2 | [RTX 3080 显卡直通：Linux 与 Windows 各用一张显卡](/posts/libvirt-rtx3080-passthrough/) | 如何让 Windows 驱动物理显卡，同时保留 Linux 桌面？ |
| 3 | [让 Linux 与 Windows 共用显示器](/posts/linux-windows-monitor-switch/) | 如何切换物理输入，并处理两个系统的桌面布局？ |
| 4 | [用 Deskflow 共享 Linux 与 Windows 的键鼠](/posts/deskflow-linux-windows-sharing/) | 如何通过网络跨系统操作，处理 Wayland 权限与剪贴板？ |
| 5 | [用 evdev/uinput 切换虚拟机键鼠，并支持自动热插拔](/posts/libvirt-evdev-input-hotplug/) | 如何用双 Shift 统一切换多把键盘与鼠标，并处理插拔和故障？ |

前 3 篇处理资源与画面。第 4 篇 Deskflow 适合网络上的多个系统，第 5 篇 evdev/uinput 针对本机虚拟机，两种输入方案按需选择。

本文记录的主环境为 Fedora 44 KDE、双路 Xeon E5-2696 v4、Libvirt 12.0.0 和 QEMU 10.2.2。CPU 编号、PCI 地址、显示器接口以及客户端名称都是本机安排，迁移到另一台机器时需要重新检查。

CPU 与显卡配置已核对，显示器和 evdev 切换有操作记录；显式 HugeTLB 与当前 Deskflow 组合仍待验收。

## 参考与致谢

系列从个人笔记与配置实践整理而来。原笔记记录的学习来源包括窝法氦镁烷的[《QEMU/KVM启动物理分区的Windows并调优》](https://www.cnblogs.com/i-am-stupid/articles/18160484)、The Passthrough POST 的 [Libvirt hook helper 教程](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/)和 [ywba.com 的显卡直通原始参考](https://www.ywba.com/655.html)。感谢原作者分享方法，也感谢 Linux、QEMU、Libvirt、VFIO-Tools、ddcutil、MultiMonitorTool、Deskflow 等项目的作者与维护者。

各篇文末分别列出借鉴范围、脚本来源和官方资料。无法访问的旧链接保留原始地址，并在对应篇目中说明。

## 写作说明

本系列笔记与文章由 AI 辅助整理、生成初稿和润色，依据个人笔记、实际配置与验证记录编写。已完成的验证与仍待验证的内容在各篇正文中分别标明。

---
title: "RTX 3080 显卡直通：Linux 与 Windows 各用一张显卡"
description: "从 IOMMU 分组和 VFIO 驱动绑定到 Libvirt PCI 配置，记录双显卡工作台的实际检查点与恢复路径。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/nvidia.jpg
categories:
  - 系统环境
tags:
  - Libvirt
  - VFIO
  - NVIDIA
  - Windows
series:
  - libvirt-workstation
series_order: 2
draft: true
---

这套工作台让 AMD RX 6800 XT 驱动 Fedora 桌面，RTX 3080 专供 Windows 虚拟机。本文接着[CPU 与内存配置](/posts/libvirt-cpu-numa-hugepages/)完成显卡分配，显示器切换见[下一篇](/posts/linux-windows-monitor-switch/)。

前提是已有能启动的 Windows 虚拟机、本机采用 Q35 与 UEFI、宿主保留独立显卡，并启用 CPU 虚拟化及 IOMMU。本文整理现有部署，没有重新演练空白系统的首次安装。

## 1. 确认设备和 IOMMU 分组

2026 年 10 月 11 日的宿主核对结果如下，地址和 ID 均需按自己的机器替换：

| 功能 | PCI 地址 | 主设备 ID | 当前驱动 |
| --- | --- | --- | --- |
| AMD 宿主图形 | `0000:05:00.0` | `1002:73bf` | `amdgpu` |
| RTX 图形 | `0000:06:00.0` | `10de:2206` | `vfio-pci` |
| RTX 音频 | `0000:06:00.1` | `10de:1aef` | `vfio-pci` |

```bash
cat /proc/cmdline
lspci -Dnnk -d 10de:
readlink -f /sys/bus/pci/devices/0000:06:00.0/iommu_group
ls /sys/bus/pci/devices/0000:06:00.0/iommu_group/devices
```

本机组 95 只有图形和音频两个功能。如果同组还有宿主依赖的设备，先调整插槽或固件安排，再继续。VFIO 隔离受整个分组约束。[内核 VFIO 文档](https://docs.kernel.org/driver-api/vfio.html#groups-devices-and-iommus)

`vfio-pci ids=` 使用主设备的 vendor/device ID。图形的 `Subsystem` ID `10de:146d` 不是另一个功能；旧 Tesla T10 参数也不能沿用。按 ID 绑定会匹配所有同 ID 设备，有同型号显卡时需重新评估选择范围。

## 2. 让 VFIO 提前接管目标卡

以下为 Fedora／dracut 示例，实施前备份相关文件和原启动参数。`/etc/modprobe.d/vfio.conf`：

```text
options vfio-pci ids=10de:2206,10de:1aef
```

本机 `/etc/dracut.conf.d/vfio.conf` 的早期加载安排：

```text
force_drivers+=" vfio_pci vfio_iommu_type1 vfio "
```

模块清单需按当前内核及 VFIO 后端核对。下面只修改并重建**当前内核**；下次启动也选择它，其他版本另行准备：

```bash
target_kernel="$(uname -r)"
sudo grubby --update-kernel="/boot/vmlinuz-$target_kernel" \
  --args='intel_iommu=on iommu=pt rd.driver.pre=vfio-pci rd.driver.blacklist=nouveau modprobe.blacklist=nouveau vfio-pci.ids=10de:2206,10de:1aef'
sudo dracut --force --kver "$target_kernel"
```

这是 Intel 平台的现用参数，AMD 平台按自身条件调整。重启后检查 `Kernel driver in use`：RTX 两个功能应为 `vfio-pci`，AMD 仍为 `amdgpu`。`Kernel modules` 只列候选驱动。[dracut 参数说明](https://dracut-ng.github.io/dracut/man/dracut.8.html)

## 3. 给 Windows 添加两个 PCI 功能

来宾关机后保存定义，再通过 virt-manager 或 `virsh edit` 添加：

```bash
virsh -c qemu:///system dumpxml win11 --inactive > win11-before-gpu.xml
virsh -c qemu:///system edit win11
```

合并到原有 `<devices>` 中：

```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
  </source>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x06" slot="0x00" function="0x1"/>
  </source>
</hostdev>
```

`source/address` 是宿主地址，来宾 PCI 地址可以不同。`managed="yes"` 管理设备分离和接回；宿主已固定 VFIO 绑定时，Windows 退出不意味着 RTX 自动回到 Linux 桌面。[Libvirt PCI 分配](https://libvirt.org/formatdomain.html#host-device-assignment)

## 4. 检查驱动、画面和再次启动

Windows 安装适合该系统的 [NVIDIA 官方驱动](https://www.nvidia.com/en-us/drivers/)，检查设备管理器，再运行：

```powershell
Get-CimInstance Win32_VideoController |
    Select-Object Name, DriverVersion, Status
nvidia-smi
```

验收只抓三个重点：图形与音频设备没有错误；线缆接在 RTX 上且物理屏收到画面；正常关机后再次启动仍能恢复。虚拟 QXL 画面可用于维护，但不能证明 RTX 接口输出正常。

2026 年 10 月 5 日 Windows 核验到 RTX 3080、20 GiB 显存、591.86 驱动。这是本机这张卡的结果，不能推广为所有 RTX 3080 的规格。

恢复时分别处理来宾定义和宿主绑定：停止直通先恢复 XML；让 Linux 使用 RTX 还需恢复 modprobe、dracut、启动参数并重建对应 initramfs。

## 参考与致谢

原笔记位于 `pphome/virtual/_virsh-libvirt/hostdev/hostdev.md`，其中引用了 [ywba.com 的直通教程](https://www.ywba.com/655.html)。感谢原作者；旧链接本次未能访问，保留便于追溯。设备参数依据本机记录，配置规则依据文中 Linux、Libvirt、dracut 官方资料。

本文由 AI 辅助整理与润色，依据个人配置和验证记录编写；待验证事项已在正文标明。

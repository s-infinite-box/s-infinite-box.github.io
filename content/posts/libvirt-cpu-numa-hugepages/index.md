---
title: "Libvirt 虚拟机的 CPU 固定、NUMA 与大页配置"
description: "以双路 Xeon 上的 Windows 虚拟机为例，核对 CPU 固定与 NUMA 分配，区分现用 memfd 内存、透明大页和待验证的 HugeTLB 配置。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - 系统环境
tags:
  - Libvirt
  - KVM
  - NUMA
  - 大页
series:
  - libvirt-workstation
series_order: 1
draft: true
---

这套工作台以 Fedora KDE 为宿主，Windows 虚拟机负责办公和桌面应用。本文先安排 CPU 与内存，再接着处理[显卡直通](/posts/libvirt-rtx3080-passthrough/)。完整目录见[工作台系列](/series/libvirt-workstation/)。

2026 年 10 月 11 日核对的环境：双路 Xeon E5-2696 v4、44 个在线逻辑 CPU、Libvirt 12.0.0、QEMU 10.2.2。CPU 固定与 NUMA 策略已经读取确认；显式 HugeTLB 尚未部署，也没有性能对照测试。

## 1. 先查拓扑，再分配资源

```bash
lscpu -e=CPU,NODE,SOCKET,CORE,ONLINE
numactl --hardware
cat /sys/devices/system/cpu/cpu10/topology/thread_siblings_list
```

结合 `SOCKET`、`CORE` 和线程同胞判断物理核心，再检查目标 NUMA 节点是否容得下内存。CPU 编号连续，不代表位于同一节点。

| 资源 | 本机分配 |
| --- | --- |
| 宿主节点 0／1 | CPU 0–21／22–43 |
| Windows 的 10 个 vCPU | CPU 10–19 |
| QEMU 模拟器线程／IOThread | CPU 20／21 |
| 宿主普通任务 | CPU 0–9、22–43 |
| Windows 内存 | XML 最大 32 GiB，限制在宿主节点 0 |

本次 CPU 10 的线程同胞只有 `10`；其他机器应按自己的 SMT 状态重新规划。

## 2. 固定线程，并匹配内存节点

来宾关机后先备份，再编辑持久定义：

```bash
virsh -c qemu:///system dumpxml win11 --inactive > win11-before.xml
virsh -c qemu:///system edit win11
```

以下片段合并进已有 `<domain>`，CPU 编号按本机替换：

```xml
<vcpu placement="static">10</vcpu>
<iothreads>1</iothreads>
<cputune>
  <vcpupin vcpu="0" cpuset="10"/>
  <vcpupin vcpu="1" cpuset="11"/>
  <vcpupin vcpu="2" cpuset="12"/>
  <vcpupin vcpu="3" cpuset="13"/>
  <vcpupin vcpu="4" cpuset="14"/>
  <vcpupin vcpu="5" cpuset="15"/>
  <vcpupin vcpu="6" cpuset="16"/>
  <vcpupin vcpu="7" cpuset="17"/>
  <vcpupin vcpu="8" cpuset="18"/>
  <vcpupin vcpu="9" cpuset="19"/>
  <emulatorpin cpuset="20"/>
  <iothreadpin iothread="1" cpuset="21"/>
</cputune>
<cpu mode="host-passthrough" migratable="off">
  <topology sockets="1" cores="10" threads="1"/>
</cpu>
<numatune>
  <memory mode="strict" nodeset="0"/>
</numatune>
```

`vcpupin`、`emulatorpin` 和 `iothreadpin` 分别约束三类线程；后两者不增加来宾核心数。`numatune` 的 `nodeset` 指宿主节点，`strict` 要求该节点能满足实际内存分配。来宾跨节点时，还需设计来宾 NUMA 单元及 `memnode` 对应关系。

本机存储为 PCI 直通，没有磁盘关联上述 IOThread。只有声明和绑定线程，不能写成已完成磁盘 I/O 优化。支持 IOThread 的 virtio-blk 磁盘还需按实际格式关联，例如 `driver` 的 `iothread="1"`。[Libvirt CPU 与 NUMA 配置](https://libvirt.org/formatdomain.html#cpu-tuning)

## 3. 核对宿主任务的有效 CPU 范围

CPU 固定不会阻止其他宿主任务使用同一核心。现有 hook 通过 systemd 收紧三个宿主单元，本次读取的有效范围均为 `0–9,22–43`：

```bash
systemctl show user.slice system.slice init.scope \
  -p Id -p AllowedCPUs -p EffectiveCPUs
```

`AllowedCPUs` 依赖 cgroup v2 的 cpuset，最终以 `EffectiveCPUs` 为准。还要确认 QEMU 所在 cgroup 允许 CPU 10–21；这些设置也不排除全部中断和内核线程。[systemd 资源控制](https://github.com/systemd/systemd/blob/main/man/systemd.resource-control.xml)

现有 helper 在 `prepare/begin` 收紧、`release/end` 恢复。复用前应保存原值、处理部分失败，并统一管理多台虚拟机的占用；旧脚本直接恢复 `0–43`，会覆盖原有限制。嵌套目录来自 helper 的约定，同步 hook 内不要调用 `virsh` 回调 Libvirt。[Libvirt hook 生命周期与限制](https://libvirt.org/hooks.html)

## 4. 大页作为单独试验

本机使用共享 `memfd`，XML 没有 `<hugepages>`，两个节点的 2 MiB HugeTLB 页池均为 0。现有清缓存、压缩内存脚本没有预留大页；不建议把 `drop_caches` 当成日常性能优化。[内核说明](https://docs.kernel.org/admin-guide/sysctl/vm.html#drop-caches)

THP 是内核按策略管理的透明大页，HugeTLB 是显式预留页池。共享 `memfd` 的 THP 要结合 `ShmemHugePages`、`ShmemPmdMapped` 和目标 QEMU 映射的 `smaps` 检查，单看 `AnonHugePages` 不够。本次没有统计 QEMU 实际 THP 用量。[THP 监测说明](https://docs.kernel.org/admin-guide/mm/transhuge.html#monitoring-usage)

若单独试验 32 GiB、2 MiB 页，需要 `16384` 张。维护窗口中先关闭来宾，保存 XML 和页池原值，再调整节点 0 的目标总页数：

```bash
hp_dir=/sys/devices/system/node/node0/hugepages/hugepages-2048kB
cat "$hp_dir/nr_hugepages" "$hp_dir/free_hugepages"
printf '16384\n' | sudo tee "$hp_dir/nr_hugepages"
cat "$hp_dir/nr_hugepages" "$hp_dir/free_hugepages"
cat /sys/kernel/mm/hugepages/hugepages-2048kB/resv_hugepages
```

目标总数、实际页数、空闲页和已有预留分别核对；对应页大小的全局 `resv_hugepages` 不是节点 0 的统计，不能把全部空闲页承诺给这台虚拟机。碎片或其他使用者也可能导致页数不足。[HugeTLB 页池规则](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html)

在原有内存后端中加入页大小，保留 `memfd` 与共享访问：

```xml
<memoryBacking>
  <hugepages><page size="2" unit="MiB"/></hugepages>
  <source type="memfd"/>
  <access mode="shared"/>
</memoryBacking>
```

若给 `<page>` 添加 `nodeset`，它指**来宾 NUMA 单元**，与 `numatune` 的宿主节点不同。还需检查 Libvirt 的后端、权限和大页挂载条件。[内存后端配置](https://libvirt.org/formatdomain.html#memory-backing)

## 验收与恢复

来宾重启后检查运行设置：

```bash
virsh -c qemu:///system vcpupin win11 --live
virsh -c qemu:///system emulatorpin win11 --live
virsh -c qemu:///system iothreadinfo win11 --live
virsh -c qemu:///system numatune win11 --live
```

再用目标 QEMU 的线程亲和性、`numastat -p PID` 和 `smaps` 核对实际执行与内存分配。正常退出和启动失败都要检查宿主 CPU 范围恢复；HugeTLB 试验还需分别恢复 XML 与页池，确认没有其他使用者依赖新增页面。性能变化最后用相同负载比较。

## 参考与致谢

原笔记位于 `pphome/virtual/_virsh-libvirt/hook/`。CPU 固定和隔离方法参考窝法氦镁烷的[《QEMU/KVM启动物理分区的Windows并调优》](https://www.cnblogs.com/i-am-stupid/articles/18160484)；hook helper 和内存整理脚本来自 [VFIO-Tools](https://github.com/PassthroughPOST/VFIO-Tools)及其[教程](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/)。感谢 SharkWipf、Rokas Kupstys、Danny Lin、yjzzjy4 及相关项目维护者。

原笔记的[龙芯大页配置资料](https://docs.loongnix.cn/kvm/kvm/loongarch-kvm/performance-tune-guide/%E5%86%85%E5%AD%98%E5%A4%A7%E9%A1%B5%E9%9D%A2%E9%85%8D%E7%BD%AE.html)本次未能访问，保留用于追溯；参数含义以上述官方文档为准。

本文由 AI 辅助整理与润色，依据个人配置和验证记录编写；待验证事项已在正文标明。

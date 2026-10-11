---
title: "用 evdev/uinput 切换虚拟机键鼠，并支持自动热插拔"
description: "把多把实体键盘汇合为一个虚拟键盘，用双 Shift 联动鼠标，并通过单一服务处理设备插拔、输入句柄与故障恢复。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - 系统环境
tags:
  - Libvirt
  - evdev
  - uinput
  - systemd
series:
  - libvirt-workstation
series_order: 5
draft: true
---

同一台宿主机上的 Windows 虚拟机，可以直接用 evdev 接收键鼠。固定设备路径会遇到缺席设备阻止启动、插拔后节点变化等问题；当前方案把多把键盘汇合为一个 uinput 键盘，再用双 Shift 联动鼠标切换。

本文是[工作台系列](/series/libvirt-workstation/)最后一篇，整理 2026 年 10 月 11 日的脚本与测试记录。它针对名为 `win11` 的虚拟机、64 位 Linux 的 24 字节 `input_event`，USB 无线接收器在范围内，直接连接的蓝牙设备未纳入自动发现。

## 输入怎样切换

```text
USB 键盘 A、B → evdev 路由 → uinput 统一键盘 → Linux 或 QEMU
USB 鼠标 A、B ─────────────────────────────→ QEMU 抓取组
```

实体键盘由路由独占读取，QEMU 只接入 `Win11 Unified Keyboard`。统一键盘使用 `grab="all"`、`grabToggle="shift-shift"`、`repeat="on"`；以下节点号只是示意，由脚本动态生成：

```xml
<input type="evdev">
  <source dev="/dev/input/event42"
          grab="all" grabToggle="shift-shift" repeat="on"/>
</input>
```

任意键盘的左右 Shift，也包括来自不同键盘的组合，都可触发切换；等汇合键盘的按键全部释放后执行。多源同时按同一键时，最后一个源释放才发松键；拔出持键源时补发释放，`SYN_DROPPED` 后查询位图恢复状态。

| 变化 | 当前处理 | 完成后的侧别 |
| --- | --- | --- |
| 键盘插拔或换节点 | 更新输入源，保留统一键盘和连接 | 保持原侧别 |
| 鼠标变化或 QEMU 输入句柄失效 | 清理旧组并重建 | 默认 Windows |
| 正常补查 | 不重建健康连接 | 保持原侧别 |
| Win11 正常关机 | 清理客机连接，保留 Linux 输入 | Linux |

重建期间短暂暂停键盘输出，恢复最终持键状态，期间完整按下再松开的击键不回放。普通键盘插拔不触发这次暂停。

## 1. 准备完整附件并核对依赖

下载[脚本快照](win11-inputs-bundle.zip)和[SHA-256 校验文件](win11-inputs-bundle.zip.sha256)。唯一维护源码位于 `pphome/_pphome/_inft/win11-inputs/`，附件保留主脚本、服务单元和设备登记。

```bash
sha256sum -c win11-inputs-bundle.zip.sha256
unzip win11-inputs-bundle.zip
bash ./win11-inputs/win11-inputs --check
bash ./win11-inputs/win11-inputs --list
```

依赖为 Bash、Perl、libvirt 客户端、udev、util-linux、coreutils、xmllint、systemd 和内核 uinput。`--check` 仅检查 Bash 与内嵌 Perl 语法，`--list` 列出输入源，均不等于运行验收。

## 2. 备份 XML，再安装服务

先保存虚拟机持久定义，明确移除本方案管理的旧固定 evdev 项，避免设备缺席或重复接入。其他输入项保留，不能整段删除：

```bash
virsh -c qemu:///system dumpxml win11 --inactive > win11-before-inputs.xml
virsh -c qemu:///system edit win11
sudo bash ./win11-inputs/win11-inputs --install
```

`--install` 必须在完整附件目录中调用，会备份原部署文件与服务状态，不重启虚拟机或 Libvirt。若检测到旧 `win11-inputs-watch` 或本方案旧 hook，会停止并提示迁移。

服务监听 udev，另在每轮对齐结束后最多五秒补查；失败退出由 systemd 按五秒间隔重启。对齐本身也有执行时间，不能承诺所有故障五秒内恢复。它不依赖同步 Libvirt hook 调用 `virsh`。

## 3. 检查日志和实际往返

```bash
systemctl status win11-inputs.service
journalctl -u win11-inputs.service -n 30 --no-pager
/usr/local/libexec/win11-inputs --list
```

用双 Shift 测试 Linux → Windows → Linux，再检查键盘、鼠标插拔及虚拟机关机启动。实体键盘始终被路由抓取，要通过统一键盘和鼠标的 QEMU 抓取状态、真实操作判断侧别。

运行记录位于 `/run/win11-inputs/` 的 `keyboards.tsv`、`proxy.tsv` 和 `state.tsv`。需要主动重新对齐时使用 `sudo /usr/local/libexec/win11-inputs --once`；它会实际调整连接，不是只读检查。

回退使用安装输出中的真实备份目录：

```bash
sudo /usr/local/libexec/win11-inputs --rollback \
  /var/lib/libvirt/win11-inputs-backups/实际备份目录
```

回退恢复文件和服务状态，XML 需按前面单独的备份恢复。停止服务可释放实体键盘抓取，但还应检查 QEMU 的旧输入连接。

## 已验证与待验证

10 月 11 日使用 HFD T88、BKB02 两把键盘及 Razer、2.4G 两只鼠标完成以下验证：

- 真实 evdev 注入触发双 Shift 往返；Linux 侧补查不切回 Windows。
- USB HID 解绑／重绑后接入恢复；键盘持键重连补发松键。
- 路由 TERM 后服务恢复；Win11 ACPI 关机再启动后自动接入，持久 XML 未变。
- 隔离内核测试覆盖同键引用计数、连发、源拔出、队列溢出和正常退出松键。

这些自动化事件不替代用户亲手按键、物理拔插验收；生产路由持键时被 SIGKILL 的清理未验证。显示器与键鼠仍是独立动作。

## 参考与致谢

本文和附件基于个人维护脚本及验证记录。感谢 Linux、QEMU、Libvirt 和 systemd 项目维护者：

- [Libvirt evdev 配置](https://libvirt.org/formatdomain.html#input-devices)
- [Linux uinput](https://docs.kernel.org/input/uinput.html)与[输入事件、SYN_DROPPED](https://docs.kernel.org/input/event-codes.html)
- [systemd 启动限流](https://github.com/systemd/systemd/blob/main/man/systemd.unit.xml)

本文由 AI 辅助整理与润色，依据个人配置和验证记录编写；待验证事项已在正文标明。

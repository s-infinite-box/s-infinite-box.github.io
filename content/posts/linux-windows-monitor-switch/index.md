---
title: "让 Linux 与 Windows 共用显示器"
description: "用 DDC/CI 切换显示器输入，配合 KDE 与 Windows 输出管理，保留分辨率、刷新率和多屏布局。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/linux_windows.jpg
categories:
  - 工具与自动化
tags:
  - DDC/CI
  - KDE
  - Windows
  - 显示器
series:
  - libvirt-workstation
series_order: 3
draft: true
---

Linux 与 Windows 各有一张显卡后，可以用不同输入共用显示器。切换要同时处理两件事：**显示器选哪个输入，以及另一系统是否仍启用这块桌面**。只切输入，窗口可能继续留在不可见区域。

本文整理 2026 年 10 月 6 日的接线与双向切换记录。本机往返后保留 Linux 4K 160 Hz、缩放 1.7，以及 Windows 4K 120 Hz 主屏；写作期间没有再次切换桌面。

## 1. 确认接口和输入值

| 项目 | Linux | Windows |
| --- | --- | --- |
| 显卡 | AMD RX 6800 XT | 直通 RTX 3080 |
| 显示器背面输入 | DP-1 | HDMI-1 |
| 系统输出标识 | KDE 的 `DP-2` | 目标显示器 ID |
| 本机 VCP 输入值 | `0x0f`（15） | `0x11`（17） |

KDE 的 `DP-2` 与显示器背面的 DP-1 属于不同编号体系。先开启显示器 OSD 中的 DDC/CI，再探测：

```bash
ddcutil detect --brief
kscreen-doctor -j
# 总线 7 换成探测结果
ddcutil --bus 7 getvcp 60 --brief
ddcutil --bus 7 capabilities
```

VCP `0x60` 用于选择输入，实际支持值以目标屏测试为准。[ddcutil 输入源说明](https://www.ddcutil.com/faq/)

## 2. 准备脚本与运行条件

下载[切换脚本附件](screen-switch-bundle.zip)并解压，包含独立 Bash 函数、Windows PowerShell 脚本和使用说明。

| 一侧 | 必要条件 |
| --- | --- |
| Linux | 已登录的 KDE 会话；Bash、ddcutil、kscreen-doctor、jq、Python 3 标准库、OpenSSH、flock、timeout；有 DDC 设备访问权限 |
| Windows | OpenSSH Server、PowerShell；[MultiMonitorTool](https://www.nirsoft.net/utils/multi_monitor_tool.html)；有交互登录桌面及另一块活动输出 |

MultiMonitorTool 二进制需单独下载，放到 `C:\Tools\MultiMonitorTool.exe`；PowerShell 脚本放到 `C:\Tools\switch_screen.ps1`。SSH 账户必须与已登录桌面用户一致：脚本通过临时计划任务进入该用户的交互会话，再用 JSON 返回结果。

附件的 DDC 值仍是本机示例，先按显示器调整；Windows 只有一块活动屏时，脚本会拒绝禁用它。

## 3. 配置连接，先查状态再切换

下面的地址是占位值，换成实际 Windows SSH 目标；`DP-2` 换成实际连接器：

```bash
export MONITOR_SSH_HOST='desktop-user@192.0.2.20'
export MONITOR_SSH_PORT=22
export MONITOR_CONNECTOR='DP-2'
source ./screen-switch/switch_screen.bash

switch_screen status
switch_screen windows
switch_screen linux
```

不带参数的 `switch_screen` 按 Windows 当前输出状态判断方向。切换顺序如下：

| 目标 | 关键顺序 |
| --- | --- |
| Windows | 确保 Linux 输出启用 → Windows 恢复目标屏 → DDC 选 HDMI → Linux 禁用目标输出 |
| Linux | Linux 恢复布局 → Windows 必要时转移主屏 → Windows 用 DDC 选 DP → Windows 禁用目标输出 |

先切输入再禁用输出，避免失去 DDC 控制所需的活动显示器句柄。Linux 函数按连接器 EDID 查总线，不固定总线 7；换线后仍要重新核对映射。

## 验收与排查

双向切换后确认实体画面正确、非目标系统的该输出已禁用、分辨率和布局恢复。当前 DDC 写入用了 `--skip-ddc-checks`、`--noverify`，没有自动读回验证；软件状态正常不能替代看实体画面。

| 现象 | 先检查 |
| --- | --- |
| 找不到屏幕或 Windows 无画面 | 线缆、显卡接口、输入源、EDID 与系统识别 |
| DDC 写入失败 | OSD 的 DDC/CI、总线和访问权限 |
| SSH 成功但桌面操作失败 | 交互登录账户、计划任务与工具路径 |
| 窗口留在看不见的位置 | 输出是否真正禁用，主屏与布局是否恢复 |

中途失败先用 OSD 选回可操作系统，再启用输出、恢复布局。脚本没有整个双系统流程的统一回滚。画面与键鼠目前独立切换，键鼠方案见 [Deskflow](/posts/deskflow-linux-windows-sharing/)和 [evdev/uinput](/posts/libvirt-evdev-input-hotplug/)。

## 参考与致谢

感谢 ddcutil、NirSoft MultiMonitorTool 和 KDE 维护者。本文基于个人脚本和切换记录；工具能力见 [ddcutil 显示器选择](https://www.ddcutil.com/display_selection/)、[常见问题](https://www.ddcutil.com/faq/)及 [MultiMonitorTool 文档](https://www.nirsoft.net/utils/multi_monitor_tool.html)。

本文由 AI 辅助整理与润色，依据个人配置和验证记录编写；待验证事项已在正文标明。

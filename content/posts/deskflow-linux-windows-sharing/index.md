---
title: "用 Deskflow 共享 Linux 与 Windows 的键鼠"
description: "从服务端、客户端和屏幕布局开始，整理 Deskflow 的网络连接、Wayland 权限、剪贴板与恢复检查。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/linux_windows.jpg
categories:
  - 工具与自动化
tags:
  - Deskflow
  - Wayland
  - Linux
  - Windows
series:
  - libvirt-workstation
series_order: 4
draft: true
---

键鼠接在 Linux 上时，Deskflow 可以通过网络操作 Windows，画面仍由各自显卡输出。本文延续[显示器切换](/posts/linux-windows-monitor-switch/)，介绍网络共享输入的最小流程。

我曾在 Linux 和 Windows 实体机之间使用 Deskflow。2026 年 10 月 11 日核对的宿主版本为 Deskflow `1.26.0.0-1`、Plasma `6.7.5`、libportal `0.10.0`；当前 Windows 虚拟机的完整往返与剪贴板测试仍待进行。

## 1. 安装并确定角色

Linux 持有键鼠，设为服务端；Windows 接收输入，设为客户端。Windows 从[官方发布入口](https://github.com/deskflow/deskflow)下载，先记录实际版本和运行模式。Fedora 可以使用：

```bash
sudo dnf install deskflow
rpm -q deskflow plasma-workspace xdg-desktop-portal \
  xdg-desktop-portal-kde libei libportal
printf '%s\n' "$XDG_SESSION_TYPE"
```

Wayland 首次运行需要授权：服务端捕获输入、客户端注入输入依赖对应 portal。按桌面弹窗完成授权；进程启动并不说明权限已就绪。

## 2. 用图形界面建立连接

1. Linux 选择服务端，记下计算机名称。
2. 在屏幕配置中添加 Windows 的名称，与客户端显示的名称一致。
3. 安排位置，例如 Linux 在左、Windows 在右。
4. Windows 选择客户端，填写能访问 Linux 的地址。
5. 保留 TLS，核对双方提示的对端指纹，启动连接。
6. 测试鼠标越界、普通文字、修饰键、滚轮和返回本地。

默认端口为 TCP 24800。连接失败先查网络，再查名称、指纹与桌面授权。[GUI 设置说明](https://github.com/deskflow/deskflow/wiki/GUI-Settings)

```powershell
# 换成 Linux 服务端的实际地址
Test-NetConnection -ComputerName 192.0.2.10 -Port 24800
```

Windows 虚拟机的 `localhost` 指向自己，不能作为宿主地址。单屏工作台首次配置时，先用维护屏或虚拟控制台让两边画面都可见。

## 3. 保存布局，检查剪贴板

GUI 连通后导出配置。下面只描述屏幕关系，名称需替换为两端真实名称：

```text
section: screens
    linux-workstation:
    windows-vm:
end
section: links
    linux-workstation:
        right = windows-vm
    windows-vm:
        left = linux-workstation
end
```

布局不包含全部角色、端口和证书设置，命令行运行时仍按当前版本帮助和 GUI 导出结果配置。[服务端配置格式](https://github.com/deskflow/deskflow/wiki/Server-Text-Config)

截至本次核对，官方说明 Wayland V2 portal 提供剪贴板能力，列出 libportal 0.10.0+、GNOME 50+ 或 Plasma 6.7.1+ 等条件。满足版本条件后仍要做双向中文、多行文字复制测试，图片和文件按需另测。[Wayland 条件与已知问题](https://github.com/deskflow/deskflow/discussions/7499)

此前的 [ClipCascade 笔记](/posts/clipcascade-clipboard-sharing/)记录了旧环境的限制；是否继续使用它，应按当前组合实测决定。

## 验收后再自动启动

先完成手动连接，再设置 KDE 用户登录自启动和 Windows 对应启动方式。复测只抓三组场景：

- 键鼠往返与双向剪贴板。
- 网络断开再连接、虚拟机停止再启动。
- 锁屏、UAC、注销登录与整机重启后的恢复。

这些项目在当前虚拟机组合中均待复测。QEMU 独占抓取同一套键盘后，Deskflow 可能收不到输入；两条路径同时送入 Windows 也可能重复。当前本机虚拟机主要采用[evdev/uinput 方案](/posts/libvirt-evdev-input-hotplug/)，两种方案的组合运行尚未验收。

## 参考与致谢

感谢 Deskflow、Wayland、libei、libportal 及桌面 portal 项目维护者。安装、GUI、布局和兼容性资料见正文官方链接；本机操作结果以实际复测为准。

本文由 AI 辅助整理与润色，依据个人配置和验证记录编写；待验证事项已在正文标明。

---
title: "用 cloud-init 制作可重复的虚拟机初始化盘"
description: "用 NoCloud 数据盘统一用户、公钥和时区配置，说明 Linux 与 macOS 的 ISO 构建、实例 ID 和登录验收。"
date: 2026-10-11T00:00:00+08:00
draft: true
image: img/covers/ubuntu.png
categories:
  - 系统环境
tags:
  - cloud-init
  - 虚拟化
  - Linux
---

用发行版 cloud image 创建测试机时，可以把用户、公钥和时区放进 NoCloud 初始化盘。系统盘提供操作系统，`seed.iso` 提供首次启动配置。

本文给出独立示例，尚未对某个固定版本的新镜像完成端到端验收。镜像需安装并启用 cloud-init、允许 NoCloud，具备 OpenSSH、sudo、Bash 和时区数据；网络沿用镜像默认行为。

## 1. 准备 user-data 和 meta-data

NoCloud 要求卷标为 `cidata`，根目录包含 `user-data`、`meta-data`，文件名没有扩展名。[数据源规则](https://docs.cloud-init.io/en/latest/reference/datasources/nocloud.html)

在项目根目录用 Bash 执行，需要 `ssh-keygen`、`uuidgen` 和自己的公钥文件：

```bash
set -euo pipefail
umask 077
mkdir -p .tmp/cloud-init-seed
cd .tmp/cloud-init-seed

public_key_file="$HOME/.ssh/id_ed25519.pub"
ssh-keygen -lf "$public_key_file"
[[ $(awk 'END {print NR}' "$public_key_file") -eq 1 ]]
public_key=$(awk '{print $1 " " $2}' "$public_key_file")

cat > user-data <<EOF
#cloud-config
users:
  - default
  - name: dev
    shell: /bin/bash
    lock_passwd: true
    sudo:
      - "ALL=(ALL) NOPASSWD:ALL"
    ssh_authorized_keys:
      - "$public_key"
disable_root: true
ssh_pwauth: false
ssh_deletekeys: true
preserve_hostname: true
timezone: Asia/Shanghai
package_update: false
package_upgrade: false
EOF

instance_id="iid-local-$(uuidgen | tr '[:upper:]' '[:lower:]')"
printf 'instance-id: %s\n' "$instance_id" > meta-data
```

示例保留发行版默认用户，创建锁定密码的 `dev`，授予免密码 sudo。公钥用于客户端登录，私钥不放进 ISO；`ssh_deletekeys` 处理服务端主机密钥。需要指定主机名时改为 `preserve_hostname: false`，在 `meta-data` 增加 `local-hostname`。[用户模块](https://docs.cloud-init.io/en/latest/reference/modules.html#users-and-groups)

`ssh_pwauth: false` 关闭 SSH 密码认证；PAM 键盘交互和镜像默认用户策略仍需核验，不能据此断言所有账户只能公钥登录。[密码模块](https://docs.cloud-init.io/en/latest/reference/modules.html#set-passwords)

构建端已安装 cloud-init 时，可以先运行 `cloud-init schema --config-file user-data --annotate`。它检查配置结构，实际登录留到最后验收。[配置校验](https://docs.cloud-init.io/en/latest/howto/debug_user_data.html)

## 2. 生成 cidata ISO

保持在配置目录中。Linux 使用 `genisoimage`，也可以换成 `mkisofs`：

```bash
genisoimage -output seed.iso -volid cidata -joliet -rock \
  user-data meta-data
```

macOS 使用系统自带的 `hdiutil`：

```bash
mkdir -p seed-files
cp user-data meta-data seed-files/
hdiutil makehybrid -o seed.iso seed-files -iso -joliet \
  -iso-volume-name cidata -joliet-volume-name cidata
```

按需生成校验值 `shasum -a 256 seed.iso > seed.iso.sha256`。每次生成的新实例 ID 会改变 ISO 内容，不承诺重复构建字节完全一致。

## 3. 用新系统盘启动并验收

从未初始化的 cloud image 创建独立系统盘，把 `seed.iso` 挂为只读光驱，仍从系统盘启动。本文验收命令面向 systemd Linux：

```bash
cloud-init status --wait
id dev
sudo -l -U dev
timedatectl
```

确认状态没有错误，再从客户端真实登录：

```bash
ssh -i "$HOME/.ssh/id_ed25519" dev@虚拟机地址
# 登录后应无需密码，输出 root
sudo whoami
```

客体内检查 sshd 的最终认证策略。以下登录条件换成实际值，尤其存在 `Match` 规则时：

```bash
sudo sshd -T -C user=dev,host=客户端主机名,addr=客户端IP | \
  grep -E '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|authenticationmethods) '
```

失败时查看 `/var/log/cloud-init.log`、`/var/log/cloud-init-output.log`，依次判断数据源、配置执行、网络和认证。制作 ISO 成功、cloud-init 完成、真实登录通过是三个不同检查点。

## 复用时注意实例 ID

新机器使用新的 `instance-id`，同一机器普通重启保持原值。仅修改 `user-data` 不保证已初始化机器重新执行所有模块；换 ID 可能触发新实例流程，不应当作普通配置更新。[首次启动与缓存](https://docs.cloud-init.io/en/latest/explanation/first_boot.html)

仓库旧封装 `virtual/cloud-init/common-iso/build_iso.sh` 缺少配套 `user-data.template`，目前不能直接复用。上面的独立流程不依赖它；不要沿用旧说明中 root 与普通用户共用密码的策略。

本文由 AI 辅助整理与润色，配置依据 cloud-init 官方文档；新镜像的实际验收仍待完成。

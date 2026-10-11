---
title: "用 VictoriaMetrics、Grafana、Telegraf 搭建多机器监控"
description: "记录云服务器、家庭 Linux 主机与 Windows 虚拟机的统一监控方案，说明指标上传、持久化、看板、告警和实际验证结果。"
date: 2026-10-11T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - 云原生
tags:
  - VictoriaMetrics
  - Grafana
  - Telegraf
  - 监控
  - SSH
draft: true
---

Telegraf 在各台机器采集，VictoriaMetrics 集中存储，Grafana 展示与评估告警。家庭 Linux 主机主动建立 SSH 隧道，Windows 虚拟机复用宿主机入口，云端服务只绑定回环地址。

```text
云端 Linux Telegraf ────────────────→ VictoriaMetrics ← Grafana
家庭 Linux Telegraf → SSH 隧道 ────────↑
Windows Telegraf → 宿主机网关 → SSH ───↑
```

本文保留 2026 年 10 月 8–10 日的 Compose 搭建流程。10 月 11 日云端已迁入 K3s，复现时使用独立目录，避免两套服务同时打开现有数据库。

| 采集对象 | 本机情况 | 指标标签 |
| --- | --- | --- |
| 云舟 | Ubuntu，2 个逻辑 CPU、约 7.51 GiB 内存 | 实际 `yunzhou`，示例 `cloud-linux` |
| 家庭宿主机 | Fedora，约 125 GiB 内存 | 实际 `inft`，示例 `home-linux` |
| Windows 客体 | Q35，10 个逻辑 CPU、约 31.93 GiB 内存、RTX 3080 | `win11` |

宿主机已经包含客体消耗，两边资源不能相加。Windows 计算机名同样是 INFT，因此另用 `win11` 区分。

## 1. 部署中心存储与看板

下载[基础配置包](monitoring-starter.zip)和[校验文件](monitoring-starter.zip.sha256)。版本固定为 VictoriaMetrics `1.153.0`、Grafana `13.2.3`、Telegraf `1.40.1`，包含中心 Compose、两种采集配置、数据源及四个基础面板。

云端需 Docker Compose、OpenSSL；解压后进入 `monitoring-starter/central`，使用独立数据目录：

```bash
sha256sum -c monitoring-starter.zip.sha256
unzip monitoring-starter.zip
cd monitoring-starter/central

MONITORING_BASE="$HOME/monitoring-blog-demo"
mkdir -p "$MONITORING_BASE"/data/{victoriametrics,grafana}
mkdir -p "$MONITORING_BASE/secrets"
chmod 700 "$MONITORING_BASE/secrets"
if ! test -s "$MONITORING_BASE/secrets/grafana-admin-password"; then
  (umask 077; openssl rand -hex 24 > "$MONITORING_BASE/secrets/grafana-admin-password")
fi
printf 'MONITORING_DATA_DIR=%s\nGRAFANA_UID=%s\nGRAFANA_GID=%s\n' \
  "$MONITORING_BASE" "$(id -u)" "$(id -g)" > .env
docker compose config --quiet
docker compose up -d
```

8428、3000 只监听云端回环地址；数据落在独立宿主目录。密码文件用于首次创建 Grafana 管理员，已有数据库的密码需按账户管理流程修改。

配置的 `7d` 是保留策略，清理由后台执行；`256MiB` 是内部缓存预算，容器 `1g` 是另一层限制，不能据此承诺进程只占 256 MiB。[VictoriaMetrics 资源与保留规则](https://docs.victoriametrics.com/victoriametrics/#resource-usage-limits)

## 2. 建立家庭侧 SSH 隧道

在家庭 Linux 上，用已有公钥登录和可信主机记录的目标替换示例：

```bash
ssh -NT \
  -o BatchMode=yes -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=15 -o ServerAliveCountMax=3 \
  -L 127.0.0.1:28428:127.0.0.1:8428 \
  -L 127.0.0.1:23000:127.0.0.1:3000 \
  monitor@metrics.example.net
```

浏览器访问 `http://127.0.0.1:23000`，Linux 采集端写入 `28428`。Windows 还需要隧道在**虚拟机可达的宿主机接口**监听：在同一条 SSH 命令增加 `-L 宿主机网关IP:28428:127.0.0.1:8428`，防火墙只允许所需客体来源。[OpenSSH 转发参数](https://man.openbsd.org/ssh)

手动连通后再交给现有 SSH 代理或 systemd，退出重启间隔可设五秒。应用停止未必让隧道退出，健康检查仍需单独做。

## 3. 接入 Linux 采集端

在该机器的配置包 `linux/` 目录执行：

```bash
printf '%s\n' 'MONITORING_HOST=home-linux' 'MONITORING_SITE=home' \
  'VICTORIAMETRICS_URL=http://127.0.0.1:28428' > .env
docker compose config --quiet
docker compose up -d
```

云端同样部署一份，改为 `cloud-linux`、`cloud`、`http://127.0.0.1:8428`。容器通过 host 网络、host PID 和 `/hostfs` 读取宿主视图，配置包含 CPU、内存、**磁盘容量**、磁盘 I/O、网络及采集端内部指标。

Telegraf 用 `outputs.influxdb` 写入 VictoriaMetrics，Grafana 则用 Prometheus 类型数据源查询；不需要另装 InfluxDB。[写入兼容说明](https://docs.victoriametrics.com/victoriametrics/integrations/influxdb/)

## 4. 接入 Windows 原生服务

按[官方安装说明](https://docs.influxdata.com/telegraf/v1/install/)下载 Telegraf 1.40.1 Windows 包并核对校验值，放到 `C:\Tools\Telegraf`。将附件 `windows/telegraf.conf` 保存为 `C:\ProgramData\Telegraf\telegraf.conf`，把写入地址的 `192.0.2.1` 换成真实宿主机网关；它是占位地址。

管理员 PowerShell 中执行，下面面向首次安装：

```powershell
New-Item -ItemType Directory -Force C:\ProgramData\Telegraf\logs
Test-NetConnection -ComputerName 宿主机网关IP -Port 28428
& 'C:\Tools\Telegraf\telegraf.exe' --config 'C:\ProgramData\Telegraf\telegraf.conf' --test
# 确认采集正常后安装并启动；已有服务先检查原配置，不重复安装
& 'C:\Tools\Telegraf\telegraf.exe' --config 'C:\ProgramData\Telegraf\telegraf.conf' service install
Start-Service telegraf
Get-Service telegraf
```

`--test` 只验证采集，不写入数据库。服务账号需能读配置、写日志和访问采集对象。附件先覆盖基础资源；原完整部署另外采集关键进程、LogicalDisk 计数器和 `nvidia-smi` CSV 的 GPU 字段，不强求所有系统使用相同插件。

## 5. 查实际指标，再补告警

中心 Compose 会自动供应 VictoriaMetrics 数据源和基础看板。家庭侧检查：

```bash
curl -fsS http://127.0.0.1:23000/api/health
curl -fsSG http://127.0.0.1:28428/api/v1/query \
  --data-urlencode 'query=system_uptime'
curl -fsSG http://127.0.0.1:28428/api/v1/export \
  --data-urlencode 'match[]=system_uptime{host="home-linux"}' \
  --data-urlencode 'start=-1m'
```

确认三个唯一的 `host` 标签、面板有数据，原始 `timestamps` 的稳定间隔为 5000 毫秒。最新点查询还受写入延迟和 VictoriaMetrics 查询延迟窗口影响，不能用图表步长替代原始样本检查。[原始数据导出](https://docs.victoriametrics.com/victoriametrics/#how-to-export-time-series)、[查询延迟](https://docs.victoriametrics.com/victoriametrics/keyconcepts/#query-latency)

告警先覆盖 CPU、内存、磁盘容量和数据缺失。在 Grafana 基于面板查询建立阈值，缺失条件可用：

```promql
absent_over_time(system_uptime{host="home-linux"}[1m])
```

原部署每 30 秒评估，CPU／内存超过 90% 持续 3 分钟、磁盘超过 85% 持续 5 分钟触发。基础附件不包含这些规则，按主机标签另建并验证；外部通知渠道也尚未配置。[Grafana 配置供应](https://grafana.com/docs/grafana/latest/administration/provisioning/)

## 实测范围与维护

| 时间 | 原完整部署的验证结果 |
| --- | --- |
| 10 月 8 日 | 两台 Linux 持续入库、5 秒采样；18 个面板有数据，5 条规则健康；数据库重启后历史数据保留 |
| 10 月 10 日 | 三个采集对象；Windows 20 个面板、27 条查询有返回；采集进程终止后约 7.08 秒恢复，整机重启未测 |
| 10 月 11 日 | 云端迁入 K3s `v1.34.12+k3s1`；合计 46 个面板、63 条查询、9 条规则保留；云端自身采集空档约 100 秒，本地两端连续并补写 |

配置包是精简复现入口，完成了静态检查，没有重新部署；表中数字来自原完整部署。两台 Linux 阶段云端三个组件短期占用约 0.66 GiB，不能用来估算长期峰值。

持续关注磁盘增长、查询耗时、采集错误、缓冲与丢弃数。默认内存缓冲上限为 10000 条指标，满后丢弃旧数据，采集进程退出也会丢失未写缓冲；尚未模拟长期断网，也未等待完整七天清理周期。[Telegraf 数据管道](https://docs.influxdata.com/telegraf/v1/concepts/data-pipeline/)

本文由 AI 辅助整理与润色，依据个人部署和验证记录编写；待验证事项已在正文标明。

# 宋明河的博客

基于 Hugo 和仓库内的个人主题 `songminghe`，通过 GitHub Actions 发布到 GitHub Pages。主题按已确认的个人博客 Demo 迁移，采用左侧菜单和右侧文章的两栏布局。

在线地址：https://s-infinite-box.github.io/

仓库名为 `s-infinite-box.github.io`，本地工作目录仍为 `_sub_mod/SongMingHe`。根地址迁移与旧链接兼容见 [迁移说明](docs/root-domain-migration.md)。

写作与平台分发见 [博客写作与发布流程](docs/blog-workflow.md)；主题迁移与验收见 [个人主题迁移](docs/theme-migration.md)。

## 当前主题

- 首页、最新文章、归档、搜索、关于五项导航，两列排列；所有文章列表不分页，置顶只影响首页。
- 桌面侧栏吸顶；手机默认收起，可通过“展开菜单”打开。文章目录位于左侧社交图标下方。
- 分类和标签使用折叠分组，展开内容高度上限为 128 像素，超出时显示“查看全部”；分类每行两项。
- 默认亮色、海岸浪花背景，保留简约山景、蓝调雾山、秋林倒影以及纯色，按钮轮换并保存偏好。
- 左侧 APlayer 播放本地歌单，默认 No Worries、75% 音量、折叠歌单；使用 Swup 连续导航，切页后音乐继续播放。浏览器限制自动播放时，使用正常播放按钮。
- 保留全文 RSS、文章系列、giscus 评论、Markdown 导出，以及旧分类标签和分页地址兼容。
- 站点信息只显示文章数、字数、运行天数和内容更新时间，不记录访客或浏览量，不需要额外统计服务器。

## 目录结构

```text
.
├── hugo.yaml                 # 正式 URL、导航、评论、输出格式和日期策略
├── Makefile                  # 构建、检查与本地预览
├── content/
│   ├── posts/<slug>/          # 文章正文和单篇配图
│   ├── page/                 # 最新文章、归档、搜索、关于
│   ├── categories/、tags/    # 分类标签入口与手工别名
│   └── series/               # 文章系列介绍
├── assets/
│   ├── img/                  # 头像与共用技术封面
│   └── diagrams/             # 可编辑图表源文件
├── data/blog.json            # 歌单、首页置顶和站点运行起点
├── static/
│   ├── music/                # 转码音频与专辑封面
│   ├── media/                # 背景图片及来源说明
│   └── .../page/<n>/         # 65 个旧分页静态跳转
├── layouts/single.markdown.md # Markdown 导出模板
├── themes/songminghe/        # 个人主题模板、样式、脚本和组件许可
├── scripts/verify-build.py   # 构建产物兼容检查，仅使用 Python 标准库
├── scripts/legacy-pages.json # 需要保留的旧分页地址清单
├── systemd/                  # 本地预览的开机启动配置与说明
├── docs/                     # 写作流程和迁移验收记录
└── .github/workflows/hugo.yml
```

## 构建与预览

建议使用与 CI 相同的 Hugo **0.165.0 extended**。本地检查脚本只需要 Python 3.9 或更新版本的标准库。

```bash
make                   # 构建正式站点到 public/
make check             # 构建并检查链接、资源、RSS、搜索和旧地址
make test              # 评论、目录、发布时间、系列排序与缓存版本回归测试
make serve             # http://localhost:1313/
make serve-drafts      # 本地预览包含草稿
make serve PORT=1314    # 使用其他端口
make clean             # 清理构建产物
```

需要每次开机自动启动预览时，使用 [systemd 用户服务配置与说明](systemd/README.md)。服务直接执行 `make serve-drafts`，默认使用 `~/.local/share/hugo/bin/hugo`，在 `http://localhost:1313/` 预览草稿及已发布内容。新增与修改自动更新，异常退出后自动重试；删除整篇文章或修改文章路径后，重启服务清除旧 URL。

可用 `HUGO=/path/to/hugo` 选择指定二进制。GitHub Actions 固定使用 0.165.0，构建后也会运行检查脚本，检查失败则不上传站点产物。

个人主题没有 npm 构建步骤，APlayer 和 Swup 浏览器资源已保存在仓库中。旧 Stack 子模块已移除，构建使用仓库内的 `songminghe` 主题，无需拉取外部主题。

## 写文章

```bash
hugo new posts/my-post/index.md
```

正文位于 `content/posts/<slug>/index.md`，URL 保持 `/posts/<slug>/`。

```yaml
---
title: "文章标题"
description: "一句话摘要"
date: 2026-10-07T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - Linux内核
tags:
  - Linux
  - Rust
series:
  - aa-kernel-study
series_order: 2
draft: true
---
```

一篇文章通常选择一个主题分类、2–5 个具体标签；连载使用 `series` 和 `series_order`，零散文章无需添加系列。现有分类为 Linux内核、云原生、编程语言、软件工程、工具与自动化、系统环境、硬件实践。

- 正文配图放在文章目录，使用 `![说明](pic.png)`；共用封面放在 `assets/img/covers/`，通过 `image` 引用。
- `comments: false` 关闭单篇评论，`toc: false` 关闭单篇左侧目录。
- 发布时间使用 `date`；内容更新时间优先使用显式 `lastmod`，否则取文章最后一次 Git 提交。CI 保留完整 Git 历史，新文章尚未提交时回退到发布时间。
- 旧分类与标签的 `aliases` 继续保留；取消分页后，原 `/page/<n>/` 等路径跳转到对应完整列表。

### 正文标题与目录

文章页会用元数据中的 `title` 显示一级标题。正文从 `##` 开始，同级主题使用同级标题，子章节依次使用 `###`、`####`，避免跳级。

- 标题只写简短的章节名；背景说明、操作提示、结果描述和待验证的猜测写成普通段落。
- 命令、报错和类型解析链放进代码块或行内代码，参考链接使用段落或列表。
- 真正需要独立导航的部分才设为章节；不要为了放大字体，把整段正文写成标题。
- 本站目录收录一到四级标题，发布前检查侧栏的条目名称、层级和跳转目标。

## 音乐、置顶与背景

修改 `data/blog.json` 的 `playlist` 维护歌单，第一项为默认曲目。文件引用 `static/music/` 中的音频和封面。用户原始 FLAC 留在本机，不纳入博客仓库。

首页置顶使用同一数据文件的 `pinned`，填写文章英文目录名。最新文章仍按发布时间排列。`startedAt` 表示当前 Hugo 博客的运行起点，不表示最早写文章的时间。

四张背景与图片许可位于 `static/media/`，默认背景和切换顺序由个人主题维护。明暗与背景选择保存在当前浏览器中。

## Markdown 导出、评论与发布

文章页提供“复制 Markdown”和“查看 Markdown 源码”，导出地址仍为 `/posts/<slug>/index.md`。正文不含 frontmatter，单篇相对配图转换为绝对 URL，便于向 CSDN 等平台分发。切换主题不改变原来的导出内容。

评论使用原仓库的 giscus / GitHub Discussions，配置集中在 `hugo.yaml`。进入文章自动加载评论组件；跨页重新加载对应文章评论，明暗配色会同步。

根地址迁移后以 `specific` 和原 `SongMingHe/posts/<slug>/` 检索词关联讨论，仓库 ID 与分类 ID 保持原值。构建后 `scripts/prepare-legacy-paths.py` 为旧项目路径补页面跳转及资源兼容文件。

全文订阅地址仍为 `/index.xml`，文章、分类、标签和系列的订阅入口也保留。分享元信息同时包含标题、摘要和封面，连续导航后同步更新。

推送到博客仓库自己的 `main` 分支会触发 GitHub Pages 部署。主题迁移先在 `codex/theme-migration` 分支完成本地验收，发布按当次授权执行。

## 来源与回退

个人主题借用与适配的 Stack 改造部分保留 GPL-3.0-only 许可，见 `themes/songminghe/LICENSE` 和主题中的来源说明。APlayer、Reimu 与 Swup 保留各自 MIT 许可；摄影背景按独立 Unsplash 许可记录。文章与用户音乐不纳入主题代码许可声明。

原版基线为 `270b88a`。回退应恢复完整主题迁移提交，包括配置和旧覆盖模板；仅将 `theme` 改回 Stack 不足以恢复原版。已有文章与音乐原始文件在此次迁移中保持不变。

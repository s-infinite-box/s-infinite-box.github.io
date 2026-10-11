# 博客写作与发布流程

本流程用于维护宋明河的 Hugo 博客，从整理素材、编写文章、处理图片，到站点发布和 CSDN、知乎分发。以 2026 年 10 月 6 日完成的内核学习文章、旧博客合并和分类整理为实践基线。

正文统一在博客仓库维护，各平台使用导出的分发稿。当前采用**仓库文档加 README 入口**的方式保存流程；后续若需要自然语言触发自动操作，再用一个轻量 skill 读取本文和仓库配置，调用已有命令。

2026-10-07 起站点接入个人主题 `songminghe`，写作、分类、封面和 Markdown 分发规则沿用。新增 `make check` 进行构建兼容检查；歌单、首页置顶和运行起点集中在 `data/blog.json`，默认 No Worries、亮色海岸背景。迁移与回退细节见 [个人主题迁移](theme-migration.md)。

## 仓库入口与日常步骤

博客仓库位于 `/home/wz/p/pphome/_sub_mod/SongMingHe`，是 `pphome` 的 Git 子模块。站点地址为 [宋明河](https://s-infinite-box.github.io/)，正式发布分支是博客仓库自己的 `main`。

| 内容 | 仓库内位置 | 用途 |
| --- | --- | --- |
| 正文和单篇配图 | `content/posts/<slug>/` | 每篇文章一个目录，正文文件为 `index.md` |
| 共用封面 | `assets/img/covers/` | 根据技术主题选择，文章通过 `image` 引用 |
| 连载介绍 | `content/series/<series>/_index.md` | 系列名称、简介和封面 |
| 分类和标签入口 | `content/categories/`、`content/tags/` | 定制入口标题，保存旧链接跳转 |
| 图表可编辑源文件 | `assets/diagrams/` | 保存本次流程图的 HTML 源文件 |
| 构建和预览命令 | [`Makefile`](../Makefile) | `make`、`make serve`、`make serve-drafts` |
| 正式部署 | [GitHub Actions 工作流](../.github/workflows/hugo.yml) | 推送 `main` 后构建并部署到 GitHub Pages |

新文章按下面的顺序推进：

1. 整理事实和配图，明确文章要回答的问题。
2. 创建文章目录，填写元数据、正文、分类和标签。
3. 选择封面，在本地检查文章页和列表页。
4. 构建正式版本，按本次发布范围提交并推送，确认部署和线上页面。
5. 有多平台分发需求时生成分发稿，同步到指定平台并核对内容。
6. 保存平台文章地址和稿源版本，后续更新回到已有文章继续维护。

只编写文章或整理文档时执行相应本地工作；同步草稿与公开发布按当次请求的目标和已有授权执行。

## 写作与连载

先确定读者能看懂的标题。学习方法文章使用《我在内核学习方式上的探索与演进》。2026 年 10 月 11 日起，博客正文、系列介绍和图表中的 Rust 教学内核统一称为 `aa_rust_kernel`；英文目录 `aa-kernel-learning-method` 和系列路径 `aa-kernel-study` 保留，已分享的文章网址继续有效。代码命令、产物路径和源码链接按实际仓库维护。

素材整理时区分已经验证的结果、当前待验证的步骤和后续计划。实验还未完成时，写清实际检查点，不把准备工作描述为已完成整个功能。学习类文章可以按阶段说明调整原因，并用书籍照片、笔记和实验记录支撑正文。

在博客仓库中创建文章：

```bash
hugo new posts/my-post/index.md
```

元数据示例：

```yaml
---
title: "文章标题"
description: "这篇文章解决什么问题"
date: 2026-10-06T00:00:00+08:00
image: img/covers/Linux.webp
categories:
  - Linux内核
tags:
  - Linux
  - Rust
draft: true
---
```

`draft: true` 用于准备中的文章，检查完成并准备发布时改为 `false`。发布时间应由作者确定；不要把写作时间、实验时间和迁移时间混为一谈。

连续文章按需增加 `series` 和 `series_order`。现有内核学习系列使用 `aa-kernel-study`，系列介绍见 [`content/series/aa-kernel-study/_index.md`](../content/series/aa-kernel-study/_index.md)。零散文章按主题归类即可。

系列页面按数字 `series_order` 升序展示，未填写序号的文章放在最后，按日期倒序排列。首页、最新文章和分类仍按原规则展示。设置 `publishDate` 可以指定实际发布时间；默认采用 `date`，时区为 `Asia/Shanghai`。GitHub Pages 在下一次构建时发布到期文章，设置未来时间不会自动触发定时部署。

## 封面与正文图片

封面集中放在 `assets/img/covers/`，在 frontmatter 中写 `image: img/covers/<文件名>`；正文照片和截图放在文章目录中，使用 `![说明](pic.png)` 引用。文件名区分大小写，引用共用封面时不加 `assets/` 前缀。

共用封面由个人主题的 [`cover.html`](../themes/songminghe/layouts/_partials/cover.html) 处理：先查找文章目录中的资源，再查找 `assets/`。可处理的位图会由主题生成适合列表与分享的缩略图；SVG 等资源直接输出。构建结果使用根路径；原 `/SongMingHe/` 页面与文件地址由构建后的兼容脚本保留。

选择封面时优先检查主体在列表裁剪后是否清楚、标题附近是否拥挤、图片尺寸是否足够。当前内核学习、IO 模型、进程线程文章引用 `Linux.webp`，Kubernetes 升级文章引用 `k8s-flower.svg`。这些是本次选图结果，各篇文章以作者最终设置的 `image` 为准。

正文图片尽量下载为本地资源，再检查迁移后的相对路径。本次迁入 9 张旧文配图，其中 8 张原为 CSDN 外链；文章目录内的图片既用于站点，也用于生成分发包。

流程图在本站保留 SVG，分发时可另导出 PNG。本次跨平台版本采用 PNG，以便在知乎编辑器中插入。导出时落实 SVG 的 CSS 变量和中文字体，再检查缩放后的文字；连续照片在分发稿中拆成独立段落，便于逐张核对。

可编辑的图表 HTML 源文件放在 `assets/diagrams/`，正文引用生成的图片。本次曾把 HTML 源文件放进文章目录，触发 Hugo 内容解析错误，移动源文件后构建通过。

## 分类标签与系列

当前规则的统一入口是 [README 分类标签与系列](../README.md#分类标签与系列)：**分类确定主要主题，标签关联具体技术和问题，系列表示连载顺序**。新增文章优先使用已有主题分类。

本次把混用的技术主题、文章用途和写作形式调整为 7 个主题分类。2026 年 10 月 6 日发布时的分布如下：

| 分类 | 篇数 |
| --- | ---: |
| Linux内核 | 3 |
| 云原生 | 3 |
| 编程语言 | 2 |
| 软件工程 | 2 |
| 工具与自动化 | 2 |
| 系统环境 | 4 |
| 硬件实践 | 1 |

每篇使用一个分类，通常选择 2–5 个重点标签，简短文章可以少于两个。技术名保持常见写法，保留有检索价值的 Cilium、eBPF、syn、ClipCascade 等具体标签；版本标签用于文章确实围绕该版本展开的情况。

本次标签调整为：

| 原写法 | 调整 |
| --- | --- |
| `linux` | 统一为 `Linux` |
| `开发工作流程` | 改为 `研发流程` |
| `学习思考`、`内核学习` | 合并为 `学习方法` |
| 磁盘只读文章的 `disk` | 改为 `NTFS`，补充 `Ubuntu`、`Windows` |
| Ubuntu 开机后关机文章 | 补充 `Ubuntu`、`systemd` |

整理后共有 40 个标签。判断是否需要合并标签时看含义和检索用途，不能只看使用次数；文章数量少时，标签只出现一次很正常。

批量修改前保存文章元数据和正文摘要，修改后核对文章数量、分类归属、大小写重复、封面和正文是否符合预期。YAML 列表可能使用有缩进或无缩进的写法，替换时应覆盖整个列表，并重新解析 frontmatter。

旧分类能对应单一新主题时跳转到新分类；旧分类拆分到多个主题时跳转到分类总览。合并后的旧标签跳转到新标签。本次通过 taxonomy 页的 `aliases` 保留了 9 个旧入口；例如：

```yaml
---
title: 系统环境
aliases:
  - /categories/troubleshooting/
---
```

## 本地预览与站点发布

在博客仓库运行：

```bash
make serve         # 预览非草稿文章
make serve-drafts  # 同时预览草稿
make serve PORT=1314
make               # 正式环境构建
```

默认预览地址为 <http://localhost:1313/>。页面检查覆盖首页卡片、文章正文、封面、图片、分类标签、系列入口和 Markdown 导出。`make serve` 启用完整重建与内存渲染，内容修改后可查看刷新效果。

正式发布前运行 `make check`，它包含 Node.js 标准测试模块及 Python 标准库回归测试，不需安装额外依赖。构建和 `hugo list published` 使用相同完整时刻，已发布清单保存在忽略提交的 `.build-published.csv`。需要复现定时发布问题时使用 `make check BUILD_CLOCK=2026-10-08T01:00:00+08:00`，不要另向 `HUGO_FLAGS` 传入不同的 `--clock`。

`make build`、`make check` 面向 `hugo.yaml` 定义的正式已发布内容；`HUGO_FLAGS` 可设置缓存或与配置一致的站点地址，不用于改变草稿、未来、过期过滤，也不用于切换源目录、输出目录、环境或配置。需要查看未来文章时使用 `make serve HUGO_FLAGS=--buildFuture`，需要查看草稿时使用 `make serve-drafts`。

没有 Makefile 的旧检出可以直接运行以下命令；Hugo 版本要求和主题初始化见 [README 本地预览](../README.md#本地预览)。

```bash
hugo server --bind 127.0.0.1 --port 1313 \
  --baseURL http://localhost:1313/ \
  --renderToMemory --disableFastRender

HUGO_ENVIRONMENT=production TZ=Asia/Shanghai hugo --gc --minify
```

发布时先确认博客仓库的分支、远端最新提交和本次文件范围。预览期间可能有人修改标题、封面或 README；需要纳入发布的新变化应先核对，再构建最终版本。

提交示例中的文件应替换为本次实际需要发布的文章和资源：

```bash
git status --short --branch
git diff --check
git add -- content/posts/my-post assets/img/covers/my-cover.jpg
git diff --cached --check
git diff --cached --stat
git commit -m "发布文章：文章标题"
git push origin HEAD:main
```

正式构建可以从待发布 Git 提交导出快照，并同时使用该提交固定的主题版本。构建通过的提交应与实际推送的提交一致；构建后又增加需要发布的修改时，重新构建相关最终版本。父仓库的子模块指针和旧目录清理按父仓库的任务范围单独维护。

推送后查看 [GitHub Actions](https://github.com/s-infinite-box/s-infinite-box.github.io/actions)，确认对应提交的构建和部署成功，再打开线上首页、分类、文章和图片检查实际内容。推送成功只说明源码已上传，部署成功和线上检查才完成站点发布。

本次曾遇到远端 README 新提交夹带无关的本机配置链接，自动审批拦截了合并提交；移除该链接后继续发布。远端 URL 若含凭据，报告中只保留仓库身份，错误输出先脱敏。

## 多平台共享

### 分发稿与本站正文

本站每篇文章有「复制 Markdown」和「查看 Markdown 源码」入口，也可访问 `/posts/<slug>/index.md`。输出由 [`layouts/single.markdown.md`](../layouts/single.markdown.md) 生成：移除 Hugo frontmatter，将普通相对图片链接改为基于文章网址的绝对链接；现有 HTTP 链接和以 `/` 开头的路径保持原样。

用于跨平台的稿件应检查图片是否能在站外访问。根路径图片、引用式图片、HTML 特殊结构等写法需要单独核对；不能假定导出模板处理了所有 Markdown 语法。

独立分发包包含 Markdown 或 HTML 正文、配图、PNG 流程图和稿源记录。文末附上本站原文链接。平台稿可以调整排版，但后续技术内容修改回到仓库正文维护，再更新平台文章。

### 本次跑通的同步方式

本次通过 **Edge 中的 Wechatsync 同步编辑器**创建 CSDN、知乎草稿。CLI 完成了 `--dry-run` 预检；实际创建草稿使用浏览器扩展。

日常分发顺序：

1. 在同一浏览器安装 Wechatsync，并登录目标平台。另一个浏览器中的登录状态不能直接作为当前扩展已登录的依据。
2. 打开已发布的本站文章，或在同步编辑器中载入分发稿，选择本次要同步的平台。
3. 确认标题、正文、配图、代码块和原文链接，再提交草稿。
4. 打开每个平台返回的编辑页，逐项检查内容；图片检查以实际可见的图片和保存状态为准。
5. 缺图时回到同一篇草稿补图，检查编辑器保存状态，收起扩展弹窗后刷新确认。
6. 按当次发布安排完成公开发布，并记录正式文章地址和稿源版本。

本次内核学习文章包含 4 张照片和 1 张学习进度图。CSDN 显示正常；知乎在扩展报告成功后漏掉全部 5 张图，通过编辑器的图片剪贴板方式补齐。这个结果要求同步后逐平台验收，尤其是图文混排、目录树和流程图。

作者随后确认完成了两平台最终发布。平台编辑入口记录如下，后续更新应先定位已有文章：

| 平台 | 本次编辑入口 |
| --- | --- |
| CSDN | [内核学习文章编辑页](https://editor.csdn.net/md?articleId=167177190) |
| 知乎 | [内核学习文章编辑页](https://zhuanlan.zhihu.com/p/2090904717346002597/edit) |

这次跨平台同步范围是内核学习方法这一篇；本站整理和部署范围是 17 篇文章。临时分发清单保留了当时的草稿状态，不能据此判断作者后来完成的最终发布状态。

### CLI 和后续平台

Wechatsync 官方提供 CLI 和浏览器桥接方式。连接条件就绪后，可使用下面的命令；本次已验证预检，真实 CLI 分发仍需单独跑通。连接和命令参数以 [官方 CLI 文档](https://github.com/wechatsync/Wechatsync/blob/v2/packages/cli/README.md) 为准。

```bash
wechatsync platforms --auth
wechatsync sync article.md -t "文章标题" -p csdn,zhihu --dry-run
wechatsync sync article.md -t "文章标题" -p csdn,zhihu
```

CLI 需要浏览器扩展连接，以及扩展设置中一致的桥接 Token；仅安装 CLI 或在浏览器登录平台不能证明桥接已可用。Token 留在运行环境中，流程文档只记录配置项名称。

除本次使用的平台，官方还列出掘金、博客园、思否、开源中国、51CTO、语雀、简书、头条、B站、WordPress 和 Typecho 等。支持列表和适配状态会变化，新平台先用一篇代表性文章试跑，完整范围见 [官方支持列表](https://github.com/wechatsync/Wechatsync/blob/v2/README.md#支持-29-主流平台)。其中 Hugo、Hexo 条目是 Markdown 下载支持，本站部署继续使用 GitHub Pages。

每次分发保存日期、文章 slug、Git 提交或稿源摘要、目标平台、编辑页或正式网址、图片核验结果和状态。把「预检通过」「草稿已创建」「草稿已核验」「已公开发布」分别记录，失败重试前检查已有文章，避免重复创建。

## 旧博客合并

本次将 `_pphome/blog/` 合并到博客仓库。现有 9 篇文章保留，8 篇旧文均未重复，新增后共 17 篇；先迁入 16 张共用封面，后续选图又补充了素材，当前封面目录有 19 个文件。

迁移流程如下：

1. 盘点旧文和现有文章，结合标题与正文比较重复内容，不能只按文件名判断。
2. 备份原目录并保存迁移清单。新内容建立英文 slug 和 page bundle；重复稿在确认本站内容和素材完整后清理。
3. 将封面集中迁入 `assets/img/covers/`，正文配图迁入文章目录，改写原有图片引用。
4. 补齐摘要、日期、分类和标签，保留旧稿技术版本的实际语境。
5. 构建核对文章数量、图片、导出文件和 URL，再清理已迁入的旧目录。

旧稿没有声明发布时间，本批使用旧目录首次进入 Git 的作者时间 `2026-04-14T15:18:15+08:00`，它是迁移时采用的日期基线。后续找到真实发表时间时再修正，新增文章不套用这个日期。

## 本次记录与临时资料

2026 年 10 月 6 日完成本站更新：17 篇文章、7 个分类、40 个标签，站点标题为「宋明河」。[本次 GitHub Pages 部署](https://github.com/s-infinite-box/s-infinite-box.github.io/actions/runs/37478273336)成功，随后核验了线上分类、学习方法标签、新文章和封面图片。

下列路径相对于 `/home/wz/p/pphome`，用于本地追溯；日常操作以当前仓库正文和配置为准。

| 本地资料 | 内容 |
| --- | --- |
| `.tmp/wechatsync/aa-kernel-learning-method-distribution.zip` | 本次分发稿与配图包 |
| `.tmp/wechatsync/article/manifest.json` | 稿源摘要、图片清单、平台编辑页和草稿核验记录 |
| `.tmp/wechatsync/prepare_article.py`、`render_svg.cjs` | 本次文章导出和 SVG 转 PNG 脚本，源路径与运行库路径为本机固定值 |
| `.tmp/blog-merge-2026-10-06/legacy-blog-backup.zip` | 迁移前 25 个文件的备份 |
| `.tmp/blog-merge-2026-10-06/migration-report.json` | 旧稿到新目录的迁移关系 |
| `.tmp/blog-taxonomy-change/before.json` | 分类标签调整前的元数据与正文快照 |
| `.tmp/blog-release-2026-10-06/` | 发布快照、构建结果、验证记录和部署状态 |

临时产物统一放在父项目 `.tmp/`，长期流程文档放在博客 `docs/`。旧分发包不会随正文自动更新；再次发布时生成与当前稿源匹配的新包。

## 文档与 skill 的选择

当前先维护本文和 README。它们跟随博客仓库版本变化，作者和其他协作者都能直接阅读；分类、路径和封面规则有明确入口。把完整流程另复制进 skill，会增加两份内容不一致的维护成本。

| 载体 | 适合保存的内容 | 当前安排 |
| --- | --- | --- |
| README | 常用命令、目录说明、分类规则和流程入口 | 作为项目入口 |
| 本文 | 完整步骤、迁移经验、多平台核验和问题处理 | 作为复用流程 |
| Makefile 与脚本 | 构建、预览、导出和检查等可重复执行的动作 | 优先沿用已有命令；文章导出脚本可后续参数化 |
| 轻量 skill | 触发条件、任务范围、读取哪些项目资料、调用哪些命令 | 后续需要持续由 AI 执行时再增加 |

skill 的收益在于让 AI 自动找到项目规则并执行合适的步骤。写作、封面、分类和本站发布已经具备封装条件；后续频繁由 AI 执行这些任务时，可以先增加 `blog-workflow` skill：只保存项目入口、按任务读取本文的方式和输出要求，详细规则继续维护在仓库文档中。

扩展到多平台自动分发前，值得先将分发脚本的文章目录和输出目录改为参数，并跑通真实 CLI 桥接、失败恢复和已有文章更新方式。Wechatsync 已有 [官方 skill 示例](https://github.com/wechatsync/Wechatsync/blob/v2/skills/wechatsync/SKILL.md)，平台部分可先评估其适配性，再补本站的约定。

下一次复用时，可以直接提出：

> 先阅读 README 和 docs/blog-workflow.md，按当前分类与封面约定整理这篇文章，启动本地预览。需要分发时生成对应稿件；公开发布按我本次指定的站点和平台执行。

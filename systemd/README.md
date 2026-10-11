# 开机启动博客预览

## 服务行为

`blog-preview.service` 是 systemd 用户服务，复用仓库的 `make serve-drafts`，默认预览地址为 <http://localhost:1313/>，包含草稿。Hugo 监听文章、主题和配置变化，修改后自动重建并通过 LiveReload 刷新已打开的页面。

服务使用前台进程，异常退出后每 5 秒重试，日志由 journal 保存。手动停止服务后不会自动重启。服务挂到用户的 `default.target`；启用用户 linger 后，可在电脑开机时启动，并在退出桌面后继续运行。它不需要 KDE 启动脚本，也不自动打开浏览器。

## 安装前提

- 本机已安装 systemd、GNU Make 和 Git。
- 博客工作目录为 `$HOME/p/pphome/_sub_mod/SongMingHe`。
- Hugo extended 安装在 `$HOME/.local/share/hugo/bin/hugo`，默认使用用户维护的这一二进制。显式路径避免服务依赖交互式 Shell 的 PATH 或临时目录。
- 1313 端口空闲。若已有终端启动的预览，先在对应终端按 Ctrl+C 停止，再由服务接管。

inft 的博客目录所在磁盘已通过 `/etc/fstab` 开机挂载，`wz` 用户已启用 `Linger=yes`。本机安装和验收结果见 [验证记录](验证记录.md)。

安装前核对默认二进制：

```bash
test -x "$HOME/.local/share/hugo/bin/hugo"
"$HOME/.local/share/hugo/bin/hugo" version
```

版本输出应包含 `extended`。本地预览版本由此路径的二进制决定，CI 版本继续由 `.github/workflows/hugo.yml` 管理。更新 Hugo 后应重新核对预览内容。

## 安装与启用

配置以博客仓库中的文件为准，安装时按 `0644` 权限复制到用户服务目录。inft 的仓库位于 NTFS 挂载中，文件呈现为统一权限；复制到用户目录可避免 systemd 对配置文件可执行、全局可写的权限提示。

```bash
blog_root="$HOME/p/pphome/_sub_mod/SongMingHe"
mkdir -p "$HOME/.config/systemd/user"
install -m 0644 "$blog_root/systemd/blog-preview.service" \
  "$HOME/.config/systemd/user/blog-preview.service"
systemd-analyze --user verify "$HOME/.config/systemd/user/blog-preview.service"
systemctl --user daemon-reload
systemctl --user enable --now blog-preview.service
```

检查开机驻留设置：

```bash
loginctl show-user "$USER" -p Linger
```

inft 已是 `Linger=yes`，无需重复设置。其他机器若为 `Linger=no`，需执行 `sudo loginctl enable-linger "$USER"` 才能在未登录桌面时开机运行用户服务。

## 日常管理与验收

```bash
systemctl --user status blog-preview.service
journalctl --user -u blog-preview.service -n 50 --no-pager
journalctl --user -u blog-preview.service -f
systemctl --user restart blog-preview.service
systemctl --user stop blog-preview.service
systemctl --user start blog-preview.service
```

安装后确认 `systemctl --user is-enabled blog-preview.service` 返回 `enabled`，`systemctl --user is-active blog-preview.service` 返回 `active`。浏览器打开 <http://localhost:1313/>，核对首页及文章内容；保存一篇文章的修改，确认页面更新。

```bash
curl --fail --silent --show-error --output /dev/null http://127.0.0.1:1313/
```

下次正常重启电脑后再次检查服务状态和页面，确认开机自动启动。若启动失败，查看 journal：端口占用、工作目录尚未挂载、Hugo 路径不存在或文章配置错误都会在日志中体现。

服务直接执行 Makefile 预览命令。新增文章、修改正文及删除正文中的段落都无需重启，自动更新与 LiveReload 已实测通过。按用户决定，删除整篇文章或修改文章路径后，执行下面的命令清除旧 URL：

```bash
systemctl --user restart blog-preview.service
```

服务保留已授权的 `HUGO_FLAGS=--cleanDestinationDir`。排查确认，Hugo 0.167.0 的普通文章变更走增量重建，该参数不会在这一流程中主动清除旧页面输出。当前采用手动重启处理删除场景，详见 [验证记录](验证记录.md)。

## 调整配置

修改本目录的 `blog-preview.service` 后，将配置同步到安装副本并重启：

```bash
install -m 0644 "$HOME/p/pphome/_sub_mod/SongMingHe/systemd/blog-preview.service" \
  "$HOME/.config/systemd/user/blog-preview.service"
systemd-analyze --user verify "$HOME/.config/systemd/user/blog-preview.service"
systemctl --user daemon-reload
systemctl --user restart blog-preview.service
```

- 更换检出位置时，修改 `WorkingDirectory`，并按新仓库路径同步服务配置。
- 不需要草稿时，将 `ExecStart` 的 `serve-drafts` 改为 `serve`。
- 更换端口时，同时修改 `PORT` 和 `PREVIEW_URL`，例如 `PORT=1314 PREVIEW_URL=http://localhost:1314/`。
- 升级默认路径下的 Hugo 后，重启服务即可加载新版本；若改用其他安装路径，则修改 `ExecStart` 的 `HUGO` 参数并同步配置。

服务默认只监听 `127.0.0.1`。Hugo 缓存沿用 Makefile 的 `resources/_gen/hugo_cache`，已由仓库忽略提交；`--renderToMemory` 使预览页面在内存中渲染。

## 停用与卸载

```bash
systemctl --user disable --now blog-preview.service
systemctl --user reset-failed blog-preview.service
rm "$HOME/.config/systemd/user/blog-preview.service"
systemctl --user daemon-reload
```

卸载命令只移除用户服务的安装副本，仓库配置保留。linger 同时供其他用户服务使用，停用博客预览时保留该设置。

## 参考

- [Hugo 预览与 LiveReload](https://gohugo.io/getting-started/usage/)
- [Hugo server 参数](https://gohugo.io/commands/hugo_server/)
- [systemd 重启策略](https://github.com/systemd/systemd/blob/v259/man/systemd.service.xml)
- [用户 linger 的开机行为](https://github.com/systemd/systemd/blob/v259/man/loginctl.xml)

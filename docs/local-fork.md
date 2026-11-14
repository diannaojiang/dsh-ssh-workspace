# 本地 fork：本仓库是 `@shiqyka/dsh-ssh-workspace` 的本地源码树

> 状态（2026-11-14）：基于上游 **v0.3.3**（`0cd4caf`）的 fork，分支 `local` 带 2 个本地提交。
> 运行中的 `web` profile 以 `link:` 依赖指向本目录——**改这里的文件就是改正在运行的插件**，
> 系统里不存在第二份被手工改过的安装副本。

## 1. 为什么要 fork

上游发布到 npm 的包只带 `lib/`，而 `lib/` 就是它的源码（无打包步骤：`package.json` 没有
`build` 脚本，`lib/client.js` 顶部明确写着"This is ONE bundle… this plugin deliberately has no
build step"）。本部署需要两处上游没有的改动，之前的做法是直接改
`$DSH_HOME/profiles/web/node_modules/@shiqyka/dsh-ssh-workspace/lib/*.js`：

| 问题 | 直接改安装副本 |
| --- | --- |
| 上游升级 | 文件被覆盖，补丁静默消失；只能靠一个 Python 脚本事后重打 |
| 追溯 | 没有 diff、没有提交信息、没有版本号 |
| 回滚 | 只能靠 `.orig` 备份 |
| 排障 | 分不清"上游行为"和"我们的改动" |
| 系统提示 | 插件把自己描述成上游版本，实际行为不同 |

fork 之后：改动进 git 提交，`git log` 能区分上游与本地，升级走 rebase，profile 用
`link:` 指过来，编辑器里改的就是运行时读的那一份。

## 2. 安装形态

`$DSH_HOME/profiles/web/package.json`：

```json
"dependencies": { "@shiqyka/dsh-ssh-workspace": "link:/root/Codes/多主机插件" }
```

`$DSH_HOME/profiles/web/node_modules/@shiqyka/dsh-ssh-workspace` → 符号链接到本目录。
`ssh2` 这类运行时依赖装在本仓库的 `node_modules/` 里（`link:` 不会代装被链接包的依赖）：

```bash
npm ci --omit=dev --cache ./.npm-cache    # 注意 --cache：本机沙箱不允许写 /root/.npm
```

## 3. 分支与上游同步

```bash
git log --oneline            # 0cd4caf 上游 v0.3.3 → 上面两条即本地提交
git fetch origin --tags      # 取上游新版本
git rebase v0.3.x            # 把两个本地补丁搬到新上游上；冲突通常只在下面两个锚点
npm run lint && npm test     # 补丁失效时测试/语法检查会先叫
```

约定：**上游提交保持原样，本地改动一律单独成 commit**，提交信息里写清"为什么上游需要这处
改动、什么时候可以删"。

## 4. 本地补丁登记

格式参照上游的 [upstream-workarounds.md](./upstream-workarounds.md)：稳定 id 可全仓库
`grep`，代码里留 `local fork runbook` 标记，注销判据写在这张表里。

| id | 文件 | 症状 | 本地修法 | 什么时候可以删 |
| --- | --- | --- | --- | --- |
| `LOCAL-1-directory-flow-shadow` | `lib/client.js` | 该部署的目录选择器只组装 `browse` 能力，而插件以 `priority: -100` 抢占"添加工作区"槽位；其"本地"分支调用 `uiWorkspace.pickDirectory()`，报 `directoryPicker.pick needs the native capability`，同时把可用的原生 browse 面板盖住 | 用 `REGISTER_DIRECTORY_FLOW = false` gate 掉整个 `directoryFlow` 注册，让"添加工作区"完整交回 DSH 原生面板；远程工作区仍在插件面板（主机行→浏览目录→挂载此目录）与 `ssh_workspace_mount` 里 | 插件的"本地"分支能在 browse-only 部署里复用原生 browse 面板（或 DSH 的 picker 组合提供 `native`）时；删掉 gate 并把该常量设为 `true` |
| `LOCAL-2-sandbox-wrapper-prefix` | `lib/patch-subprocess.js` | `workspace-write` 下 DSH 把命令包成 `[landlock-run, …grants, '--', cmd, …]`，桥只按 `argv[0]` 的 basename 做 git/rg 白名单，于是抛 `cannot run …/landlock-run with a remote ssh:// cwd — only git and ripgrep (rg) are supported` | 按沙箱约定在 `'--'` 处剥掉启动器前缀再走白名单；只对 `landlock-run` / `bwrap` / `sandbox-exec` 生效，**不**改成"找第一个 git/rg"（会误匹配 `--ro-bind /usr/bin/rg` 这类授权路径） | 宿主给 `ctx.subprocess` 提供"逻辑 argv"（沙箱包装对 remote 目标自动跳过），或桥能拿到未包装的命令时 |

两条补丁都**只在本部署的宿主行为下**成立：被剥掉的沙箱保护的是本机进程，而这些进程本来就在
远端执行。

## 5. 改完怎么生效

| 改动 | 生效方式 |
| --- | --- |
| `lib/client.js`（浏览器侧） | 刷新页面（客户端模块由 Host 从本目录读取；HMR 接收器在，但自动热更未验证，刷新最稳） |
| 其它 `lib/*.js`（Host 侧） | 插件安装/卸载会即时重载；**直接改文件**不一定被感知。没生效就 `systemctl restart dsh-web`（会断开当前页面连接，重连即可；会话与工作区记录保存在 `$DSH_HOME`） |
| `package.json` 依赖 | `npm ci --omit=dev --cache ./.npm-cache`，再重启 host |

重启命令（本机部署方式见 `/etc/systemd/system/dsh-web.service`）：

```bash
systemctl restart dsh-web && journalctl -u dsh-web -n 20 --no-pager
```

## 6. 验证

```bash
npm run lint                 # 156 个模块可解析
npm test                     # 77 passed / 0 failed / 6 skipped（缺真机环境的套件自动跳过）
DSH_WS_TEST_ALIAS=910b-05 \
DSH_CORE_ROOT=/usr/local/lib/node_modules/@deepseek-ai/dsh/node_modules \
  node tests/live-smoke.mjs  # 真机 SFTP 冒烟：55 passed，只在远端 /tmp 建临时目录并自清理
```

安装侧自检：`project_manager` 给的 `application: applied` 表示宿主已按新文件重载；
`cordis_inspect_query`/`ssh_workspace_status` 可确认工具在。

## 7. 回滚

```bash
# 回到上游发布版（会同时删掉 profile 里的 link: 依赖）
#   plugin_manager remove_bundle  target: @shiqyka/dsh-ssh-workspace
#   plugin_manager install_bundle target: @shiqyka/dsh-ssh-workspace@0.3.3
```

补丁前的现场备份在 `/tmp/shiqyka-backup/`（profile 的 `package.json` / `cordis.patch.yml` /
`pnpm-*.yaml`、打过补丁的 0.3.3 安装副本、`dsh-ssh.json`）。**`/tmp` 重启可能被清空**，
需要长期保留就复制到别处。

## 8. 数据位置（不随插件装卸变化）

- `$DSH_HOME/dsh-ssh.json`：主机库（本机 31 台）、密钥引用、`known_hosts`、最近使用。
- `$DSH_HOME/storages/workspace.json`：工作区登记；`ssh://<alias>/<path>` 形式的工作区
  依赖本插件提供服务，插件缺席时它们不可用（记录仍在）。

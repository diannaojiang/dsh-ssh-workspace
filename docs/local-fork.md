# 本地 fork：本仓库是 `@shiqyka/dsh-ssh-workspace` 的本地源码树

> 状态（2026-11-14）：基于上游 **v0.3.3**（`0cd4caf`）的 fork，分支 `local` 带 3 个本地补丁提交
> （另有 1 个登记文档提交）。
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
git log --oneline            # 0cd4caf 上游 v0.3.3 → 上面三条即本地补丁提交（第 2 条已被第 3 条取代）
git fetch origin --tags      # 取上游新版本
git rebase v0.3.x            # 把本地补丁搬上新上游；flow 的两条是同一处锚点，冲突以最新一条为准
npm run lint && npm test     # 补丁失效时测试/语法检查会先叫
```

约定：**上游提交保持原样，本地改动一律单独成 commit**，提交信息里写清"为什么上游需要这处
改动、什么时候可以删"。

## 4. 本地补丁登记

格式参照上游的 [upstream-workarounds.md](./upstream-workarounds.md)：稳定 id 可全仓库
`grep`，代码里留 `local fork runbook` 标记，注销判据写在这张表里。

| id | 文件 | 症状 | 本地修法 | 什么时候可以删 |
| --- | --- | --- | --- | --- |
| `LOCAL-1-directory-flow-two-step` | `lib/client.js` | 该部署的目录选择器只组装 `browse` 能力；上游的"添加工作区"占用 `directoryFlow` 槽位（`priority: -100`）后，"本地"分支先调 `uiWorkspace.pickDirectory()`，报 `directoryPicker.pick needs the native capability`，同时把可用的原生 browse 面板盖住 | 恢复 `directoryFlow` 双槽位注册：第一步是本地/远程二选一菜单；**本地**不再先调 picker，而是从 `ctx.slots.entries(hole)` 取已安装 browse occupant 自己的组件与注入（同一个对话框、同一套 locale 与列目录调用）按 owner 对话渲染，只有部署里没有 browse occupant 时才回落 `pick()`（桌面 preload 桥 / `uiWorkspace.pickDirectory()`）；**远程**保持原有主机选择 + 远程目录浏览 + 挂载 | DSH 给 flow occupant 提供正式的"交给已安装 browse 流"入口（例如 picker 服务的 `browse.open`、或单槽位的"让位给下一个 occupant"API），或上游"本地"分支自己能识别能力并复用 browse 时；届时删掉 `resolveBrowseFlow` 辅助与 `localFlow` 注入，本地分支回到 `pick()` |
| `LOCAL-2-sandbox-wrapper-prefix` | `lib/patch-subprocess.js` | `workspace-write` 下 DSH 把命令包成 `[landlock-run, …grants, '--', cmd, …]`，桥只按 `argv[0]` 的 basename 做 git/rg 白名单，于是抛 `cannot run …/landlock-run with a remote ssh:// cwd — only git and ripgrep (rg) are supported` | 按沙箱约定在 `'--'` 处剥掉启动器前缀再走白名单；只对 `landlock-run` / `bwrap` / `sandbox-exec` 生效，**不**改成"找第一个 git/rg"（会误匹配 `--ro-bind /usr/bin/rg` 这类授权路径） | 宿主给 `ctx.subprocess` 提供"逻辑 argv"（沙箱包装对 remote 目标自动跳过），或桥能拿到未包装的命令时 |

| `LOCAL-3-undefined-env` | `lib/patch-subprocess.js` | Forge 面板用 16 个 `GIT_* = undefined` 清掉继承的 Git 选择器。Node 的 spawn 把 `undefined` 当"不设置变量"，远端桥却把 env 序列化成 `export KEY=...` 并用 `String(value)` 强转，于是导出 `GIT_DIR='undefined'`；远端 git 把 `GIT_DIR` 当 `./undefined`，exit 128 `fatal: not a git repository: 'undefined'`，面板显示「当前目录不是 Git 仓库」 | `buildScript()` 的 env 循环跳过 `undefined` 值（保持 Node 的 unset 语义）；`null` 等其它值保持原有强转语义，argv/cwd 翻译不动 | 上游 `buildScript()` 同样跳过 `undefined`（或宿主给桥提供"逻辑 env"）时；删补丁 + `tests/remote-git-env-smoke.mjs` + `docs/remote-git-diagnosis.md` 登记 |

**退役记录**：`LOCAL-1-directory-flow-shadow`（用 `REGISTER_DIRECTORY_FLOW = false` 整个关掉
`directoryFlow`）已退役——它解决了"报错"，但代价是二选一菜单和远程建工作区入口一起消失。
现在由上面 `LOCAL-1-directory-flow-two-step` 取代，upstream 的 flow 注册与远程分支重新生效。

三条有效补丁都**只在本部署的宿主行为下**成立：被剥掉的沙箱保护的是本机进程，而这些进程本来就在
远端执行；browse occupant 的组件只在本部署的 DSH 组合里注册，且只经它的**原始**槽位条目
（`component` + `inject`）复用，不复制对话框实现；`undefined` env 的语义差只存在于"本地 spawn →
远端 `export`"这条转换里。

## 5. 改完怎么生效

| 改动 | 生效方式 |
| --- | --- |
| `lib/client.js`（浏览器侧） | 浏览器模块图在宿主启动时按文件内容算 `rev` 并缓存包体，刷新页面即可拿到新版本（浏览器若仍用旧缓存就 Ctrl+Shift+R） |
| 其它 `lib/*.js`（Host 侧） | **必须重启宿主**：本 profile 的 hmr 配置是 `root: []`（`@deepseek-ai/dsh-base/cordis.patch.yml` 的 `hmr` 行），宿主侧源码不做热重载，运行中的进程一直持有旧模块——刷新浏览器不够 |
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

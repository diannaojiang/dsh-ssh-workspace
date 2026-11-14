# 远程工作区 Forge 面板误报「当前目录不是 Git 仓库」：诊断与修复

> 一句话定位：宿主 Forge 面板用 `env: { GIT_DIR: undefined, … }` 清掉继承的 Git 选择器，
> 本地 `spawn` 把 `undefined` 当作「不设置该变量」，而插件的远端桥把它序列化成
> `export GIT_DIR='undefined'`，远端 git 于是去读 `./undefined` 并以 128 退出——
> 面板把「git 说这里不是仓库」当成了「这里确实不是仓库」。本页记录证据、
> LOCAL-3 补丁（`lib/patch-subprocess.js`）与退役判据。

- 状态：已修复（本地补丁 `LOCAL-3-undefined-env`，2026-11-14）
- 影响面：所有 `ssh://` 远程工作区的**服务端** Forge 面板检查（本地工作区不受影响）
- 上游基线：`@shiqyka/dsh-ssh-workspace` v0.3.3（`0cd4caf`）；宿主 `@deepseek-ai/dsh` 0.2.2-rc.1
- 相关：[local-fork.md](./local-fork.md)（本地补丁登记与升级流程）、
  [troubleshooting.md](./troubleshooting.md)、
  [upstream-workaround-pattern.md](./upstream-workaround-pattern.md)（退役模式）

## 1. 现象

在 DSH Web UI 里打开远程工作区 `ssh://910b-05/mnt/data/Codes/vllm-ascend`，
Forge 面板显示「当前目录不是 Git 仓库」，而该远端目录确实是 Git 仓库：

```console
$ ssh 910b-05 'cd /mnt/data/Codes/vllm-ascend && git rev-parse --show-toplevel && git rev-parse --abbrev-ref HEAD'
/mnt/data/Codes/vllm-ascend
fix/rwkv7-api-compat
```

该中文串的唯一来源是宿主 Forge 面板的 `checkout.none` 文案
（`@deepseek-ai/dsh-experimental-client-ui-forge/lib/client.js:8450`）；
服务端用 `RemoteError('forgePanel/not-a-checkout')` 映射到它
（`…/dsh-experimental-client-ui-forge/lib/index.js:3082`，
客户端映射见 `client.js:6298`，渲染见 `client.js:8036-8038`）。
因此这条提示**只**来自 Forge 面板的 checkout 检查，与「本地变更」列表
（`dsh-workspace-changes`）的判空不是同一条路径。

## 2. 链路：哪条 git 调用、什么 spec

面板每次检查都走这段服务端代码：

1. `ForgePanel.inspectOnce()`（`dsh-experimental-client-ui-forge/lib/index.js:1739`）
   以 Session 的当前目录调用 `inspectCheckout()`；该目录就是会话 cwd 字符串
   `ssh://910b-05/mnt/data/Codes/vllm-ascend`。
2. `inspectCheckout()`（同文件 `:219`）先跑
   `git rev-parse --show-toplevel`，`exitCode !== 0` 且 stderr 匹配
   `NOT_A_CHECKOUT = /not a git repository|must be run in a work tree/i`
   （同文件 `:203`）时返回 `undefined`（`:221-223`）→ `checkout.state = "none"`
   （`:1741`）→ 客户端显示 `checkout.none`。
3. `gitIn()`（`:336`）→ `runGit()`（`:359`）用**Session 的沙箱策略**包装命令，
   再交给 `ctx.subprocess.spawn`（`:378-392`）。它显式传的环境是：

   ```js
   env: {
     ...GIT_ENV,                    // 16 个键，值全是 undefined
     GIT_CONFIG_COUNT: String(overrides.length),
     // …overrides 的 GIT_CONFIG_KEY_n / GIT_CONFIG_VALUE_n
   }
   ```

   `GIT_ENV`（`:172-195`）由 `Object.fromEntries([…].map(key => [key, void 0]))`
   构造，`GIT_DIR`、`GIT_WORK_TREE`、`GIT_COMMON_DIR`、`GIT_INDEX_FILE`、
   `GIT_OBJECT_DIRECTORY`、`GIT_ALTERNATE_OBJECT_DIRECTORIES` 等 16 个键**存在但值为
   `undefined`**——注释写明用途是「清掉继承来的仓库/配置/替换引用选择器」。
4. `cwd` 是 `ssh://`，于是插件的 `ctx.subprocess` 接管生效
   （`lib/patch-subprocess.js:716-731`，只对 `isRemotePath(spec.cwd)` 分流），
   `buildScript()`（`:299`）把 spec 翻译成远端 shell 脚本。
   LOCAL-2 已保证 `[landlock-run, …grants, '--', git, …]` 被剥回逻辑 argv，
   所以命令确实在远端跑——这也解释了为什么报错从「桥拒跑 landlock-run」
   变成了「git 说不是仓库」。

## 3. 根因：`undefined` 环境值被序列化成字符串 `undefined`

Node 自己的 `child_process` 对 `env` 中值为 `undefined` 的项是**跳过**（等于不设置该变量），
所以本地工作区里这 16 个键不会到达 git：

```console
$ node -e 'const {spawnSync}=require("node:child_process");console.log(spawnSync(process.execPath,["-p","process.env.GIT_DIR"],{env:{...process.env,GIT_DIR:undefined}}).stdout.toString())'
undefined        # 打印的是字符串 "undefined"，即进程里根本没有 GIT_DIR
```

插件的远端半不是 `spawn`，而是 `export KEY=…`。修复前
`buildScript()` 的 env 循环（`lib/patch-subprocess.js:302-318`）只对**非字符串**值做
`String(value)`（`translateEnvValue`，`:269-271`），于是 `undefined` 变成字面量
`'undefined'` 导出：

```text
export GIT_DIR='undefined'; export GIT_WORK_TREE='undefined'; …; export LC_ALL='C'; …
cd '/mnt/data/Codes/vllm-ascend' && { …; exec 'git' … 'rev-parse' '--show-toplevel'; }
```

远端 git 把 `GIT_DIR=undefined` 当成相对于 cwd 的路径 `./undefined`，直接失败：

```console
$ ssh 910b-05 "cd /mnt/data/Codes/vllm-ascend && export GIT_DIR=undefined && git rev-parse --show-toplevel"
fatal: not a git repository: 'undefined'        # exit 128
```

（Forge 的 spec 还导出 `LC_ALL='C'`，所以面板侧看到的是英文
`fatal: not a git repository: 'undefined'`，正好命中 `NOT_A_CHECKOUT` 正则。）

一句话：**宿主用 `undefined` 表达「unset」，桥把它翻译成了「set 成字符串 undefined」**。
这不是 LOCAL-2 引入的回归，而是上游 v0.3.3 的桥在「本地 spawn 语义 ↔ 远端 shell 导出」
之间漏掉的一层语义；LOCAL-2 只是让它第一次有机会暴露成「不是 Git 仓库」。

## 4. 复现与证据

三层证据，从下到上：

### 4.1 远端事实（与会话无关）

```console
$ ssh 910b-05 'cd /mnt/data/Codes/vllm-ascend && git rev-parse --show-toplevel --absolute-git-dir --git-path objects; echo exit=$?'
/mnt/data/Codes/vllm-ascend
/mnt/data/Codes/vllm-ascend/.git
.git/objects
exit=0
```

### 4.2 桥生成的脚本（不连网也能看）

`tests/remote-git-env-smoke.mjs` 用 subprocess 服务替身 + engine 替身捕获
`record.client.exec()` 收到的脚本。回退 `lib/patch-subprocess.js:315` 的守卫再跑，
脚本里会出现 `export GIT_DIR='undefined' …` 且三条断言失败；
打上修复后脚本只剩已定义键（`GIT_CONFIG_COUNT` / `LC_ALL` / …）。

### 4.3 真实 SSH 端到端（本次诊断用的一次性 harness）

用同一个 `installRemoteSubprocess()`，把 `record.client.exec()` 换成真 `ssh -o BatchMode=yes 910b-05 <script>`
的通道替身，喂给桥的 spec 与 Forge `runGit()` 完全一致
（`argv = [git, --no-optional-locks, -c core.hooksPath=/dev/null, -c core.fsmonitor=false, rev-parse, --show-toplevel]`、
`cwd = ssh://910b-05/mnt/data/Codes/vllm-ascend`、`env = GIT_ENV + GIT_CONFIG_COUNT: '0'`）：

| | 桥脚本里的 GIT_DIR | 远端 exit | 面板结论 |
| --- | --- | --- | --- |
| 修复前 | `export GIT_DIR='undefined'` | 128，`fatal: not a git repository: 'undefined'` | `checkout.none`（当前目录不是 Git 仓库） |
| 修复后 | 不导出 | 0，stdout `/mnt/data/Codes/vllm-ascend` | checkout 就绪 |

修复后把 `inspectCheckout()` 需要的 7 条查询全部走桥跑一遍，全部命中允许的退出码：

```text
ok   rev-parse  exit=0 stdout="/mnt/data/Codes/vllm-ascend\n"
ok   common-dir exit=0 stdout="/mnt/data/Codes/vllm-ascend/.git\n"
ok   git-dir    exit=0 stdout="/mnt/data/Codes/vllm-ascend/.git\n"
ok   branch     exit=0 stdout="fix/rwkv7-api-compat\n"
ok   commit     exit=0 stdout="7f9fc618f\n"
ok   upstream   exit=0 stdout="origin/fix/rwkv7-api-compat\n"
ok   remote     exit=0 stdout="origin\thttps://github.com/TimeMobius/vllm-ascend.git (fetch)\n…"
```

## 5. 修复

`lib/patch-subprocess.js`（`buildScript()` 的 env 循环，`:302-318`）——只加一条「与本地语义对齐」
的守卫，外加注释和稳定 id：

```js
for (const [key, value] of Object.entries(env)) {
  if (!ENV_NAME_RE.test(key)) continue
  // local fork runbook: docs/remote-git-diagnosis.md · id: LOCAL-3-undefined-env
  if (value === undefined) continue      // ← LOCAL-3：Node 的 unset 语义
  segments.push(`export ${key}=${shellQuote(translateEnvValue(key, value, parts.alias, mkdirs))}`)
}
```

边界与取舍：

- **只跳过 `undefined`**，不跳过 `null`：Node 对 `null` 会强转成字符串 `"null"`，
  桥保持一致（这是本地/远端行为等价，不是本次缺陷）。
- 不动 `translateEnvValue()`：是否**发出**这条 export 是 `buildScript` 的决定；
  `translateEnvValue` 仍负责路径翻译，行为不变。
- 不动 LOCAL-2 的 argv 剥离、不动 argv/cwd 翻译、不动 `lib/client.js`（该文件由
  `ws-flow-ui` 负责，本修复不需要它）。
- 本地工作区零影响：本地 spec 不进这个桥，`undefined` 仍由 Node 自己丢弃。

## 6. 验证

```console
$ node tests/remote-git-env-smoke.mjs
=== 16 passed, 0 failed ===

$ npm run lint
all 157 modules parse (lib, scripts, tests)

$ npm test
=== run.mjs: 78 passed, 0 partial, 6 skipped, 0 failed (84 suite(s)) ===
```

防回归测试 `tests/remote-git-env-smoke.mjs`（已登记进 `tests/run.mjs` 的 `SUITES`，
紧跟 `phase6`）钉住四件事：

1. Forge 形状的 spec 里 `undefined` 值不得出现在远端脚本中（`GIT_DIR=` 等一个都不许有）；
2. 已定义值仍原样导出（`GIT_CONFIG_COUNT`、`LC_ALL`、`GIT_TERMINAL_PROMPT`…）；
3. cwd / argv 翻译不受影响（`cd '<remotePath>'`、`exec 'git' …`）；
4. 反证：把 `:315` 的守卫删掉再跑同一套件 → `13 passed, 3 failed`，失败项正是上面第 1 条。

`docs/development.md` 的套件表与生成区（`scripts/doc-numbers.mjs`）已同步：
套件数 84、新增 `remote-git-env-smoke` 一行。

## 7. 注销判据（什么时候可以删 LOCAL-3）

满足**任一**即可删：

1. **上游版本修好**：上游 `lib/patch-subprocess.js` 的 `buildScript()` 在序列化 env 时
   不再导出 `undefined` 值（等价于本补丁的 `if (value === undefined) continue`），
   且发布的版本 ≥ 该修复版本。升级/rebasse 到该版本后：
   ```bash
   grep -n "value === undefined" node_modules/@shiqyka/dsh-ssh-workspace/lib/patch-subprocess.js
   ```
   命中即视为上游已包含；同时用上游代码跑一次反向复现（删掉测试的 fork 指向，
   把 `tests/remote-git-env-smoke.mjs` 的 import 临时指向上游包）确认脚本里没有
   `'undefined'` 导出。
2. **宿主侧不再用 `undefined` 表达 unset**：`dsh-experimental-client-ui-forge` 的
   `GIT_ENV` 改成不包含这些键（或改成显式空值并自行处理），且 `runGit()` 不再把
   `undefined` 交给 `ctx.subprocess`。该信号比上游修桥更弱——桥仍应正确处理
   `undefined`（任何宿主代码都可能在 spec 里用它表达 unset），所以优先等条件 1。

退役步骤：删 `lib/patch-subprocess.js:305-315`（注释 + 守卫）→ 删
`tests/remote-git-env-smoke.mjs` → 从 `tests/run.mjs` 的 `SUITES` 移除 →
从 `docs/development.md` 套件表删行 → `node scripts/doc-numbers.mjs` 重生成 →
`npm run lint && npm test`。

## 8. 残留风险与未处理项

1. **宿主进程需要加载新代码才生效**。`dsh web`（systemd `dsh-web.service`）在本机
   07:00:56 启动，而插件 host 端模块是在启动时 `import` 的；浏览器端 `lib/client.js`
   的 HMR 只更新客户端 bundle，管不到 host 端的 `lib/patch-subprocess.js`。
   本次任务约定不重启宿主，因此**验证时如果面板仍报旧结论**，说明当前进程还持有旧模块，
   需要重启 `dsh-web.service`（或宿主自身的插件热重载）后才看得到修复。
   判断依据：修复生效后该面板应显示分支 `fix/rwkv7-api-compat`、commit `7f9fc618f`、
   remote `origin/…`，而不是「当前目录不是 Git 仓库」。
2. **Forge 的「本地变更」卡片在远端工作区仍不可用（与本次缺陷无关）**。
   `localChanges()` 把后续 git 绑定到 `inspection.checkout.root`（`…/index.js:755-760`），
   而 `root` 来自 `ctx.fs.processPath()`，远端目标被剥成不带 `ssh://` 的
   `/mnt/data/Codes/vllm-ascend`（本插件 `lib/remote-fs.js:434-440`）；这样的 cwd 不满足
   桥的分流条件（`lib/patch-subprocess.js:728`），会落到本地 spawn 并失败，
   而 `localOf()` 把该错误吞掉（`…/index.js:1805-1811`），于是卡片直接不显示。
   checkout 信息（分支 / commit / remote）不受影响。
3. **`dsh-workspace-changes`（Changes 记录器）在远端工作区同样有路径形状问题**：
   `locateGitWorkspace()` 对 `git rev-parse --git-path objects` 的相对输出
   `.git/objects` 做 `resolve(cwd, line)`（`dsh-workspace-changes/lib/types/git.js:87`），
   `cwd` 是 `ssh://…`，于是得到一个宿主本地绝对路径，随后作为
   `GIT_ALTERNATE_OBJECT_DIRECTORIES` 传给远端 git；后续快照命令又以
   `workspace.root`（同样是不带 scheme 的远端路径）为 cwd 派发，无法被桥接管。
   这是宿主侧实现与「远端执行世界」之间的形状不匹配，插件内无法在不猜测
   alias 的前提下安全修复（同一路径可能存在于多台主机）；本补丁不扩大范围。
4. **其它可能传 `undefined` 的宿主调用方**：修复对它们一律按 Node 语义「不导出」，
   不会比本地更宽松或更严格；数值/布尔等非字符串值仍按 `String()` 处理（与 v0.3.3 一致）。

## 9. 相关

- [local-fork.md](./local-fork.md)：fork 形态、补丁登记、升级流程
- [troubleshooting.md](./troubleshooting.md)：远程工作区常见错误与限制
- [upstream-workaround-pattern.md](./upstream-workaround-pattern.md)：本补丁采用的退役模式
- [architecture.md](./architecture.md)：`ctx.subprocess` 接管与远端执行世界

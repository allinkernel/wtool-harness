# 05 下一步

> 按顺序做。第 0 步不做完，后面每一步都会把下一次 publish 堵死。
>
> 最后对齐现实：2026-09-17（对着 `bootstrap` 的代码和 git log 核过一遍：
> 做完的标 ✅，没做的保留，部分做的写清"做到哪了"）。

---

## 0. `generated.tsv` —— 脏检查豁免 ✅ **已完成（2026-09-15）**

> 实现：`wt_atomic_write`（唯一的写文件入口）里统一登记到
> `$WTOOL_STATE/generated.tsv`；`wt_git_dirty` 把 `git status --porcelain`
> 减去登记过的文件。`cmd_publish` 改用它，被拒时打印真实改动的前几行。
> 已测两种情形：只有 wtool 写过的文件变了 → 放行；用户手改了别的 → 拒绝。
>
> 下面留原文，讲清楚它当初为什么必须最先做。


**问题**：`wtool publish` 会改写受版本控制的文件（`scripts/release.json`、
`scripts/pre_release.json`、`wtool-base/README.md` 的下载块）。改完这些文件就"脏"了，
**下一轮 publish 会以"有未提交改动"拒绝这个项目** —— 一次发布把下一次发布堵死。

**现状**：`cmd_publish` 里对 README 做了一次性特判（`_doc_path` + 只豁免那一个文件）。
要推广成通用机制。

**做法**：wtool 每次写文件时登记到 `$WTOOL_STATE/generated.tsv`（`路径<TAB>写入者<TAB>时间`）；
`cmd_publish` 的脏检查里，把 `git status --porcelain` 的结果减去这张表里的文件，
剩下的才算真脏。要保证**只豁免 wtool 自己写过的**，用户手改的内容照样算脏。

**验收**：连续跑两次 `wtool publish <项目>`，第二次不能报"有未提交改动"。

---

## 1. `release_mgr.py` + `scripts/release.json`（download 的整套机制）—— **未做**

### 已定的设计

**现状（2026-09-17）**：`bootstrap/lib/` 里**没有** `release_mgr.py`，
仓库里也一个 `release.json` / `pre_release.json` 都没有。
所以现在能 `download` 的只有自带 `scripts/download.sh` 的项目（目前只有
`editor/astronvim_v5`）；`wtool download` 对别的项目只会说"没有 scripts/download.sh，
这个项目只能自己编"。引擎里唯一认 `scripts/release.json` 的地方是
`cmd_bootstrap` 的就绪判断（见了就不要求先 build/download）—— 那是给这套机制预留的钩子。

- **`release_mgr.py` 住引擎**（`bootstrap/lib/`），静态工具。`publish.sh` 和 `download.sh`
  都要用它，每个项目各带一份 = N 份实现。
- **`scripts/release.json` 每个项目一份**，由**引擎生成**，脚本只声明元数据
  （`target` / `glibc` / `arch`）。sha256、URL、JSON 结构全归引擎 —— 加字段不用改所有项目。
- **`release.json` 存在 = 这个项目可以直接 download**。这是关键：文件提交在仓库里，
  所以 wtool 刚 clone 下来就知道有没有现成的包，**不需要查 GitHub、不需要本地状态**。
- **URL 是可预测的**：`https://github.com/<owner>/<repo>/releases/download/<tag>/<asset>`。
  所以 release.json 可以在上传**前**写好，它描述的是"我打算发什么、发到哪"。
  publish 照着它上传，两者天然一致。上传后用 `gh` 校验，不一致就不写 release.json。
- **选包按 glibc，不按发行版名字**：glibc 单向兼容（老环境编的能在新环境跑，反过来不行），
  所以选"glibc ≤ 本机"里最新的那个。只按 `ubuntu-24.04` 这种名字匹配的话，
  别的发行版会匹配到跑不起来的包。
- **`download.sh` 降级为可选覆盖**：通用逻辑（读 release.json → 选包 → 下载 → 解包 →
  写 artifacts）由引擎提供，只有解包方式特殊的项目才自带脚本（目前只有 astronvim_v5 的分卷 + payload）。

### 内容

```json
{
  "project": "editor/astronvim_v5",
  "repo": "allinkernel/wtool-astronvim_v5",
  "tag": "snapshot-2026-09-15",
  "commit": "b52b231...", "commit_time": "...", "dirty": false,
  "built_at": "...", "published_at": "...", "wtool_engine": "1.0.0",
  "targets": [{"target":"ubuntu-24.04","glibc":"2.39","arch":"x86_64","image":"ubuntu:24.04"}],
  "assets": [{"name":"...tar.zst","bytes":0,"sha256":"...","target":"ubuntu-24.04","kind":"payload"}],
  "verify": {"checked_at":"...","ok":true}
}
```

---

## 2. pre / post 两份记录 + 两个 diff —— **未做**

| 记录 | 放哪 | 含义 |
|---|---|---|
| `scripts/pre_release.json` | 项目里，**提交** | 最近一次**构建**产出了什么（哪怕上传失败了也在） |
| `scripts/release.json` | 项目里，**提交** | 发布页上**现在**有什么（上传校验通过才写） |
| `$WTOOL_STATE/<id>/{pre_,}build.json` 等 | 状态目录，**不提交** | 机器私有：这台机器编了什么 |

**两个 diff，各有各的用处**：

- `pre_release.json` vs `release.json` → 上传有没有按计划完成。不一致 = 有一次发布没走完
- 上次 `release.json` vs 这次 `pre_release.json` → **两次发布之间变了什么**
  → **写进 release notes**（`gh release create --notes`），发布页自己就说清了差异

**其他动作也照这个通则**：每个动作留"打算做什么"和"实际做了什么"两份记录。
install 的"实际"就是现成的 `journal.tsv`。

---

## 3. 同名 commit 重发的处理 —— **未做**（依赖第 1/2 条的 `release.json`）

准备发布时先比对 `release.json` 里记的 commit 和当前 HEAD：

- 相同 → 提示"发布页上的归档就是这个 commit 编的，内容不会有变化，确定重发吗？"
- **交互时问人**；**非交互（stdin 不是终端，比如被 wtool 在容器里调用）直接拒绝并要求 `--force`** ——
  和 astronvim 的 `publish.sh` 处理"目标系统"是同一套规矩，不会让脚本卡在等输入上。

---

## 4. `wtool docs sync` —— **部分完成**

扫全仓找带 `<!-- >>> wtool:downloads >>>` 标记的 Markdown 文件，用 **release.json 的数据**
（不再是 GitHub API，离线可用且和 download.sh 同一源头）重写标记之间的内容。
纯脚本、不碰 LLM。

- 支持**多个块各管各的**：标记带参数，如 `wtool:downloads project=astronvim_v5`
- 提成显式命令 `wtool docs sync`，`wtool publish` 结尾自动调

**现状（2026-09-17）**：机制已经有了（`wtool_plan.py` 的 `render_downloads` / `splice_block`），
也已经有 CLI 入口 —— 但名字不是 `docs sync`，而是
`wtool docs refresh`（`wtool refresh-downloads` 等价，`wtool docs` 不带参数也会刷新），
而且**这两个都不在 `--help` 里**。
数据源仍是 `gh` 查 GitHub（不是 `release.json`），标记也**不支持参数**（只有一个全局块）。
要做的是：换数据源 + 参数化多块 + 把命令名字和 `--help` 统一。
（顺带：`refresh-downloads` 是在 `e0a2950` 里为了能单独测"空结果不许洗掉下载表"才加出来的。）

---

## 5. 四目标矩阵

- 镜像已验：`ubuntu:20.04` focal glibc 2.31 / `22.04` jammy 2.35 / `24.04` noble 2.39 /
  `26.04` resolute 2.43（**rhel9 已放弃**，单独做）
- ⚠️ **代理很慢**：容器里 `apt-get install` 走宿主的 7897 代理约 171 kB/s，
  装一次构建依赖要十几分钟。这不是卡住，是慢 —— 别误判成死锁
- `publish.sh` **一次只做一个目标**：`--target=ubuntu-24.04`（单数）。
  矩阵写在 `wtool.xml` 的 `<publish><target .../>` 里（现在声明了
  focal / noble / jammy 三个），脚本从清单读，读不到才用内置的两个。
  **没有** `--targets=a,b` 这种一次多发的写法；也**没有"中断后续跑"** ——
  每次都是新容器，断了要重跑（`--keep` 只是保留容器方便事后排查）。
  断点续传只在下载侧（`download.sh` 的 `curl -C -`，按分卷保留）
- 构建时间：一次 astronvim 构建 40–90 分钟，串行四轮 3–6 小时。**并行不了** ——
  24G 内存同时跑两个容器就到顶
- 20.04 编出来的包在四个版本上都能跑，所以它最通用；另外三个是"原生环境 + 新工具链"。
  风险：focal 上某些 mason 包可能已经没有对应构建
- 容器必须 `--network=host`：宿主代理是 `http://127.0.0.1:7897`，
  而**容器里的 127.0.0.1 是容器自己**，直接透传 proxy 变量所有下载都会失败
  （`publish.sh` 会看代理变量自动切到 host 网络；容器脚本另有 `container-proxy.sh`
  自动探测，见 `AGENTS.md`）

---

## 6. 文档 —— **wtool-base 部分未做；harness 部分本轮做了**

### `wtool-base/`（给人看，**必须让所有人看懂**）—— **未做**

现状：`wtool-base/` 里只有 `README.md` / `guide.md` / `wtool.xml` / `LICENSE`，
**没有 `install.md`，也没有 `原理.md`**。README 里那个下载块是 wtool 自动生成的，
块**外面**还留着一段手写说明，而且已经过期（见下面第 8 节）。

- **不要出现本地路径**（`~/self/wtool` 这种），别人没有你的上下文
- **不要莫名其妙冒出一个名词**，概念第一次出现就要解释
- `README.md` 加一节指向 `install.md`
- `install.md`：从 `allinkernel/wtool-git-repo` 拿 repo 启动器 → `repo init` / `repo sync`
  → `wtool build` / `install` / `bootstrap` 的完整路线
  （`w_manifests` 目前是 PRIVATE，用户说先不用管，他后续考虑开源）
- 同步本轮所有变化：rc 收敛、`scripts/` 目录、四种状态、`all`、`wtool download`、
  `wtool init`、install/provision 的真实区别

### `原理.md`（用户要求新增）—— **未做**

讲清楚**每个命令的原理**，不限于：
- 为什么配置文件只放软链、系统里不留副本
- install 与 provision 为什么分开（用"没记账"的说法，不是"不可逆"）
- journal 和 actions 的区别
- rc 为什么要收敛到一个 loader 块
- 产物契约（build 和 download 必须同路径）
- 发布流水线（pre/post 记录、commit 绑定、按 glibc 选包）

### `harness/`（给助手看）—— ✅ **2026-09-17 这轮做完**

- `01-context.md`：结构树、契约表、状态目录、表格语义、当前状态都对着代码更新了
- `02-decisions.md`：补了 ADR-013 ~ ADR-018（含 ADR-010 被取代的说明）
- `03-hazards.md`：补了 H1 ~ H13（`71b4b0d`）

---

## 7. 本轮新踩的坑（要补进 03-hazards.md）—— ✅ **已完成（`71b4b0d`）**

已经全部进 `03-hazards.md` 的 H 节（H1–H13，另外补了 `$!`/`set -u`、
"别把一次失败当规律"两条）。下面这份清单保留作索引：

1. **`tar --transform` 会连符号链接的指向一起改写** → 包里的相对软链全变断链。
   加 `S` 标志。`tar -tf` 看不出来，只有真解压去读才暴露
2. **`set -e` 下 `x=$(失败的命令)` 会静默退出** → 第三方仓保护从来没生效过。
   放进 `||` 列表里
3. **`grep -c 'x' || echo 0`** → 匹配 0 时 grep 自己也输出 0，变成两行 "0"
4. **`${var// /}` 是 bashism**，脚本是 `#!/bin/sh`（dash）
5. **压缩器按"有没有装"选而不是按扩展名选** → 没装 zstd 时静默回退 gzip，
   但文件名还叫 `.tar.zst`，用户解不开会以为包坏了
6. **project id 要相对工作区根算**，不能靠 `os.environ['WTOOL_ROOT']` ——
   那个变量在 shell 里没 export，读不到会退化成 basename，嵌套项目全认错
7. **`scan_projects` 里"项目内部不再嵌套项目"的剪枝是错的** —— 项目可以嵌套
8. **助手函数污染调用方变量**（POSIX sh 没有 local）：`wt_pack_source` 的内部 `_out`
   覆盖了调用方的 `_out`，上传路径变成 `x.tar.zst/x.tar.zst`
9. **测试用真工作区跑会把真文档洗掉** —— 测试必须用临时工作区，并加一条断言防这个
10. **`cmd_uninstall` 只手工处理 plan 里的 `rc` 行，从没跑过 `wt_plan_exec`** ——
    新增的动作被静默丢掉
11. **改工作区时容器正挂着它**：脚本正被逐行读取，改一半会炸。先停容器再改
12. **约 6 次尝试都没测出 nvim `-j32` 的真实内存**：死在代理上
    （archive/security.ubuntu.com 502、GitHub TLS 中断），不是我该反复重试的事

---

## 8. 顺手要修的文档问题（都在 `harness/` 之外，别混进这个仓库的提交）

1. **`bootstrap/scripts/container-shell.sh:18`** 的注释写着
   "见 harness/doc/06-排错.md"，而 `harness/doc/` 整个目录在 `cebe7c9`
   （2026-09-15，"删掉用户文档，只留 agent 需要的东西"）里被删了。
   要么把注释改成指 `harness/notes/03-hazards.md` / `wtool-base/guide.md`，
   要么把那份排错文档找回来。
2. **`wtool-base/README.md`** 下载块**下面**的手写段落让人跑
   `./bootstrap/install.sh` —— 这个路径在 `scripts/` 迁移后不存在了
   （`bootstrap/` 下没有 `install.sh`）。现在应该是根目录入口 `./install.sh`，
   或者 `bootstrap/scripts/install.sh`。这段在自动生成的
   `wtool:downloads` 标记**之外**，`wtool docs refresh` 刷不到它，只能手改。
3. **`bootstrap/docs/spec.md` §2**（约 46–59 行）还在说 `templates/stub.sh`、
   `scaffold`，CLI 列表也只写到 install/uninstall/list/status/doctor/scaffold/validate。
   实际是 `templates/*.tpl`、命令叫 `init`（`scaffold` 只是告警别名），
   另外有 build/download/provision/publish/bootstrap/table/env。
   spec.md 是**权威契约**（按"契约优先"的规矩），要改得改它本身，
   不能只在 harness 里记一笔。
4. **`wtool uninstall` 没有 `--no-script`**（代码小缺口）：`cmd_uninstall`
   判断的是环境变量 `WTOOL_NO_SCRIPT=1`，而没有任何地方 export 它
   （`--no-script` 只被 `cmd_install` 认，且只存进局部变量）。
   要么给 uninstall 也加上这个参数，要么把这个判断删掉 —— 见 ADR-015 的括号。

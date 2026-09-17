# AGENTS.md

> 每次对话开始前先读这一份。它只讲**操作这个项目必须知道的事**，
> 不讲设计（设计看 `notes/01-context.md`）。

---

## 这个工作区是多仓库，根目录不是 git 仓库

```
~/self/wtool/          ← repo 客户端。**不是 git 仓库**，`git add` 在这里会静默失败
├── bootstrap/         ← 独立 git 仓库
├── wtool-base/        ← 独立 git 仓库
├── harness/           ← 独立 git 仓库
├── os/ubuntu/         ← 独立 git 仓库
├── shell/zsh/         ← 独立 git 仓库
└── ...                ← 每个项目一个仓库
```

**每个项目是独立的 git 仓库，用 `repo`（git-repo）统一管理。**
根目录只是它们拼在一起的地方，仓库清单在 `.repo/manifests/default.xml`。

### 因此

- **不要**在 `~/self/wtool` 里跑 `git add` / `git commit` —— 那里没有 `.git`，
  命令会失败，而且失败得很难看（`fatal: not a git repository`）或者被自己忽略
- **要提交就进到具体项目目录里**：`cd bootstrap && git add -A && git commit`
- 一个改动可能涉及多个仓库，**每个都要单独提交、单独推送**
- 各仓库的 remote 名不统一：手工 clone 的是 `origin`，
  repo 客户端建的是 `github`。推之前先 `git remote -v` 看一眼
- 有的项目是 repo 管理的，**HEAD 是 detached 的**（不是分支）—— 这是正常的，
  它们跟着清单里钉的 revision 走

### 清单仓

`.repo/manifests/` 指向**另一个仓库** `allinkernel/w_manifests`（分支 `wtool`），
`default.xml` 里写着所有项目、它们的 remote、以及根目录那几条软链（linkfile）。
**改项目列表或者根目录软链，要改那里。**

---

## 绝对不能做的事

| 别做 | 为什么 |
|---|---|
| 在 `~/self/wtool` 里跑 `repo` 命令 | 会就地重写这个客户端。已经毁过一次：`repo init` 把 `.repo/repo` reset 到旧版本，多仓库工具直接坏掉 |
| 容器正在跑的时候改工作区里它读的文件 | 容器把工作区只读挂着，脚本正被**逐行读取**，改一半会炸（实测报过 `Syntax error: ";;" unexpected`） |
| 在真 `$HOME` 上跑测试 | 测试必须用 `WTOOL_HOME`/`WTOOL_STATE` 指向临时目录。有一次测试用真工作区跑，把真的 `README.md` 洗掉了 |
| 用 `git add` 提交根目录 | 见上。根目录不是仓库 |
| 把 apt/编译说成"install" | 那是 `provision`，两条命令分开是有意的（理由见 `notes/01-context.md`） |

---

## 改完代码必须做的

```bash
cd bootstrap/tests && ./run_all.sh          # 5 组，146 条，应该全绿
cd editor/astronvim_v5 && ./tests/astronvim_test.sh   # 52 条
```

**容器相关的测试只能人工跑**（跑不了 docker 里那套完整流程时，
至少要人工确认两个容器脚本没被改坏）：

```bash
# 从零走一遍（什么都不装）
docker run --rm -it --network=host -v ~/self/wtool:/wtool:ro \
  ubuntu:24.04 bash /wtool/bootstrap/scripts/container-raw.sh
# 一步到位（装依赖 + 引擎 + bootstrap）
docker run --rm -it --network=host -v ~/self/wtool:/wtool:ro \
  -v wtool-apt-cache:/var/cache/apt \
  ubuntu:24.04 bash /wtool/bootstrap/scripts/container-shell.sh
```

**改完代码要同步文档**，两个地方，别混：
- `wtool-base/` —— **给所有人看**。不写本地路径、不写内部实现、名词第一次出现要解释
- `harness/` —— **给助手看**。写机制、写坑、写为什么

---

## 权威数据在哪

| 想知道 | 看 |
|---|---|
| 引擎有哪些命令 | `bootstrap/wtool.sh --help` |
| 当前项目表 | `python3 bootstrap/lib/wtool_plan.py publish-list --root .` |
| 全貌 + 三条核心契约 | `harness/notes/01-context.md` |
| 为什么这么设计 | `harness/notes/02-decisions.md` |
| 踩过哪些坑 | `harness/notes/03-hazards.md` |
| 下一步做什么 | `harness/notes/05-next.md` |
| 引擎的接口契约 | `bootstrap/docs/spec.md` |

---

## 脚本地图：哪个脚本是谁调用的

同名 `.sh` 很多，**改之前先确认它在哪一层**——改错层会"看起来生效了"但实际没跑。

### 第一层：工作区根目录（都是 linkfile 软链）

| 根目录 | 指向 | 作用 |
|---|---|---|
| `install.sh` | `bootstrap/scripts/install.sh` | 装 wtool **自己**。**不装任何项目**，做完就停 |
| `uninstall.sh` | `bootstrap/scripts/uninstall.sh` | 卸 wtool 自己 |
| `README.md` / `guide.md` | `wtool-base/` | 用户文档 |

**根目录的 `install.sh` 和项目的 `scripts/install.sh` 是两码事**，
这个混淆已经害过一次（改项目脚本时以为在改引擎）。

### `install.sh` 的四步（2026-09-17 重构）

用户明确要求：**它只负责让 `wtool` 这条命令能用，做完就停**，
把"装项目"交给 `wtool bootstrap`。理由：两件事的失败原因完全不同
（系统环境 vs 某个项目），混在一条命令里用户分不清该修哪一头。

```
第 0 步  准备运行环境  ← 探测发行版，派发给 install-<发行版><版本>.sh
第 1 步  自举引擎到 ~/.wtool/bootstrap
第 2 步  建工作区入口软链（install.sh / uninstall.sh / README.md / guide.md）
第 3 步  wtool install <引擎自己> --force   ← 让 wtool 出现在 PATH 里，到此为止
```

**发行版的差异这样组织**（加新发行版 = 加一个小文件）：

| 文件 | 内容 |
|---|---|
| `scripts/install-env.sh` | 共用逻辑：装包、挑源、git ownership、复查依赖 |
| `scripts/install-ubuntu20.sh` | 只写不一样的地方：`ENV_NAME` + `ENV_ANSIBLE="ansible"` |
| `scripts/install-ubuntu22/24/26.sh` | 同上，`ENV_ANSIBLE="ansible-core ansible"` |

`install.sh` 先 source `install-env.sh`（提供函数），再 source 选中的 profile
（只声明变量并调 `env_prepare`）。**profile 里不重复写装包逻辑。**

三件必须记住的事（都踩过）：
- **`ENV_ANSIBLE` 要按候选名依次试**：`ansible-core` 是 22.04 才有的包名，
  focal 上只有 `ansible`。写死一个名字 = 另一版本上 "Unable to locate package"
- **`git safe.directory` 必须在 git 装完之后设**：`container-raw.sh` 里
  曾经在 git 还不存在时设它，命令静默失败，用户后面照样撞 dubious ownership
- **第 3 步的输出绝不能吞**：这里原来是 `>/dev/null 2>&1`，
  install 因为工作区有改动而拒绝时用户看到一片安静 ——
  一个可能静默失败的安装步骤比会报错的更坏

### 第二层：引擎

| 文件 | 约束 |
|---|---|
| `bootstrap/wtool.sh` | 命令分发 + 所有"写"的动作 |
| `bootstrap/lib/wtool_plan.py` | **只算不写**（除 scratch）：扫项目、算计划、画表、算 env |
| `bootstrap/lib/wtool_fs.sh` | **只写不算**（除读 journal）：落盘、记账、链接、rc |
| `bootstrap/lib/wtool_os.sh` | 系统探测 / provision |

分成两半是有意的：计划能在动手之前完整算出来，`--dry-run` 才有意义。
**新增动作要同时改两边**（py 出计划、sh 执行），只改一边的表现是
动作被静默丢弃。

### 第三层：项目自己的 `scripts/`

| 文件 | 被谁调用 | 契约 |
|---|---|---|
| `build.sh` | `wtool build` | 产物落到最终位置 + 写 `$WTOOL_ARTIFACTS` |
| `download.sh` | `wtool download` | **和 build.sh 落到完全相同的路径** |
| `install.sh` | `wtool install`（引擎铺完 link/rc 之后） | 登记、软链、shell 集成 |
| `install.sh --uninstall` | `wtool uninstall`（**逆放 journal 之前**） | 撤掉自己装的实体 |
| `publish.sh` | `wtool publish` | 构建 + 打包 + 上传 |
| `extract.sh` | 手动（只有浏览器的机器） | 校验分卷、铺到 `$HOME`，**不装** |

**文件存在即能力声明**：`wtool_plan.py` 就是按 `scripts/<名字>` 在不在
来填表格那几列的。所以"新建一个空的 build.sh"会立刻让表格显示"可执行"。

### 容器脚本（两个，别搞混）

| 脚本 | 状态 |
|---|---|
| `container-shell.sh` | 装依赖 → 装引擎 → `wtool bootstrap` → 进 zsh。**一步到位** |
| `container-raw.sh` | **什么都不装**，只挂工作区 → 进 bash。等价于"刚 `repo sync` 完" |

`container-raw.sh` 刻意不装 python3 —— 装了 `wtool` 就能跑，
而真机器刚同步完时本来就没有。**测试"从零走一遍"必须用这个**，
用 `container-shell.sh` 测会把要验证的前提条件提前满足掉。

`container-raw.sh` **不设** `safe.directory` —— 它进去时 git 还没装，
设了也是静默失败（踩过）。它把这条命令放进"第 0 步"让用户跟着 git 一起装，
`install.sh` 则在自己的第 0 步里（装完 git 之后）自动设好。

---

## 项目脚本的三条硬契约

改任何 `scripts/*.sh` 之前先过一遍这个清单。**三条都踩过。**

1. **产物落在 `$WTOOL_PREFIX`（默认 `~/.wtool/usr`）下面。**
   不是 `~/.local`。引擎通过 `wt_run_project_script` 导出这个变量。
   `$HOME` 里只允许留**软链**（登记进清单，可撤销）。
   违反的后果：`wtool uninstall` 撤不掉（实测 plan 出 `actions: 0`），
   `$HOME` 被污染且没人知道是谁放的。astronvim_v5 犯过，
   详见 `notes/01-context.md` §3.1。

2. **`build.sh` 和 `download.sh` 必须产出完全相同的路径。**
   这是 `build+install` 与 `download+install` 等价的前提。

3. **shell 集成要靠"环境变量块"，而且要带全。**
   最容易漏的是 `PATH` —— 只写 `NVIM_APPNAME` 那种，表现为
   "装完了但敲命令是 command not found"，而 `~/.config` 里又看得到东西，
   看起来像装了一半。**$WTOOL_PREFIX/bin 一定要进 PATH。**

---

## 代理（这台机器）

宿主代理是 `http://127.0.0.1:7897`。

- **容器里 `127.0.0.1` 是容器自己**，直接透传 `HTTP_PROXY` 会让容器内所有下载失败。
  必须 `--network=host`（`publish.sh` 已经会自动判断）
- `archive.ubuntu.com` / `security.ubuntu.com` 走这个代理**经常 502**。
  容器里装包失败先换国内镜像（`mirrors.ustc.edu.cn`）
- GitHub 的 `git clone` 走这个代理会偶发 TLS 中断，重试或改用 tarball
- **代理对 GitHub 有时是坏的，而直连是好的**（实测直连 `api.github.com` 200/0.4s，
  走代理 `SSL_ERROR_SYSCALL`）。所以下载/上传都写成"先按现状试、失败后绕开代理"
- **`github.com` 这个域名可能整个不通，而 `api.github.com` 通**。
  release 资产的常规 URL 第一步就要访问 `github.com` 拿 302，会永远卡住。
  绕开的办法是走 API 的资产端点（`Accept: application/octet-stream`），
  它跳到 `release-assets.githubusercontent.com`。`download.sh` 就是这么做的
- **代理坏掉时的表现是"慢慢磨"而不是立刻报错**，所以要**先探一次**
  （`gh api /rate_limit` 或一次小请求），别拿几百 MB 去赌

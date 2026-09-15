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
cd editor/astronvim_v5 && ./tests/astronvim_test.sh   # 45 条
```

**容器相关的测试只能人工跑**（这个环境跑不了 docker 里那套完整流程时，
至少要人工确认 `container-shell.sh` 没被改坏）：

```bash
docker run --rm -it \
  -v ~/self/wtool:/wtool:ro \
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

## 代理（这台机器）

宿主代理是 `http://127.0.0.1:7897`。

- **容器里 `127.0.0.1` 是容器自己**，直接透传 `HTTP_PROXY` 会让容器内所有下载失败。
  必须 `--network=host`（`publish.sh` 已经会自动判断）
- `archive.ubuntu.com` / `security.ubuntu.com` 走这个代理**经常 502**。
  容器里装包失败先换国内镜像（`mirrors.ustc.edu.cn`）
- GitHub 的 `git clone` 走这个代理会偶发 TLS 中断，重试或改用 tarball

# 03 禁区与坑

## A. 绝对禁区

| 禁区 | 说明 |
|---|---|
| **不要对用户的 git 仓库做写操作** | 不 `add/commit/push/checkout/reset/stash/branch/tag`。用户明确要求自己提交。只读命令可用 |
| **不要写 `~/source/mytool`** | 只读来源，任何改动都不要落在那 |
| **不要在真实 `$HOME` 上做安装实验** | 一律 `WTOOL_HOME=$(mktemp -d)`，见 `bootstrap/tests/pairing_test.sh` |
| **不要把 `~/.zshrc` 软链进仓库** | 它含明文 `GEMINI_API_KEY`（见 `notes/01-context.md`） |
| **不要改 `~/self/wtool/.repo/`** | repo client 内部状态 |
| **不要用 `--force` 掩盖失败** | 它只该用于"首次试用未提交的仓库"或明确的冲突接管 |

## B. 仓库层面的坑（迁移前必须处理）

| 坑 | 证据 | 影响 |
|---|---|---|
| **wtool 清单未提交** | `w_manifests` 的 `wblog`/`wtool` 都指向 `036aa0f`，内容还是博客清单 | 一次 `repo sync` 就回退，wtool 清单消失 |
| **子仓 detached HEAD** | `git branch -a` 显示 `* (no branch)` | 本地提交容易丢；建议 manifest 加 `dest-branch` |
| **readme/gbb 与实际不符** | readme 说跑 `./build.sh`（不存在）、`-b wblog`；`gbb` push 到 `wblog` | 新机器按文档操作会失败 |
| **mytool 的 `install_all.sh` 有未提交脏改动** | 被改成只跑 `emacs`（该仓不存在） | 迁移时不要照抄 |
| **mytool `astronvim_v5.xml` 指向已改名的仓** | `allinkernel/astronvim_v5_config` 已 404 | 不要搬这个 xml |
| **gitee 的 7 个 `wsw_*` 在 github 不存在** | 逐个 `ls-remote` 验证 | 迁移需新建仓 + `git push --mirror` |

## C. 引擎实现里踩过的坑（改代码时对照）

| 坑 | 症状 | 正确做法 |
|---|---|---|
| 块里放时间戳 | 重复 install 会重写 rc；测试间歇性失败 | 溯源信息放 `meta.tsv`（ADR-006） |
| install 截断 journal | 重复 install 后 uninstall 残留软链/目录 | journal 只去重、永不截断（ADR-009） |
| 用 `git symbolic-ref` 判断分支 | repo 的 detached HEAD 直接卡死 | 只查 `status --porcelain -uno`（ADR-011） |
| `git status --porcelain` 不加 `-uno` | nvim 生成的 data/state 让安装永远失败 | 必须 `-uno` |
| 写 rc 用 `mv` 覆盖 | 把 `~/.zshrc` 的软链毁成普通文件 | 先 `readlink -f` 再写穿 |
| 块 append 而非排序插入 | 安装顺序影响 rc 顺序 | 按 `(prio,id)` 插入（ADR-008） |
| 在管道子 shell 里改全局变量 | 计数器/状态丢失 | 用重定向 `< file` 而非管道 |
| `awk -v` 传含反斜杠的路径 | 路径被转义 | 目前路径不含特殊字符，若将来支持需改 `ARGV` 方式 |

## D. 上游工具的已知陷阱（若将来换方案）

| 工具 | 陷阱 |
|---|---|
| GNU Stow | 目录折叠会把 `~/.config` 整体变成软链，别的程序写入会直接写进你的 git 仓 → 用 `--no-folding` |
| GNU Stow | `--adopt` 会把 `$HOME` 里的现有文件移进仓库并覆盖仓内容 → 永远别用 |
| 编辑器原子写 | 写临时文件再 rename 可能把软链替换成普通文件（neovim#23808、yui 专门做了 absorb 分类器）→ 定期 `wtool status` 检查 |
| stow 版本 | Ubuntu noble apt 里是 2.3.1，手册最新 2.4.1，行为有差异 |

## E. 本机环境注意

| 项 | 说明 |
|---|---|
| `repo` 工具 | `~/bin/repo`；mytool 的 `.repo` 是只读挂载（`repo list` 会因写 TRACE_FILE 失败） |
| sandbox | 当前会话只允许写 `/home/mindul/self/wtool`；写 mytool 会被拒绝，这是策略不是 bug |
| zsh | `~/.zshrc` 第 1 行仍 source 旧 mytool；迁移时要替换成 wtool 块 |

## F. 全新系统上的 bootstrap 顺序（2026-09-10 实测踩到）

`ubuntu:24.04` **docker 镜像**里缺的东西比想象的多：

| 组件 | docker 镜像 | server/desktop ISO 安装 |
|---|---|---|
| `git` | ❌ 缺 | ❌ 缺（默认不装） |
| **`python3`** | ❌ **缺** | ✅ 有（priority important） |
| `ca-certificates` | ❌ 缺 | ✅ 有 |
| apt 源 | 镜像自带，是 **HTTP** 的 `archive.ubuntu.com` | HTTP 官方源 |

**踩到的坑**：先换 HTTPS 镜像源、再装 `ca-certificates` → apt 全线
`Certificate verification failed: The certificate issuer is unknown`，什么都装不上。

**正确顺序（必须遵守）**：

```sh
# 1) 先用"系统自带源"装齐最小依赖（此时是 HTTP，不需要证书）
apt-get update
apt-get install -y --no-install-recommends ca-certificates git python3
# 2) 再换 HTTPS 镜像源（现在证书能验证了）
# 3) 然后才能跑 wtool
```

**推论**：wtool 的 `start.sh`（todo.md 第 0 条）第一步必须是这条裸 apt；
引擎依赖 `python3`，而最小化系统上它不一定在。

## G. ⚠️ 事故记录（2026-09-14）：误在用户工作区跑了 repo init

**事故**：写一个 E2E 测试时，变量 `t`（临时目录）为空，导致

```sh
cd "$t/work"                       # → cd "/work" 失败
~/bin/repo init -u "file://$t/manifest" -b main   # → URL 变成 file:///manifest
```

命令在**当前工作目录**（也就是 `~/self/wtool` 这个 repo client）里执行了，
把用户的 repo client 重新 init 了一遍。

**实际损害（2 处）**：
1. `.repo/repo` 被 `reset --hard` 到 `v2.9^0`（2020 年的老版本，`help.py` 里
   `from formatter import ...` 在 Python 3.10+ 已删除）→ **repo 命令直接不可用**。
2. `.repo/manifests.git` 的 `remote.origin.url` 被改成 `file:///manifest`。

**未受损**：`.repo/manifests/default.xml`（用户自己的改动）、`.repo/manifest.xml`、
所有项目仓。

**恢复方法**：
```sh
# 1) 从 reflog 找到被我改之前的提交
git -C ~/self/wtool/.repo/repo reflog | head
#    HEAD@{1} = 6d2260b  ← 用户 fork 的 wsw 分支 tip
git -C ~/self/wtool/.repo/repo reset --hard 6d2260b

# 2) 还原 manifest 仓的 remote（原 URL 保存在 syncstate 里，没丢）
git -C ~/self/wtool/.repo/manifests.git config --get repo.syncstate.remote.origin.url
git -C ~/self/wtool/.repo/manifests.git config remote.origin.url \
    'ssh://git@github.com/allinkernel/w_manifests.git'
```

**防复发纪律（今后必须遵守）**：
1. **`repo init` 绝不能在"当前目录"裸跑**。必须先 `cd` 到专用目录，
   并**断言** `[ "$(pwd)" = "$expected" ]`，否则中止。
2. **测试目录放在工作区内**（如 `~/self/wtool/.e2e-test/`），不要放 `/tmp`——
   agent 的 bash 调用之间 `/tmp` **不保留**（每次调用是新的挂载命名空间），
   跨调用引用 `/tmp` 路径必然踩空。
3. 变量用在路径里之前先检查非空：`[ -n "$t" ] || exit 1`。
4. 涉及 repo 的破坏性实验，先在 **manifest 仓的副本**上做，不要碰真 client。

### G.2 第二次事故（同日）：测试目录放在工作区内部

修完 G 节后重做 E2E，把测试目录放在 `~/self/wtool/.e2e-work/` —— **仍然中招**。原因：

> **repo 会从当前目录向上逐级查找 `.repo`**，一旦找到就"复用那个 client"
> （日志原话：`repo: reusing existing repo client checkout in /home/mindul/self/wtool`）。

于是 `repo init` 又作用到了用户的 client 上：`.repo/manifests` 的 HEAD 变成
unborn、`refs/heads/default` 被清掉、`remote.origin.url` 又被改成临时路径。
（`default.xml` 因为 git 拒绝覆盖未提交改动而**幸免**。）

**恢复**：`origin/wtool` 与 `packed-refs` 里还留着 `036aa0f`，所以
```sh
git -C .repo/manifests.git update-ref refs/heads/default 036aa0f
git -C .repo/manifests.git config remote.origin.url 'ssh://git@github.com/allinkernel/w_manifests.git'
git -C .repo/manifests.git update-ref -d refs/remotes/origin/main   # 删掉泄漏进来的
git -C .repo/manifests reset -q 036aa0f                            # 索引回位，工作区不动
```

**新增纪律（最重要的一条）**：
> **绝不在任何 repo client 的目录树内部运行 `repo` 命令。**
> 测试工作目录必须用 `mktemp -d`（`/tmp` 下没有 `.repo` 祖先），
> 且整个测试要在**同一次进程**里跑完（agent 的 `/tmp` 不跨调用保留）。

`tests/e2e_repo_sync_test.sh` 里已加**硬性安全闸**：WORK 若在工作区内、
或其任一祖先存在 `.repo`，直接拒绝执行。

### G.3 顺带发现：分支名不一致
`bootstrap` 与 `harness` 在 **`master`**，其余 6 个仓在 **`main`**，
而清单的 `<default revision="main"/>`。把它们加进真清单时，
`repo sync` 会报 `couldn't find remote ref refs/heads/main`。
（E2E 测试里已改成按各仓真实分支生成 revision。）

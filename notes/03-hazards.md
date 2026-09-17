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

## H. 2026-09-17 这一轮的坑（都是"看起来在正常工作"的那种）

这一轮的共同点：**失败都不报错，只是安静地做错事**，
所以人会被引到完全错误的方向去查。按发现顺序：

### H1. apt 没有下载超时 → build.sh 的重试逻辑永远轮不到执行

`_apt` 里写了"失败就重试、就换源"，但 **apt 默认没有下载超时**：
代理后面的连接**停滞**不算失败，apt 会一直挂着等。
于是重试和换源那段代码根本没机会跑，构建卡了 20 分钟、`partial/` 一个字节都没有。
表现和"网络断了"一模一样，但其实是"在等一个永远不会到的响应"。

修：`-o Acquire::http::Timeout=20 -o Acquire::https::Timeout=20 -o Acquire::Retries=3`。
**把"挂死"变成"失败"，退路逻辑才生效。**

### H2. 换源判断错了 → 索引好不代表包能下来

`_apt_prepare` 只在 `apt-get update` 失败时才换源。
可坏的是**包下载**、`update` 是好的，于是重试还是走同一个卡死的源。
修：抽 `_apt_switch_mirror()`，重试时**强制**换，不再问 update 的意见。

### H3. 国内镜像走代理 → 换了域名还是同一条坏路

报错里 `502 Bad Gateway [IP: 127.0.0.1 7897]` —— 对 ustc 的请求也在走代理。
容器里设了 `HTTP_PROXY`，apt 默认对所有 http 都用它。
修：把镜像主机加进 `no_proxy`，并写 `Acquire::http::Proxy::<host> "DIRECT"`（双保险）。

### H4. `docker cp` 不创建中间目录 → 一条产物都收不到

`docker cp ctr:/a/b/c  /host/x/a/b/c`，如果 `/host/x/a` 不存在，它报
`invalid output path: directory "..." does not exist`，**不会替你建**。
publish.sh 只 `mkdir -p` 了 `payload/home` 一层，于是 `.local/bin`、`.config`
全都不存在 —— 五条产物一条都复制不出来，最后死在
"安装清单里一条 payload 都没有"，看起来像 install.sh 没产出东西。
**报错把人往完全无关的地方引。**
修：`docker cp` 前 `mkdir -p` 目标父目录；失败时把 docker 的原始报错露出来。

### H5. curl 的 `--retry` 会让断点续传白做

```
curl: (28) Operation timed out ... 21441650 out of 33554432 bytes received
Warning: Transient problem: timeout Will retry in 5 seconds.
Throwing away 21441650 bytes          ← 关键
```
curl 内部重试**不续传**，把已下的 21MB 整个丢掉。在 20 KB/s 的链路上
那是十五分钟的成果。`-C -` 只在 curl 启动时用一次，它自己的重试不走这个逻辑。
修：去掉 `--retry`，重试一律交给外层循环（它保留断点文件）。
加 `--speed-limit 2048 --speed-time 90` 让停滞快速暴露，不傻等 15 分钟。

### H6. 分卷 300M 在这条链路上永远传不完

实测直连上传只有 ~237 KB/s，而链路每隔几分钟断一次 ——
314M 的单次 POST 注定完不成，重试就是从头再来。
修：默认切成 32M（每个约 2 分钟）。
**卷大小不是随便定的，它要和网络的可靠性窗口匹配。**

### H7. 上传失败会连带删掉产物 / 失败却退出 0

`cmd_publish` 的 trap 无条件 `rm -rf` 临时目录，而 576M 产物就在里面 ——
构建跑了 30 分钟，上传断了，产物也没了，重试要重编一遍。
而且 `wt_publish_gh_upload` 里是 `|| wt_die`，一失败直接退整个脚本，
调用点的判断根本轮不到；`publish.sh` 失败那条路径也没算进 `_failed`，
于是**发不出去也退出 0**。
修：失败时保留产物并打印补传命令；上传函数返回非零；两条失败路径都计数。

### H8. 进度显示永远 0% → 用户以为下载卡死

`_launch` 把卷从 `.todo.tsv`（队列）里摘掉，而算进度的 `_bytes_now`
读的**正是** `.todo.tsv` —— 在下载的卷已经不在里面了，永远算出 0。
用户看到 `0M / 483M（0%）` 纹丝不动，合理地判断成卡死，然后去查网络、查 URL。
**下载其实一直正常。**
修：队列和清单分成两个文件 —— `.todo.tsv` 会消费，`.want.tsv` 不动、用来算总进度。

> 这一类（H8）最值得记住：**显示错了比逻辑错了更坏。**
> 逻辑错了会报错、会停下来；显示错了会让人安静地朝错误方向查下去。

### H9. "/dev/tcp" 探测污染 shell → 交接后不给提示符

`container-proxy.sh` 里原本用 `(exec 3<>"/dev/tcp/$host/$port")` 探代理，
之后脚本 `exec bash -i` **不给提示符**：进程不退、光标不动，看起来就是卡死。
二分定位：`WTOOL_NO_PROXY=1` 跳过探测 → 提示符回来了；
只留那段 fd 操作 → 提示符又没了。
修：改成在独立子进程里探（`timeout 3 bash -c "exec 3<>..."`），父 shell 不碰 fd。

**并且加了一句显式交接**（"已经进到容器里了，下面这个提示符就是在等你"）——
"卡住"和"在等你输入"分不清，本来就是脚本的责任。

### H10. `grep -c ... || echo 0` 会输出两行

查不到时 `grep -c` 打印 `0` 并返回 1，`|| echo 0` 再打印一个 `0` —— 两行。
用 `awk 'END{print NR}'` 之类替代。

### H11. 三元运算符是 bash 扩展，但 `sh -n` 查不出来

`$(( a > 0 ? a : 1 ))` 在 dash 下**能通过 `sh -n` 的语法检查**，
却会在**运行时**报错。语法检查过了不等于能跑，脚本要**真跑一遍**。

### H12. `set -u` 下引用未设置的 `$!` 会让整个脚本立刻退出

表现是"16 个卷全下完、清单也写好了，却报 download 失败" ——
**活干完了，报错说没干成**，最气人的失败模式。
取 `$!` 前后临时 `set +u`，取不到就当没有（那个 pid 只用于收割和显示）。

### H13. 别把一次失败当成规律

我在某个时刻量到 `走代理 → SSL_ERROR_SYSCALL`，就写下了
"代理对 GitHub 是坏的、直连才好"，还进了三处文档。
后来复测：代理完全正常（`api.github.com` 200/1.1s、`github.com` 200/2.75s）。
那只是**一次临时故障**。

**一次失败只证明"这次失败了"，不证明"这条路不行"。**
要下"哪条路更好"的结论，至少隔一段时间复测几次。
留着错的结论比没有结论更坏 —— 它会让人主动绕开本来能用的路。


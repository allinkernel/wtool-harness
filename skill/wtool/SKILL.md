---
name: wtool
description: Use when working in /home/mindul/self/wtool (the wtool tool collection, its bootstrap engine, project manifests, or migrating tools from ~/source/mytool).
---

# wtool 项目

`/home/mindul/self/wtool` 是用户用 `repo` 管理的个人工具集合，正在把
`~/source/mytool`（旧集合，gitee + 复制式安装）迁移过来（github + 纯软链）。

## 三条铁律

1. **不要碰用户的 git 仓库。** 不 `git add/commit/push/checkout/reset/stash`，不建分支、不建 tag。
   用户自己提交。只读命令（`status`/`log`/`ls-files`/`rev-parse`）可以用。
2. **不写 `~/source/mytool`。** 它是只读来源。要改就改 `~/self/wtool` 下的东西。
3. **不碰真实 `$HOME` 的 dotfile 做实验。** 所有验证用 `WTOOL_HOME=<临时目录>` 跑
   （见 `bootstrap/tests/pairing_test.sh`）。

## 先读什么

| 想了解 | 读 |
|---|---|
| 设计契约（代码的真相） | `bootstrap/docs/spec.md` |
| 清单字段 | `bootstrap/docs/manifest-schema.md` |
| 为什么这么设计 / 否决了什么 | `harness/notes/02-decisions.md` |
| 禁区与已知坑 | `harness/notes/03-hazards.md` |
| 仓库清单与现状 | `harness/notes/01-context.md` |
| 下一步 | `harness/notes/05-next.md` |

## 当前架构（一句话）

> bootstrap 是引擎（全机器唯一一份），项目是数据（每个仓一份）；
> Python 只算不写，Shell 只写不算；状态目录既是日志也是注册表；
> "install 后 uninstall 必须字节级回到原样"由测试证明。

```
wtool-bootstrap/          引擎
  wtool.sh                CLI: build/download/install/uninstall/provision/publish/
                               bootstrap/table/list/status/doctor/env/init/validate/version
  lib/wtool_plan.py       规划器（只写 scratch）
  lib/wtool_fs.sh         执行器（唯一写 $HOME 的地方）
  templates/*.tpl         wtool init 生成 scripts/ 用的模板（stub.sh 已废，见 ADR-013）
  tests/                  7 个 *_test.sh；run_all.sh 跑 5 组 147 条，全在临时 $HOME 里跑

<项目>/
  wtool.xml               声明：env / link / priority / publish
  env.zsh                 被 source 的部分（只导出，无副作用）
  scripts/                动作脚本：build.sh / download.sh / install.sh / publish.sh
                          **文件存在即能力声明**（根目录老位置引擎仍认，但会警告）

$WTOOL_STATE/  (~/.local/state/wtool)
  registry.tsv            dest -> 项目 id / kind
  generated.tsv           wtool 自己写过的文件（publish 脏检查豁免）
  <id>/journal.tsv        撤销的唯一依据（按 (action,dest) 去重，永不截断）
  <id>/meta.tsv           版本/来源/时间
  <id>/actions.tsv        做过什么（build / download，只追加）
  <id>/artifacts.tsv      当前产物来自 build 还是 download
  <id>/publish.tsv        本地发布历史
```

## 项目清单（12 个，2026-09-17）

`python3 bootstrap/lib/wtool_plan.py publish-list --root .` 是权威列表：

| 路径 | id | prio | 形态 |
|---|---|---|---|
| `bootstrap` | bootstrap | 5 | 引擎自己（install 让 `wtool` 进 PATH） |
| `os/ubuntu` | os/ubuntu | 5 | system-file + provision(ansible) |
| `shell/oh-my-zsh` | shell/oh-my-zsh | 10 | env |
| `shell/zsh` | shell/zsh | 20 | env |
| `tools/repo` | tools/repo | 40 | env |
| `tools/android_repack` | tools/android_repack | 45 | `scripts/install.sh`（装到 `$WTOOL_PREFIX/bin`） |
| `terminal/tmux` | terminal/tmux | 50 | env + link |
| `terminal/fzf` | terminal/fzf | 60 | env(zsh) + env(bash) |
| `editor/astronvim_v5` | editor/astronvim_v5 | 70 | publish kind="script"；子树 config/nvim 由它 `<sub>` 声明 |
| `harness` | harness | 100 | 只可发布（给助手的记忆） |
| `themes/typora/lightmind` | lightmind | 100 | 只可发布（Typora 主题） |
| `wtool-base` | wtool-base | 100 | 只可发布（用户文档） |

已发布 source 包的有 10 个（见 `wtool-base/README.md` 的下载块）；
`editor/astronvim_v5` 还是"待构建下载"（要容器构建，小时级）；
`tools/android_repack` 有发布能力但还没发过。
上游 `nvim`（neovim/neovim）在 astronvim_v5 的 wtool.xml 里声明为 `kind="none"` ——
我们既没权限推它，也不能往它里面塞 wtool.xml。

## provision 层（不可逆操作，与 install 分离）

```xml
<system-file kind="apt-mirror" mirror="ustc" dest="auto" mode="replace" backup="true"
             when="os:ubuntu" desc="换源"/>
<source url="https://github.com/x/y.git" ref="v1.0" dir="$WTOOL_SRC/y"
        branch="wsw" overlay="overlay"/>
<provision src="wsw.sh" runner="shell" marker="y-{ref}" when="os:ubuntu" desc="编译"/>
```

- 三阶段顺序：**system-file → source → task**；`wtool provision <项目> --with-system`
- `system-file` **可逆**（备份/还原，进 journal）；`source`/`task` 不可逆（只记 marker/日志）
- 编译型项目约定：`overlay/wsw.sh`，只往 `$WTOOL_PREFIX`（= `~/.wtool/usr`）装
- 任务环境变量：`WTOOL_PREFIX` / `WTOOL_SOURCE_DIR` / `WTOOL_SOURCE_REF` / `WTOOL_OS_*` / `WTOOL_JOBS`
- 一条命令装完：`wtool bootstrap --with-system`

## 迁移一个项目（含保留 git 历史）

```sh
# 1. 目标目录里放好 wtool 化后的文件（wtool.xml / env.* / README / 存根）
# 2. 把源仓历史搬过来（源 .git 常是符号链接，必须 -L 解引用）
cp -RL ~/source/mytool/<项目>/.git <目标>/.git
cd <目标>
git checkout -b main          # 源多为 detached HEAD
git add -A
git commit                    # 迁移改动作为新提交叠在历史之上
```

先确认源仓 `objects/info/alternates` 为空（否则复制 .git 会丢对象）。
**不要 `git push`**，remote 留给用户改。

## 常用命令

```sh
cd ~/self/wtool/bootstrap
./tests/run_all.sh                      # 必须全绿再交付（5 组 147 条）
./wtool.sh doctor
./wtool.sh table --verbose              # 项目表 + 每个项目"装过没/发布过没"
./wtool.sh install   ../terminal/tmux --dry-run
./wtool.sh list
python3 -m py_compile lib/wtool_plan.py # 改 py 后
sh -n wtool.sh && sh -n lib/wtool_fs.sh # 改 sh 后（只查语法：bashism 它查不出来，见 H11）
cd ../editor/astronvim_v5 && ./tests/astronvim_test.sh   # 52 条
```

## 改代码的规矩

| 改什么 | 必须同时 |
|---|---|
| 清单格式 / 块格式 / plan.tsv 列 | 改 `bootstrap/docs/spec.md`（契约优先），并考虑向后兼容 |
| 引擎行为 | 在 `tests/pairing_test.sh` 加一条断言，然后跑全绿 |
| 任何决策 | 在 `harness/notes/02-decisions.md` 追加 ADR |
| 发现新坑 | 写进 `harness/notes/03-hazards.md` |
| 每轮结束 | 在 `harness/journal.md` 追加流水 |

## 反复踩过的坑（别再犯）

- 块里放时间戳 → 破坏幂等（重复 install 会重写 rc）。溯源信息放 `meta.tsv`。
- install 时截断 journal → 重复 install 后 uninstall 残留软链。journal 只去重不截断。
- 块里不导出 `WTOOL_PROJECT_*` → env 文件里这些变量是空的（块必须在 source 前导出）。
- 判断"这条软链是不是我们的"只看目标相等 → 仓库搬家后旧的中转链接无法更新而报冲突。
  要用 journal 判断归属（`wt_journal_owns`）。
- 依赖"当前分支"判断 git 状态 → repo 工具默认 detached HEAD，会直接卡死。
- `git status --porcelain` 不加 `-uno` → nvim 生成的 data/state 会让安装永远失败。
- 写 rc 文件用 `mv` 覆盖 → 会毁掉软链。必须 `readlink -f` 后写穿。
- 块插入用"append" → 安装顺序会影响结果。必须按 `(prio,id)` 排序插入。
- 在当前 shell 里做 `/dev/tcp` 的 fd 操作 → 之后 `exec bash -i` 不给提示符，看起来就是卡死。
  探测要放独立子进程（`timeout 3 bash -c ...`）。
- `grep -c 'x' || echo 0` → 查不到时 grep 自己也输出 0，变成两行。用 `awk 'END{print NR}'`。
- `docker cp` 不创建中间目录 → 先 `mkdir -p` 目标父目录，否则一条产物都收不到。
- apt 默认没有下载超时 → 连接停滞时一直挂着等，重试/换源逻辑永远轮不到执行。
  要显式设 `Acquire::http::Timeout` / `Acquire::https::Timeout`，把"挂死"变成"失败"。
- `sh -n` 过得去不等于能跑：三元运算符是 bash 扩展，dash 下**运行时**才报错。

完整清单在 `harness/notes/03-hazards.md`（H1–H13，每条都有当时的症状和怎么发现的）。

## 验证要求

任何改动后至少跑：

```sh
./bootstrap/tests/run_all.sh   # 期望 5 组全绿：pairing 30 / provision 24 /
                               # publish 41 / table 35 / release-copy 17 = 147 条
cd editor/astronvim_v5 && ./tests/astronvim_test.sh   # 52 条
```

并连续跑 3 次确认不 flaky（历史上出现过时间戳导致的间歇性失败）。
`run_all.sh` **不跑** `container_test.sh`（要 docker）和 `e2e_repo_sync_test.sh`（慢），
涉及容器/同步逻辑时人工跑；涉及 shell 的改动别只靠 `sh -n`（H11：三元等 bashism 能过语法检查）。

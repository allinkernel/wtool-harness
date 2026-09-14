# 01 项目全貌与现状

> 快照时间：2026-09-09 / 2026-09-10 会话。任何 agent 接手前先看这里。

## 1. 两套集合

| | `~/source/mytool`（旧） | `~/self/wtool`（新） |
|---|---|---|
| 管理方式 | `repo`，清单仓 `gitee.com:mindulmindul/wsw_manifests`（分支 `ubuntu_20`/`blog`） | `repo`，清单仓 `ssh://git@github.com/allinkernel/w_manifests.git`（分支 `wtool`） |
| 远端 | 全部 gitee | 全部 github（ssh） |
| 部署方式 | 复制 / `install_all.sh` 递归调用各子项目 `wsw_install.sh` | **纯软链 + 受管块**（本次重构的核心变化） |
| 环境变量 | `WSW_REPO_TOP`，根目录软链 `source_all_env.sh` / `install_all.sh` | `WTOOL_*`，引擎在 bootstrap，项目只放声明 |
| 规模 | 7 个项目，~700MB（多为上游 checkout） | 目前 2 个项目 + bootstrap + harness |

## 2. mytool 资产盘点（迁移依据）

| mytool 项目 | 体积 | 内容 | 迁移目标 |
|---|---|---|---|
| `start` | 56K | `.init/{start,source_all_env,install_all}.sh` + zsh/proxy/node 安装器 | → `bootstrap`（已被新引擎取代） |
| `os` | 60K | ustc/tuna × 5 个 Ubuntu 版本 sources.list + apt 批量安装 | → `os/ubuntu`，**需要 provision + system scope** |
| `zsh/wsw-zshrc` | 4 文件 | `wsw.zsh` + env/install | ✅ **已迁移为 `shell/zsh`** |
| `zsh/oh-my-zsh` | 13M | vendored 的 oh-my-zsh 副本（个人 fork） | ✅ **已迁移为 `shell/oh-my-zsh`**（独立 wtool 项目） |
| `nvim/wsw_nvim` | 小 | `wsw_configs/wsw_nvim/*.lua` + 源码编译 neovim | → `editor/nvim`，**需要 provision（编译）** |
| `tmux` | 32K | tmux.conf + cpu/mem/disk/net 脚本 | ✅ **已迁移为 `terminal/tmux`** |
| `fzf` | 4.3M | 4.4MB 二进制入库 + 补全脚本 | ✅ **已迁移为 `terminal/fzf`**（多 shell 示例） |
| `android` | 456K | `my_repo.py` + 一大套 zsh 函数（cs/ct/cnp/cnn/cdd/gb/gbb/rs/rscur/wninja） | ✅ **已迁移为 `tools/repo`**（纯 env） |
| 未跟踪垃圾 | 11M | `wsw-configs.tar.gz`、`nvim/nvim-important.tar.gz` | 不迁移，删除 |

> 迁移进度（2026-09-10）：**8 个 mytool 项目已迁 5 个**。剩余 `start`(→bootstrap 已覆盖)、`os`、`nvim` 需要 provision/system scope 能力后才能迁。

## 3. 当前 wtool checkout 内容

```
~/self/wtool/
├── .repo/                       repo client（清单仓 w_manifests@wtool）
├── bootstrap/                   ★ 引擎（git 仓，用户提交）
├── harness/                     ★ 本项目记忆（git 仓，用户提交）
├── shell/oh-my-zsh/             ★ 迁移（prio 10，独立项目）
├── shell/zsh/                   ★ 迁移（prio 20，纯 env）
├── tools/repo/                  ★ 迁移（prio 40，纯 env）
├── terminal/tmux/               ★ 迁移（prio 50，env + link）
├── terminal/fzf/                ★ 迁移（prio 60，env zsh + env bash）
├── editor/vim/astronvim_v5_config/    repo 管理的项目
├── themes/typora/lightmind/           repo 管理的项目
├── astronvim_v5/                空目录（残留）
├── typora-theme/                手工克隆残留（与 themes/typora/lightmind 重复）
└── .mypy_cache/                 残留
```

## 4. GitHub 上已有的东西（allinkernel）

| 仓库 | 状态 |
|---|---|
| `w_manifests` | 分支 `wblog` 与 `wtool` **指向同一个 commit `036aa0f`**，内容仍是博客清单；wtool 清单改动只在工作区（未提交）⚠️ |
| `wtool-git-repo` | 用户 fork 的 git-repo，`wsw` 分支加了 `repo manifest -R`（并行查远端锁版本） |
| `wtool` | 只有一个 LICENSE（占位） |
| `wtool-astronvim_v5_config` | 已在用 |
| `typora-LightMindTheme` | 已在用（fork） |
| `wblog-*` | 博客项目，与 wtool 无关 |

mytool 的 7 个 `wsw_*` 仓在 github 上**都不存在**（已逐个 `ls-remote` 验证），迁移时需要新建 + `git push --mirror`。

## 5. 本机 $HOME 现状（与设计相关的部分）

| 项 | 现状 | 影响 |
|---|---|---|
| `~/.zshrc` | 1311 字节，第 1 行是 `source /home/mindul/source/mytool/source_all_env.sh`；含明文 `GEMINI_API_KEY` | ① 绝不能把 `~/.zshrc` 软链进公开仓 ② 迁移就是把这一行换成 wtool 块 |
| `~/.zshenv` | **不存在** | 若将来想用 `ZDOTDIR` 接管，成本是"新建一个文件"，不改任何现有文件 |
| `~/.config/astronvim_v5` | 完整克隆（带自己的 `.git`） | 正是"两份都要改"的根源，应改为软链 |
| `~/.wswtool` | 已存在 → `~/source/mytool` | todo.md 里的想法已落地；wtool 侧对应 `~/.wtool/links/<id>` |
| stow / chezmoi 等 | 均未安装；apt 里 `stow` 是 2.3.1 | 目前方案零外部依赖 |

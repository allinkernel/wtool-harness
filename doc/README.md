# wtool —— 我的工具集合

> **一句话**：用软链接 + shell 注入管理我所有的配置和工具。
> 仓库搬到哪都能用；装了什么、改了什么，随时能精确撤掉。

这套东西不用"安装"（不会把文件复制到 `$HOME`），
而是**在 `$HOME` 里建软链**、**在 `~/.zshrc` 里写一段带标记的块**。
所以你改配置永远只改一处，也不会有"仓库和家目录两份、改了一处忘了另一处"的问题。

---

## 30 秒上手

### 已经有这套目录（比如 `~/self/wtool`）

```sh
cd ~/self/wtool
./bootstrap/install.sh        # 装引擎自己（往 ~/.zshrc / ~/.bashrc 各写一个受管块）
exec zsh                      # 重开 shell，wtool 命令和变量就位

wtool doctor                  # 看当前状态：装了什么、系统是什么
wtool bootstrap --with-system # 一条命令：把所有项目装完（换源+装包+软链+注入）
```

### 全新机器

**别急着跑上面的命令**——先看 `02-快速开始.md`。
最小系统（比如 docker 镜像）里没有 `python3`，而 wtool 引擎依赖它，
顺序搞反了会卡在证书验证上。

### 只想看看它干了什么

```sh
wtool provision os/ubuntu --dry-run --with-system   # 只打印计划，什么都不改
WT_SKIP_SETUP=1 bash bootstrap/tests/container_test.sh   # 在假 HOME 里全流程演练
```

---

## 文档索引

| 我想知道 | 看这篇 |
|---|---|
| 它到底怎么工作的、为什么这么设计 | `01-架构与原理.md` |
| 在一台新机器上从零装好 | `02-快速开始.md` |
| 某个命令怎么用 | `03-命令速查.md` |
| 每个项目是什么、迁移时改了什么 | `04-项目清单与变更导读.md` |
| 我要加一个新的工具/配置 | `05-写一个新项目.md` |
| 报错了怎么办 | `06-排错.md` |
| 清单仓（.repo/manifests）要做什么 | `07-清单仓待办.md` |

> 更深的参考资料（设计契约、字段全表）在 `bootstrap/docs/`：
> `spec.md`（接口契约）、`manifest-schema.md`（wtool.xml 全字段）、`roadmap.md`（未来方向）。

---

## 目录地图

```
~/self/wtool/                     ← 整个集合（用 repo 管理）
├── README.md      → harness/doc/README.md      （软链，就是本文）
├── docs/          → harness/doc/               （软链，全部文档）
├── bootstrap/        引擎：wtool 命令、规划器、测试
├── harness/          项目记忆与文档（含 doc/、notes/、journal.md）
├── os/ubuntu/        系统层：换源 + 装基础软件包
├── shell/oh-my-zsh/  oh-my-zsh（个人 fork）
├── shell/zsh/        我的 zsh 别名与函数
├── tools/repo/       repo/git 辅助命令（cs/ct/cdd/gb/rscur…）
├── terminal/tmux/    tmux 配置 + 状态栏脚本
├── terminal/fzf/     fzf 二进制 + 补全
├── editor/… themes/…  原有的（repo 管理）
└── .repo/            repo 的工作目录

$HOME 里被它创建的东西：
├── ~/.wtool/links/<项目id>   每个项目的"稳定地址"（软链，指向仓库真实位置）
├── ~/.wtool/usr/             编译安装前缀（wsw.sh 只准往这里装，删了就是卸载）
└── ~/.local/state/wtool/     状态：谁占了哪个路径、装过什么（撤销的依据）
```

---

## 三个核心概念

先记住这三条，其余都能推导出来。

### 1. `$HOME` 里只允许两种东西：软链 + 受管块

- **软链**：`~/.tmux.conf -> ~/.wtool/links/terminal/tmux/tmux.conf`（应用去固定位置找配置）
- **受管块**：`~/.zshrc` 里一段带 `# >>> wtool:<id> >>>` / `# <<< wtool:<id> <<<` 标记的代码

没有第三种形态。所以"卸载"就是删软链 + 删块，精确、可验证。

### 2. `install` 可逆，`provision` 不可逆——两者永不互相调用

| | 做什么 | 能撤吗 | 需要 root |
|---|---|---|---|
| `wtool install` | 建软链、写受管块、换系统配置文件 | **能**（字节级还原） | 否（换系统文件时除外） |
| `wtool provision` | 装 apt 包、编译源码、拉第三方仓库 | 不能（删了重来即可） | 视情况 |

之所以分开：apt 装包和源码编译本来就不需要"回退"，
而 `~/.zshrc` 里加过哪几行是**记不住的**，那才需要精确撤销。

### 3. 配置里互相引用时，写"稳定地址"而不是仓库真实路径

```sh
# ✗ 不要这样：仓库一搬家就全废
/home/mindul/self/wtool/terminal/tmux/bin/cpu.sh

# ✓ 这样写：不管仓库在哪台机器、哪个目录，都指向同一个地方
$HOME/.wtool/links/terminal/tmux/bin/cpu.sh
```

`~/.wtool/links/<项目id>` 是引擎自动创建的软链，指向该项目的真实位置。
仓库搬家后只要重跑一次 `install.sh`，配置文件一个字都不用改。

# 01 项目全貌与现状

> 接手前先看这一篇。它讲**现在**是什么样，不讲历史。
> 历史决策看 `02-decisions.md`，踩过的坑看 `03-hazards.md`，下一步看 `05-next.md`。
>
> 快照：2026-09-17 会话末。

---

## 一句话

`wtool` 是一个**脚本调用器 + 状态管理者**：把一堆个人配置和工具拆成独立的小项目，
每个项目只声明"我有什么""我要做什么"，由 wtool 负责安装、构建、发布，以及**精确撤销**。

## 代码结构（每层只讲一件事）

```
wtool/                              工作区（repo 客户端，名字随安装位置变）
├── bootstrap/                      引擎。改行为来这儿，其他目录都是数据
│   ├── wtool.sh                    命令分发 + 各命令实现（唯一入口，1600+ 行）
│   ├── lib/wtool_plan.py           Python 侧：只算不写（解析清单、算差异、渲染，1900+ 行）
│   ├── lib/wtool_fs.sh             Shell 侧：只写不算（软链、原子写、journal，800+ 行）
│   ├── lib/wtool_os.sh             系统探测
│   ├── scripts/                    引擎自己的动作脚本
│   │   ├── install.sh              装 **wtool 自己**：四步做完就停（不装任何项目）
│   │   ├── install-env.sh          第 0 步的共用逻辑（装包、挑源、git ownership、复查）
│   │   ├── install-ubuntu{20,22,24,26}.sh  各发行版只写"自己不一样的地方"
│   │   ├── uninstall.sh            反向
│   │   ├── container-shell.sh      一条命令进容器开发环境（依赖 + 引擎 + bootstrap）
│   │   ├── container-raw.sh        只挂工作区、什么都不装（等价"刚 repo sync 完"）
│   │   └── container-proxy.sh      不是入口；被上面两个 source，探一次宿主代理
│   ├── templates/                  wtool init 用的模板（build/download/install/publish .tpl）
│   ├── tests/                      7 个 *_test.sh；run_all.sh 跑其中 5 组、147 条
│   └── docs/                       spec.md / manifest-schema.md / roadmap.md
├── wtool-base/                     用户文档。只有文档，没有工具
│   ├── README.md                   总览 / 没 git clone 怎么装 / 怎么用
│   └── guide.md                    结构 / 命令 / 子项目索引
├── harness/                        你正在看的：给 AI 助手的项目记忆
├── os/ shell/ terminal/ editor/ themes/    各具体项目
└── .wtool/                         安装痕迹（在 $HOME 里，不在仓库里）
```

**测试怎么跑**：`bootstrap/tests/run_all.sh` 跑 5 组 147 条（全程临时 `$HOME`）；
`container_test.sh`（要 docker）和 `e2e_repo_sync_test.sh`（用本地裸仓真跑 repo sync，慢）
不在里面，人工按需跑。astronvim 自己还有一组 52 条。

**分层原则**：`bootstrap/` 是唯一有行为的地方；其他项目要么是纯数据（配置文件 + 声明），
要么带 `scripts/`。文档分两处 —— `wtool-base/` 给人看，`harness/` 给助手看。
**两者不要互相掺**：用户文档里不写本地路径和内部实现，harness 里不写教程。

## 核心契约（改代码前必须理解的三条）

### 1. 声明式是默认，脚本是例外

| 动作 | 默认（声明式） | 例外（脚本） |
|---|---|---|
| install | `wtool.xml` 的 `<link>` / `<env>` | `scripts/install.sh` |
| download | —— **通用下载还没实现**（`release.json` 只是 `05-next.md` 里的设计） | `scripts/download.sh` |
| build | —— 构建没有声明式的可能 | `scripts/build.sh` |
| publish | `<publish kind="source">` 打源码包 | `scripts/publish.sh` |

**表格里"能不能做"就是查这张表**：左栏有就是引擎的通用能力，右栏有就是项目自带脚本。
判定的实现是 `wtool_plan.py` 的 `pipeline_states()`：它只看
`scripts/<名字>`（或项目根的老位置）在不在 —— **文件存在即能力声明**。
所以现阶段 "download" 那一列不是"有 release.json 就能下"，
而是"有 `scripts/download.sh` 才能下"。引擎里唯一认 `scripts/release.json`
的地方是 `cmd_bootstrap` 的就绪判断（见了就认为这个项目不用 build 也能装）。

### 2. install 和 provision 分开，理由不是"可逆性"

`uninstall` 实际能撤的东西（见 `$WTOOL_STATE/<id>/journal.tsv`）：软链、rc 块、
它创建的空目录、改过的系统文件（从备份还原）、克隆的源码树。

**apt 装的包没有记账，撤不回来** —— 不是"不可逆"，是"**没记**"。对外文档必须这么说，
不要说成"provision 不可逆"。

那为什么还分两条命令？三个理由，都和可逆性无关：

- **权限**：install 永远不需要 root，provision 需要。混在一起，`wtool install foo`
  会在某一刻突然问你要 sudo
- **失败爆炸半径**：install 失败，journal 里每条都在，能精确收拾；provision 失败
  （dpkg 半配置、apt 锁残留），系统处于引擎推理不了的状态
- **幂等成本**：install 秒级可随便重跑，provision 分钟级

### 3. 产物契约：脚本要声明"我产出了什么"

```
$WTOOL_STATE/                     默认 ~/.local/state/wtool（在 $HOME 里，不在仓库里）
├── registry.tsv                  全局：dest → 项目 id / kind（跨项目冲突检测）
├── generated.tsv                 全局：wtool 自己写过的文件（publish 脏检查豁免）
├── created-rc.tsv                全局：当初由 wtool 创建的 rc 文件（卸空了就删）
└── <项目 id>/
    ├── meta.tsv         项目元信息（priority / project_root / head / installed_at / engine）
    ├── journal.tsv      "当前该撤销什么"，uninstall 逆着做，**永不截断**
    ├── actions.tsv      "做过什么"的时间线（build / download），只追加，不参与回滚
    ├── artifacts.tsv    当前磁盘上的产物是谁产出的（build 还是 download）
    ├── publish.tsv      本地发布历史（离线也能在表格里看到发过没有）
    ├── env.zsh / env.bash   这个项目的环境变量块（按 priority 汇总进 ~/.wtool/.zshrc）
    ├── provision.log    provision 任务的输出（只追加）
    ├── provisioned/     provision 任务的幂等 marker（文件名 = 任务的 marker）
    └── system/          system-file 的原文件备份（uninstall 时还原）
```

**journal 和 actions 是两件事，别混。** 前者描述"该撤销什么"，后者是时间线。

**`build.sh` 和 `download.sh` 必须把产物放到同样的路径** —— 这是
`wtool build X && wtool install X` 与 `wtool download X && wtool install X`
等价的前提。脚本通过 `$WTOOL_ARTIFACTS` 拿到清单文件路径，往里追加
`kind<TAB>相对$HOME的路径<TAB>来源<TAB>时间`。

### 3.1 产物必须落在 `$WTOOL_PREFIX` 下面（2026-09-17 补记）

**这条原来漏写了，后果是 astronvim_v5 装完撤不回来。**

引擎给出 `WTOOL_PREFIX`（默认 `$HOME/.wtool/usr`，`wtool.sh:51`，
并通过 `wt_run_project_script` 导出给项目脚本）。契约是：

> 项目脚本把**自己产出的东西**装进 `$WTOOL_PREFIX`。
> `$HOME` 里只允许出现 **wtool 管的软链**（在 `~/.wtool/links/` 下）
> 和**那一个 rc loader 块**。

为什么必须这样：

- **可撤销**：卸载 = 删 `~/.wtool/`，不用去猜"这个项目往 $HOME 撒了什么"
- **不打架**：`~/.local/bin`、`~/.config` 是用户和别的工具共用的地方，
  往里倒东西就是在制造"谁装的、能不能删"的糊涂账
- **journal 才有意义**：journal 记的是"该撤销什么"，而撤销的目标必须
  是 wtool 自己拥有的路径

**反面教材（真实发生过）**：`astronvim_v5/scripts/install.sh` 写的是
`PREFIX=${PREFIX:-$HOME_DIR/.local}`，完全无视 `WTOOL_PREFIX`，
于是往 `~/.local/bin/nvim`、`~/.config/astronvim_v5`、
`~/.local/share/astronvim_v5` 里倒。结果：

- `~/.wtool/usr` 是空的 → `wtool uninstall` 撤不掉
- env 块只写了 `NVIM_APPNAME`、**没写 PATH** → 新开的 shell 找不到 nvim
- 用户 `$HOME` 被污染，且没有任何一条记录说这些东西是谁放的

**新项目上手时先检查这一条**，它比 build/download 同路径更容易漏，
因为漏了以后 install 看起来是成功的。

## 环境变量汇总（rc 收敛）

用户 rc 里**只有一段** loader 块，指向 `~/.wtool/.zshrc` / `~/.wtool/.bashrc`；
那两份文件由 wtool **整份生成**（按 priority 拼接各项目的 `env.<shell>` 块）。

好处：wtool 不再需要在**用户的文件**里做排序插入；删掉那一个块就能彻底去掉影响。
每次 install/uninstall 后由 `wt_env_sync` 全量重算，所以不会堆积，早期散落的
`# >>> wtool:<id>` 块会被自动清掉（迁移路径）。

## 表格的四种状态

```
│ editor/astronvim_v5  │ 70   │ 可执行     │ 可执行     │ 待构建下载 │ 待构建下载 │
    ↑ 项目                 ↑ prio  ↑ build     ↑ download   ↑ install    ↑ publish
```

行列顺序是 `项目 / prio / build / download / install / publish`；
一格是"能力 + 状态"，不是"命令"。

- **不支持**（红）—— 没这项能力
- **可执行**（黄）—— 现在就能跑
- **待构建下载**（蓝）—— 能力有，但要先 build 或 download
- **已完成**（绿）—— 跑过了

流水线：`build 或 download → install → publish`，后面的依赖前面的。
`kind="source"` 的 publish 没有前置依赖（它打的就是源码）。

**引擎不自动串联这些动作**：`wtool bootstrap` 遇到"有 build/download 脚本但一轮都没跑过"
的项目会**跳过**，把该跑的命令列出来交给用户（构建门槛在那里）；
`wtool install` 本身不拦，前置没做时是**项目自己的 `install.sh`** 报错告诉你去跑哪条。
另外 `wtool table --verbose` 会补上"什么时候装的 / 当前产物来自 build 还是 download"。

## 当前状态（截至 2026-09-17）

已完成：

- rc 收敛、`scripts/` 迁移、`wtool download`、`all` 参数、构建门槛、
  表格四态 + 表框、`wtool init`、bootstrap 只装能装的
- **`generated.tsv` 脏检查豁免**：publish 改写过的文件不再把下一次 publish 堵死
- **`install.sh` 重构成四步**（环境 → 自举 → 入口软链 → 让 wtool 进 PATH），
  发行版差异拆成 `install-env.sh` + `install-ubuntu{20,22,24,26}.sh`
- **`wtool uninstall` 先跑项目自己的 `install.sh --uninstall`**，再逆放 journal
- **容器三个脚本**：`container-shell.sh`（一步到位）/ `container-raw.sh`（什么都不装）/
  `container-proxy.sh`（自动接宿主代理，修掉"容器里裸网"）
- **publish 的失败语义**：上传失败保产物 + 非零退出 + `_failed` 计数；
  分卷默认 32M；下载走 `api.github.com` 资产端点绕开 `github.com`；
  发布包里的 `install.sh` 换成了 `extract.sh`
- 12 个项目在表里，10 个已发布 source 包（`wtool-base/README.md` 的下载块自动生成）
- 5 组测试共 147 条全过（pairing 30 / provision 24 / publish 41 / table 35 / release-copy 17）；
  `editor/astronvim_v5` 另有 52 条

**没做**（详见 `05-next.md`）：

- `release_mgr.py` + `scripts/release.json`（download 的整套机制）
- pre/post 两份记录（`pre_release.json` / `release.json`）与两个 diff
- 四目标矩阵（ubuntu 20.04 / 22.04 / 24.04 / 26.04，按 glibc 选包）
- `astronvim_v5` 的发布包（要跑一次容器构建，小时级）
- `原理.md` / `install.md`；`wtool-base` 文档同步
- 两处跨仓文档问题（改代码的人顺手修）：
  `bootstrap/scripts/container-shell.sh` 的注释指向已删除的 `harness/doc/06-排错.md`；
  `wtool-base/README.md` 让人跑 `./bootstrap/install.sh`，而脚本已迁到
  `bootstrap/scripts/install.sh`（根目录入口是 `./install.sh`）

## 权威数据在哪

| 想知道 | 看 |
|---|---|
| 引擎有哪些命令、怎么用 | `bootstrap/wtool.sh --help`（文件头注释） |
| 接口契约、不变量 | `bootstrap/docs/spec.md` |
| `wtool.xml` 能写什么 | `bootstrap/docs/manifest-schema.md` |
| 当前项目表 | `python3 bootstrap/lib/wtool_plan.py publish-list --root .` |
| 踩过哪些坑 | `03-hazards.md` |
| 为什么这么设计 | `02-decisions.md` |
| 测试怎么跑 | `bootstrap/tests/run_all.sh` |

# 01 项目全貌与现状

> 接手前先看这一篇。它讲**现在**是什么样，不讲历史。
> 历史决策看 `02-decisions.md`，踩过的坑看 `03-hazards.md`，下一步看 `05-next.md`。
>
> 快照：2026-09-15 会话末。

---

## 一句话

`wtool` 是一个**脚本调用器 + 状态管理者**：把一堆个人配置和工具拆成独立的小项目，
每个项目只声明"我有什么""我要做什么"，由 wtool 负责安装、构建、发布，以及**精确撤销**。

## 代码结构（每层只讲一件事）

```
wtool/                              工作区（repo 客户端，名字随安装位置变）
├── bootstrap/                      引擎。改行为来这儿，其他目录都是数据
│   ├── wtool.sh                    命令分发 + 各命令实现（唯一入口）
│   ├── lib/wtool_plan.py           Python 侧：只算不写（解析清单、算差异、渲染）
│   ├── lib/wtool_fs.sh             Shell 侧：只写不算（软链、原子写、journal）
│   ├── lib/wtool_os.sh             系统探测
│   ├── scripts/                    引擎自己的动作脚本
│   │   ├── install.sh              自举引擎 + 装整个工作区（也是"没 git clone"的入口）
│   │   ├── uninstall.sh            反向
│   │   └── container-shell.sh      一条命令进容器开发环境
│   ├── templates/                  wtool init 用的模板（.tpl 后缀）
│   ├── tests/                      5 组测试，136 条断言
│   └── docs/                       spec.md / manifest-schema.md / roadmap.md
├── wtool-base/                     用户文档。只有文档，没有工具
│   ├── README.md                   总览 / 没 git clone 怎么装 / 怎么用
│   └── guide.md                    结构 / 命令 / 子项目索引
├── harness/                        你正在看的：给 AI 助手的项目记忆
├── os/ shell/ terminal/ editor/ themes/    各具体项目
└── .wtool/                         安装痕迹（在 $HOME 里，不在仓库里）
```

**分层原则**：`bootstrap/` 是唯一有行为的地方；其他项目要么是纯数据（配置文件 + 声明），
要么带 `scripts/`。文档分两处 —— `wtool-base/` 给人看，`harness/` 给助手看。
**两者不要互相掺**：用户文档里不写本地路径和内部实现，harness 里不写教程。

## 核心契约（改代码前必须理解的三条）

### 1. 声明式是默认，脚本是例外

| 动作 | 默认（声明式） | 例外（脚本） |
|---|---|---|
| install | `wtool.xml` 的 `<link>` / `<env>` | `scripts/install.sh` |
| download | `scripts/release.json` 存在即可下载 | `scripts/download.sh` |
| build | —— 构建没有声明式的可能 | `scripts/build.sh` |
| publish | `<publish kind="source">` 打源码包 | `scripts/publish.sh` |

**表格里"能不能做"就是查这张表**：左栏有就是引擎的通用能力，右栏有就是项目自带脚本。

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
$WTOOL_STATE/<项目 id>/
├── meta.tsv         项目元信息（priority / project_root / head / installed_at）
├── journal.tsv      "当前该撤销什么"，uninstall 逆着做，**永不截断**
├── actions.tsv      "做过什么"的时间线（build / download），只追加，不参与回滚
├── artifacts.tsv    当前磁盘上的产物是谁产出的（build 还是 download）
├── env.zsh/.bashrc  这个项目的环境变量块（汇总进 ~/.wtool/.zshrc）
└── provisioned/     provision 任务的幂等 marker
```

**journal 和 actions 是两件事，别混。** 前者描述"该撤销什么"，后者是时间线。

**`build.sh` 和 `download.sh` 必须把产物放到同样的路径** —— 这是
`wtool build X && wtool install X` 与 `wtool download X && wtool install X`
等价的前提。脚本通过 `$WTOOL_ARTIFACTS` 拿到清单文件路径，往里追加
`kind<TAB>相对$HOME的路径<TAB>来源<TAB>时间`。

## 环境变量汇总（rc 收敛）

用户 rc 里**只有一段** loader 块，指向 `~/.wtool/.zshrc` / `~/.wtool/.bashrc`；
那两份文件由 wtool **整份生成**（按 priority 拼接各项目的 `env.<shell>` 块）。

好处：wtool 不再需要在**用户的文件**里做排序插入；删掉那一个块就能彻底去掉影响。
每次 install/uninstall 后由 `wt_env_sync` 全量重算，所以不会堆积，早期散落的
`# >>> wtool:<id>` 块会被自动清掉（迁移路径）。

## 表格的四种状态

```
│ editor/astronvim_v5  │ 70 │ 可执行 │ 不支持 │ 待构建下载 │ 待构建下载 │
```

- **不支持**（红）—— 没这项能力
- **可执行**（黄）—— 现在就能跑
- **待构建下载**（蓝）—— 能力有，但要先 build 或 download
- **已完成**（绿）—— 跑过了

流水线：`build 或 download → install → publish`，后面的依赖前面的。
`kind="source"` 的 publish 没有前置依赖（它打的就是源码）。

**四个命令都不自动串联**：前置没做时 install/publish 直接报错告诉你去跑哪条。
`wtool bootstrap` 只装"不需要决策"的那部分，剩下的列成命令清单交给用户。

## 当前状态（截至 2026-09-15）

已完成：

- rc 收敛、`scripts/` 迁移、`wtool download`、`all` 参数、构建门槛、
  表格四态 + 表框、`wtool init`、bootstrap 只装能装的
- 8 个项目的 release 已发布（source 包），`wtool-base/README.md` 的下载块自动生成
- 5 组测试共 136 条全过

**没做**（详见 `05-next.md`）：

- `release_mgr.py` + `scripts/release.json`（download 的整套机制）
- 四目标矩阵（ubuntu 20.04 / 22.04 / 24.04 / 26.04，按 glibc 选包）
- `astronvim_v5` 的发布包（要跑一次容器构建，小时级）
- `原理.md` / `install.md`；`wtool-base` 文档同步
- **`generated.tsv`（脏检查豁免）必须最先做** —— 否则 publish 改完
  `release.json` / README 会把下一次 publish 堵死（已经踩过一次）

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

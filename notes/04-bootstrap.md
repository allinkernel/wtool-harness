# 04 bootstrap 引擎现状

**代码的真相在 `bootstrap/docs/`，这里只记概览和状态。**
（最后一次对着代码核对：2026-09-17。行数会漂，看数量级就行。）

| 文档 | 内容 |
|---|---|
| `bootstrap/docs/spec.md` | 接口契约（不变量、数据流、块格式、hash 策略、兼容承诺） |
| `bootstrap/docs/manifest-schema.md` | `wtool.xml` 字段表 |
| `bootstrap/docs/roadmap.md` | 预留设计（`provision` 那条已标"已实现"，`--prune`/`--exact`/锁还没做） |
| `bootstrap/README.md` | 快速上手 |

## 文件与规模

| 文件 | 行数（约） | 角色 |
|---|---|---|
| `bootstrap/wtool.sh` | 1670 | CLI 分发 + 所有命令实现（唯一入口） |
| `bootstrap/lib/wtool_plan.py` | 1930 | 清单解析、校验、冲突检测、计划、表格、env 渲染、下载块 |
| `bootstrap/lib/wtool_fs.sh` | 880 | 软链、原子写、journal、registry、发布上传 |
| `bootstrap/lib/wtool_os.sh` | 40 | 系统探测（`/etc/os-release` + `uname` + `nproc`） |
| `bootstrap/templates/*.tpl` | 4 个 | `wtool init` 生成 `scripts/` 用的模板（**没有 stub.sh**，见 ADR-013） |
| `bootstrap/scripts/` | 10 个 | 引擎自己的动作脚本：`install.sh` / `install-env.sh` / `install-ubuntu{20,22,24,26}.sh` / `uninstall.sh` / `container-{shell,raw,proxy}.sh` |
| `bootstrap/tests/` | 7 个 `*_test.sh` | `run_all.sh` 跑 5 组 147 条；`container_test.sh`（要 docker）和 `e2e_repo_sync_test.sh`（慢）人工跑 |

## 版本

| 项 | 值 |
|---|---|
| 引擎版本 | `1.0.0` |
| 清单 schema | `1` |
| `plan.tsv` 列 | 6（action/kind/dest/source/sha256/extra，只增不改） |
| 状态文件格式 | TSV，按列读（加列不影响老版本解析） |

## 已实现（对着 `wtool.sh --help` 核过）

- `install`（`--dry-run` / `--force` / `--no-script`）—— 跑完项目自己的
  `scripts/install.sh`，最后全量重算 env 汇总
- `uninstall`（`<项目目录>` 或 `--id`）—— **先跑项目 `install.sh --uninstall`**，
  再逆放 journal（ADR-015）
- `build` / `download`（`<项目>...|all`、`--dry-run`）—— 跑
  `scripts/build.sh` / `scripts/download.sh`，记 `actions.tsv`
- `provision`（`--with-system`）—— `<system-file>` / `<source>` / `<task>` 三件套，
  见 `spec.md` §11
- `publish`（`--tag=` / `--out=` / `--force` / `--allow-foreign`）——
  source 包 / 项目 `publish.sh` / 声明为不发布，三条分支
- `bootstrap`（`--with-system` / `--no-system` / `--install-only`）——
  只装"不需要决策"的项目，需要先 build/download 的跳过并列出来
- `table`（四态 + `--verbose` / `--summary` / `--color=`）、`list`、`status`、`doctor`
- `env`（`--quiet` / `--json`）、`validate`、`init`（`--all` 生成 `scripts/` 模板）、
  `version`
- 跨项目 dest 冲突检测（靠 registry）；rc 块被手改的检测（靠 journal 里的块 sha）；
  **仓库搬家**：旧的中转链接会被识别为"自己的"并更新（`wt_journal_owns`）
- 脏检查豁免 `$WTOOL_STATE/generated.tsv`（ADR-018）
- `docs refresh` / `refresh-downloads` —— 重写文档里 `wtool:downloads` 标记之间的内容
  （**没有**出现在 `--help` 里；`wtool docs` 不带参数也会刷新）

## 未实现（按优先级）

| 优先级 | 能力 | 触发场景 |
|---|---|---|
| 中 | `--prune` | 从清单删掉 link 后重装，旧软链仍残留到 uninstall |
| 中 | `release_mgr.py` + `scripts/release.json`（通用 download） | 见 `05-next.md` 第 1 条 |
| 低 | `--exact`（用历史 `wtool.xml` 推导逆操作） | journal 够用 |
| 低 | 并发锁 | 多终端同时装 |

## 状态目录布局

```
$WTOOL_STATE/                    默认 ~/.local/state/wtool（$XDG_STATE_HOME 优先）
├── registry.tsv                 dest \t 项目id \t kind
├── generated.tsv                路径 \t 项目id \t 时间（wtool 自己写过的文件）
├── created-rc.tsv               当初由 wtool 创建的 rc 文件（卸空了就删）
└── <项目id>/
    ├── meta.tsv                 project_id/project_root/schema/priority/manifest_sha/head/at/engine/installed_at
    ├── journal.tsv              action \t kind \t dest \t target \t sha256
    │                            action ∈ {link, mkdir, rc, rccreate, backup, sysfile, srcdir}
    │                            （`reg` 不记 journal —— 它只进 registry）
    ├── actions.tsv              动作 \t 时间 \t 说明（build / download，只追加）
    ├── artifacts.tsv            kind \t 相对$HOME路径 \t 来源 \t 时间
    ├── publish.tsv              时间 \t repo \t tag \t 资产数 \t 说明
    ├── env.zsh / env.bash       这个项目的 env 块（被汇总进 ~/.wtool/.zshrc）
    ├── provision.log            provision 任务的输出（只追加，用于审计/判断跑没跑过）
    ├── provisioned/<marker>     provision 任务的幂等标记
    └── system/<slug>/           system-file 的备份：original（原文件）、dest（目标路径）
```

# 04 bootstrap 引擎现状

**代码的真相在 `bootstrap/docs/`，这里只记概览和状态。**

| 文档 | 内容 |
|---|---|
| `bootstrap/docs/spec.md` | 接口契约（不变量、数据流、块格式、hash 策略、兼容承诺） |
| `bootstrap/docs/manifest-schema.md` | `wtool.xml` 字段表 |
| `bootstrap/docs/roadmap.md` | provision / system scope / prune / exact / 锁 等预留设计 |
| `bootstrap/README.md` | 快速上手 |

## 文件与规模

| 文件 | 行数（约） | 角色 |
|---|---|---|
| `bootstrap/wtool.sh` | 320 | CLI 分发 + install/uninstall 流程 + list/status/doctor/scaffold |
| `bootstrap/lib/wtool_plan.py` | 650 | 清单解析、校验、冲突检测、rc 块渲染与排序插入 |
| `bootstrap/lib/wtool_fs.sh` | 200 | 软链、原子写、journal、registry |
| `bootstrap/templates/stub.sh` | 45 | 项目存根模板 |
| `bootstrap/tests/pairing_test.sh` | 215 | 8 场景 / 20 断言 |

## 版本

| 项 | 值 |
|---|---|
| 引擎版本 | `1.0.0` |
| 清单 schema | `1` |
| `plan.tsv` 列 | 6（action/kind/dest/source/sha256/extra，只增不改） |

## 已实现

- `install` / `uninstall`（含 `--id` 方式，仓库删了也能卸）
- `--dry-run` / `--force`
- `list`（全局 registry）/ `status`（缺失软链检测）/ `doctor`
- `scaffold`（生成 `wtool.xml` + `env.zsh` + 两个存根）
- `validate`
- 跨项目 dest 冲突检测（靠 registry）
- rc 块被手改的检测（靠 journal 里的块 sha）
- **仓库搬家**：旧的中转链接会被识别为"自己的"并更新（`wt_journal_owns`）

## 未实现（按优先级）

| 优先级 | 能力 | 触发场景 |
|---|---|---|
| 高 | `provision`（apt/编译） | `os/ubuntu`、`editor/nvim` 迁移时会立刻需要 |
| 高 | `system scope`（写 `/etc`） | `os/ubuntu` 换镜像源 |
| 中 | `--prune` | 从清单删掉 link 后重装 |
| 低 | `--exact`（历史版推导逆操作） | journal 够用 |
| 低 | 并发锁 | 多终端同时装 |

## 状态目录布局

```
$WTOOL_STATE/                    默认 ~/.local/state/wtool
├── registry.tsv                 dest \t 项目id \t kind
└── <项目id>/
    ├── meta.tsv                 project_id/project_root/schema/priority/manifest_sha/head/at/engine/installed_at
    └── journal.tsv              action \t kind \t dest \t target \t sha256
                                 action ∈ {reg 不记, link, mkdir, rc, backup}
```

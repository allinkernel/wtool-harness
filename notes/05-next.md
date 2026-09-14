# 05 下一步

## 等用户决策（阻塞项）

| # | 问题 | 现状 |
|---|---|---|
| 1 | `w_manifests` 是继续与博客共用，还是给 wtool 单开清单仓？ | 两个分支内容还一样，最容易踩坑 |
| 2 | 仓库命名统一成 `wtool-<路径>`，还是保留 `wsw_*` 原名只换 remote？ | 影响后续所有仓的创建 |
| 3 | shell 注入用 `ZDOTDIR` 接管还是"受管块追加"？ | 已实现后者；`~/.zshenv` 不存在，前者成本也很低 |

## 用户自己要做的事（agent 不要代劳）

1. 提交 wtool 清单到 `w_manifests` 的 `wtool` 分支（**最紧急**，否则一次 sync 就回退）。
2. 给 5 个迁移仓建 github 仓并 push（本地分支是 `main`，remote 仍指向 gitee）。
3. 给 `bootstrap`、`harness` 首次提交（这两个仓 agent 未提交）。
4. 决定是否清理 `~/self/wtool/{astronvim_v5,typora-theme,.mypy_cache}` 残留。

## 建议的下一步（按价值排序）

| 顺序 | 事项 | 说明 |
|---|---|---|
| 1 | 真机试用 5 个已迁移项目 | `cd <项目> && ./install.sh`（现在都已提交，不需要 `--force`），开新 shell 验证 |
| 2 | ~~实现 `provision`~~ | ✅ 已完成（system-file / source / provision + bootstrap） |
| 3 | ~~迁移 `os/ubuntu`~~ | ✅ 已完成（换源自动生成 + Ansible 装包） |
| 4 | 迁移 `editor/nvim` | 引擎已支持 `<source>`+`wsw.sh`，等建仓 |
| 5 | 处理大文件 | fzf 二进制 4.4M（改 release 下载）、oh-my-zsh 11M（可接受） |
| 6 | `shell/oh-my-zsh` 与上游同步策略 | 现在是 fork，升级需要 rebase/merge |

## 迁移一个项目的标准流程（照抄即可）

```sh
# 1. 在目标路径建目录（例如 tools/repo）
mkdir -p ~/self/wtool/tools/repo

# 2. 复制原项目内容（不要带 .git）
cp -r ~/source/mytool/android/. ~/self/wtool/tools/repo/

# 3. 写清单
$EDITOR ~/self/wtool/tools/repo/wtool.xml

# 4. 生成存根
~/self/wtool/bootstrap/wtool.sh scaffold ~/self/wtool/tools/repo --id tools/repo

# 5. 校验
~/self/wtool/bootstrap/wtool.sh validate ~/self/wtool/tools/repo

# 6. 干跑
WTOOL_HOME=$(mktemp -d) ~/self/wtool/bootstrap/wtool.sh install ~/self/wtool/tools/repo --dry-run --force

# 7. 真装（用户自己提交后再去掉 --force）
~/self/wtool/bootstrap/wtool.sh install ~/self/wtool/tools/repo
```

**改写 env 文件的要点**：把原来靠 `get_this_dir` / `WSW_*_DIR` 算路径的写法，
换成直接用 `$WTOOL_PROJECT_DIR` / `$WTOOL_PROJECT_ROOT`；删掉 `echo` 等副作用。

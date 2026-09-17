# 02 设计决策（ADR）

每条：背景 → 决策 → 理由 → 被否决的方案。
**新增决策请追加编号，不要改写历史结论**（结论变了就新加一条并注明取代了哪条）。

---

## ADR-001 放弃"安装/复制"，改为纯软链 + 受管块

**背景**：旧的 `install_all.sh` 把仓库内容复制/安装到 `$HOME`。改了家目录里的文件，
仓库那份就过期，两边都要改。

**决策**：仓外只允许两种改动形态：① 软链；② 带标记的"受管块"（往 rc 文件里注入）。

**理由**：软链天然可撤销；受管块有明确边界，能精确删除；二者都可用脚本证明"配对"。

**否决**：继续用复制 + 同步脚本（永远有漂移）；用 chezmoi/yui 这类"目标即真相"的工具
（会推翻"仓库是真相"这个前提）。

---

## ADR-002 过程仍叫 install，但 install 只做可逆的事

**背景**：项目里有不可逆操作（apt 安装、源码编译、写 `/etc`）。

**决策**：`install` = 软链 + rc 注入，**保证可逆**；不可逆的东西放未来的 `provision` 子命令，
`install` 永不自动调用它。

**理由**："完全配对"是这套设计的核心卖点，混入不可逆操作就毁了它。

**否决**：把编译/apt 塞进 install（会让 uninstall 变成谎言）。

---

## ADR-003 引擎与数据分离：bootstrap 一份，项目只放声明

**背景**：如果每个项目都自带完整的 `install.sh`，逻辑会复制 N 份，改一处要改 N 处。

**决策**：`wtool-bootstrap` 持有引擎；每个项目只放 `wtool.xml` + `env.*` + 两个存根。

**理由**：引擎唯一 → 行为一致、可单测、可演进；项目只声明 → 解耦、可单独下载。

**代价**：项目不能脱离 bootstrap 独立安装（存根找不到引擎时会打印明确的克隆命令）。

**否决**：把引擎代码复制进每个项目（重复且必然漂移）。

---

## ADR-004 Python 只算不写，Shell 只写不算

**背景**：解析 XML / 计算 rc 最终内容 / 排序插入，用 shell 写会非常痛苦；
但落盘（原子写、跟随软链、日志）用 shell 更直接。

**决策**：
- `lib/wtool_plan.py`：读清单/读 rc/读 registry → 输出 `plan.tsv` + 新 rc 内容到 scratch，
  **不写 $HOME**。
- `lib/wtool_fs.sh`：逐行执行 plan，**不做文本逻辑判断**。

**理由**：py 可单测（纯输入输出）；破坏性操作只有一处实现；不会出现两套不一致的文本逻辑。

**否决**：py 直接写文件（两处实现）；sh 用 sed/awk 改 rc（易错、难测）。

---

## ADR-005 清单格式选 XML（不是 TOML/YAML）

**背景**：bootstrap 要在**全新机器**上运行，那时 Python 版本未知。

**决策**：`wtool.xml`，用标准库 `xml.etree` 解析。

**理由**：`xml.etree` 从 Python 3.0 起就在标准库；TOML 需要 3.11+ 的 `tomllib`；
YAML 需要 PyYAML（外部依赖，直接排除）。且 XML 与 repo manifest 同构，用户已有经验。

**否决**：TOML（可读性更好但版本门槛）、YAML（依赖）、JSON（无注释）。

---

## ADR-006 受管块里记 `head` 但不记时间戳

**背景**：用户要求块里带上仓库 hash，uninstall 时校验。

**决策**：块内字段 = `id / schema / engine / prio / head / manifest`。
**不记时间戳**；时间等溯源信息写 `$WTOOL_STATE/<id>/meta.tsv`。

**理由**：块是**契约**不是日志。实测发现带时间戳会让"重复 install"重写 rc 文件，
破坏幂等（回归场景 2）。另外只记 `head` 做硬校验会导致"改一行配置就卸不掉"，
所以 `head` 只用于**提示**，真正做硬校验的是 rc 块内容 sha。

**否决**：块里放 `at=`（破坏幂等）；只用 HEAD 做硬校验（误报）。

---

## ADR-007 两级软链（中转链接 + 项目内链接）

**背景**：仓库位置会变（`~/self/wtool` ↔ `~/source/wtool` ↔ `~/.wtool` 软链）。

**决策**：
```
$HOME/.tmux.conf -> $HOME/.wtool/links/terminal/tmux/tmux.conf -> 仓库真实文件
```
所有 rc 块也只引用 `$HOME/.wtool/links/<id>/...`。

**理由**：仓库搬家后，**rc 块和已建的软链都不用改**，重跑 install 重建中转链接即可；
块内容在不同机器上一致。

**代价**：`readlink` 多一跳；`id` 变成对外契约（改了要重装）。

**否决**：块里直接写仓库绝对路径（搬家即失效）。

---

## ADR-008 块顺序按 `(priority, id)` 排序插入，而不是 append

**背景**：多个项目各自独立安装，顺序不可控；但 `oh-my-zsh` 之类必须靠前。

**决策**：插入点 = "最后一个排序小于我的块之后"；没有更小的块则插到最前面。

**理由**：数学上保证最终顺序只取决于 `(priority, id)`，与安装顺序无关（测试场景 3 验证）。

**否决**：直接 append（顺序依赖安装历史）。

---

## ADR-009 journal 是"撤销依据"，按 `(action,dest)` 去重且永不截断

**背景**：最初实现里 install 会清空 journal 再重建。

**决策**：journal 只做**去重更新**，从不截断；uninstall 逆序重放它。

**理由**：实测 bug —— 重复 install 后 journal 被清空，导致 uninstall 留下软链和目录。
去重保证"当前该撤销什么"是一个集合，而不是历史流水。

**否决**：uninstall 时按当前清单重新推导（清单可能已改，推不出旧状态）。

---

## ADR-010 项目存根只做"找引擎 + exec"

**背景**：用户要求"下载项目目录后直接执行 `install.sh`"。

**决策**：`install.sh`/`uninstall.sh` 是同一模板的副本，靠自身文件名判断子命令；
查找顺序 `$WTOOL_BOOTSTRAP` → 向上 4 层找 `bootstrap/` 或 `wtool-bootstrap/` → `~/.wtool/bootstrap`。

**理由**：满足"直接执行"的诉求，同时把逻辑集中在引擎；模板唯一，scaffold 直接复制。

**否决**：项目里写完整逻辑（ADR-003 已否决）。

> 后续：这套"存根"方案已经作废，见 **ADR-013**（动作脚本统一住 `scripts/`，
> 存根全部删除）。这里保留原文，讲清楚当初为什么这么做。

---

## ADR-011 git 检查：要"干净"，但不要"分支"

**背景**：用户要求"仓库没有提交拒绝执行 install.sh"。

**决策**：要求在工作区内且 `git status --porcelain -uno` 为空；
**不要求有分支**，也不因 untracked 文件而拒绝。非 git 目录默认拒绝，`--force` 放行（仅测试用）。

**理由**：
- repo 工具默认 **detached HEAD**（用户现有两个 wtool 仓就是 `* (no branch)`），查分支会卡死；
- nvim 等项目会在仓库里生成 `data/state` 目录，用不带 `-uno` 的检查会永远装不上。

**否决**：要求分支；把 untracked 也当脏。

---

## ADR-012 演示项目选 `terminal/tmux`

**背景**：需要一个真实迁移样例验证整条链路。

**决策**：迁移 mytool 的 `tmux`：1 个 link + 1 个 env，体量小但覆盖了全部机制。

**理由**：能同时验证软链、rc 块、优先级、uninstall 配对，且改动风险低。

**注意**：`bin/disk.sh` 的管道子 shell bug **故意保留未修**，避免混入行为变更。

---

## ADR-013 动作脚本统一住 `scripts/`，项目存根作废（取代 ADR-010）

**背景**：ADR-010 让每个项目放 `install.sh` / `uninstall.sh` 存根（同一模板的副本），
靠文件名判断子命令。同名 `.sh` 到处都是，改错层"看起来生效了"但实际没跑（已经害过一次）。
而且引擎后来改成"`wtool install` 会跑项目自己的 `install.sh`"——存根里再调
`wtool install` 就成了无限递归。

**决策**：
- 一个项目的能力**由 `scripts/<动作>.sh` 在不在声明**（build / download / install / publish）；
  项目根的老位置仍然认，但引擎会警告"应该挪到 `scripts/` 下"。
- 所有存根删除；需要自定义安装的项目才提供 `scripts/install.sh`。
- 项目不需要 `uninstall.sh`：卸载钩子是同一个 `scripts/install.sh --uninstall`。
- `wtool init --all` 从 `templates/*.tpl` 生成 `scripts/` 下的模板。

**理由**：一个项目目录里最多只有一个 `install.sh`，不会再和根目录入口、引擎脚本混淆；
"文件存在即能力声明"让表格的格子有唯一判据（`pipeline_states()` 只看文件在不在）。

**证据**：`606bac7`（动作脚本迁进 `scripts/`）、`c7ba44c`（职责重构）、
`cf79722`（存根时代的最后一个补丁：让存根被软链调用时也能找到引擎）；
`cmd_install` 里的注释写着"存根已经全部删除"。

**否决**：保留存根（递归风险 + 混淆）。

---

## ADR-014 `install.sh` 只装 wtool 自己：四步 + 发行版 profile 派发

**背景**：老的 `install.sh` 既装系统环境、又自举引擎、还顺手把整个工作区装掉。
用户看到一屏输出，失败时分不清是系统环境的问题还是某个项目的问题。

**决策**：拆成四步，做完就停：
```
0 准备运行环境（探测发行版 → 派发给 install-<发行版><主版本>.sh）
1 自举引擎到 ~/.wtool/bootstrap
2 建工作区入口软链
3 wtool install <引擎自己> --force   ← 让 wtool 进 PATH
```
发行版差异组织成：`install-env.sh`（共用逻辑）+ 每个发行版一个小文件
（只声明 `ENV_NAME` / `ENV_ANSIBLE` / `ENV_EXTRA_PKGS` 并调 `env_prepare`）。
被引擎调用时（`WTOOL_PROJECT_ID` 已设）**只自举、立即退出**，不能往下跑第 0 步。

**理由**：两件事失败原因不同（系统环境 vs 项目自身），分开用户才知道该修哪一头；
加新发行版 = 加一个小文件；第 3 步的输出绝不能吞（静默失败比报错更坏）。

**证据**：`365f210`。

**否决**：继续"一条命令装完"（用户明确要求拆开）。

---

## ADR-015 uninstall 先跑项目钩子，再逆放 journal

**背景**：install 会跑项目自己的 `install.sh`，uninstall 却只逆放引擎的 journal ——
项目脚本装的东西（编译产物、下载的包、铺到 `$HOME` 的配置）不在那本 journal 里。
实测 astronvim_v5：`wtool uninstall` 跑得"成功"，`plan-uninstall` 出来 `actions: 0`，
东西一个没少（用户报的"装完撤不回来"）。

**决策**：`wtool uninstall` 的步骤固定为
**项目 `install.sh --uninstall`（存在才跑）→ rc 回退 → plan 里其余动作 → 逆序重放 journal
→ 重算 env 汇总 → 清状态目录**。钩子失败默认 `die`，要 `--force` 才继续。
（`cmd_uninstall` 里判断的是环境变量 `WTOOL_NO_SCRIPT=1`；`--no-script` 那个 CLI 参数
目前只有 `install` 认 —— 想跳过卸载钩子得自己 `WTOOL_NO_SCRIPT=1 wtool uninstall ...`。）

**理由**：顺序不能反 —— 项目脚本删的是实体（大件），引擎删的是软链和状态；
软链先没了，项目脚本可能就找不到自己装的东西。卸载没干净比装失败更危险：
后者看得见，前者是"以为清干净了"。

**证据**：`ef586b9`。

**否决**：uninstall 只信 journal；钩子失败继续往下走。

---

## ADR-016 分卷大小按"网络可靠性窗口"定（默认 32M），失败保产物

**背景**：实测直连上传只有 ~237 KB/s，链路每隔几分钟断一次。314M / 576M 的单次 POST
注定完不成，重试就是从头再来。同时 `cmd_publish` 的 trap 无条件删临时目录，
而半小时才编出来的产物就在里面；上传函数还是 `|| wt_die`，调用点根本轮不到判断：
**传失败连产物一起没，还退出 0。**

**决策**：
- 分卷默认 `VOLUME_SIZE=32M`（每个约 2 分钟），值记进 `dist.json`；
  下载侧不看这个值，它照 `dist.json` 的 `volumes` 清单（名字 + 字节数 + sha256）
  逐个下，所以两边天然一致。
- `wt_publish_gh_upload` **返回非零而不是 die**；调用方决定处置。
- `cmd_publish` 用 `_failed` 计数，有项目没发出去就**非零退出**；
  失败时 `_keep_scratch=1` 保留产物并打印补传命令。
- 上传前用 `gh api /rate_limit` **探一次路**（选代理或直连走到底），
  中途失效再换另一条试一次。

**理由**：卷大小不是随便定的，它要和网络的可靠性窗口匹配；
退出码是调用方（脚本、CI、后台任务）唯一的信号，报"成功"比报错更坏。

**证据**：`ad3d24a`、`e73f0fd`、`c5e33b8`；分卷默认值在
`editor/astronvim_v5/scripts/publish.sh`（`VOLUME_SIZE=${VOLUME_SIZE:-32M}`）。

**否决**：一个整包（断了从头再来）；上传失败即 die（产物一起毁、退出码还是 0）。

---

## ADR-017 容器网络：自动探测宿主代理，不让人去猜

**背景**：`docker run` **不会**把宿主 shell 的环境变量带进容器（除非显式 `-e`）。
于是宿主 `curl` 什么都通、容器里全失败，人很容易归因成"网络坏了""GitHub 连不上"，
去查完全错误的方向。

**决策**：新增 `container-proxy.sh`，两个容器脚本都 source：
1. 容器里已有代理变量（`-e` 传进来的）→ 直接用，不干预；
2. 没有 → 探一次宿主代理（默认 `127.0.0.1:7897`，依赖 `--network=host`），
   通了就设上大小写四个变量 + `no_proxy=127.0.0.1,localhost`，并**明确说出来**；
3. 都没有 → 说清后果并给出该加的 `-e` 参数。
可关：`WTOOL_NO_PROXY=1`；换端口：`-e WTOOL_HOST_PROXY=...`。
**探测必须在独立子进程里做**（`timeout 3 bash -c 'exec 3<>/dev/tcp/...'`）——
在当前 shell 里开 fd 会让之后 `exec bash -i` 不给提示符，看起来和卡死一模一样。

**理由**：这种"环境差异"最贵的不是失败本身，是它把人引到错误的方向；
另外"卡住"和"在等你输入"必须能区分开。

**证据**：`5a3133d`、`b99d5e3`（提示符那条）。

**否决**：默认直连（容器里全失败）；无条件透传 `HTTP_PROXY`（`127.0.0.1` 在容器里是容器自己）。

---

## ADR-018 生成物登记表 `generated.tsv`：脏检查豁免

**背景**：`wtool publish` 会改写受版本控制的文件（文档里的下载块，以后的
`release.json` / `pre_release.json`）。改完这些文件就"脏"了，**下一轮 publish
会以"有未提交改动"拒绝这个项目** —— 一次发布把下一次发布堵死。
最初只对 README 做了一个文件名的特判。

**决策**：`wt_atomic_write`（唯一的写文件入口）统一登记到
`$WTOOL_STATE/generated.tsv`（路径 / 项目 / 时间）；
`wt_git_dirty` 把 `git status --porcelain` 减去登记过的文件，剩下的才算真脏。
`cmd_publish` 用它，被拒时打印真实改动的前几行。

**理由**：生成物只会越来越多（`release.json`、下载块……），逐文件特判堆不下去，
也分不清"这是 wtool 写的"还是"用户改的"；只豁免 wtool 自己写过的，用户手改照样算脏。

**证据**：`24bf6e2`。"连续跑两次 publish，第二次不能报有未提交改动"已经验证。

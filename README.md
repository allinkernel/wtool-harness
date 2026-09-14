# harness —— wtool 项目的"外部记忆"

这个目录**不是**代码，是给**任何 agent**（DSH / Gemini / 人）读的项目记忆。
目的：换一个 agent、换一台机器，读这里就能掌握项目全貌并接着干活。

## 怎么读（建议顺序）

| 顺序 | 文件 | 读它你会得到 |
|---|---|---|
| 1 | `skill/wtool/SKILL.md` | 可执行的技能卡片：项目是什么、禁区是什么、怎么验证 |
| 2 | `notes/01-context.md` | 仓库清单、两套 checkout 的现状、GitHub 上已有什么 |
| 3 | `notes/02-decisions.md` | 所有关键设计决策 + 被否决的方案 + 理由 |
| 4 | `notes/03-hazards.md` | **禁区**：哪些操作绝对不能做 |
| 5 | `notes/04-bootstrap.md` | 引擎现状（细节在 `bootstrap/docs/`） |
| 6 | `notes/05-next.md` | 下一步待办 |
| 7 | `journal.md` | 按时间追加的操作流水（谁做了什么、验证结果） |

## 约定

1. **每轮对话/每次操作结束都要在 `journal.md` 追加一条**，包含：时间、操作、命令、结果、遗留问题。
2. 新的设计决策写进 `notes/02-decisions.md`（编号递增，ADR 风格：背景 → 决策 → 理由 → 被否决的方案）。
3. 新的坑/禁区写进 `notes/03-hazards.md`。
4. 代码的真相在 `bootstrap/docs/spec.md`；这里只记**为什么**和**历史**。
5. 本目录和 `bootstrap/` 都是 git 仓库，但**由用户自己提交**，agent 不要执行任何 git 写操作（见 `notes/03-hazards.md`）。

## 关于 skill 的加载

技能卡片在 `harness/skill/wtool/SKILL.md`。DSH 的技能发现路径不包含 `harness/`，
所以两种加载方式：

```sh
# 方式一：直接读
cat harness/skill/wtool/SKILL.md

# 方式二：软链到 DSH 会发现的位置（可选，用户自己决定）
ln -s "$PWD/harness/skill/wtool" "$PWD/skills/wtool"
```

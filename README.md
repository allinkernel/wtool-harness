# harness —— wtool 的项目记忆

**这个目录不是代码，也不是给用户看的文档。** 它是接手这个项目的 AI 助手（或几个月后的我自己）需要的东西：设计为什么这么定、踩过哪些坑、发生过什么。

用户文档在另一个仓库：[wtool-base](https://github.com/allinkernel/wtool)（工作区里的 `wtool-base/`），由它通过软链接暴露成工作区根目录的 `README.md` 和 `guide.md`。**用户文档不要写在这里。**

## 目录

| 路径 | 是什么 |
|---|---|
| `notes/` | 交接笔记。`01-context` 讲全貌与现状，接手前先看它；`02-decisions` 记设计取舍；`03-hazards` 记踩过的坑（**改代码前必读**）；`04-bootstrap`、`05-next` 是具体进度 |
| `skill/wtool/SKILL.md` | 给 AI 助手用的技能定义，说明什么时候该介入、按什么流程做 |
| `journal.md` | 时间线：哪次会话做了什么、为什么 |

## 和其他仓库的关系

```
wtool-base      用户文档（README.md / guide.md）
bootstrap       引擎本体 —— 改行为去这里
harness         你正在看的：给助手的上下文
os/ shell/ terminal/ editor/ themes/    具体项目
```

改 `bootstrap/` 之前先看 `notes/03-hazards.md`。那里记着的坑，基本都是"看起来无关紧要的一行改动，毁掉了一整个已装好的环境"这类。

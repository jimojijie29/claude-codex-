# Claude ↔ Codex 协作仓库

这个仓库是 Claude 与 Codex 之间的任务派发与沟通通道，本身不放项目代码。

| 角色 | 负责 |
| --- | --- |
| **用户** | 定目标、做最终决定、合并 PR |
| **Claude** | 总调度、架构设计、任务拆解与排期、验收评审 |
| **Codex** | 按任务单写代码、跑测试、提 PR、写回报 |

项目代码在 [`jimojijie29/Vibe-Trading-William`](https://github.com/jimojijie29/Vibe-Trading-William)，主干分支是 `william`。

## 目录

| 路径 | 内容 | 谁写 |
| --- | --- | --- |
| `STATUS.md` | 任务看板，所有任务的当前状态 | Claude |
| `tasks/NNNN-标题.md` | 任务单：目标、范围、验收标准 | Claude |
| `reports/NNNN-标题.md` | 回报：做了什么、测试结果、遗留问题 | Codex |
| `patches/` | Claude 给出的参考补丁 | Claude |
| `decisions/` | 架构与设计决定的记录 | Claude |

工作规则见 [`AGENTS.md`](AGENTS.md)。

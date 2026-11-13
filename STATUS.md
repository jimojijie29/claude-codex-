# 任务看板

状态取值：`待领取` → `进行中` → `待验收` → `已完成`，另有 `受阻`、`已取消`。

| 编号 | 任务 | 状态 | 依赖 | PR | 回报 |
| --- | --- | --- | --- | --- | --- |
| 0001 | [yfinance 限流误报修复](tasks/0001-yfinance-preflight.md) | 验收通过，待用户合并 | 无 | [Vibe-Trading-William#1](https://github.com/jimojijie29/Vibe-Trading-William/pull/1) | [回报](reports/0001-yfinance-preflight.md) |
| 0002 | [实测美股、港股的后备数据源](tasks/0002-fallback-chain-verification.md) | 待领取 | 无 | | |
| 0003 | [让 CI 在 william 分支上运行](tasks/0003-ci-on-william.md) | 待领取 | 无 | | |

## 验收记录

### 0001（2026-11-13，通过）

- PR 提交 `9b32b81`，改动 4 个文件，均在任务范围内，与 `william` 无冲突。
- Claude 在独立检出中复跑 `tests/test_preflight.py`：6 个全部通过。
- 五条验收标准均满足。Codex 对提示文案的改动（不再保证限流是临时的，分别列出美股、港股的后备链）比参考补丁更准确，予以采纳。
- PR 上没有自动检查：仓库有 `test.yml`，但只对 `main` 触发，而 PR 的目标分支是 `william`（见任务 0003）。验证依据是 Codex 的本地运行和 Claude 的复跑。
- 遗留：用户环境的 yfinance 为 1.4.1 仍被限流，说明是出口 IP 被 Yahoo 限流，代码无法消除。后备数据源在限流期间能否实际取到数据未经验证。

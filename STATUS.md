# 任务看板

状态取值：`待领取` → `进行中` → `待验收` → `已完成`，另有 `受阻`、`已取消`。

| 编号 | 任务 | 状态 | 依赖 | PR | 回报 |
| --- | --- | --- | --- | --- | --- |
| 0001 | [yfinance 限流误报修复](tasks/0001-yfinance-preflight.md) | 验收通过，待用户合并 | 无 | [Vibe-Trading-William#1](https://github.com/jimojijie29/Vibe-Trading-William/pull/1) | [回报](reports/0001-yfinance-preflight.md) |
| 0002 | [实测美股、港股的后备数据源](tasks/0002-fallback-chain-verification.md) | 待领取（返工 R1，见任务单末尾） | 无 | [Vibe-Trading-William#2](https://github.com/jimojijie29/Vibe-Trading-William/pull/2) | [回报](reports/0002-fallback-chain-verification.md) |
| 0003 | [让 CI 在 william 分支上运行](tasks/0003-ci-on-william.md) | 待领取 | 无 | | |

## 验收记录

### 0001（2026-11-13，通过）

- PR 提交 `9b32b81`，改动 4 个文件，均在任务范围内，与 `william` 无冲突。
- Claude 在独立检出中复跑 `tests/test_preflight.py`：6 个全部通过。
- 五条验收标准均满足。Codex 对提示文案的改动（不再保证限流是临时的，分别列出美股、港股的后备链）比参考补丁更准确，予以采纳。
- PR 上没有自动检查：仓库有 `test.yml`，但只对 `main` 触发，而 PR 的目标分支是 `william`（见任务 0003）。验证依据是 Codex 的本地运行和 Claude 的复跑。
- 遗留：用户环境的 yfinance 为 1.4.1 仍被限流，说明是出口 IP 被 Yahoo 限流，代码无法消除。后备数据源在限流期间能否实际取到数据未经验证。

### 0002（2026-11-13，返工 R1）

- PR 提交 `64f5332`，改动 `runner.py` 和一个新测试文件，在任务范围内。
- Claude 在独立检出中禁网复跑：`test_fallback_chain_task.py`、`test_registry.py`、`test_backtest_runner_security.py` 全部通过。`test_akshare_loader.py` 有 5 个外汇测试失败，原因是 Claude 的容器没装 akshare，在基线 `william` 上同样失败，与本 PR 无关。
- 逐只补取的逻辑正确，修掉了"只取回一部分时缺失股票被静默丢弃"的缺陷。
- 返工原因：补取后一次回测可混用两个数据源，但运行卡片仍只记录推断的来源，与实际不符。详见任务单末尾。
- 实测结论（Codex 如实报告）：用户当前网络下，美股经 akshare 后备成功过一次，之后不稳定；港股一次都没跑通。Yahoo 被限流，东方财富接口经代理频繁断连。**后备链在代码上成立，但在当前网络下不可靠。**
- 待用户决定的设计问题：两个数据源的复权口径不一致（yfinance 不复权，akshare 前复权）；缺失标的时是告警继续还是直接失败。


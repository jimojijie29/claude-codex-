# 任务看板

建议领取顺序：0004 → 0006 → 0005 → 0007 → 0008。依赖未满足的任务跳过。

状态取值：`待领取` → `进行中` → `待验收` → `已完成`，另有 `受阻`、`已取消`。

| 编号 | 任务 | 状态 | 依赖 | PR | 回报 |
| --- | --- | --- | --- | --- | --- |
| 0001 | [yfinance 限流误报修复](tasks/0001-yfinance-preflight.md) | 验收通过，待用户合并 | 无 | [Vibe-Trading-William#1](https://github.com/jimojijie29/Vibe-Trading-William/pull/1) | [回报](reports/0001-yfinance-preflight.md) |
| 0002 | [实测美股、港股的后备数据源](tasks/0002-fallback-chain-verification.md) | 验收通过，待用户合并 | 无 | [Vibe-Trading-William#2](https://github.com/jimojijie29/Vibe-Trading-William/pull/2) | [回报](reports/0002-fallback-chain-verification.md) |
| 0003 | [让 CI 在 william 分支上运行](tasks/0003-ci-on-william.md) | 验收通过，待用户合并 | 无 | [Vibe-Trading-William#3](https://github.com/jimojijie29/Vibe-Trading-William/pull/3) | [回报](reports/0003-ci-on-william.md) |
| 0004 | [诊断并修复东方财富接口的连接失败](tasks/0004-eastmoney-network-diagnosis.md) | 待领取 | 无 | | |
| 0005 | [美股、港股价格统一为前复权](tasks/0005-unify-forward-adjusted-prices.md) | 待领取 | PR #2 合并后才可开始 | | |
| 0006 | [修复依赖版本漂移导致的后端测试失败](tasks/0006-dependency-drift-failures.md) | 待领取 | PR #3 合并后才可开始 | | |
| 0007 | [处理图表 HTML 测试的失败](tasks/0007-chart-html-tests.md) | 待领取 | PR #3 合并后才可开始 | | |
| 0008 | [处理前端测试与界面文字不一致的失败](tasks/0008-frontend-test-text.md) | 待领取 | PR #3 合并后才可开始 | | |

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
- 用户决定（2026-11-13）：美股、港股统一为前复权，见任务 0005。用户网络为代理的绕过大陆模式，东方财富断连的原因待诊断，见任务 0004。缺失标的的处理用户未表态，维持告警后继续。
- 更正：yfinance 的 `Close` 已按拆股调整，只是不含分红，并非完全不复权。

### 0002 返工 R1（2026-11-13，通过）

- 追加提交 `f8a55fa`，只改 `runner.py` 和测试文件，符合返工范围。
- 两条取数路径都按实际返回的数据记录每只股票的来源，`_run_card_effective_sources` 写入实际来源的排序列表，类型与 `engines/base.py` 的读取方式一致。同一市场混用来源时有 warning。没有使用全局变量。
- Claude 禁网复跑 `test_fallback_chain_task.py`、`test_registry.py`、`test_backtest_runner_security.py`、`test_preflight.py`、`test_market_detection.py`、`test_engine_robustness.py`、`test_hypothesis_registry.py`、`test_run_card.py`：77 通过，1 跳过，2 失败。2 个失败都是 `test_run_card.py` 里的 `ModuleNotFoundError`，原因是 Claude 的容器没装全项目依赖，与本 PR 无关。
- 遗留：真实联网验证未完成（美股不稳定，港股未跑通），由任务 0004 处理。

### 0003（2026-11-13，通过）

- PR 提交 `ac00f8a`，只改 `.github/workflows/test.yml`：增加 `william` 触发，后端与前端拆成并行 job，超时调到 20 分钟。没有改业务代码，没有新增 `skip` 或 `xfail`。
- Claude 读取了 [CI 运行](https://github.com/jimojijie29/Vibe-Trading-William/actions/runs/43347475219) 的日志，与回报一致：后端 3496 通过、18 失败、6 跳过；本地 MCP 集成测试 14 个通过；前端 198 通过、13 失败。完整运行约 5 分钟。
- 被排除的 9 个联网测试原本就带 `integration` 标记，不是本次新加的。
- CI 是红的，这是 `william` 的真实基线，符合任务单的要求。31 个失败拆成任务 0006、0007、0008 处理。
- 更正：0002 验收时，Claude 把 `test_akshare_loader.py` 的 5 个外汇测试失败归因于自己的容器没装 akshare。CI 里装了 akshare，这 5 个同样失败，说明新版 akshare 下外汇识别确实失效，是真实缺陷，见任务 0006。


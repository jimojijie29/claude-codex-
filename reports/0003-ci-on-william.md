# 0003 让 CI 在 william 分支上运行

- **结果**：部分完成。CI 已触发并完整运行；后端和前端均有既有失败，检查保持红色。
- **PR**：https://github.com/jimojijie29/Vibe-Trading-William/pull/3
- **最终运行**：https://github.com/jimojijie29/Vibe-Trading-William/actions/runs/43347475219 ，提交 `ac00f8a`。
- **耗时**：完整运行 5 分 08 秒（01:47:22 至 01:52:30 UTC）；后端 job 5 分 06 秒，前端 job 57 秒。

## 改动与验证

`.github/workflows/test.yml`：保留 `main`，增加 `william` push 和目标为 `william` 的 PR 触发；后端、前端拆为并行 job，超时从 10 分钟调整为 20 分钟。后端排除已标记的外网 integration 测试；14 个使用本地进程或 localhost 的 MCP integration 测试在独立步骤继续执行，即使主测试失败也会运行。未修改业务代码、测试断言或新增 skip/xfail。

第一次 [运行](https://github.com/jimojijie29/Vibe-Trading-William/actions/runs/43346356877) 后端 `3519 passed, 18 failed, 6 skipped`，job 耗时 8 分 21 秒；串行结构导致后端失败后前端未运行。拆分后的最终运行：

```text
后端语法检查：通过
后端主测试：18 failed, 3496 passed, 6 skipped, 23 deselected, 14 warnings in 168.12s
本地 MCP 集成测试：14 passed, 1 warning in 23.70s
前端 npm ci / npm run build：通过
前端 Vitest：Test Files 4 failed | 17 passed (21)
              Tests 13 failed | 198 passed (211)
```

主测试的 23 个 deselected 中，14 个本地 MCP 测试在下一步骤运行并全部通过，剩余 9 个是下列外网测试。前端 job 独立运行；前端代码未改动。

## 后端失败清单

以下均为第三类：`william` 现有代码、测试或依赖接口不一致；不是本次工作流变更引入的失败，也未排除。原因按日志归类，尚需分别确认应改实现还是测试。

- `factors/test_alpha_compare.py::test_run_bench_only_restricts_to_subset`、`test_run_bench_only_none_benches_everything`：测试 stub 不接受 `on_fetch_progress`。
- `factors/test_gtja191_part1_samples.py::test_gtja191_sample_matches_golden[gtja191_001]`、`[gtja191_005]`：数值与 golden 不一致。
- `test_akshare_loader.py::TestIsForex::test_eurusd_matches`、`test_lowercase_matches`、`test_fx_suffix_matches`；`TestRouting::test_forex_routes_to_forex_hist_em`、`test_forex_strips_fx_suffix`：外汇识别返回 False，因而路由断言失败；需检查已安装 AKShare 与使用的接口兼容性。
- `test_cn_treasury_redesign.py::test_cn_treasury_html_smoke`、`test_macro_echarts.py::test_html_inline_echarts`、`test_macro_echarts_more.py::test_ust_html_smoke`：生成 HTML 长度约 64–65 KB，低于测试要求的 500 KB。
- `test_etf_broad_history.py::test_render_html_default_window_and_range_buttons`、`test_render_html_index_delta_pct_in_tooltip`、`test_tooltip_amount_delta`：生成 HTML 未包含测试预期的 JS 片段。
- `test_macro_echarts.py::test_html_table_tab`：HTML 缺少测试期望的 `\\uFEFF`。
- `test_mcp_client_adapter.py::test_build_mcp_tool_wrappers_retries_transient_discovery_failure`、`test_oauth_token_cache.py::test_cache_expiry_surfaces_reauth_no_silent_stale_call`：构造 `MCPError` 时缺少必需参数 `message`，需检查 MCP 依赖与测试接口。

## 前端失败清单

以下 13 项均为第三类：组件实际渲染内容未匹配现有测试期待文字；尚需判断是中文界面变更导致断言过期，还是组件行为需要调整。

- `ThinkingTimeline.test.tsx`：`shows summary text when done`、`shows running state with spinner`、`shows error icon when a step failed`、`handles multiple tool steps`。
- `ToolProgressIndicator.test.tsx`：`renders multiple running tools with header`、`shows overflow indicator for > 3 running tools`。
- `WelcomeScreen.test.tsx`：`renders the title`、`renders capability chips`、`renders example categories`、`calls onExample with prompt when an example button is clicked`、`renders the helper text`。
- `ConnectionBanner.test.tsx`：`shows reconnecting message with attempt number`、`defaults to attempt 1 when retryAttempt is not provided`。

## 排除的外网测试

以下 9 项原本已标记 `integration`，并非本次新增标记；CI 的 `-m "not integration"` 排除它们：

- `test_commodities.py::test_fetch_eia_spr_live`：美国 EIA 战略石油储备 XLS。
- `test_commodities.py::test_fetch_eia_rigs_live`：美国 EIA 钻机 XLS。
- `test_commodities.py::test_fetch_wti_live`：AKShare WTI 期货历史接口。
- `test_commodities.py::test_fetch_oih_live`、`test_fetch_ieo_live`：Yahoo Finance OIH、IEO 行情。
- `test_gold_holdings_vs_xauusd.py::test_akshare_macro_cons_gold_live`：AKShare 黄金 ETF 持仓接口。
- `test_silver_holdings_vs_xagusd.py::test_akshare_macro_cons_silver_live`：AKShare 白银 ETF 持仓接口。
- `test_vix_move.py::test_fetch_vix_live`：CBOE VIX，失败时使用 FRED。
- `test_vix_move.py::test_fetch_move_live`：Yahoo Finance MOVE 行情。

原工作流已有排除 `agent/tests/e2e_backtest` 和 `agent/tests/test_e2e_harness_v2.py`，本次保持不变。

## 建议

分别立项处理：因子测试与 golden；AKShare 外汇识别；图表 HTML 及断言；MCP SDK 兼容；前端组件与测试文字。保持当前 PR 为红色直到这些问题得到核实解决。GitHub 还提示 checkout/setup actions 的 Node 20 运行时将被强制切换到 Node 24，本次不是失败原因。

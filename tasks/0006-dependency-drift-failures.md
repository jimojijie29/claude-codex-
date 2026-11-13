# 0006 修复依赖漂移与接口不一致导致的后端测试失败

- **代码仓库**：`jimojijie29/Vibe-Trading-William`，基于 `william`
- **分支**：`codex/0006-dependency-drift`
- **依赖**：PR #3（任务 0003）合并后开始，这样本任务的 PR 上能看到 CI 结果
- **来源**：任务 0003 的 CI 基线，[运行记录](https://github.com/jimojijie29/Vibe-Trading-William/actions/runs/43347475219)

## 背景

CI 用 `pip install -e ".[dev]"` 装的是各依赖当前最新版，与项目开发时的版本不同。下面 11 个后端失败里，A、B、C 三组指向依赖接口或数值行为的变化，D 组是项目内部接口与测试替身不一致。四组互相独立，放在一张任务单里。

## A. akshare 外汇识别失效（5 个失败，疑似真实缺陷）

`test_akshare_loader.py` 的 `TestIsForex` 三个用例和 `TestRouting` 两个外汇用例失败：`_is_forex('EURUSD')` 返回 `False`。

`agent/backtest/loaders/akshare_loader.py` 的 `_is_forex` 这样写：

```python
try:
    from akshare.forex.cons import symbol_market_map
except Exception:
    return False
return upper in symbol_market_map
```

导入失败时静默返回 `False`，外汇代码于是落到 A 股接口。这不只是测试问题：在新版 akshare 下，外汇回测会取错数据或取不到数据。

要求：

1. 查清 CI 里装的 akshare 版本，以及 `akshare.forex.cons.symbol_market_map` 在哪个版本被移走或改名。
2. 修复 `_is_forex`，使它在项目声明支持的 akshare 版本范围内都能正确识别外汇代码。不要只依赖 akshare 的内部常量；内部常量拿不到时要有不依赖它的判断方式（例如按代码格式判断，可参考 `agent/backtest/engines/base.py` 里的 `_FOREX_RE`）。
3. 同时确认 `forex_hist_em` 这个取数接口在新版里还在、参数没变。接口也变了的话一并适配，并在回报里说明。
4. 导入失败不应再被静默吞掉：至少记一条 debug 日志。

## B. MCP SDK 接口变化（2 个失败）

`test_mcp_client_adapter.py::test_build_mcp_tool_wrappers_retries_transient_discovery_failure` 和 `test_oauth_token_cache.py::test_cache_expiry_surfaces_reauth_no_silent_stale_call` 报 `TypeError: MCPError.__init__() missing 1 required positional argument: 'message'`。CI 日志里还有 `Tool.inputSchema is deprecated; MCP SDK v2 renamed this field to input_schema` 的警告，说明装到的是 MCP SDK v2。

要求：

1. 查清 `MCPError` 是在测试代码里构造的，还是在 `agent/src/` 的业务代码里构造的。
2. 只是测试构造方式过期时，改测试的构造方式，断言不变。
3. 业务代码也按旧签名构造 `MCPError`，或依赖其他已变化的 v2 接口时，这是真实缺陷，要修业务代码。全仓搜索 `MCPError` 和 `inputSchema` 的使用点，列出清单。
4. 做出选择并说明理由：适配 v2，还是在 `pyproject.toml` 里给 MCP SDK 加版本上限。优先适配；适配范围很大时改为加上限，并在回报里建议后续任务。

## C. gtja191 因子的数值比较（2 个失败）

`factors/test_gtja191_part1_samples.py` 的 `gtja191_001`、`gtja191_005` 与 golden 不一致。日志显示差异都在 `1e-15` 量级，发生在期望值为 `0` 或接近 `0` 的位置，测试用的是 `rtol=1e-06, atol=0`。`atol=0` 时，任何接近零的浮点噪声都会判为不相等。

要求：

1. 确认 150 个元素里不一致的只有这类接近零的位置，其余位置在 `rtol=1e-6` 内一致。贴出证据。
2. 确认属实后，给比较加一个很小的绝对容差（建议 `atol=1e-12`），并在测试里用注释说明原因。这是修正比较方式，不是放宽对因子正确性的要求。
3. 有任何一个位置的差异超出浮点噪声的量级，就不要改容差，把它当成真实的数值回归写进回报。
4. 不要重新生成 golden 文件。

## D. alpha_compare 的测试替身缺少参数（2 个失败）

`factors/test_alpha_compare.py::test_run_bench_only_restricts_to_subset` 和 `test_run_bench_only_none_benches_everything` 失败，原因是测试里的替身函数不接受 `on_fetch_progress` 参数，而被测代码现在会传这个参数。

要求：

1. 找到是哪次改动给被测代码加了 `on_fetch_progress`，确认这是有意的功能（取数进度回调）。
2. 属实的话，更新测试替身让它接受这个参数，断言不变。顺便看一下有没有测试覆盖"进度回调确实被调用"；没有就补一个。
3. 同一个测试文件里其他替身有同样问题但碰巧没被触发的，一并改掉。

## 范围

`agent/backtest/loaders/akshare_loader.py`、`agent/src/` 下与 MCP 相关的代码、上述测试文件、必要时的依赖版本约束。D 组只改测试文件。

范围外：其余 7 个后端失败（图表 HTML）和 13 个前端失败属于任务 0007、0008，不要碰。

## 验收标准

1. 上述 11 个测试在 CI 上通过，回报附 CI 运行链接。
2. CI 的后端失败数从 18 降到 7，剩下的 7 个正是任务 0007 的那些。有其他变化要解释。
3. 没有新增 `skip`、`xfail`，没有删除测试。
4. A、B、D 三组各给出"是真实缺陷还是测试过期"的结论和依据。

## 交付

向 `william` 提 PR，标题 `[0006] fix: adapt to akshare and MCP SDK changes`，并写回报 `reports/0006-dependency-drift-failures.md`。

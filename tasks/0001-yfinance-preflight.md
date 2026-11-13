# 0001 yfinance 限流误报修复

- **代码仓库**：`jimojijie29/Vibe-Trading-William`，基于 `william`
- **分支**：`codex/0001-yfinance-preflight`
- **参考补丁**：[`patches/0001-yfinance-preflight.patch`](../patches/0001-yfinance-preflight.patch)

## 背景

项目启动时的自检表出现：

```
FAIL │ yfinance │ YFRateLimitError: Too Many Requests. Rate limited. Try after a while. (US/HK equity backtest unavailable)
```

这是 Yahoo 返回 HTTP 429，不是项目代码出错。两个可能的来源：

1. yfinance 低于 0.2.58 时会被 Yahoo 的 TLS 指纹识别拦截，表现为永久限流。项目目前只要求 `>=0.2.30`。
2. 出口 IP 被临时限流，过一段时间自动恢复。

自检把它显示成红色 `FAIL` 和 "backtest unavailable" 是误报：`agent/backtest/runner.py` 在主数据源返回空时会按 `FALLBACK_CHAINS` 换用后备数据源。

## 目标

限流不再显示为失败；用户能从提示里知道该做什么；旧版本 yfinance 不再能被装上。

## 范围

- `agent/src/preflight.py`：识别限流错误，返回新状态 `degraded`，显示为黄色 `WARN`；检测到 yfinance 低于 0.2.58 时提示升级。其他错误仍为 `error`。
- `pyproject.toml`、`agent/requirements.txt`：yfinance 下限改为 `>=0.2.58`。
- `agent/tests/test_preflight.py`：覆盖限流、旧版本提示、其他错误、`WARN` 渲染四种情况。

参考补丁已实现以上全部内容，并在模拟异常下通过了 6 个测试。它没有对真实的 Yahoo 接口验证过，这是你要补上的部分。

范围外：不改 `yfinance_loader.py` 的下载与重试逻辑，不改后备链的顺序。

## 验收标准

1. `cd agent && python -m pytest tests/test_preflight.py -q` 全部通过。
2. 安装 `yfinance>=0.2.58` 后实际运行一次 `run_preflight()`，把自检表输出贴进回报。yfinance 一行应为 `OK` 或 `WARN`，不应为 `FAIL`。
3. 确认 0.2.58 这个下限是否准确：查 yfinance 的发布记录，找到引入 `curl_cffi` 浏览器指纹伪装的版本。若不是 0.2.58，改成正确的版本号并在回报里给出依据。
4. 确认项目里读取 `CheckResult.status` 的地方都能正确处理新增的 `degraded`。已知读取点是 `agent/cli/_legacy.py` 的 `cmd_run` 和 `cmd_interactive`，它们只判断 `critical`，应当不受影响；请全仓搜索确认没有遗漏，包括前端。
5. 提示文案里写了"回退到 akshare"。确认美股和港股在 yfinance 返回空时确实会走到后备源；港股的链是 `yfinance → futu → akshare`，文案如与实际不符就改文案。

## 交付

向 `william` 提 PR，标题 `[0001] fix(preflight): report Yahoo rate limit as warning`，并写回报 `reports/0001-yfinance-preflight.md`。

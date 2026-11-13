# 0002 实测美股、港股的后备数据源

- **代码仓库**：`jimojijie29/Vibe-Trading-William`，基于 `william`
- **分支**：`codex/0002-fallback-chain`
- **依赖**：无（不依赖 0001 是否已合并）

## 背景

任务 0001 把 Yahoo 限流改成了 `WARN`，提示里写着后备链：美股 `yfinance → akshare`，港股 `yfinance → futu → akshare`。这个说法目前只有代码核查支持，没有实际跑过。用户的出口 IP 正被 Yahoo 限流，所以后备链现在就是美股、港股回测的实际路径，必须确认它真的能用。

相关代码：

- `agent/backtest/loaders/registry.py`：`FALLBACK_CHAINS`、`resolve_loader`
- `agent/backtest/runner.py`：指定数据源路径（约 455–480 行）和 `_fetch_auto`（约 590–640 行）里各有一段 "Runtime fallback"，只在主数据源返回**空字典**时触发
- `agent/backtest/loaders/yfinance_loader.py`：`is_available()` 恒为 `True`，下载异常被吞掉后返回空结果
- `agent/backtest/loaders/akshare_loader.py`：`_fetch_us`、`_fetch_hk`

## 目标

弄清在 yfinance 取不到数据时，美股和港股回测能否靠后备源跑通；跑不通就找出原因并修好，修不了就说清楚卡在哪里。

## 要回答的问题

1. **实测**：在 Yahoo 限流的状态下，分别对一只美股（如 `AAPL.US`）和一只港股（如 `700.HK`）跑一次日线回测，数据源分别用 `yfinance` 和 `auto`。每次是否成功？日志里是否出现 `Runtime fallback: ... -> akshare`？取回的行数和日期范围是否合理？
2. **部分失败**：一次请求多只股票，yfinance 只取回其中一部分时，结果字典非空，后备不会触发，缺失的股票会被静默丢掉。确认这个行为是否存在，以及回测会因此出错还是悄悄用更少的股票跑完。
3. **代码转换**：`_normalize_codes(codes, "akshare")` 对 `AAPL.US`、`700.HK`、`0700.HK` 给出的代码，akshare 的 `stock_us_hist`、`stock_hk_hist` 是否认得。
4. **数据口径**：akshare 返回的价格与 yfinance 是否同口径（复权方式、币种、成交量单位）。只需指出差异，不需要统一。
5. **futu**：港股链里的 futu 在没有安装或没有配置时是否被正确跳过，不会抛错中断。

## 范围

- 先调查并回答以上问题，这是本任务的主要产出。
- 发现的缺陷如果局限在 `agent/backtest/runner.py` 的两段后备逻辑、`agent/backtest/loaders/akshare_loader.py` 或 `_normalize_codes` 内，直接修复并补测试。
- 补一组不联网的单元测试（标记 `pytest.mark.unit`）：用假的加载器模拟 yfinance 返回空，断言美股、港股都会走到下一个可用后备源；如果修了问题 2，也要覆盖部分失败的情况。

范围外：不改 `FALLBACK_CHAINS` 的顺序，不改 yfinance 加载器的下载逻辑，不引入新的数据源或新依赖。需要这些改动才能解决时，写进回报由 Claude 决定。

## 验收标准

1. 回报逐条回答上面 5 个问题，每条附实际命令和输出；没能实测的要写明原因。
2. 新增的单元测试通过，且不访问网络。
3. `cd agent && python -m pytest tests/test_akshare_loader.py tests/test_preflight.py -q` 以及你新增的测试文件全部通过。
4. 有代码修复时，修复前后的行为差异在回报里用实际输出说明。
5. 如果实测结论与 0001 的提示文案不符（例如港股实际走不到 akshare），在回报里指出，不要在本任务里改 `preflight.py`。

## 交付

有代码改动就向 `william` 提 PR，标题以 `[0002]` 开头；只有调查结论、没有代码改动时不必提 PR。两种情况都要写回报 `reports/0002-fallback-chain-verification.md`。

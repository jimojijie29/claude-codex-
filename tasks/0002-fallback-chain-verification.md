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

---

## 返工 R1（2026-11-13，Claude 验收意见）

PR #2 的补取逻辑和测试通过验收，回报如实。合并前需要补一处，在同一个 PR 分支 `codex/0002-fallback-chain` 上追加提交即可。

### 要改的问题：运行卡片记录的数据源与实际不符

`runner.py` 的 `main` 在取数后写入：

```python
if source == "auto":
    config["_run_card_effective_sources"] = sorted(_group_codes_by_source(codes))
else:
    config["_run_card_effective_sources"] = [source]
```

这两个分支都是按代码**推断**来源，不是按**实际**取到数据的来源。本 PR 之前，指定源模式下要么全部来自主源、要么全部来自一个后备源（此时 `source` 被改写），记录是准的。本 PR 引入逐只补取后，一次回测可以同时用 yfinance 和 akshare 的数据，而卡片只写 `["yfinance"]`。两个源的复权口径不同（你在回报第 4 点已指出），卡片不如实记录的话，事后无法判断一次回测的价格口径。`auto` 模式在发生后备时原本就有同样的偏差，一并修掉。

### 要求

1. 两条取数路径都要记下每只股票实际来自哪个数据源。`_fetch_auto` 目前只返回数据字典，可以改返回值或另加参数带回这份映射，方式由你定，但不要用模块级全局变量。
2. `config["_run_card_effective_sources"]` 写入实际用到的数据源集合（排序后的列表）。先看 `agent/backtest/engines/base.py` 第 39 行附近对这个字段的用法，保持它期望的类型不变。
3. 同一市场内的股票来自不止一个数据源时，输出一条 warning，写明哪些代码来自哪个源，并说明不同来源的复权口径可能不同。
4. 补单元测试：全部来自主源、全部来自后备源、混合来源三种情况下，`_run_card_effective_sources` 的值正确；混合时有 warning。指定源和 `auto` 两种模式都要覆盖。

### 不要做的

- 不统一复权口径，不改 `yfinance_loader.py` 或 `akshare_loader.py` 的取数参数。这是独立的设计决定，会另开任务。
- 不改"缺失标的只告警、回测继续"的现有策略。
- 不重跑真实联网验证。网络问题会另行处理。

### 验收

- `cd agent && python -m pytest tests/test_fallback_chain_task.py tests/test_registry.py tests/test_akshare_loader.py tests/test_preflight.py -q --disable-socket` 全部通过，输出贴进回报。
- 在 `reports/0002-fallback-chain-verification.md` 末尾追加 "返工 R1" 一节，不要改动原有内容。

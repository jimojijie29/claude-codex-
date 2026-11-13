# 0005 美股、港股价格统一为前复权

- **代码仓库**：`jimojijie29/Vibe-Trading-William`，基于 `william`
- **分支**：`codex/0005-forward-adjusted`
- **依赖**：0002 返工 R1 的 PR 合并之后再开始（两者的告警文案有关联，避免冲突）

## 背景

任务 0002 发现两个数据源的价格口径不一致：

- `agent/backtest/loaders/yfinance_loader.py` 调 `yf.download(..., auto_adjust=False)` 并取 `Close`，价格已按拆股调整，但**不含分红**。
- `agent/backtest/loaders/akshare_loader.py` 的美股、港股接口传 `adjust="qfq"`，是前复权，**含分红**。

同一个策略换一个数据源，收益率就不一样；0002 的逐只补取还会让一次回测里混用两种口径。

## 设计决定（用户已确认）

美股、港股回测统一使用**前复权**价格，理由是回测收益应当包含分红。yfinance 一侧改成前复权，akshare 一侧保持不变。

## 范围

1. **yfinance 加载器改为前复权**。`open`、`high`、`low`、`close` 四列都要调整，不能只调 `close`，否则 K 线内部关系会错。`yf.download` 的 `auto_adjust=True` 会按复权因子同时调整四列，优先用它；用之前先确认当前安装版本下它返回的列名和多标的时的列结构，`_extract_symbol_frame`、`_normalize_frame` 要能正确处理。成交量保持 yfinance 返回的原值。
2. **缓存失效**。加载器缓存的键由 `agent/backtest/loaders/base.py` 的 `_loader_cache_payload` 生成，不含复权方式，因此旧的不复权数据会被当成新数据返回。必须让旧缓存失效。把 `_LOADER_CACHE_VERSION` 加一最简单，但会让所有数据源的缓存一起失效；更精确的做法是把复权方式放进键里。两种都可以，在回报里说明选了哪种和理由。
3. **加密货币不受影响**。yfinance 加载器也服务 `crypto` 市场。确认改动后加密货币的价格与改动前一致，并有测试覆盖。
4. **调查其他使用点**。全仓搜索直接使用 yfinance 或依赖其价格口径的代码，例如 `agent/backtest/benchmark.py`、`agent/backtest/metrics.py`、`agent/backtest/correlation.py`、`agent/src/shadow_account/backtester.py`、`agent/src/tools/` 下的工具。列出每一处用的是什么口径。基准收益与策略收益口径不一致会直接让超额收益算错：这类地方如果只需改一个取数参数，一并改掉；需要更大改动的，列进回报由 Claude 决定。
5. **更新 0002 引入的混用来源告警**。口径统一后，告警文案不应再说"复权口径可能不同"；仍然保留"数据来自多个来源"的提示。
6. **文档**。在加载器的模块文档字符串里写明口径；项目里有说明数据口径的文档时同步更新。

范围外：不改 A 股（tushare、mootdx、akshare 的 A 股路径）的复权方式，但请在回报里列出它们当前各用什么口径，供后续决定。不改后备链顺序。

## 验收标准

1. 单元测试（不联网，`pytest.mark.unit`）覆盖：
   - 加载器请求的是前复权数据
   - 四个价格列都被调整，且每一行满足 `low <= open, close <= high`
   - 单标的、多标的两种返回结构都能正确解析
   - 加密货币路径行为不变
   - 旧口径的缓存不会被命中
2. 实测对比（Yahoo 限流时写明没能做，不要编造数据）：挑一只在测试区间内有分红的美股，贴出改动前后同一区间的首末收盘价和区间收益率，并与 akshare 前复权的同区间数据对比。两者不必完全相等，但区间收益率应当接近；差异超过 1 个百分点时要找出原因。
3. `cd agent && python -m pytest tests/ -q --disable-socket -k "yfinance or loader or cache or benchmark"` 全部通过，输出贴进回报。原本就失败的测试单独列出，并证明它们在基线 `william` 上同样失败。
4. 回报包含第 4 点的使用点清单和 A 股各加载器的口径清单。

## 需要注意

这个改动会让已有的美股、港股回测结果发生变化：含分红后，高股息股票的历史收益会变高。这是预期的。请在 PR 描述里明确写出这一点，方便用户知道旧结果与新结果不可直接比较。

## 交付

向 `william` 提 PR，标题 `[0005] feat(data): use forward-adjusted prices for US/HK equities`，并写回报 `reports/0005-unify-forward-adjusted-prices.md`。

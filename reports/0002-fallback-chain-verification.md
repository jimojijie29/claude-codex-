# 0002 实测美股、港股的后备数据源

- **结果**：部分完成。离线缺陷修复完成；真实港股回测及美股 auto 回测在当前代理网络下未跑通，不能证明后备源稳定可用。
- **PR**：https://github.com/jimojijie29/Vibe-Trading-William/pull/2 （未合并）
- **提交**：`64f5332`，基于 `origin/william`。

## 改动摘要

- `agent/backtest/runner.py`：两条回退路径对美股、港股按缺失标的补取数据；记录仍然缺失的代码；后备源初始化异常时跳过；指定源模式将已取回的数据交给引擎，避免再次请求导致数据变化。
- `agent/tests/test_fallback_chain_task.py`：离线模拟空结果、部分结果、港股跳过不可用 futu，以及两种回测模式使用预取数据。

## 1. 四次真实回测

在项目原有虚拟环境、干净的 `william` 检出上，通过 `backtest.runner.main` 执行日线回测，日期 `2024-01-02` 至 `2024-03-01`，零仓位策略。临时驱动脚本位于本机 `C:/Users/Administrator/.codex/worktrees/fallback-chain/live_check_0002.py`；执行方式（在 `agent` 下）：

```powershell
$env:PYTHONPATH='C:\Users\Administrator\.codex\worktrees\fallback-chain\Vibe-Trading\agent'
& D:\AI_Tools\finance\Vibe-Trading\.venv\Scripts\python.exe -X utf8 C:\Users\Administrator\.codex\worktrees\fallback-chain\live_check_0002.py <yfinance|auto> <AAPL.US|700.HK>
```

初次结果，逐次调用：

```text
yfinance AAPL.US: YFRateLimitError; Runtime fallback: yfinance -> akshare; LIVE RESULT success
auto AAPL.US:     YFRateLimitError; {"error": "No data fetched"}; LIVE RESULT systemexit=1
yfinance 700.HK:  YFRateLimitError; akshare ProxyError / RemoteDisconnected; {"error": "No data fetched"}; LIVE RESULT systemexit=1
auto 700.HK:      YFRateLimitError; akshare ProxyError / RemoteDisconnected; {"error": "No data fetched"}; LIVE RESULT systemexit=1
```

首次美股成功时尚未加加载器行数记录，不能断言该次的行数或日期范围。后续加记录后再次运行 `yfinance AAPL.US` 和 `auto AAPL.US` 均失败，出现 `No data fetched`。单独调用 akshare 原始接口一次成功：`stock_us_hist(symbol='105.AAPL', period='daily', start_date='20240102', end_date='20240301', adjust='qfq')` 返回 **42 行**、首日 `2024-01-02`、末日 `2024-03-01`；再次调用出现 `ProxyError / RemoteDisconnected`。这是原始接口结果，不代表成功回测的行数。

## 2. 部分失败

修改前执行离线回归：

```text
python -X utf8 -m pytest tests/test_fallback_chain_task.py -q
FAILED ...test_auto_falls_back_for_missing_symbols[True-codes2]
assert {'AAPL.US'} == {'AAPL.US', 'MSFT.US'}
1 failed, 5 passed
```

即使 `MSFT.US` 缺失，非空结果也不会触发回退。`BaseEngine.run_backtest` 以信号与 `data_map` 的交集构造 `valid_codes`，因此正常生成信号的策略可用较少标的继续运行；假定所有标的都存在的策略也可能自行报错。修复后同一场景两个代码均进入引擎；没有任何后备源返回的代码会输出 warning，仍可能用部分标的继续运行。混用来源可能导致价格口径不同，见第 4 点。

## 3. 代码转换

实际执行 `C:/Users/Administrator/.codex/worktrees/fallback-chain/inspect_0002.py` 输出：

```text
normalized ['AAPL.US', '700.HK', '0700.HK']
us 105.AAPL: rows=42
us 106.AAPL: TypeError: 'NoneType' object is not subscriptable
us AAPL: TypeError: 'NoneType' object is not subscriptable
hk 00700: ProxyError ... RemoteDisconnected
```

`_normalize_codes` 对上述代码不修改；akshare loader 内部将美股转换为 `105.AAPL`、`106.AAPL`、`AAPL` 依次尝试，将两个港股写法都转换为 `00700`。实测 `105.AAPL` 被接口接受；两个其他美股格式当次未返回数据；港股由于连接失败无法确认接口接受与否。

## 4. 数据口径

代码检查：`yfinance_loader.py` 传 `auto_adjust=False` 并使用 `Close`；`akshare_loader.py` 传 `adjust="qfq"`，即前复权。两者不一致，混用时价格与收益率可能不同。代码均未做币种转换或成交量单位转换；Yahoo 被限流、港股接口代理失败，无法实测同一标的的币种、成交量单位是否一致，也不能仅凭代码保证两数据源的单位一致。akshare 单次原始美股结果首日成交量为 `82488674`，未能与 Yahoo 同日数据核对。

## 5. Futu

`inspect_0002.py` 在清除 `FUTU_HOST`、`FUTU_PORT` 后输出 `futu unconfigured available False`。测试中的不可用 Futu 被跳过并到达 akshare。`FutuLoader.is_available()` 在包缺失或连接失败时捕获异常返回 False；本机未配置场景已实际运行，未安装场景为代码核查及离线模拟。新代码还捕获实例构造异常（例如无效端口）。

## 验证

以下命令均在 `agent` 目录，用原项目虚拟环境并显式禁网：

```text
python -X utf8 -m pytest tests/test_akshare_loader.py tests/test_preflight.py tests/test_fallback_chain_task.py -q --disable-socket --basetemp=.../pytest-final
.................................                                        [100%]
33 passed in 1.85s

python -X utf8 -m pytest tests/test_registry.py tests/test_futu_loader.py -q --disable-socket --basetemp=.../pytest-extra
.........................................                                [100%]
41 passed in 0.84s
```

## 偏离、问题与建议

- 未改动 0001 的提示。提示中的后备链顺序与代码一致，但实际可用性受代理影响；应避免将其理解为回测一定能成功。
- 本任务未统一复权和成交量口径，建议单独决定数据标准；新修复可在同一组合内混用不同来源的价格，回测解释应考虑此风险。
- Yahoo 限流、东方财富代理断连均属于环境/上游连接问题，不能仅靠本任务范围内的后备逻辑保证真实回测成功。建议在网络恢复后重跑四个组合并记录行数。
- 新增缺失标的 warning；若要求缺标的时回测强制失败，应另行确定产品策略。

## 返工 R1

- **结果**：完成，已追加到 [PR #2](https://github.com/jimojijie29/Vibe-Trading-William/pull/2)，提交 `f8a55fa`，尚未合并。
- `agent/backtest/runner.py`：在指定源和 auto 路径按实际返回数据记录每个标的的来源；运行卡片字段 `_run_card_effective_sources` 使用实际来源去重排序后的列表。同一市场混用来源时输出来源、对应代码和复权口径可能不同的 warning。
- `agent/tests/test_fallback_chain_task.py`：两种模式分别覆盖全部主源、全部后备源和混合来源，并检查混合来源告警。
- 先运行新测试，修改前结果为 `3 failed, 13 passed`：auto 全回退仍记为 yfinance，两种混合模式均漏记 akshare。
- 修改后在 `agent` 下以项目虚拟环境禁网运行：

```text
D:\AI_Tools\finance\Vibe-Trading\.venv\Scripts\python.exe -X utf8 -m pytest tests/test_fallback_chain_task.py tests/test_registry.py tests/test_akshare_loader.py tests/test_preflight.py -q --disable-socket --basetemp=C:/Users/Administrator/.codex/worktrees/fallback-chain/pytest-r1-final
.......................................................                  [100%]
55 passed in 2.26s
```

- 按要求未重跑联网验证，未修改价格复权方式或缺失标的处理策略。混合来源的复权问题由任务 0005 处理。

# 0004 东方财富网络诊断

- **结果**：受阻。默认代理、强制直连、1 秒与 3 秒间隔以及升级 AKShare 均未达到 9/10 成功率。没有足够证据判断哪一项改动能稳定解决问题。
- **PR**：无；未改项目代码、依赖、系统代理或代理软件配置。

## 代理与接口

Python 进程中的 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY` 及小写形式均未设置。`urllib.request.getproxies()` 返回系统代理：`http`、`https`、`ftp` 均为 `http://127.0.0.1:10808`；未返回 `no`。地址仅为本机代理监听地址，不含凭据。

安装的 AKShare `1.18.64` 源码显示：美股使用 `63.push2his.eastmoney.com`，港股使用 `33.push2his.eastmoney.com`。美股 `secid=105.AAPL`，港股 `secid=116.00700`。系统代理说明默认请求先到本机代理，但这不能证明代理程序的出口分流路径。

## 请求对照

本机执行 `D:\AI_Tools\finance\Vibe-Trading\.venv\Scripts\python.exe -X utf8 C:\Users\Administrator\.codex\worktrees\ci-on-william\diagnose_0004.py`。参数与上述 AKShare kline 请求相同；每组 10 次，成功要求 HTTP 成功且 JSON 中 `data.klines` 非空。默认采用 `requests.Session` 的系统配置；直连设置 `session.trust_env=False`，每次超时 5 秒。组内连续请求的间隔在表中注明。

| 市场 | 方式 | 间隔 | 次数 | 成功 | 失败 |
| --- | --- | ---: | ---: | ---: | --- |
| 美股 | 默认 | 0 秒 | 10 | 0 | 10 `ProxyError` |
| 美股 | 直连 | 0 秒 | 10 | 0 | 10 `ConnectionError` |
| 美股 | 直连 | 1 秒 | 10 | 2 | 8 `ConnectionError` |
| 美股 | 直连 | 3 秒 | 10 | 1 | 9 `ConnectionError` |
| 港股 | 默认 | 0 秒 | 10 | 2 | 8 `ProxyError` |
| 港股 | 直连 | 0 秒 | 10 | 3 | 7 `ConnectionError` |
| 港股 | 直连 | 1 秒 | 10 | 0 | 10 `ConnectionError` |
| 港股 | 直连 | 3 秒 | 10 | 2 | 8 `ConnectionError` |

这些顺序执行的短样本不能排除此前请求对后续限流的影响。默认错误包装为 `ProxyError`，直连也有连接失败；因此不能据此断定绕过代理、间隔 1 秒或间隔 3 秒可稳定解决。诊断没有观察或修改代理软件内部规则，也没有证明断连的唯一原因。

## AKShare 版本

`pip index versions akshare` 显示已安装 `1.18.64`、最新 `1.20.38`。在独立目录用 `pip install --no-deps --target ... akshare==1.20.38` 安装新版，复用现有虚拟环境依赖，未升级用户环境。脚本 `C:/Users/Administrator/.codex/worktrees/ci-on-william/check_akshare_latest_0004.py` 设置 5 秒超时，默认系统代理下各调用对应 AKShare API 10 次：

| AKShare 版本 | 市场 | 次数 | 成功 | 其他结果 |
| --- | --- | ---: | ---: | --- |
| 1.20.38 | AAPL | 10 | 5 | 5 `Push2RateLimitError` |
| 1.20.38 | 00700.HK | 10 | 0 | 10 `Push2RateLimitError` |

成功的 AAPL 返回 42 行，2024-01-02 至 2024-03-01。新版 [发布记录](https://github.com/akfamily/akshare/releases/tag/v1.20.38)没有列出该接口的断连修复；[官方 FAQ](https://github.com/akfamily/akshare/blob/main/docs/faq.md)描述 push2 断连可能是 IP 限流，建议冷却期间停止重复请求，并指出反复请求可能延长限流。新版将断连识别为此类异常，不代表本次已证明全部失败都由东方财富限流引起，也未证明升级能使接口稳定。代码历史还显示 1.20.0 有内部模块迁移，升级需独立评估兼容性。

## 项目加载器实测

未做修复；以现有 `1.18.64` 运行 `C:/Users/Administrator/.codex/worktrees/ci-on-william/check_loader_0004.py`，禁用项目缓存并对请求设置 5 秒超时，每个标的调用项目 `DataLoader.fetch` 10 次：

| 标的 | 次数 | 成功 | 空结果 | 成功结果行数和日期 |
| --- | ---: | ---: | ---: | --- |
| AAPL.US | 10 | 2 | 8 | 每次 42 行，2024-01-02 至 2024-03-01 |
| 700.HK | 10 | 0 | 10 | 无 |

离线回归：`cd agent && python -X utf8 -m pytest tests/test_akshare_loader.py -q --disable-socket --basetemp=.../pytest-0004` 输出 `21 passed in 0.86s`。未新增逻辑，因此没有新增开关测试。

## Tushare 补充测试

根据用户要求，使用本机已配置的 Tushare 令牌只读查询，不输出令牌。脚本 `C:/Users/Administrator/.codex/worktrees/ci-on-william/check_tushare_0004.py`：

```text
us_daily AAPL: rows=42 date=20240102..20240301
hk_daily 00700.HK: rows=42 date=20240102..20240301
```

这是各一次调用，不代表稳定性或全部标的都有权限。Tushare 官方文档说明 [us_daily](https://tushare.pro/document/2?doc_id=254) 和 [hk_daily](https://tushare.pro/document/2?doc_id=192) 返回未复权行情；现有项目 Tushare 加载器调用的是 A 股 `daily`，其注册的市场不包括美股和港股。将 Tushare 接入美港股后备链、确认复权及权限属于本任务范围外，建议单独决定。

## 建议

先停止反复访问东方财富并按上游建议冷却，再在同一代理模式下间隔较长时间重测；同时检查代理软件日志确认实际出口。若仍不稳定，建议单独评估已实测可用的 Tushare 美港股接口作为后备方案，明确前复权、权限和配额。当前数据不支持将默认代理绕过、短间隔重试或依赖升级作为可靠修复。

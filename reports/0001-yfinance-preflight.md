# 0001 yfinance 限流误报修复

- **结果**：完成
- **PR**：https://github.com/jimojijie29/Vibe-Trading-William/pull/1
- **提交**：`9b32b81`

## 改动摘要

- `agent/src/preflight.py`：识别 Yahoo 限流并返回 `degraded`，以黄色 `WARN` 显示；旧版本提示升级，其他异常继续显示 `FAIL`。
- `pyproject.toml`、`agent/requirements.txt`：将 yfinance 下限提高到 `0.2.58`。
- `agent/tests/test_preflight.py`：添加限流、旧版本、其他异常及 WARN 显示测试。

## 验证

项目原有虚拟环境已安装 yfinance `1.4.1`，满足要求。Windows 下使用 `-X utf8`：

```text
cd agent
D:\AI_Tools\finance\Vibe-Trading\.venv\Scripts\python.exe -X utf8 -m pytest tests/test_preflight.py -q
......                                                                   [100%]
6 passed in 0.11s
```

实现前运行同一测试：`3 failed, 3 passed`，失败原因分别为限流仍返回 `error` 和显示表不认识 `degraded`。

真实网络自检使用同一 Python，执行 `from src.preflight import run_preflight; run_preflight()`。LLM 使用合成测试配置，Tushare token 置空；Yahoo 和 OKX 检查均实际联网。输出：

```text
Preflight Check
 OK   | LLM (openai-codex) | preflight-test via API key
      |                    | (https://example.invalid)
 OK   | OKX API            | reachable
 WARN | yfinance           | rate limited by Yahoo; retry later (fallbacks if
      |                    | available: US: akshare; HK: futu -> akshare)
 N/A  | Tushare            | TUSHARE_TOKEN not set (optional) (A-share data
      |                    | unavailable)
 OK   | akshare            | installed
 OK   | ccxt               | installed

4/6 services ready
```

表格的竖线以上用 ASCII 表示；LLM 行仅验证显示，未连接 LLM。

官方 [0.2.58 发布记录](https://github.com/ranaroussi/yfinance/releases/tag/0.2.58)明确说明切换到 `curl_cffi` 以修复 TLS 指纹导致的误限流；[0.2.59 发布记录](https://github.com/ranaroussi/yfinance/releases/tag/0.2.59)另有后续修复。按任务要求保留引入版本 `0.2.58`。

全仓搜索 `CheckResult`、`run_preflight`、`src.preflight`：CLI 两处仅在 `critical` 且非 `ready` 时阻止启动；API 和异步 CLI 入口只调用显示，前端没有消费该结果。`registry.py` 的美股链为 `yfinance → akshare`，港股链为 `yfinance → futu → akshare`；`runner.py` 在数据为空时遍历可用后备源。这是代码核查，未验证后备源在本次网络环境中实际返回数据。

## 偏离

参考补丁称限流是临时性的并会按需重试，且称美港股均回退至 akshare。实际提示改为建议稍后重试，列出两条市场各自的后备链并注明需要可用，避免保证限流持续时间或后备源成功。

## 问题与建议

Yahoo 当前仍返回限流，真实自检已覆盖 WARN；后备源是否有数据取决于其可用性及标的。PR 尚未合并。

# 0007 处理图表 HTML 测试的失败

- **代码仓库**：`jimojijie29/Vibe-Trading-William`，基于 `william`
- **分支**：`codex/0007-chart-html-tests`
- **依赖**：PR #3（任务 0003）合并后开始
- **来源**：任务 0003 的 CI 基线，[运行记录](https://github.com/jimojijie29/Vibe-Trading-William/actions/runs/43347475219)

## 背景

CI 上有 7 个图表 HTML 相关的后端测试失败：

| 测试 | 失败现象 |
| --- | --- |
| `test_cn_treasury_redesign.py::test_cn_treasury_html_smoke` | HTML 约 64 KB，测试要求大于 500 KB |
| `test_macro_echarts.py::test_html_inline_echarts` | 同上 |
| `test_macro_echarts_more.py::test_ust_html_smoke` | 同上 |
| `test_macro_echarts.py::test_html_table_tab` | HTML 里没有 `\\uFEFF` |
| `test_etf_broad_history.py::test_render_html_default_window_and_range_buttons` | 没有 `dispatchAction({type:'dataZoom'` |
| `test_etf_broad_history.py::test_render_html_index_delta_pct_in_tooltip` | 没有 `fmtDelta(d.idx_chg[i],'%')` |
| `test_etf_broad_history.py::test_tooltip_amount_delta` | 没有 `fmtDelta(d.amt_chg[i],' 亿')` |

## 已知线索（Claude 的初步判断，需要你核实）

"大于 500 KB" 的三个测试是在检查 `echarts.min.js` 是否被整份内联进 HTML。但 `agent/macro_workflows/echarts_common.py` 的注释写着，从 2026-08-27 起图表改为外链共享库 `macro_workflows/assets/echarts.min.js`，只有在不传 `out_path` 或共享库文件缺失时才走旧的全文内联。所以这三个很可能是产品行为已经改了、测试没跟上。

另外四个的原因还不清楚，有两种可能：

- 同样是测试过期：图表脚本改成了延迟加载或改了写法，相关 JS 片段不再出现在生成的 HTML 里。
- 真实缺陷：某次改版（近期提交里有 "layout A redesign"）把 CSV 导出的 BOM、缩放按钮、悬浮提示里的变化量这些功能弄丢了。

这两种情况的处理完全不同，所以必须逐个查清。

## 要求

对 7 个失败逐一回答：

1. 这个测试原本要保护的功能是什么？
2. 这个功能现在还在不在？用代码位置和实际生成的 HTML 片段证明。
3. 测试是从哪次提交开始失败的？用 `git log`、`git bisect` 或逐个检出相关提交定位。仓库是完整克隆时直接做；浅克隆时先拉取足够的历史。
4. 结论：**测试过期**（功能还在或被有意替换）还是**功能回归**（功能被无意弄丢）。

然后按结论处理：

- **测试过期**：更新测试，让它检查现在的正确行为。例如外链方案下应当断言 HTML 引用了共享的 `echarts.min.js` 且该文件存在，而不是断言体积。更新后的测试必须仍然能在功能真的坏掉时失败，不能改成恒真的断言。
- **功能回归**：不要改测试。能在图表生成代码里小范围修复的就修；修复需要判断设计意图的，写进回报，把结果标为 `部分完成`，由 Claude 和用户决定。

另外确认一件事：这 7 个测试在你本机（用户的 Windows 环境）是通过还是失败？本机通过而 CI 失败时，说明测试依赖了本机才有的文件或状态，要找出是什么并写进回报。

## 范围

上述三个测试文件，以及 `agent/macro_workflows/` 下对应的图表生成代码（仅限修复已确认的功能回归）。

范围外：不改图表的视觉设计，不改 `/macro/file` 等接口的行为，不动任务 0006、0008 的失败项。

## 验收标准

1. 回报包含一张 7 行的表：测试、保护的功能、功能是否还在、开始失败的提交、结论、处理方式。
2. 处理后这 7 个测试在 CI 上通过，或者明确列出哪些因属于功能回归而保持失败。
3. 被更新的每个测试，说明它现在在什么情况下会失败。
4. 没有新增 `skip`、`xfail`，没有删除测试。

## 交付

向 `william` 提 PR，标题以 `[0007]` 开头，并写回报 `reports/0007-chart-html-tests.md`。

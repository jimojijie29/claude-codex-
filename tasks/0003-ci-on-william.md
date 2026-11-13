# 0003 让 CI 在 william 分支上运行

- **代码仓库**：`jimojijie29/Vibe-Trading-William`，基于 `william`
- **分支**：`codex/0003-ci-on-william`
- **依赖**：无

## 背景

仓库里已有 `.github/workflows/test.yml`，但它只对 `main` 触发：

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

这个仓库的默认分支和 PR 目标分支都是 `william`，所以 CI 从来不跑。PR #1 上没有任何检查就是这个原因。这个工作流继承自上游项目，它在 `william` 上能不能通过，目前没人知道。

工作流现有内容：Python 3.11 安装 `.[dev]`、语法检查、`pytest`（带覆盖率，忽略 `e2e_backtest` 和 `test_e2e_harness_v2.py`）、前端 `npm ci`、`npm run build`、`vitest run`，整体超时 10 分钟。

## 目标

向 `william` 提的每个 PR 都自动跑测试，并且结果可信：绿色代表测试真的通过了。

## 范围

1. 修改 `test.yml` 的触发条件，使其对 `william` 的 push 和以 `william` 为目标的 PR 生效。保留对 `main` 的触发。
2. 在 PR 上看 CI 的实际运行结果，把失败项逐个归类：
   - **工作流自身的问题**（超时太短、依赖装不上、路径不对、Node 或 Python 版本不合适）：在本任务里修。
   - **测试依赖网络或外部服务**：项目已装 `pytest-socket` 并定义了 `unit`、`integration` 两个标记。按现有约定处理，把需要联网的测试标成 `integration` 并在 CI 里排除，同时在回报里列出被排除的测试。
   - **`william` 分支上真实存在的代码缺陷或过期测试**：不要在本任务里修，也不要跳过。列进回报，由 Claude 另开任务。
3. 工作流超时不够时可以调大，并把 Python 与前端拆成两个并行 job，前提是不减少检查内容。

范围外：不改业务代码，不改测试的断言，不删除测试，不为了变绿而给失败的测试加 `skip` 或 `xfail`。不动 `wiki.yml`、`wiki-deploy.yml`。

## 验收标准

1. 本任务的 PR 上能看到 CI 检查实际运行，回报里附运行链接。
2. 回报给出一份完整清单：CI 跑了多少测试、通过多少、失败多少，每个失败项属于上面哪一类、原因是什么。
3. 属于第三类（真实缺陷）的失败项如果存在，CI 允许是红的。**如实报告红色比做出一个虚假的绿色更重要。** 这种情况下把结果标为 `部分完成`，并建议后续任务怎么拆。
4. 被标成 `integration` 而排除的测试，每一个都要说明它访问了什么外部资源。
5. 回报里写明一次完整 CI 运行的耗时。

## 交付

向 `william` 提 PR，标题 `[0003] ci: run tests on william branch`，并写回报 `reports/0003-ci-on-william.md`。

## 提示

修改 `.github/workflows/` 下的文件需要推送凭据带 `workflow` 权限。推送被拒时不要绕过，把结果标为 `受阻` 并说明，由用户处理权限。

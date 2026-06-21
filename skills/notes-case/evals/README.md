# notes-case 最小回归测试集

> 这是**纯手工对照**用的最小回归集——12 条 prompt,覆盖触发判定的四类轴。**不跑 skill-creator 的完整 eval / viewer / assertion**,只在你改 SKILL.md description 后用作 checklist 跑一遍。

## 用法

改完 description(或调整白/黑名单)后,准备一个**全新会话**(确保 skill 状态新鲜),依次说每条 `prompt`,对照 `expected_behavior` 是否吻合。

### `expected_behavior` 三种值

| 值 | 期望表现 |
|---|---|
| `trigger` | agent 应直接进入 skill 流程(可能先做问答收集,但意图已明确) |
| `no-trigger` | agent 不应进 skill,按普通调试/对话回应 |
| `ask-first` | agent 应先反问"是要归档还是只是会话备忘",得到归档意图才进 |

### 通过标准

- **trigger / no-trigger** 错判 → 改 description 引发回归,必须修
- **ask-first 退化为 no-trigger** → 漏掉了模糊场景的安全网,要修
- **ask-first 退化为 trigger** → 误触发风险变高,要修

## 四类测试轴

| 轴 | 条数 | 测什么 |
|---|---|---|
| white-list | 6(id 1-6) | 5 个流程 + 强制点名是否都能触发 |
| black-list | 4(id 7-10) | "记一下"、调试动作、会话总结、关键词在代码正文里 — 都不应触发 |
| ambiguous | 2(id 11-12) | 模糊指令必须先问,不要闷头开干 |

## 跑完后

- 全部吻合 → description 合格,正常 commit
- 有 1-2 条不吻合 → 看是 description 措辞需要微调,还是测试集本身需要更新预期(比如这个用法你新版本就是不想保留)
- 有 ≥3 条不吻合 → 这次 description 改坏了,考虑 revert

## 扩展

每次实际使用中发现**新的边界 case**(我误触发了 / 该触发的没触发),就加一条到 `evals.json`。这个文件会随时间长出更准的回归网。

不需要写 assertion、不需要装 skill-creator——它就是个**人肉对照的 checklist**。要不要升级成跑 skill-creator 的完整 eval(viewer + 量化指标),等真正出现"description 反复改、记不清哪版好"的时候再说。

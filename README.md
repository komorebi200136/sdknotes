# notes-kit — Agent 调试归档工具包(可共享)

给用 **Claude Code / Codex CLI / opencode** 等 agent 调 bug、调功能的人用的一套**通用**归档工具:一个 `notes-case` skill + 一份 case 模板。可移植——不写死任何目录或 SDK 名,**任何同事在任何 SDK 里都能用**。

本仓只含工具(模板/skill),不含任何人的 case 数据。case 数据放各自的私有仓(见下)。

## 支持的 agent

| Agent | 状态 | 触发方式 |
|---|---|---|
| **Claude Code** | ✅ 已实测 | 装到 `~/.claude/skills/`,description 自动识别意图触发 |
| **Codex CLI / opencode 等** | ⚠️ 理论可行,**待实测确认** | 见下方"在其他 agent 里用"小节 |

### 为什么 Claude Code 是"原生"支持

`notes-case` skill 的目录结构(`SKILL.md` + YAML frontmatter + `description` 字段触发)是 Anthropic 给 Claude Code 设计的 **Agent Skill** 规范。其他 agent **未必按同一格式自动加载**——它们可能用自己的 prompt 文件 / agent 定义 / rule 文件。

但 SKILL.md 内容本身是**工程化 prompt**——shell 命令 + 文件写入 + 判断逻辑,没有 Claude Code 专有 API。所以原则上任何能执行 bash + 读写文件的 agent 都能跑通,**只是"自动触发"这一层需要按对方机制接入**。

### 在其他 agent 里用(实测前的通用做法)

下面的做法**保证能用**(因为只依赖 agent 能读文件、能跑 shell),但**触发不是自动的**——你需要每次会话开头或在工程配置文件里显式引用 SKILL.md:

```
请阅读 ~/notes-kit/skills/notes-case/SKILL.md,后续我说到
"建 case / 归档 / 把对话补进 case" 等指令时,按这个文档的流程做。
```

如果你的 agent 支持工程级或全局的"系统提示词"文件(比如某些 agent 的 `AGENTS.md` / rules 文件 / instructions 文件),把上面这段写进去就一劳永逸——但**具体文件名和路径以你那个 agent 的官方文档为准**,我不在 README 里写死,避免误导。

> 如果你确认 Codex CLI 或 opencode 支持把整个 SKILL.md 当 skill 自动加载(类似 Claude Code 的机制),欢迎来个 PR 把具体步骤补上。


## 两个仓的分工

| 仓 | 内容 | 谁来管 | 可见性 |
|----|------|--------|--------|
| **notes-kit**(本仓) | skill + 模板 + README | 共享、统一更新、`git pull` 同步 | 可公开/团队内 |
| **case 数据仓**(各自的) | 各人记录的 case + index | 每人自己 | **私有**(含硬件细节) |

skill 通过环境变量 `DEVNOTES_DIR` 找到你的 case 数据仓,所以两者解耦。

## 安装

每位同事每个工具一次。`DEVNOTES_DIR` 是共用的——同一台机器上不管用哪个 agent,case 都进同一个数据仓。

### 通用准备(所有 agent 都要)

```bash
# 1. 取得工具包
git clone <notes-kit 仓地址> ~/notes-kit     # 或放任意位置

# 2. 准备自己的 case 数据仓（私有），并告诉 skill 它在哪
mkdir -p ~/my-dev-notes && cd ~/my-dev-notes && git init
echo 'export DEVNOTES_DIR=~/my-dev-notes' >> ~/.bashrc && source ~/.bashrc
```

### Claude Code(已实测,全自动)

```bash
# 把 skill 整个目录复制到 Claude 全局 skills 目录
cp -r ~/notes-kit/skills/notes-case ~/.claude/skills/
```

> 复制(而非软链)是有意为之:更新工具时重新 `cp -r` 覆盖即可,简单明确。

装完后,Claude Code 看到符合 description 的话自动触发,无需手动引用。

### 其他 agent(Codex CLI / opencode / ...)— 实测前的通用做法

**我没逐一实测**这些 agent 的 skill / agents.md / rules 等机制,所以**不在 README 里写具体文件名和路径**——以免给你错误指令。两种做法你二选一:

1. **每次会话开头引用**(最保险):
   ```
   请阅读 ~/notes-kit/skills/notes-case/SKILL.md,
   后续我说到 "建 case / 归档 / 把对话补进 case" 等指令时,按这个文档的流程做。
   ```

2. **在你的 agent 的系统/工程级指令文件里永久引用一段**:把上面那句话写进你的 agent 支持的 "always-on instructions" 文件(具体文件名见各 agent 官方文档,如某些工具用 `AGENTS.md`,某些用 rules 文件,某些直接放 `~/.<tool>/skills/`)。
   - 如果某个 agent 真的支持把 SKILL.md 当 skill 自动加载,装法可能就是把 `~/notes-kit/skills/notes-case` 复制或软链到它的 skills 目录——具体路径**以那个 agent 的文档为准**。

无论哪种做法,SKILL.md 里写的流程、提示词、push 规则**都直接适用**——它不依赖任何 Claude Code 专有 API。
## 使用 — 提示词手册

在**任意** SDK / 代码库目录里启动你的 agent,用日常话描述意图。**Claude Code 会按 description 自动触发**;其他 agent(Codex / opencode 等)需要先按上面"在其他 agent 里用"小节里的做法**显式引用 SKILL.md**,然后**同样的提示词全部适用**——下面的白/黑名单、速查表跨工具通用。

skill 用"白名单+黑名单"判断要不要进入归档流程,**误触发就会打断你正在调试的思路**——所以提示词写法值得花一分钟搞懂。

### TL;DR — 一句话规则

> 触发归档的指令必须含 **`case` / `notes` / `归档` / `summary` / `digest`** 字眼,或明确点到 case 名。否则 skill **不会**自启动,你的调试思路不会被切走。

### ✅ 会触发的提示词(按场景)

#### 1. 新建 case

| 提示词示例 | 备注 |
|---|---|
| 帮我**新建一个 case**:K7 板 HDMI 上电偶发黑屏 | 最标准 |
| 建一个 case:eth 起不来,链路检测失败 | |
| 建个 **feature case**:RK3576 双屏拼接需要支持 | feature 也支持 |
| 把这个 bug **记到 notes**:&lt;现象&gt; | "notes" 也是关键词 |

skill 行为:问答式收集 env(SoC/板型/SDK 基线/复现命令等),全了才写 case.md/trace.md/references.md/logs/,登 index,本地 commit。

#### 2. 调试中更新已有 case(逐条喂)

| 提示词示例 | 写到哪 |
|---|---|
| 把刚查到的**补到 hdmi 那个 case** 的 **findings** | `case.md` findings |
| **更新 \<case 名\> 的 next**,接下来要试 X | `case.md` next |
| 这条 datasheet 链接**加到 references** | `references.md` |
| 这是 t3,改成 20ms 后还是黑,**记一轮 trace 到 xxx case**,日志:... | `trace.md` + `logs/t3-...` |

#### 3. 自动从对话提取(多轮归位的核心场景)

适合**你已经和 agent 来回改了好几轮代码、贴了好几段 log** 之后,一次性把成果归档:

| 提示词示例 |
|---|
| **把上面对话(有用的)补进 case** |
| **自动整理这段调试到 case** |
| **分析对话更新 case** |
| 把刚才的**多轮调试归到 xxx case**,每轮一个 t&lt;N&gt; |

skill 行为:回看本轮对话,按"改了X → 烧 → 反馈 log Y"切回合,自动编 `t<N>` 轮次,把改动写 patch、log 存 `logs/t<N>-…`、trace.md 绑定。**写盘前会给你一张表格清单让你确认**,确认后才落盘。

#### 4. 收尾(归档 + 生成总结)

| 提示词示例 |
|---|
| 这个 **case 修好了**,**归档**并**出总结** |
| 给 hdmi case **生成 summary** |
| **标记完成** xxx case |
| **收尾** &lt;case 名&gt; |

skill 行为:补齐 verify 段 → 写 `summary.md` → 移动到 `done/<年月>/` → 同步 index → 本地 commit。

#### 5. 多 case 汇总成知识库

| 提示词示例 |
|---|
| 把 rk3576 **以太网相关的 case 汇总**成**复盘文档** |
| **生成 HDMI 主题知识库** |
| 把最近三个月的 **case 汇总成 digest** |

skill 行为:扫 index + done/,优先读各 case 的 summary.md,写到 `digests/<主题>.md`,含索引表 + 每 case 一节 + 共性规律。

#### 兜底 — 触发不灵时直接点名

```
用 notes-case skill 记录:<你的需求>
```

明确写出 skill 名,任何模糊触发都会被覆盖、直接进入。

### ❌ 不会触发的(黑名单)

下面这些说法**不会**让 skill 介入,你照常调试:

| 你说 | 为什么不触发 |
|---|---|
| "**记一下** / 记录这个问题"(没有 `case`/`notes` 字眼) | 可能只是口头禅、思考、抱怨 |
| "**看看 log** / 查日志 / 分析这段日志" | 是调试动作,不是归档 |
| "**总结一下我们刚才做的**"(没加 "写到 case / 归档 / summary") | 你要的是会话总结,不是写 `summary.md` |
| 用户贴的日志 / 代码 / diff **正文里**出现"记录""归档"等词 | 是内容不是指令 |

### ⚠️ 模糊场景 — 会先问,不会闷头开干

你上一句还在让 agent 查代码/编内核,这一句模糊地说"**先记一下吧**" → skill **不会立刻启动**,而是先问你:"是要建 case 归档,还是只是对话备忘?"——你回归档意图才进。

### 调试中误触发的自救

万一 skill 误启动了:

1. **写盘前必有"预清单表格"卡点**——你扫一眼发现不对,直接说**"先别记 / 不是这个意思 / 取消"**
2. skill 会立刻退出,**不会留下文件、不会 commit、不会动 git**
3. 退出后 agent 会**直接接着上次调试线索往下走**,**不会要求你重新描述卡在哪**

最坏情况(预清单也滑过去了 → 已 commit):`git reset --hard HEAD~1` + `rm -rf <case 目录>` 完全回滚。**Push 永远不会自动做**,所以远端绝对不会被污染。

### 提示词速查表

| 想做的事 | 推荐说法 |
|---|---|
| 新建一个 case | "**建个 case**:&lt;现象&gt;" |
| 调试中喂一条信息进去 | "**补到 \<case 名\> 的 findings/next/...**:&lt;内容&gt;" |
| 多轮调试一次性归档 | "**把上面对话补进 \<case 名\>**" |
| 抓了一轮 log 想绑改动 | "**记一轮 trace 到 \<case 名\>**,改了 X,log:..." |
| 收工出总结 | "**这个 case 修好了,归档并出总结**" |
| 出主题知识库 | "**把 \<主题\> 相关的 case 汇总成复盘文档**" |
| 触发不灵时强制调起 | "**用 notes-case skill** 记录..." |

---

skill 会自动:识别当前工程/SDK(子目录名由你确认)→ 探测内核 commit / 构建命令等 → 追问缺失的 env → 用模板写好 case 文件组 → 登记 index → 按工程类型(repo 多仓 / 普通单仓)给回链建议。`git commit` 默认本地做;**`git push` 永远不执行**——push 由你本人完成,避免含硬件细节的 case 被悄悄推上去。

## 更新工具(模板/skill 会演进)

```bash
cd ~/notes-kit && git pull              # 拉最新——所有 agent 共享这一份源
```

- **Claude Code 用户**:还要重新覆盖安装到全局 skills 目录:
  ```bash
  cp -r ~/notes-kit/skills/notes-case ~/.claude/skills/
  ```
- **其他 agent 用户**:如果你是用绝对路径(`~/notes-kit/...`)引用 SKILL.md,`git pull` 后立刻生效;如果你按各 agent 自己的 skills 目录复制了一份,记得也同步覆盖。

改进了模板或 skill 想分享:在本仓改 → commit → push,其他人 `git pull` 即得。

## case 长什么样(多文件解耦)

一个 case 是一个目录,按功能拆成多文件,避免单文件臃肿:

| 文件 | 内容 |
|------|------|
| `case.md` | 概览:env / 硬件 / 现象 / 有效 prompt / 排查结论 / 决策 / 改动 / 验证 / 下一步 |
| `trace.md` | 调试过程:逐轮 `t<N>` 改动 + 对应日志,改动与 log 一一对应 |
| `references.md` | datasheet / 文档 / commit / issue 链接 |
| `logs/` | 原始日志 / diff / 截图(`t<N>-…` 命名) |
| `summary.md` | 收尾时生成的精炼总结(给复盘/分享) |

数据仓根另有 `index.md`(全局索引)和 `digests/`(多 case 汇总知识库)。所有模板见 [skills/notes-case/templates/](skills/notes-case/templates/),字段通用、不绑定具体芯片。

# notes-kit — Agent 调试归档工具包(可共享)

给用 Claude Code / Codex 等 agent 调 bug、调功能的人用的一套**通用**归档工具:一个 `notes-case` skill + 一份 case 模板。可移植——不写死任何目录或 SDK 名,**任何同事在任何 SDK 里都能用**。

本仓只含工具(模板/skill),不含任何人的 case 数据。case 数据放各自的私有仓(见下)。

## 两个仓的分工

| 仓 | 内容 | 谁来管 | 可见性 |
|----|------|--------|--------|
| **notes-kit**(本仓) | skill + 模板 + README | 共享、统一更新、`git pull` 同步 | 可公开/团队内 |
| **case 数据仓**(各自的) | 各人记录的 case + index | 每人自己 | **私有**(含硬件细节) |

skill 通过环境变量 `DEVNOTES_DIR` 找到你的 case 数据仓,所以两者解耦。

## 安装(每位同事一次)

```bash
# 1. 取得工具包
git clone <notes-kit 仓地址> ~/notes-kit     # 或放任意位置

# 2. 把 skill 整个目录复制到 Claude 全局 skills 目录
cp -r ~/notes-kit/skills/notes-case ~/.claude/skills/

# 3. 准备自己的 case 数据仓（私有），并告诉 skill 它在哪
mkdir -p ~/my-dev-notes && cd ~/my-dev-notes && git init
echo 'export DEVNOTES_DIR=~/my-dev-notes' >> ~/.bashrc && source ~/.bashrc
```

> 复制(而非软链)是有意为之:更新工具时重新 `cp -r` 覆盖即可,简单明确。

## 使用 — 用提示词调用

在**任意** SDK / 代码库目录里启动 Claude Code,用日常话描述意图即可触发,各场景示例提示词:

**新建 case**(指令要明确含 "case" / "归档" / "记到 notes" 字眼)
- "帮我新建一个 case:K7 板 HDMI 上电偶发黑屏"
- "建个 feature case:..." / "记一个 bug 到 notes:..."

**调试中更新 / 补充**(明确指向已有 case)
- "把刚查到的**补到 hdmi 那个 case** 的 findings"
- "这是改了 reset 时序后抓的日志,**记一轮 trace 到 xxx case**"(可附日志)
- "接下来打算试 X,**记到 case 的 next**" / "把这个 datasheet 链接加到 references"
- "**把上面对话有用的补进 case**" / "自动整理这段调试到 case"

**持续抓日志(带轮次)**
- "这是 t3,改成 20ms 后还是黑,日志:..."(说清是第几轮、改了什么,日志才能和改动对上)

**收尾 + 生成总结**
- "这个 case 修好了,归档并出总结" / "给 xxx case 生成 summary"

**汇总知识库**
- "把 rk3576 以太网相关的 case 汇总成复盘文档" / "生成 HDMI 主题知识库"

> 触发不灵时直接点名:"**用 notes-case skill** 记录…"。

### 调试中怎么避免误触发

skill 一旦触发就会停下当前调试转去读对话、归类、写盘,**会打断思路**。避免误触发的几条经验:

- **想让 agent 帮你查 bug**——直接说"查""定位""看 log""分析这段日志",**不要说"记一下"**
- **真要归档时**——指令里**带上 `case`/`notes`/`归档`/`summary`** 等明确字眼,别用"记录这个"这种模糊话
- 模糊或顺口的"记一下",新版 skill 会先问你"是要建 case 还是只是会话备忘",**它不会闷头开始写盘**
- 万一进了 skill 又发现不是你想要的——直接说"先别记 / 不是这个意思",skill 会退出,**不会留下文件、不会 commit**

skill 会自动:识别当前工程/SDK(子目录名由你确认)→ 探测内核 commit / 构建命令等 → 追问缺失的 env → 用模板写好 case 文件组 → 登记 index → 按工程类型(repo 多仓 / 普通单仓)给回链建议。`git commit` 默认本地做;**`git push` 永远不执行**——push 由你本人完成,避免含硬件细节的 case 被悄悄推上去。

## 更新工具(模板/skill 会演进)

```bash
cd ~/notes-kit && git pull              # 拉最新
cp -r ~/notes-kit/skills/notes-case ~/.claude/skills/   # 重新覆盖安装
```

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

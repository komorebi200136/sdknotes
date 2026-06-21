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

## 使用

在**任意** SDK / 代码库目录里启动 Claude Code,说一句:

- “帮我新建一个 case：<现象/目标>”
- “把这个 bug 记到 notes”
- “这个 case 修好了，归档”

skill 会自动:识别当前工程/SDK → 探测内核 commit 等 → 追问缺失的 env → 用模板写好 case → 登记 index → 提醒在子仓库 commit 里写 `Refs notes/<case>` 回链。收尾时自动 `git commit && push` 到你的数据仓备份。

## 更新工具(模板/skill 会演进)

```bash
cd ~/notes-kit && git pull              # 拉最新
cp -r ~/notes-kit/skills/notes-case ~/.claude/skills/   # 重新覆盖安装
```

改进了模板或 skill 想分享:在本仓改 → commit → push,其他人 `git pull` 即得。

## case 模板里有什么

见 [skills/notes-case/templates/case.md](skills/notes-case/templates/case.md)。要点:env(SoC/板型/SDK 基线/内核 commit/复现命令)、硬件信息、现象+复现概率、有效 prompt、排查/决策、改动落点、验证。针对嵌入式 + agent 协作场景设计,字段通用、不绑定具体芯片。

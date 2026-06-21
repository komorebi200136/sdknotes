---
name: notes-case
description: >-
  Record a debugging/development case into the user's private notes data repo (located via
  the DEVNOTES_DIR environment variable). Use this whenever the user wants to file, log, record,
  archive, or "建 case / 记一下 / 归档" a bug they're chasing or a feature they're bringing up —
  in ANY SDK or codebase, regardless of directory layout — phrases like "新建一个 case",
  "把这个 bug 记到 notes", "记录这个问题", "归档这次调试", "建个 feature case", or when they paste
  a symptom/log and ask to start tracking it. Also use to CLOSE a case (mark done, move to done/,
  commit+push). This skill auto-detects the current project/SDK, gathers complete reproducible info
  (SoC/board, SDK baseline, kernel commit, relevant files, repro command), writes a filled-in
  case.md from the bundled template, registers it in the data repo's index, and reminds about
  commit back-links. Prefer this over hand-writing notes files so nothing important (especially env)
  gets skipped. Portable: no hard-coded paths or SDK names — works for any teammate who sets DEVNOTES_DIR.
---

# notes-case — 把一个 case 完整落到 notes 数据仓

帮用户把一个新 bug / 功能调试归档到 **case 数据仓**(由环境变量 `DEVNOTES_DIR` 指向),或收尾一个已完成的 case。本 skill 可移植:不写死任何目录/SDK 名,任何同事设好 `DEVNOTES_DIR` 即可用。

核心原则:**能自动探测的信息自己跑命令填,填不出的才问用户**。env 段(尤其 SoC/板型、SDK 基线)缺了就无法复现、也分不清属于哪块板——所以 env 必须填全才算完成。

## 关键路径(全部动态解析,不要硬编码)

- **case 数据仓根**: 环境变量 `$DEVNOTES_DIR`。第一步就检查它存在:
  ```bash
  : "${DEVNOTES_DIR:?未设置 DEVNOTES_DIR——请先 export DEVNOTES_DIR=<你的 case 数据仓路径>}"
  ```
  若未设置,停下来提示用户在 `~/.bashrc` 里加 `export DEVNOTES_DIR=...` 并 source,别瞎猜路径。
- **模板**: `~/.claude/skills/notes-case/templates/case.md`(随本 skill 一起安装)。
- **当前工程/SDK 名**: 见下方第 1 步自动识别。
- **case 落点**: `$DEVNOTES_DIR/<sdk>/active/<目录名>/`
- **索引**: `$DEVNOTES_DIR/index.md`

## 新建一个 case

### 1. 先自动探测,减少向用户提问

```bash
: "${DEVNOTES_DIR:?未设置 DEVNOTES_DIR}"
# 工程根 = 从当前目录向上找到第一个含 .repo/.git/build.sh/Makefile 的目录
root=$PWD
while [ "$root" != "/" ]; do
  if [ -e "$root/.repo" ] || [ -d "$root/.git" ] || [ -e "$root/build.sh" ]; then break; fi
  root=$(dirname "$root")
done
SDK=$(basename "$root"); echo "SDK=$SDK (root=$root)"
# 内核 commit：依次尝试常见内核目录（非内核工程则跳过）
for k in "$root"/kernel "$root"/kernel-* ; do
  [ -d "$k/.git" ] && (cd "$k" && echo "kernel $(basename $k) @ $(git rev-parse --short HEAD)") && break
done
# 若是 Rockchip 风格 SDK，顺带探测当前板型（其它 SDK 无此文件，自然跳过）
cat "$root"/device/.BoardConfig.mk 2>/dev/null | grep -iE "DEFCONFIG|TARGET_PRODUCT" || true
```

识别不出工程根就退回用 `basename $PWD` 并**让用户确认 SDK 名**。不同 SDK 的内核目录名/构建系统不同,探测不到的字段退回问用户,别瞎填。

### 2. 向用户收集填不出来的信息

按优先级问,**env 段不全(尤其 SoC/板型、SDK 基线、复现命令)不要写文件**:

- **类型**: bug 还是 feature
- **现象 / 目标**(必填): bug 写预期 vs 实际 + **复现概率/触发条件**(间歇性 bug 尤其要问"多大概率、什么条件触发");feature 写要实现什么 + 验收标准
- **SoC / 板型 + SDK 基线**(必填,探测不到才问)
- **相关 DTS / 源文件**(嵌入式 bug 几乎必填)
- **复现 / 构建命令**(必填)
- **给 agent 的有效指令**: 尤其那条约束 agent 别乱改的范围限定 prompt——最值得存
- 硬件 bug 再问: 原理图页码、关键网络/引脚、外设型号(填进 hw 段)
- 已有的 findings / 决策 / 改动落点: 有就填,没有留空待补

用户贴的大段日志别塞进正文,留到第 4 步存附件。

### 3. 生成目录并写 case.md

目录名: `bug-<现象>-<根因关键词>` 或 `feat-<模块>-<短描述>`,小写中划线、带可检索词。

```bash
mkdir -p "$DEVNOTES_DIR/$SDK/active/<目录名>/logs"
```

读模板 `~/.claude/skills/notes-case/templates/case.md` 作骨架,**把收集到的信息真正填进去**(不是原样拷贝)。日期用 currentDate 上下文里的今天,别取系统时间。状态 `active`,复现状态按现象填。没拿到的字段保留模板占位提示。

### 4. 存附件

大段 dmesg / 串口日志 / 完整 diff / 截图 → 写进 case 目录的 `logs/`,并在 case.md 「附件」段引用。

### 5. 登记 index.md

在 `$DEVNOTES_DIR/index.md` 表格追加一行(表尾空行之前):

```
| [<目录名>](<sdk>/active/<目录名>/case.md) | <SoC> | <板型/SDK> | bug/feature | active | <kernel@commit> | <一句话现状> |
```
若 `index.md` 不存在(新数据仓),先按这个表头建。

### 6. 提醒回链(通用坑)

很多 SDK 顶层是 repo 工具聚合、非单一 git 仓。告诉用户:改动提交时,在对应子仓库(kernel 等)的 commit message 写一行 `Refs notes/<目录名>`,以后 `cd kernel && git log --grep="notes/<目录名>"` 能找回。

## 收尾一个 case(标记完成 + 备份)

用户说"修好了 / 归档 / 标记完成"时:

1. 确认 case.md 的 verify 段已填(实测步骤+结果+残留边界)、patch 段有 commit sha,复现状态改 `已解决`。缺了先补问。
2. 移动并按年月归档(年月用今天):
   ```bash
   mkdir -p "$DEVNOTES_DIR/$SDK/done/<YYYY-MM>"
   mv "$DEVNOTES_DIR/$SDK/active/<目录名>" "$DEVNOTES_DIR/$SDK/done/<YYYY-MM>/"
   ```
3. 更新 index.md 该行:状态 `active → done`,路径前缀改 `done/<YYYY-MM>/`,结论改最终结论。
4. 提交并备份(数据仓含硬件细节,保持私有):
   ```bash
   cd "$DEVNOTES_DIR" && git add -A && git commit -m "<sdk>: done <目录名> — <一句话结论>" && git push
   ```

## 注意

- case 数据仓是**私有**的(含原理图/硬件细节),不要往公开仓推。模板/skill 本身不含敏感信息,放共享仓无妨。
- 不要污染各 SDK 自带的官方 `docs/`,个人记录只进数据仓。
- 信息不足时**先问再写**,宁可多问一句,也别写出无法复现的残缺 case。

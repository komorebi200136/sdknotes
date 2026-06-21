---
name: notes-case
description: >-
  Record a debugging/development case into the user's private notes data repo (located via
  the DEVNOTES_DIR environment variable). Use this whenever the user wants to file, log, record,
  archive, or "建 case / 记一下 / 归档" a bug they're chasing or a feature they're bringing up —
  in ANY SDK or codebase, regardless of directory layout — phrases like "新建一个 case",
  "把这个 bug 记到 notes", "记录这个问题", "归档这次调试", "建个 feature case", or when they paste
  a symptom/log and ask to start tracking it. Also use to UPDATE an existing case (append new
  findings/logs/prompts to it), to CLOSE a case (mark done, move to done/, commit+push) which also
  produces a polished summary.md, or to DIGEST several finished cases into a knowledge-base / 复盘
  doc — phrases like "总结成 md", "出个调试总结", "把这些 case 汇总", "生成知识库/复盘文档". This skill auto-detects the current project/SDK, gathers complete reproducible info
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
- **当前工程/SDK 名**: 新建/更新时**由用户手动指定**(见第 1 步,探测值仅作建议)。
- **case 落点**: `$DEVNOTES_DIR/<sdk>/active/<目录名>/`
- **索引**: `$DEVNOTES_DIR/index.md`

## 新建一个 case

### 1. 确定 SDK 子目录(手动指定)+ 自动探测其余 env

**case 归到哪个 SDK 子目录,由用户手动确认/输入——不自动采用探测值**,避免在 `kernel` 等子仓库里启动时落错目录。先给出参考信息让用户拍板:

```bash
: "${DEVNOTES_DIR:?未设置 DEVNOTES_DIR}"
# 向上找工程根（仅用于探测内核/板型，并给个 SDK 名建议，不直接当落点）
root=$PWD
while [ "$root" != "/" ]; do
  if [ -e "$root/.repo" ] || [ -e "$root/build.sh" ] || [ -d "$root/.git" ]; then break; fi
  root=$(dirname "$root")
done
echo "数据仓已有子目录:"; ls -1 "$DEVNOTES_DIR" 2>/dev/null | grep -vE '^(index\.md|README.*|\.git.*)$' || true
echo "建议 SDK 名(工程根目录名,仅供参考): $(basename "$root")"
# 内核 commit
for k in "$root"/kernel "$root"/kernel-* ; do
  [ -d "$k/.git" ] && (cd "$k" && echo "kernel @ $(git rev-parse --short HEAD)") && break
done
# 若是 Rockchip 风格 SDK，顺带探测当前板型（其它 SDK 无此文件，自然跳过）
cat "$root"/device/.BoardConfig.mk 2>/dev/null | grep -iE "DEFCONFIG|TARGET_PRODUCT" || true
# 构建方式：工程根有 build.sh（Rockchip 等 SDK）就建议标准编译流程
[ -e "$root/build.sh" ] && echo "构建命令(建议): cd $root && ./build.sh <defconfig>  # 选板型后 ./build.sh 全量，或 ./build.sh kernel 等增量"
```

**问用户要 `$SDK`**:展示上面"已有子目录"和"建议名",让用户选一个已有的、或输入新的。用户给定后才记为 `$SDK`。其余探测不到的字段(板型/SDK 基线/复现命令等)同样退回问用户,别瞎填。

### 2. 向用户收集填不出来的信息

按优先级问,**env 段不全(尤其 SoC/板型、SDK 基线、复现命令)不要写文件**:

- **类型**: bug 还是 feature
- **现象 / 目标**(必填): bug 写预期 vs 实际 + **复现概率/触发条件**(间歇性 bug 尤其要问"多大概率、什么条件触发");feature 写要实现什么 + 验收标准
- **SoC / 板型 + SDK 基线**(必填,探测不到才问)
- **相关 DTS / 源文件**(嵌入式 bug 几乎必填)
- **复现 / 构建命令**(必填): 工程根有 `build.sh`(Rockchip 风格 SDK)时,**编译在 SDK 根目录进行**——先 `cd <SDK根> && ./build.sh <defconfig>` 选板型,再 `./build.sh` 全量编译(或 `./build.sh kernel`/`uboot` 等增量);其它 SDK 按其构建方式填
- **给 agent 的有效指令**: 尤其那条约束 agent 别乱改的范围限定 prompt——最值得存
- 硬件 bug 再问: 原理图页码、关键网络/引脚、外设型号(填进 hw 段)
- 已有的 findings / 决策 / 改动落点: 有就填,没有留空待补

用户贴的大段日志别塞进正文,留到第 4 步存附件。

### 3. 生成目录并写 case.md

先查重(避免重复建或覆盖同名):
```bash
ls "$DEVNOTES_DIR/$SDK/active/" 2>/dev/null
grep -i "<现象关键词>" "$DEVNOTES_DIR/index.md" 2>/dev/null || true
```
若已有同名或明显相关的 case,先问用户:是「更新已有」(转下方"更新现有 case")还是确实新建。

目录名: `bug-<现象>-<根因关键词>` 或 `feat-<模块>-<短描述>`,小写中划线、带可检索词。

```bash
mkdir -p "$DEVNOTES_DIR/$SDK/active/<目录名>/logs"
```

读模板 `~/.claude/skills/notes-case/templates/case.md` 作骨架,**把收集到的信息真正填进去**(不是原样拷贝)。日期用 currentDate 上下文里的今天,别取系统时间。状态 `active`,复现状态按现象填。没拿到的字段保留模板占位提示。

### 4. 存附件

大段 dmesg / 串口日志 / 完整 diff / 截图 → 写进 case 目录的 `logs/`,并在 case.md 「附件」段引用。

### 5. 登记 index.md

在 `$DEVNOTES_DIR/index.md` 表格**最后一个数据行之后**追加一行(不要依赖表尾空行——它可能已被删):

```
| [<目录名>](<sdk>/active/<目录名>/case.md) | <SoC> | <板型/SDK> | bug/feature | active | <kernel@commit> | <一句话现状> |
```
若 `index.md` 不存在(新数据仓),先按这个表头建。

### 6. 提醒回链(通用坑)

很多 SDK 顶层是 repo 工具聚合、非单一 git 仓。告诉用户:改动提交时,在对应子仓库(kernel 等)的 commit message 写一行 `Refs notes/<目录名>`,以后 `cd kernel && git log --grep="notes/<目录名>"` 能找回。

### 7. 提交备份(避免误删丢失)

新建完就提交一次,别等到收尾:
```bash
cd "$DEVNOTES_DIR" && git add -A && git commit -q -m "<sdk>: new <目录名> — <一句话现状>"
git push 2>/dev/null || echo "（未配 remote，已本地提交;配好 remote 后再 push）"
```

## 更新现有 case(边查边补)

调试是持续的——中途有了新 findings、新日志、有效 prompt,随时往已有 case 追加。用户说"把这个补到 case""更新那个 xxx 的 case""这条加到 findings"时:

1. **定位 case**: 扫 `$DEVNOTES_DIR/$SDK/active/` 与 `index.md`;不唯一就列出让用户选,别猜。
2. **只更新相关段,不重写整个文件**:
   - 新结论 → findings 段;关键决策 → decisions 段;改动落点/commit → patch 段
   - 有效的范围限定 prompt → prompts 段
   - 大段日志/diff → 写进该 case 的 `logs/`,在「附件」段加一行引用
   - 复现状态有变(偶发→稳定复现/已解决)就同步顶部字段
3. 若进展改变了 index 的"一句话现状",同步更新 index 那一行。
4. **提交备份**: `cd "$DEVNOTES_DIR" && git add -A && git commit -q -m "<sdk>: update <目录名> — <补了什么>"`。

## 收尾一个 case(标记完成 + 生成总结 + 备份)

用户说"修好了 / 归档 / 标记完成 / 总结一下"时:

1. 确认 case.md 的 verify 段已填(实测步骤+结果+残留边界)、patch 段有 commit sha,复现状态改 `已解决`。缺了先补问。
2. **生成调试总结 `summary.md`**:读模板 `~/.claude/skills/notes-case/templates/summary.md`,从 case.md 提炼写进 case 目录的 `summary.md`——TL;DR + 问题 + 根因 + 解决方案 + 验证 + 避坑经验。这是面向复盘/分享的成品:**连贯成文、去掉占位**,既讲清"怎么解决的"(给人复用)又留下"避坑教训"(给人省时)。
3. 移动并按年月归档(年月用今天):
   ```bash
   mkdir -p "$DEVNOTES_DIR/$SDK/done/<YYYY-MM>"
   mv "$DEVNOTES_DIR/$SDK/active/<目录名>" "$DEVNOTES_DIR/$SDK/done/<YYYY-MM>/"
   ```
4. 更新 index.md 该行:状态 `active → done`,路径前缀改 `done/<YYYY-MM>/`,结论改最终结论。
5. 提交并备份(数据仓含硬件细节,保持私有):
   ```bash
   cd "$DEVNOTES_DIR" && git add -A && git commit -m "<sdk>: done <目录名> — <一句话结论>" && git push
   ```

## 汇总多个 case 成知识库 / 复盘文档

用户说"把这些 case 汇总""生成知识库""出个复盘文档"时:

1. **定范围**: 按 SDK、主题(如"以太网""HDMI")、或时间段筛选——扫 `index.md` 和各 `done/` 下的 case;不明确就问用户要哪一批。
2. **取材**: 优先读每个 case 的 `summary.md`(没有就读 case.md 提炼)。
3. **生成汇总**: 写到 `$DEVNOTES_DIR/digests/<主题或日期>.md`(目录没有就建)。结构:
   - 顶部一个目录/索引表(case 名 → 一句话结论 → 链接)
   - 每个 case 一小节:问题 / 根因 / 方案(带 commit)/ 避坑要点,末尾链接回原 case
   - 若多个 case 有共性(同一子系统反复出问题),单开一段"共性与规律"——这是知识库比单篇总结更值钱的地方
4. 提交备份: `cd "$DEVNOTES_DIR" && git add -A && git commit -m "digest: <范围> 汇总" && git push`。

## 注意

- case 数据仓是**私有**的(含原理图/硬件细节),不要往公开仓推。模板/skill 本身不含敏感信息,放共享仓无妨。
- 不要污染各 SDK 自带的官方 `docs/`,个人记录只进数据仓。
- 信息不足时**先问再写**,宁可多问一句,也别写出无法复现的残缺 case。

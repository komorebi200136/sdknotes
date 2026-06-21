---
name: notes-case
description: >-
  Record a debugging/development case into the user's private notes data repo (located via
  the DEVNOTES_DIR environment variable). Use this whenever the user wants to file, log, record,
  archive, or "建 case / 记一下 / 归档" a bug they're chasing or a feature they're bringing up —
  in ANY SDK or codebase, regardless of directory layout — phrases like "新建一个 case",
  "把这个 bug 记到 notes", "记录这个问题", "归档这次调试", "建个 feature case", or when they paste
  a symptom/log and ask to start tracking it. Also use to UPDATE an existing case (append new
  findings/logs/prompts to it) — INCLUDING auto-extracting useful info from the current conversation
  when the user says "把上面对话有用的补进 case", "自动整理这段调试", "分析对话更新 case", "刚才查的都记一下",
  to CLOSE a case (mark done, move to done/, commit+push) which also
  produces a polished summary.md, or to DIGEST several finished cases into a knowledge-base / 复盘
  doc — phrases like "总结成 md", "出个调试总结", "把这些 case 汇总", "生成知识库/复盘文档". This skill auto-detects the current project/SDK, gathers complete reproducible info
  (SoC/board, SDK baseline, kernel commit, relevant files, repro command), writes a filled-in
  case.md from the bundled template, registers it in the data repo's index, and reminds about
  commit back-links. Prefer this over hand-writing notes files so nothing important (especially env)
  gets skipped. The case is split into multiple files (case.md overview + trace.md + references.md + logs/)
  so no single file gets bloated. Portable: no hard-coded paths or SDK names — works for any teammate who sets DEVNOTES_DIR.
---

# notes-case — 把一个 case 完整落到 notes 数据仓

帮用户把一个新 bug / 功能调试归档到 **case 数据仓**(由环境变量 `DEVNOTES_DIR` 指向),或收尾一个已完成的 case。本 skill 可移植:不写死任何目录/SDK 名,任何同事设好 `DEVNOTES_DIR` 即可用。

核心原则:**能自动探测的信息自己跑命令填,填不出的才问用户**。env 段(尤其 SoC/板型、SDK 基线)缺了就无法复现、也分不清属于哪块板——所以 env 必须填全才算完成。

## 推送授权(硬规则)

> **不要在未经用户当次明确同意的前提下执行 `git push`**。case 数据仓含硬件细节,推错地方/推错时机都不可逆。

- **`git commit` 可以自动做**(新建/更新/收尾/汇总流程里写的提交,默认就 commit,无需问)——本地提交是安全的,即使写错也能 amend/reset。
- **`git push` 必须每次显式问一次**:commit 完后用一句话报告"本地已提交 \<sha\>,要不要 push?",等用户回答 yes 才执行。哪怕用户上一次 push 同意了,这一次也要再问——单次同意不延续到下次。
- 例外:用户**本轮消息里**就明说"提交并推送"/"commit and push"/"推上去"等等价指令时,可一次性 commit+push 不再二次确认。
- 不要写 `git push 2>/dev/null || ...` 这种"静默尝试推一下"的兜底,因为它绕过了用户决策。

写命令时模式固定如下:

```bash
cd "$DEVNOTES_DIR" && git add -A && git commit -q -m "<msg>"
git log --oneline -1   # 报告 sha 给用户,然后等用户决定 push
```

## 关键路径(全部动态解析,不要硬编码)

- **case 数据仓根**: 环境变量 `$DEVNOTES_DIR`。第一步就检查它存在:
  ```bash
  : "${DEVNOTES_DIR:?未设置 DEVNOTES_DIR——请先 export DEVNOTES_DIR=<你的 case 数据仓路径>}"
  ```
  若未设置,停下来提示用户在 `~/.bashrc` 里加 `export DEVNOTES_DIR=...` 并 source,别瞎猜路径。
- **模板目录**: `~/.claude/skills/notes-case/templates/`(含 case.md / trace.md / references.md / summary.md / index.md / digest.md)。
- **当前工程/SDK 名**: 新建/更新时**由用户手动指定**(见第 1 步,探测值仅作建议)。
- **case 落点**: `$DEVNOTES_DIR/<sdk>/active/<目录名>/`,内部按功能拆成多文件:
  - `case.md` — 概览:元信息/env/hw/现象/prompts/findings/decisions/patch/verify/next
  - `trace.md` — 调试过程:逐轮 `t<N>` 改动 + 对应日志引用
  - `references.md` — 参考资料链接
  - `logs/` — 原始日志/diff/截图(按 `t<N>-…` 命名)
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

### 2. 向用户收集填不出来的信息(问答式)

**以问答形式收集**:把下面探测不到的字段一次性列成一张简短问题清单抛给用户(标清哪些必填),让用户一条条补——用户可以一次答完,也可以分几轮补。**env 段不全(尤其 SoC/板型、SDK 基线、复现命令)不要写文件**。问的时候别一个字段一条消息地挤牙膏,合并成一组问完;但也别问已经能探测到或用户已说过的:

- **类型**: bug 还是 feature
- **现象 / 目标**(必填): bug 写预期 vs 实际 + **复现概率/触发条件**(间歇性 bug 尤其要问"多大概率、什么条件触发");feature 写要实现什么 + 验收标准
- **SoC / 板型 + SDK 基线**(必填,探测不到才问)
- **相关 DTS / 源文件**(嵌入式 bug 几乎必填)
- **复现 / 构建命令**(必填): 工程根有 `build.sh`(Rockchip 风格 SDK)时,**编译在 SDK 根目录进行**——先 `cd <SDK根> && ./build.sh <defconfig>` 选板型,再 `./build.sh` 全量编译(或 `./build.sh kernel`/`uboot` 等增量);其它 SDK 按其构建方式填
- **给 agent 的有效指令**: 尤其那条约束 agent 别乱改的范围限定 prompt——最值得存
- 硬件 bug 再问: 原理图页码、关键网络/引脚、外设型号(填进 hw 段)
- 已有的 findings / 决策 / 改动落点: 有就填,没有留空待补

用户贴的大段日志别塞进正文,留到第 4 步存附件。

### 3. 生成目录并写 case 文件组

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

读模板目录的三个文件作骨架,**把收集到的信息真正填进对应文件**(不是原样拷贝):
- `case.md`(模板 templates/case.md)— 填 env/hw/现象/prompts/findings/decisions/patch/verify/next;顶部"关联文件"链接保留
- `trace.md`(模板 templates/trace.md)— 调试刚开始可只留骨架,第一份日志到了再写 `t1`
- `references.md`(模板 templates/references.md)— 有参考就填,没有留骨架

日期用 currentDate 上下文里的今天,别取系统时间。状态 `active`,复现状态按现象填。没拿到的字段保留模板占位提示。

### 4. 存日志 + 记一轮 trace

大段 dmesg / 串口 / ADB 日志 / 完整 diff / 截图 → 写进 `logs/`,**按轮次命名** `t<N>-<标识>-<类型>.txt`;同时在 `trace.md` 加一条 `t<N> 改了X → 结果Y → 日志 logs/t<N>-...`,把改动和日志绑定。原始大段日志只进 logs/,trace.md 里只留结论。

### 5. 登记 index.md

在 `$DEVNOTES_DIR/index.md` 表格**最后一个数据行之后**追加一行(不要依赖表尾空行——它可能已被删):

```
| [<目录名>](<sdk>/active/<目录名>/case.md) | <SoC> | <板型/SDK> | bug/feature | active | <kernel@commit> | <一句话现状> |
```
若 `index.md` 不存在(新数据仓),用模板 `templates/index.md` 建。

### 6. 提醒回链(通用坑)

很多 SDK 顶层是 repo 工具聚合、非单一 git 仓。告诉用户:改动提交时,在对应子仓库(kernel 等)的 commit message 写一行 `Refs notes/<目录名>`,以后 `cd kernel && git log --grep="notes/<目录名>"` 能找回。

### 7. 提交备份(避免误删丢失)

新建完就本地提交一次,别等到收尾:
```bash
cd "$DEVNOTES_DIR" && git add -A && git commit -q -m "<sdk>: new <目录名> — <一句话现状>"
git log --oneline -1
```
报告 commit sha 给用户,**然后问一句"要不要 push?"——按推送授权硬规则,push 必须显式同意才执行**。

## 更新现有 case(边查边补)

调试是持续的——中途有了新 findings、新日志、有效 prompt,随时往已有 case 追加。用户说"把这个补到 case""更新那个 xxx 的 case""这条加到 findings"时:

1. **定位 case**: 扫 `$DEVNOTES_DIR/$SDK/active/` 与 `index.md`;不唯一就列出让用户选,别猜。
2. **只更新相关段/文件,不重写整个文件**:
   - 新结论 → `case.md` findings 段;关键决策 → decisions 段;改动落点/commit → patch 段
   - 有效的范围限定 prompt → `case.md` prompts 段
   - 试过的命令/实验过程 → `trace.md`,**用轮次号 `t<N>` 起头**
   - 参考链接(datasheet 页/内核文档/相似 commit/issue/社区帖)→ `references.md`
   - 中途待办/待验证假设/当前卡点 → `case.md` next 段(多轮接续全靠它,下次开 agent 先看这里)
   - **持续抓到的日志(串口/ADB/dmesg)→ 关键步骤**:先确认它对应哪次改动(接着上一轮就 `t<N+1>`),按 `logs/t<N>-<标识>-<类型>.txt` 存,并在 `trace.md` 加/补一条 `t<N> 改了X → 结果Y → 日志 logs/t<N>-...`,做到**改动与日志一一对应**;原始大段日志只进 logs/,trace.md 里只留结论
   - 复现状态有变(偶发→稳定复现/已解决)就同步 `case.md` 顶部字段
3. 若进展改变了 index 的"一句话现状",同步更新 index 那一行。
4. **提交备份**: `cd "$DEVNOTES_DIR" && git add -A && git commit -q -m "<sdk>: update <目录名> — <补了什么>"`,然后报告 sha 给用户,**按推送授权硬规则**问要不要 push,别直接 push。

### 自动从当前对话提取(无需用户逐条口述)

用户说"**把上面对话有用的补进 case**""自动整理这段调试""分析对话更新 case""刚才查的都记一下"时——不要等用户一条条喂,**自己回看本轮对话**,把散落的有用信息抽出来归位:

1. **定位 case**(同上;不唯一就问)。
2. **通读对话,按类型抽取**——只抽"对复现/定位/解决有用"的,忽略闲聊和走了弯路又被推翻的内容:
   - **改动**: 改了哪个文件/DTS/配置、改成什么(若有 commit sha 一并记)→ `case.md` patch 段
   - **结论/根因线索**: 量到的值、确认的因果、排除的可能 → `case.md` findings 段
   - **决策**: "选了 A 不选 B + 原因" → `case.md` decisions 段
   - **有效 prompt**: 那条管用的范围限定/复现指令 → `case.md` prompts 段
   - **贴进对话的日志**(dmesg/串口/ADB/diff): 存 `logs/t<N>-<标识>-<类型>.txt`,并在 `trace.md` 加一条 `t<N> 改了X → 结果Y → 日志 logs/t<N>-...`,把**改动和它对应的那段日志绑定**(这是自动提取最容易漏的——务必判断每段日志是哪次改动后抓的)
   - **待办/卡点/下一步要试的**: → `case.md` next 段
   - **参考链接**(datasheet/文档/相似 commit): → `references.md`
3. **轮次推断**: 对话里若已有 t<N>,接着往后编号;一段对话里出现多次"改→抓日志"就拆成多轮 t<N>、t<N+1>,别揉成一条。
4. **先给清单、再写盘**: 写文件前,用一张**表格**告诉用户"我从对话里提取到这些 → 准备写到哪",每行一条,列至少含 `内容摘要 | 类型(改动/结论/假设/日志/...) | 目标文件:段 | 轮次号`。表格比纯文本一行清单更密、好扫读。拿不准是不是该记的(可能是临时尝试)就在表格里标 "?" 列出来问一句,别擅自当结论写进去。
5. **口述结论但没贴原始日志怎么办**:用户只说"日志显示 X"而没贴出来时——**不要凭印象编造 `logs/t<N>-…txt` 文件**。做法是:
   - `trace.md` 该轮 t\<N\> 末尾写"日志 `logs/t<N>-<标识>-<类型>.txt`(待补)",保留引用占位
   - `case.md` findings 段照写口述结论,但**只把可推断的事实**写上(如"PLL lock 成功"是事实);**用户用"可能/也许/估计"等措辞的留给 next 段做"待验证假设"**,不要升格为 findings
   - 在给用户的预清单表格里**显式列一行**"t\<N\> 原始日志未贴,logs/ 仅占位"——让用户决定是否回去补,而不是悄悄略过
6. **区分事实与假设**(每次提取都要做的判断):
   - 用户陈述句("PLL lock 成功""换显示器仍黑")→ `findings`
   - 用户带"可能/也许/或许/感觉是/可能是 vop 时序问题"等不确定措辞 → `next` 的"待验证假设",**附"由用户口头提出"标注**,不要升格为 findings
   - 用户的下一步打算("接下来想试 X")→ `next` 的"接下来要试"
7. 确认/默许后按上面的路由写入各文件,同步 index 现状行,`git commit`(本地)——然后按推送授权硬规则问用户要不要 push。

> 自动提取≠瞎猜:提取的是用户在对话里**已经说过/做过**的事,只是替他归纳归档;凡是对话里没有依据的字段(比如根因还没定),保持留空,不要替他下结论。



## 收尾一个 case(标记完成 + 生成总结 + 备份)

用户说"修好了 / 归档 / 标记完成 / 总结一下"时:

1. 确认 case.md 的 verify 段已填(实测步骤+结果+残留边界)、patch 段有 commit sha,复现状态改 `已解决`。缺了先补问。
2. **生成调试总结 `summary.md`**:读模板 `~/.claude/skills/notes-case/templates/summary.md`,从 `case.md` + `trace.md` 提炼写进 case 目录的 `summary.md`——TL;DR + 问题 + 根因 + 解决方案 + 验证 + 避坑经验。这是面向复盘/分享的成品:**连贯成文、去掉占位**,既讲清"怎么解决的"(给人复用)又留下"避坑教训"(给人省时)。
3. 移动并按年月归档(年月用今天):
   ```bash
   mkdir -p "$DEVNOTES_DIR/$SDK/done/<YYYY-MM>"
   mv "$DEVNOTES_DIR/$SDK/active/<目录名>" "$DEVNOTES_DIR/$SDK/done/<YYYY-MM>/"
   ```
4. 更新 index.md 该行:状态 `active → done`,路径前缀改 `done/<YYYY-MM>/`,结论改最终结论。
5. 本地提交(数据仓含硬件细节,保持私有):
   ```bash
   cd "$DEVNOTES_DIR" && git add -A && git commit -m "<sdk>: done <目录名> — <一句话结论>"
   git log --oneline -1
   ```
   报告 sha,**按推送授权硬规则问用户要不要 push**——收尾这一步尤其要确认,因为是阶段性归档。

## 汇总多个 case 成知识库 / 复盘文档

用户说"把这些 case 汇总""生成知识库""出个复盘文档"时:

1. **定范围**: 按 SDK、主题(如"以太网""HDMI")、或时间段筛选——扫 `index.md` 和各 `done/` 下的 case;不明确就问用户要哪一批。
2. **取材**: 优先读每个 case 的 `summary.md`(没有就读 case.md 提炼)。
3. **生成汇总**: 读模板 `templates/digest.md`,写到 `$DEVNOTES_DIR/digests/<主题或日期>.md`(目录没有就建)。结构(模板已含):
   - 顶部一个目录/索引表(case 名 → 一句话结论 → 链接)
   - 每个 case 一小节:问题 / 根因 / 方案(带 commit)/ 避坑要点,末尾链接回原 case
   - 若多个 case 有共性(同一子系统反复出问题),单开一段"共性与规律"——这是知识库比单篇总结更值钱的地方
4. 本地提交: `cd "$DEVNOTES_DIR" && git add -A && git commit -m "digest: <范围> 汇总"`,报告 sha,**按推送授权硬规则问用户要不要 push**。

## 注意

- case 数据仓是**私有**的(含原理图/硬件细节),不要往公开仓推。模板/skill 本身不含敏感信息,放共享仓无妨。
- 不要污染各 SDK 自带的官方 `docs/`,个人记录只进数据仓。
- 信息不足时**先问再写**,宁可多问一句,也别写出无法复现的残缺 case。

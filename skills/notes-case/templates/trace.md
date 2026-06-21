# 调试过程 / 实验 — <case 标题>

> 每次"改动 → 烧录 → 抓日志"配成一条,用轮次号 `t<N>` 串起来:日志按 `t<N>` 存进 `logs/`,这里引用,做到**哪次改动对应哪个 log 可追溯**。
> 日志命名: `logs/t<N>-<简短标识>-<类型>.txt`(如 `t2-reset10ms-dmesg.txt`)。原始大段日志只进 logs/,这里每条只留结论。
>
> 关联: [case.md](case.md) · [references.md](references.md)

- **t1** 改动 `<baseline / 命令 / 改动>` → 结果 <…> → 日志 `logs/t1-<标识>-dmesg.txt`
- **t2** 改动 `<…>` → 结果 <…> → 日志 `logs/t2-<标识>-uart.txt`

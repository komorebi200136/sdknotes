# <case 标题：一句话说清现象或目标>

> 类型: bug | feature   ·   状态: active | done
> 复现状态: 稳定复现 | 偶发 | 已解决 | 无法复现
> 创建: YYYY-MM-DD

## 环境 (env)
> 多人 / 多 SDK 共用同一套模板,所以 SoC/板型 与 SDK 基线必填——否则分不清这条 case 属于谁、哪块板。
> 嵌入式调试缺了 env 就无法复现。

- SoC / 板型: <如 RK3576 / K7、RK3568 / 某板>
- SDK 基线: <版本号或 release tag>（构建配置: <如 defconfig 名>）
- rootfs 后端: <debian | buildroot | yocto | ubuntu | 其它>
- 内核 commit: <cd kernel* && git rev-parse --short HEAD>
- 相关子仓库 & commit: <kernel @ xxxx / u-boot @ xxxx / ...>
- 相关文件: <如 arch/arm64/boot/dts/.../xxx.dtsi 的某节点>
- 复现/构建命令:
  ```bash
  <你的构建/烧录/复现命令>
  ```

## 硬件信息 (hw)
> 硬件相关 bug 才填;能省下反复翻原理图的时间。

- 原理图: <文件名 第 X 页 / 模块>
- 关键网络/引脚: <网络名 / GPIO / 总线>
- 外设型号: <相关器件、屏、转接头等>

## 目标 / 现象
> bug: 预期 vs 实际 + 复现概率(间歇性 bug 必写,如"约 1/5 冷启动")+ 关键日志。
> feature: 想实现什么 + 验收标准。

- 预期:
- 实际:
- 复现概率/触发条件:
- 关键日志:
  ```
  ```

## 给 agent 的有效指令 (prompts)
> 只记"有效且可复用"的;尤其那条约束 agent 不乱改的范围限定。

- 任务句:
- 范围限定:
- 失败的尝试:

## 排查 / 设计 (findings)
> bug: 排除链路 + 根因 + 关键变量/日志含义。
> feature: 调用链、依赖、设计要点。

- 排除: 不是 X（证据 …）
- 定位: 根因在 Y（文件:行 + 为什么）
- agent 一度错判在:

## 调试过程 / 实验 (trace)
> 过程性记录,区别于上面 findings 的结论:试过哪些命令/手段、各自结果。多轮调试按时间/轮次累加,方便回看走过的路。

- <日期/轮次> 试了 `<命令/改动>` → 结果 <…>

## 决策 (decisions)
- DECIDE: 选 A 而非 B
  - 理由:
  - 风险:
  - 回退条件:

## 改动 (patch)
- 落在子仓库: <kernel / u-boot / device / app / ...>
- commit sha: <commit message 写 "Refs notes/<本case目录名>">
- diff 摘要 / PR:

## 验证 (verify)
- 实测步骤与结果:
- 残留边界 / 没覆盖的情况:
- 待补单测 / 待加监控:

## 下一步 / 待办 (next)
> 多轮/多天调试的**接续点**——下次开 agent 先看这里。修复完成后清空或转入 verify。

- [ ] 接下来要试:
- [ ] 待验证的假设:
- 上次查到哪 / 当前卡点:

## 参考资料 (references)
> datasheet 页码、内核文档、相似 commit/issue、社区帖等,边查边记。

- <标题/说明> — <链接或路径#页码>

## 附件
> 大段串口日志、截图、完整 diff 放本目录的 logs/ 子目录。

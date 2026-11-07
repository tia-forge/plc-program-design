# PLC 程序设计五步 · PLC Program Design (5 stages)

**一个只讲方法的技能（knowledge-only skill）：把一句口头需求，走成能验收的 PLC 程序。**

Most "AI + PLC" content stops at "here is an SCL block". This skill is about the part that actually fails projects: the requirement is never written down as something checkable, the IO table has no spare capacity, one FB quietly drives two devices, and the state machine has no timeouts. Five stages, five deliverable artefacts, four gates — nothing gets to skip ahead.

| 步 | 交付物 | 过关判据 |
|---|---|---|
| 0 需求 | 动作矩阵（谁/何时动/多久/算完成/出错怎么办/谁能复位 + 模式矩阵） | 每行动作能指到一个传感器和一个执行器 |
| 1 IO 设计 | 点位表（地址、符号、类型、量程、正常态、所属模块） | 分段无冲突、余量 ≥20%、安全回路不在表内 |
| 2 模块拆分 | 模块清单 + 接口表（`i_`/`q_`/`stat_`）+ 块号规划 | 一设备一 FB，工艺层不直接写 Q |
| 3 实现 | SCL 源 + 接口读回 | 块尾统一赋值、每步超时、故障一条复位路径 |
| 4 验证 | 编译原文 + 用例表 PASS/FAIL + 回归子集 | 编译 → 一致性 → 用例 → 变更回归，顺序不可换 |

Written against Siemens **TIA Portal / SCL**, but the structure applies to any IEC 61131-3 controller.

## 装

```bash
npx skills add tia-forge/plc-program-design      # skills.sh 兼容的客户端
```

或直接把 [`SKILL.md`](./SKILL.md) 放进你的 agent 的 skills 目录（Claude Code、Cursor、pi、WorkBuddy 都认 `SKILL.md` + frontmatter）。

## 里面有什么

| 文件 | 内容 |
|---|---|
| [`SKILL.md`](./SKILL.md) | 五步全文：交付物模板、反模式表、用例表、三套提示词模板、安全边界 |
| [`examples/conveyor-sort.md`](./examples/conveyor-sort.md) | 完整示例：分拣工位的 SCL 外部源 + 9 条仿真用例（真机跑过：编译 0 错 0 警、下载 0 错、用例全 PASS） |

## 它不是什么

- 不是博途插件，不含任何可执行程序；不装博途也能用完第 0–4 步（纸面 + 云仿真）。
- 不保证"设计是对的"，只保证"程序按设计工作"。设计与验收要人签字；急停、安全门、光幕这类安全功能由安全继电器或安全 PLC 实现，不进 AI 生成物。
- 与 Siemens AG 无隶属或背书关系；TIA Portal、SIMATIC、S7、PLCSIM 是其商标。

## 落到真实工程

第 3 步"落进博途工程"和第 4 步"下载到 PLCSIM 跑用例"需要一台装了博途的 Windows 机器和一个网关（[TiaLink](https://tia.hubwiz.com/tia-link)，商业软件，未授权只读）。没网关就退成云仿真跑用例，方法完全不变。

同系列（可选，非必需）：`tialink-tia-portal`（在真实工程里编译/下载/读回）、`plc-cloud-sim`（不装博途跑用例）、`tia-portal-docs`（指令参数与版本行为）、`laddoc-cloud`（梯形图交付）、`lad-render-cloud`（导入前看图）。

## 许可

文档 CC BY 4.0。转载与二次分发请保留出处 <https://github.com/tia-forge/plc-program-design> 与文末商标声明。

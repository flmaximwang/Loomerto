---
name: loomerto-plan
description: "Use when 与人/其他 agent 协同时要维护一份共享 plan。用 loomerto CLI（loomerto --plans-root <库> <子命令> <slug>）改状态、渲染三视图、发提醒；plan.json 是唯一真相，只经 CLI 改。上下文变长（跨 3+ 回合 / 要向接手的人解释背景 / 多条线并行）时先把后续规划、子代理分配与执行落到 plan 上。"
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [plan, collaboration, multi-agent, loomerto, visualization]
    category: loomerto
    related_skills: [loomerto-remind, handle-a-recurring-progress-instruction, agent-to-agent-handoff]
---

# 维护一份共享 plan（协作计划记录员的核心动作）

## When to Use

- 与用户**或另一个 agent** 协同做一件事，并且这件事会跨多轮 / 多天 / 多个人。
- 有人问「现在到哪了」「下一步谁做」——这时答案应当来自 plan，而不是来自你的记忆。
- 你要把一段对话的产出固化下来，让下一个接手的人（或 agent）不必读聊天记录。

不适用：一次性能在一个回合里做完的事；只需要一句待办的时候。

## 硬规则 0（先看这条）：改 plan 只走 `loomerto` 命令

**不许自己写脚本改 plan。** 三不许：

1. **不许写 python 去操作 plan** —— 没有 `python3 -c "import loomerto…"`、没有 heredoc、没有新写的 `.py`。
   改状态 / 建块 / 接线 / 登记线程 / 渲染，CLI 全都有命令：敲 `loomerto …`。
2. **不许直接编辑 `plan.json`** —— 它是 JSON，手改看着能跑，但会漏掉**三视图同步**（`PLAN.md` /
   `plan.html` / `plan.canvas` 停在旧内容）、**事件日志**（后人查不到谁在什么时候改了什么）、以及
   **打回 / 收工自动清线程登记**这些语义。唯一写入漏斗是包里的 `edits` + `store.commit()`，
   `loomerto` 是它唯一的对外入口。
3. **不许 `import loomerto` 去写数据** —— 包的核心 API 是给**别的程序**用的（画布服务 `serve.py` 这类），
   不是给你绕开 CLI 用的。验证一份包能不能 import 可以，**改 plan 一律发命令**。

**CLI 现在做不到的动作 → 报给用户，别绕道写脚本**（例：`task` 只建不改 —— 改泳道标题 / 重排泳道顺序
没有命令）。把缺口说清楚（要什么动作、代价多大）交给用户拍板；正路是**给包加一条命令**
（见 `maintain-the-loomerto-package`），不是手改 `plan.json`。

**常用功能没有命令时，派一个子代理去本仓库提 issue（用户 2026-10-11 要求）**：一发现自己反复要用的动作
没有 CLI（改任务标题、改 plan 级 `goal`、改块标题这类），就 `delegate_task` 一个子代理，让它在
`flmaximwang/Loomerto` 上开一条 issue，内容三样：**想要的命令形状**（`loomerto <组> <动词> <参数>`）·
**现在的绕道代价**（手改 `plan.json` 会丢三视图/事件日志，或整份重建）· **复现命令**。issue 号带回来，
写进当前 plan 的一条 `note --kind decision`。子代理只跑前台命令、跑完即结束。
**提 issue 不替代报给用户**：缺口仍要在汇报里说一句，只是不必停在「工具做不到」上。

## 会话一长，先把它落到 plan 上 → 见 `references/plan-extra-会话一长-先把它落到-plan-上.md`
## 唯一真相与目录

```
<profile>/workspace/plans/<slug>/
├── plan.json     机器模型 = 唯一真相（任务图 + 块文档 + 事件日志）
├── PLAN.md       人读摘要（含 mermaid 图），自动生成
├── plan.html     自包含离线可视化（泳道 = 任务），自动生成
└── plan.canvas   Obsidian JSON Canvas 视图，自动生成
```

`plan.json` **只通过 `loomerto` 命令改**；`PLAN.md / plan.html / plan.canvas` **永远不要手改**
（下一次 `loomerto` 落盘就会覆盖）。唯一入口 = **装好的 `loomerto` 命令**
（`uv tool install --editable <repo>` 装出来的；本机在 `~/.local/bin/loomerto`）。
**一份 plan 在哪，永远由调用方说清**（包不认任何 harness 的目录，也没有 profile 这个概念）：

```bash
loomerto --plans-root <目录> <子命令> <slug> …      # 一份库里有好几份时才用
loomerto --plan <plan 数据文件> <子命令> …          # 只认这一份；此后命令里**不写 slug**
```

定位旗标必须写在**子命令之前**。本 profile 的 plans 根 = `<profile home>/workspace/plans`
（default = `~/.hermes/workspace/plans`，其余 profile = `~/.hermes/profiles/<名字>/workspace/plans`）；
在 plan 自己的目录里跑（那里有 `plan.json`）可以省掉旗标。

**项目开发时，plan 必须持久化在项目里**（用户 2026-10-11 硬要求，原话「在项目开发时，loomerto 计划必须
持久化存储到项目中」）：只要这份计划服务的是**某个项目/仓库的开发**，plan 就住在那个项目里，而不是 profile 的
`workspace/plans` —— `loomerto --plans-root <项目>/plans …` 建出 `<项目>/plans/<slug>/`，并随项目一起提交推送
（`git add plans/`）。判据：那个提交的 `git show --stat` 里能看到 `plans/<slug>/plan.json` 与 `plan.html`；
给这个用户的看板 URL 也就顺着变成 `file://<项目>/plans/<slug>/plan.html`。
项目还不存在时先把它建出来（`mkdir -p <项目>/plans && git init`）再建 plan —— 计划是项目的第一件产物，
不是等脚手架齐了才补的记录。profile 的 `workspace/plans` 留给「不属于任何项目」的协作计划。

**实现在仓库根的 loomerto 包里**（纯 stdlib、零依赖）：model（模型与派生规则，纯函数）/ store（磁盘 +
**唯一写入漏斗** `commit()`）/ render（三视图）/ workers（线程探活）/ cli（唯一 print 与退出码）。
这个 skill 自带的 `scripts/plan.py` 是**旧写法**的薄壳（它替你交代 plans 根再调同一个 CLI，与
`loomerto` 等价；脚本位置见 `skill_view` 给的 `skill_dir`）—— **新写的命令一律直接用 `loomerto`**。
包没装的话先装：`uv tool install --editable <repo>`。

## 模型 → 见 `references/plan-model.md`
## 状态表（图例顺序 = 一个块的一生）

**流程**：待批准? → 等前置 → 待认领 → 已认领 → 进行中 → 待评审 → 已完成；旁支只有 `已取消`。
**图例按这个顺序排**（模板里的 `C`/`ZH` 两个对象的键序即图例序，改顺序就是改那两个对象的键序）。

| 中文名 | 内部值 | 是什么 | 谁该动 |
|---|---|---|---|
| 待批准 | `blocked` | **动不了，等 agent 以外的人/外部条件**。两种进入方式：建块时就置 `--status blocked`（计划内的关卡），或开工后卡住。解锁 = `set <块> pending`（或直接回 `running`），之后按依赖自动变 等前置/待认领 | 被点名的人 |
| 等前置 | `waiting`（派生） | 依赖还没完成，轮不到它（**owner 可以先派好**，轮到他就接手） | — |
| 待认领 | `ready`（派生） | 依赖全 done、**还没人接**（有 owner 的块不是待认领，是已认领） | 谁有空谁接；`loomerto current` 列的就是它 |
| 已认领 | `claimed` | 有人认领了（owner 已定）、还没开干 —— 存储的 `claimed`，或 `pending` + 依赖已就绪 + **有 `owner`**（派生）。**评审打回也是回到这里**：块没换人、也没变新块，只是重做一遍，打回原因进 `feedback`、次数显示成 `⟲N` | 认领人（作用是占位：两个人别抢同一个块） |
| 进行中 | `running` | 正在做 | 认领人 |
| 待评审 | `review` | 做完了、已送审，等别人给结论（送审 = 把做块置 `review`） | 评审人（下游的评审块） |
| 已完成 | `done` | 判据满足、通过 | — |
| 已取消 | `cancelled` | 这块不做了；不计入进度分母、不画依赖边。**图上与「等前置」同色**，靠虚线左框 + 图例空心方框区分（不靠颜色） | — |

**状态 × 负责人是一张表**（`model.STATUS_OWNER`，三档：`never` 不能有 / `required` 必须有 / `may` 可有）：

| 档 | 哪些状态 | 为什么 |
|---|---|---|
| 不能有 | `待认领`（`ready`） | 它的定义就是「还没人接」；有负责人 ⇒ 显示成 `已认领`（不是同一个状态） |
| 必须有 | `已认领` / `进行中` / `待评审` | 「有主」是这三个状态的一部分（谁接的 / 谁在做 / 等谁给结论） |
| 可有 | `待批准` / `等前置` / `已完成` / `已取消` | 待批准 = 等**这个人**点头；等前置 = 可以先派好；已完成 = 留着「谁做的」 |

`不能有` 那档由**派生**兑现（不必人手工同步）；`必须有` 那档由 `check` 发 ⚠ 盯着（硬校验属 R-04）。
**任务是泳道**，它的 `owner` 是「谁负责这条线」，不归这张表管。

**没有「待返工」这个状态**：打回不是"换个人接手"，默认就是原 owner 重做一遍 —— 那正是「已认领」的
语义（有人在手、还没开干）。所以打回 = `set <块> claimed --note "<为什么打回>"`，块回到已认领，
`--note` 落进 `feedback`。状态只有"谁在做/做完没有"，原因和往返次数是属性，不是状态。

**「待批准」只有一个状态**（内部 `blocked`）：无论是"开工前计划好的关卡"还是"开工后卡住"，都是同一
件事 —— 这块离了某个人/某个外部条件就动不了，表现和处置完全一样。区别写在 `--note` 里，不要再拆出
第二个状态（曾经试过 `needs_approval`，两个状态行为相同、只是逼记录员多选一次，已合并）。
**新建块默认就是「待批准」**（`block` 的 `--status` 默认 `blocked`）—— 由 AI 判断这块无需审批就能干时，
才显式写 `--status pending` 放行；`expand --step` 追加的步骤不在默认之列（它是已批准那条活的后续步骤）。
`pending`（待排）不进图例 —— 图上有依赖时它会显示成「等前置」，依赖全就绪后按有没有负责人显示成
「待认领 / 已认领」。

**评审不是回边，返工才是那个 loop —— 但它长在块的「状态」上。** 评审是下游的**独立块**（`kind=review`
+ `--review-of <被审块>`），由评审人持有；被审块送审后停在「待评审」等结论。通过 → 被审块 `done`；
打回 → 被审块回 `claimed`（原 owner，带 `feedback`）。

重做的环是：待评审 →（打回）已认领 →（重做）进行中 →（再送审）待评审。`plan.html` 在节点状态后用
`⟲N` 标出被打回次数（数该块 runs 里"从 `review` 走出去且不是 `done`"的次数）。**别把它画成块之间的
回边**：块自己没换、owner 也没换，画边是范畴错误；而且依赖图一旦有环，`loomerto check` 会报错，
"谁在等谁"（深度/拓扑）也没法算。返工不会波及下游 —— 打回发生在做块 `done` 之前，下游一直卡在
「等前置」，这也正是把评审卡在 done 之前的意义。

## 粒度调整：块（B）⇄ 任务（T）与定点插入 → 见 `references/plan-extra-粒度调整-块-B-任务-T-与定点插入.md`
## 谁认领了 / 谁在做 / 线程在哪 → 见 `references/plan-extra-谁认领了-谁在做-线程在哪.md`
## 一次协作回合的固定动作

> 下面写的 `loomerto <子命令>` 都要带定位旗标，完整形态是
> `loomerto --plans-root <本 profile 的 workspace/plans> <子命令> <slug>`（在 plan 目录里跑可省；
> `--plan <数据文件>` 模式则命令里不写 slug）。

1. **读**：`loomerto current <slug>` —— 现在能动的块；先看这个再说话。
2. **总结**：`loomerto note <slug> "<这一段发生了什么>" --actor <谁>`
   —— 只写真正发生的；拿不准的写成「待确认」。
3. **改状态**：`loomerto block set_status <slug> T-002#B-002 running --by <谁> --note "<一句话>"`
   - 认领 → `claimed`；开干 → `running`；送审 → `review`；通过 → `done`；打回 → `claimed`
     （`--note` 会存成 `feedback`；打回默认就是原 owner 重做，所以不换人、不建新块）。
   - 参数按状态卡：线程登记（`--delegation` / `--task-index` / `--transcript`）只在 `claimed`/`running`/`review`
     收，`--artifact` 只在 `done` 收 —— 给错状态会退 2 并列出该状态收什么。
   - 补记过去的时间用 `--at <ISO8601>`，不要假装是现在。
4. **验证**：`loomerto check <slug>` —— 环 / 悬空依赖 / 已认领却没写负责人 / 缺判据 / 悬置超时（悬置只数**存储**状态在途的块）。
   **有错误就别往下走**；告警要念给用户听。
5. **登记线程**（把块派给子代理时）：`loomerto block set_status <slug> <ref> running --by <谁> --delegation <deleg_id>
   [--task-index N] [--transcript <路径>]` —— 之后随时 `loomerto workers <slug>` 就能看出这条线程是不是还在动
   （`⚠ 线程已结束` / `❌ 号记错` 退 1：先对账再往下走）。收工或换人时 `block set_status … --unset`
   （`set … done` 也会自动清）。
6. **提醒**：`loomerto digest <slug> --to <参与方>`，纪律见 skill `loomerto-remind`。
7. **交付前两条校验**（送审 / 交接 / 收尾时跑，不是每次改状态都跑）：
   - `loomerto-check-commands`：每个块有没有可直接执行的命令、变量有没有定义 —— 缺则**不批准**（exit 1）。
   - `loomerto-check-temps`：这份 plan 会不会留下没人清的临时文件 —— `❌ 不闭环` 时按它打印的
     `loomerto task add` / `loomerto block add` / `loomerto block set_status` 命令补一个收尾任务节点与「临时文件：…」声明。
   改完重跑；两条都要 `exit 0` 才往下走。

改状态时 `loomerto` 会自动重渲染三个视图（`--no-render` 可跳过）。

## 踩过的坑 → 见 `references/plan-scars.md`
## 命令速查 → 见 `references/plan-command-reference.md`
## 可编辑的画布（`loomerto open`） → 见 `references/plan-extra-可编辑的画布-loomerto-open-.md`
## 交付给人的默认包（默认就给，不用等他要） → 见 `references/plan-extra-交付给人的默认包-默认就给-不用等他要-.md`
## 几条硬规则 → 见 `references/plan-hard-rules.md`
## 坑 → 见 `references/plan-pitfalls-a.md` 与 `references/plan-pitfalls-b.md`
## Support files

| 文件 | 承担什么 |
|---|---|
| `scripts/plan.py` | skill 侧的**旧写法薄壳**：交代 plans 根 → 找包 → 调 `loomerto.cli.main()`（找不到包时打印装法，退出码 2）。**新写命令请直接用 `loomerto`** |
| loomerto 包（仓库根） | 实现在那里：model（模型/派生）/ store（磁盘 + 唯一写入漏斗 `commit()`）/ render（三视图）/ workers（线程探活）/ cli（唯一 print、唯一退出码）。**改实现去那里，改「怎么用」才改本文件** |
| 兄弟 skill | `loomerto-check-commands`（每个块的命令与变量定义）、`loomerto-check-temps`（临时文件闭环）、`loomerto-remind`（提醒纪律） |

静态图（给聊天/群用，**只在被明确索取时才做**）落在 plan 目录的 `plan.png`；默认交付是 `file://`
看板 URL（**不发附件、不起服务**），生成方法见文末「坑」。

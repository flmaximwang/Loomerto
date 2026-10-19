# Spec：Loomerto v2 —— 统一节点模型 + Electron 桌面画布

> 状态：**已拍板，待实现**（2026-10-19，与用户逐轮对齐 Q1–Q30）。工单拆分见
> [`desktop-canvas-tickets.md`](desktop-canvas-tickets.md)。
> 本文是 v2 的**唯一设计依据**：实现与本文冲突时，先改本文（并找用户确认），再改代码。
> 现有的 [`../REQUIREMENTS.md`](../REQUIREMENTS.md) / [`../docs/architecture.md`](../docs/architecture.md) /
> [`../docs/canvas-sync.md`](../docs/canvas-sync.md) 描述的是 **v1**，M1 落地时按本文重写。

---

## 0. 一页纸

| 维度 | v1（现在） | v2（本文） |
|---|---|---|
| 真相 | `plan.json`（文件名固定） | **`<名字>.json`**（如 `PlanA.json`），仍是唯一真相，**含空间信息** |
| 派生物 | `PLAN.md` + `plan.html` + `plan.canvas` | **只剩 `<名字>.md`**（如 `PlanA.md`）；html / Obsidian canvas 全删 |
| 结构 | 两层：任务（泳道）→ 块 | **一种节点，无限嵌套**：有子节点 = 框（frame），没有 = 卡片（card） |
| id | `T-001#B-002`（位置即身份，移动要换 id） | **随机稳定 id** `n-7f3kq9xa`，永不改变；移动 = 换父节点 |
| 边 | `deps` + `review_of`，全图不许成环 | **`dep`（之后）** 不许成环；**`loop`（打回到）** 可以成环，表示实现-评审循环 |
| 人用的前端 | `loomerto open` 起的单页 html | **Electron 桌面应用**（React + React Flow），一个窗口一份 plan |
| agent 用的前端 | CLI（`task` / `block` 两组） | CLI（**一个 `node` 组**），无别名、无向前兼容 |
| 旧数据 | — | **不迁移**：v1 文件直接拒读，用户按旧 `PLAN.md` 重建 |

不变的东西（硬约束，v2 继续成立）：

1. **Python 核心纯 stdlib、零依赖、`>=3.9`**，任何 harness / agent 不装 Node 也能跑全部命令。
   桌面应用是**给人用的可选前端**，它依赖 Node/Electron，但核心不依赖它。
2. **写只有一条路**：`edits`（唯一改动实现）→ `store.commit()`（唯一落盘漏斗）。CLI 与 HTTP 服务都是薄适配层。
3. **派生一律由 Python 算**（就绪/等待/框的汇总状态/有效依赖/参与方候选），前端不实现第二份。
4. **服务只绑 `127.0.0.1`**，本机单用户。
5. 核心层不 print、不 `sys.exit`，错误只抛 `PlanError(code)`。

---

## 1. 决策记录（Q1–Q30 的结论）

| # | 决策 |
|---|---|
| Q1 | 保留 Python 核心为唯一引擎；桌面端是给人的前端，agent 不需要它 |
| Q2 | 只认 `plan.json`（现改名为 `<名字>.json`）；它存尺寸、坐标等空间信息；`render.py` 的 html / canvas 输出删除 |
| Q3 | 无限平移缩放 + 嵌套；自由便签等「非任务节点」留到以后 |
| Q4/Q18 | 一种递归节点：有子 = 框，无子 = 卡片；拖进框 = 换父；把卡片丢到另一张卡片上 = 目标变成框 |
| Q5 | 画布上画边/删边 = 改依赖（带查环） |
| Q6 | 画布随 `json` 变化**实时刷新**（保留视口与选中） |
| Q7/Q14 | Electron + Electron Forge + Vite + TypeScript + React + React Flow（`@xyflow/react`） |
| Q8/Q17 | 派生只留 `<名字>.md`；plan 库 = 扁平目录里的 `<名字>.json` + `<名字>.md` |
| Q9 | 坐标写在对象自己身上，**相对父框左上角** |
| Q10 | 视口（平移/缩放）**不进** json，由应用本地存 |
| Q11 | 挪位置不许让文件膨胀：只在松手时写、整数、单行数组、不进日志 |
| Q12 | 没坐标的节点（agent 建的）由画布自动排版，首次放置后写回；人摆过的永不自动挪 |
| Q19/Q26 | 稳定随机 id：`n-` + 8 位小写 Crockford base32；CLI 接受唯一前缀 |
| Q20/Q25 | 两种边：`dep` 无环，决定就绪；`loop` 可成环，评审打回时重开循环段；`review_of` 删除 |
| Q21 | 任何节点可有 `owner`；`exec` / `runs` / 送审只在卡片上 |
| Q22/Q23 | 不写迁移、不留别名、不兼容 v1：v1 文件拒读 |
| Q24 | 先重写模型（M1），再做画布 |
| Q27 | 画布 v1 支持增 / 改 / 拖 / 改尺寸 / 画删边 / 删节点；撤销 = **再提交一次反向改动** |
| Q28 | 应用打开**一个 json 文件**（不浏览文件夹）；侧栏 = 最近打开过的 plan（记住各自视口）；一窗一 plan，可多窗 |
| Q29 | 五个里程碑 M1–M5（见 [`desktop-canvas-tickets.md`](desktop-canvas-tickets.md)） |
| Q30 | 本 spec + 工单两份文件，中文 |

---

## 2. 文件与定位

### 2.1 命名

- 一份 plan = 一个数据文件 `<名字>.json` + 同目录的派生文件 `<名字>.md`。名字 = 文件名去掉 `.json`（`stem`）。
  `PlanA.json` → `PlanA.md`；`my-plan.json` → `my-plan.md`。同一目录可以放任意多份 plan，互不覆盖。
- `plan.json` 里**不再存 `slug`**（名字就是文件名，存两份会漂）。`title` 是给人看的标题，可以与文件名不同。
- 原子写照旧（`atomic_write`：同目录临时文件 + `os.replace`）。

### 2.2 定位（`store.plan_path(name)`）

只认显式给的，按顺序：

1. `--plan <文件>` 或 `$LOOMERTO_PLAN_FILE` ⇒ 就是这一份；命令里**不写名字**。给的必须是以 `.json` 结尾的文件路径（给目录 ⇒ 退 2，提示「v2 不再按目录找 plan.json」）。
2. `--plans-root <目录>` 或 `$LOOMERTO_PLANS_ROOT` / `EMBEDDED_PLANS_ROOT` ⇒ `<目录>/<名字>.json`，命令的第一个位置参数是名字。
3. 都没有 ⇒ 退 2，打印该给什么。**v1 的「当前目录有 plan.json 就用它」删除**（不靠猜）。

`list`：`--plan` ⇒ 只列这一份；`--plans-root` ⇒ 列目录下所有 `*.json` 中 `schema == "loomerto/plan@2"` 的（其余 json 静默跳过 —— 库目录里可能有别的 json）。

### 2.3 版本与拒读

- `schema` 固定为 `"loomerto/plan@2"`。
- `store.load()` 读到 `schema` 不是它（包括 v1 的 `"plan-weave/plan@1"`，或含 `tasks` 键）⇒
  `PlanError(code=2)`：「这是 v1 的 plan（tasks/blocks 结构），v2 不读。按它旁边的 PLAN.md 用 `loomerto node add` 重建。」
- **不写任何迁移代码。**

### 2.4 json 的写法（Q11：不膨胀）

`store.save()` 用自定义序列化（仍是 stdlib `json`）：

- 缩进 2，`ensure_ascii=False`，键按 §3 的声明顺序（不排序，便于人读 diff）。
- **只含数字或短字符串的列表写成一行**：判据 = 列表元素全是 `int/float/str`，且整行 `json.dumps(lst)` ≤ 100 字符。
  所以 `"pos": [120, 40, 240, 96]`、`"deps": ["n-7f3kq9xa", "n-p2m8c0de"]`、`"children": [...]`（短时）都是一行。
- 坐标、尺寸一律**取整**（`round()` 后存 int）。
- 结果：拖动一张卡片，git diff 恰好改 `pos` 一行 + `rev` / `updated_at` 两行。

---

## 3. 数据模型 v2

### 3.1 顶层

```json
{
  "schema": "loomerto/plan@2",
  "title": "把 Loomerto 改成桌面应用",
  "goal": "…",
  "status": "active",
  "created_at": "2026-10-19T16:00:00+08:00",
  "updated_at": "2026-10-19T16:05:12+08:00",
  "rev": "a1b2c3d4e5f6",
  "cadence": {"digest_hours": 24, "quiet_hours": ""},
  "participants": [{"id": "you", "kind": "human", "label": "本人", "channel": ""}],
  "roots": ["n-7f3kq9xa", "n-p2m8c0de"],
  "nodes": {
    "n-7f3kq9xa": { …节点… },
    "n-p2m8c0de": { …节点… }
  },
  "log": []
}
```

- **`nodes` 是扁平对象**（id → 节点），**树形由 `roots` 与每个节点的 `children` 表达**（都是有序 id 列表）。
  为什么扁平：移动一个子树只改两个 `children` 列表各一行，不会把整棵子树在文件里搬家（diff 小、不膨胀）；查找 O(1)。
- **节点不存 `parent`**（与 `children` 两份真相会漂）。`model.parent_index(plan)` 在读时派生 `{id: 父 id 或 ""}`。
- `participants` / `cadence` / `log` 的形状与 v1 相同。

### 3.2 节点（`model.NODE_FIELDS`，唯一一份声明）

| 键 | 默认 | 谁有意义 | 说明 |
|---|---|---|---|
| `id` | — | 全部 | `n-xxxxxxxx`，见 §3.4 |
| `title` | — | 全部 | 不能为空 |
| `kind` | `"impl"` | 全部 | `impl` / `review` / `decision` / `research` |
| `status` | `"blocked"` | **卡片** | 存储状态，取值 `NODE_STATUS` = v1 的 `BLOCK_STATUS`：`pending/claimed/running/review/done/blocked/cancelled`。框上这个键**存为 `""`**，读时忽略（框的状态是派生的，§4.3） |
| `owner` | `""` | 全部 | 认领人；框上 = 「这一组的协调人」（Q21） |
| `doc` / `input` / `output` / `command` | `""` | 全部 | 同 v1 |
| `done_when` | `[]` | 全部 | 判据 |
| `artifacts` | `[]` | 卡片 | 只在 `done` 时追加 |
| `deps` | `[]` | 全部 | **dep 边（入边）**：本节点要等的节点 id。见 §4 |
| `loops` | `[]` | 卡片 | **loop 边（出边）**：本节点被打回时重开到哪。见 §5 |
| `max_loops` | `0` | 卡片 | 0 = 不限；>0 = 打回次数达到上限时停住（§5.3） |
| `rework` | `0` | 卡片 | 被打回（重开）过几次 |
| `feedback` | `""` | 卡片 | 最近一次打回的意见 |
| `exec` | `{}` | 卡片 | `{by, started, delegation, task_index, transcript, note}`，同 v1 |
| `status_since` | 当前时间 | 卡片 | |
| `runs` | `[]` | 卡片（框上保留历史，不再追加） | `[{at, by, from, to, note}]` |
| `children` | `[]` | 全部 | 有序子节点 id；**非空 = 框** |
| `pos` | `null` | 全部 | `[x, y, w, h]` 整数，**相对父框左上角**（根节点相对画布原点）；`null` = 未放置（§7.4） |

- 建节点只走 `model.new_node(...)`（唯一字面量）；`store.load()` 里 `model.normalize_node()` **只补缺键、不改不删**（同 v1 的纪律）。
- 删掉的 v1 字段：`review_of`、`folded_from`、`expanded_from`、任务级 `status`/`note`。**不读、不写、不提。**
- 卡片默认尺寸 `[*, *, 240, 96]`；框默认尺寸由自动排版算（§7.4）。

### 3.3 卡片 ⇄ 框 的转换（Q18）

「是不是框」**只由 `children` 是否非空决定**，没有单独的类型字段。转换时：

- **卡片 → 框**（第一次给它放进子节点：`node add --parent X` / `move` 到 X / 画布上把卡片丢到 X 上 / `group`）：
  - 前提：X 的存储状态 ∈ {`pending`, `blocked`, `cancelled`}。在途（`claimed/running/review`）或 `done` ⇒ `PlanError`：
    「X 正在做 / 已完成，不能往里放东西 —— 先收工或新建一个框」。
  - 转换后：`status` 置 `""`、`exec` 置 `{}`、`status_since` 留着；`runs` / `feedback` / `rework` / `artifacts` 原样保留作历史。
  - X 原来的 `deps` 保留 —— 它现在是**框级依赖**（§4.2），语义自然延续：「X 要等的」变成「X 里所有东西都要等的」。
  - X 原来的 `loops`：若非空 ⇒ `PlanError`（loop 只能在卡片之间，§5.1）。
- **框 → 卡片**（最后一个子节点被移走或删掉）：`status` 置 `"pending"`，`status_since` = 现在，记一条 log。

### 3.4 id

- 格式：`n-` + 8 位，字母表 Crockford base32 小写 `0123456789abcdefghjkmnpqrstvwxyz`（去掉 `i l o u`）。
  用 `secrets.choice` 生成；与已有 id 撞了就重抽。
- **永不改变**：移动、换父、改类型、卡片⇄框都不换 id。
- 引用解析 `model.resolve(plan, ref)`：
  1. 完整 id 命中 ⇒ 它；
  2. `ref` 可省略 `n-` 前缀；按**前缀**匹配，要求 ≥ 3 个字符；
  3. 唯一命中 ⇒ 它；多个 ⇒ `PlanError`，列出候选（id + 标题）；没有 ⇒ `PlanError`「找不到」。
- 画布上从不需要人输入 id。

### 3.5 结构不变式（`model.validate(plan)`，`check` 与每次 `commit` 前都跑）

1. `roots` ∪ 所有 `children` 里，每个 id **恰好出现一次**，且都在 `nodes` 里；`nodes` 里每个 id 都被挂在树上（无孤儿）。
2. 包含关系无环（树）。
3. `deps` / `loops` 指向的 id 都存在。（悬空 ⇒ `check` 报错；`commit` 拒写）
4. 节点**不能 dep 自己的祖先或后代**（「框里的东西等框」是自相矛盾）。
5. dep 图（§4.4 展开后）无环。
6. `loops` 合法（§5.1）。
7. 框的 `status == ""`；卡片的 `status ∈ NODE_STATUS`。

`commit()` 里不满足 ⇒ `PlanError`，**一个字节都不写**。

---

## 4. dep 边与派生状态

### 4.1 卡片的显示状态（`model.effective(plan, id)`）

与 v1 的 `block_effective` 同一套规则，只是依赖换成「有效依赖」（§4.2）：

- 存储状态 ≠ `pending` ⇒ 显示存储状态；
- `pending` 且有效依赖里有任一个未 `done` ⇒ `waiting`；
- `pending` 且依赖全 `done`：有 `owner` ⇒ `claimed`，没有 ⇒ `ready`。

`STATUS_OWNER` 表照搬 v1。

### 4.2 有效依赖（`model.effective_deps(plan, id)` → 卡片 id 列表）

卡片 C 的有效依赖 = 把下面这些 id 各自**展开成卡片**后取并集（去重、保序）：

- C 自己的 `deps`；
- C 的**每一个祖先框**的 `deps`（框级依赖适用于框内一切 —— Q20）。

展开规则 `expand(id)`：卡片 ⇒ 它自己；框 ⇒ 它所有**后代卡片**中存储状态 ≠ `cancelled` 的那些（「等一个框」= 等框里所有东西做完 —— Q20）。

### 4.3 框的汇总状态（`model.rollup(plan, id)`）

对框的所有**后代卡片**的显示状态 `eff` 应用 v1 `task_status` 的同一套规则：

1. 全 `done` ⇒ `done`；2. 全 `cancelled` ⇒ `cancelled`；3. 任一在途（`claimed/running/review`）⇒ `running`；
4. 有 `blocked` 且没有 `ready/waiting` ⇒ `blocked`；5. 有 `ready` ⇒ `ready`；6. 否则 `pending`。
空框不存在（没有子节点就是卡片）。

`progress(plan)` = 所有非 `cancelled` 卡片中 `done` 的个数 / 总数；`progress(plan, frame_id)` 同理限定在框内。

### 4.4 dep 环检测

- 图：节点 = 所有卡片；边 = 每张卡片 → 它的每一个有效依赖（§4.2）。**`loops` 不进这张图。**
- `guard_acyclic(plan, what)` 照 v1：有环 ⇒ `PlanError`，把环路打印出来（`n-aaa「标题」 → n-bbb「标题」 → …`）。

### 4.5 画边用的边集（`model.edges(plan)`）

给前端画图（**不是**就绪判定）：每个节点的每条 `deps` 原样一条 `{"from": 前置 id, "to": 本节点 id, "kind": "dep"}`；
每个节点的每条 `loops` 一条 `{"from": 本节点 id, "to": 目标 id, "kind": "loop"}`。
**框级依赖就画在框与框（或框与卡片）之间**，不展开 —— v1 的 `edge_deps`「取收尾块」那套特殊处理删掉。

---

## 5. loop 边（实现-评审循环，Q20/Q25）

### 5.1 合法性

`R.loops = [I, …]` 表示「R 被打回时，回到 I 重做」。要求：

1. R 与 I 都是**卡片**；R ≠ I。
2. I 在 R 的**上游**：沿 dep 图（§4.4，有效依赖）从 R 出发能走到 I。否则不是循环 ⇒ `PlanError`：「n-I 不在 n-R 的上游，loop 只能指回它等过的东西」。
3. `kind` 不限，但 CLI/画布在 R 的 `kind != "review"` 时给一行 ⚠ 提示（不阻止）。

### 5.2 打回（`edits.reject(plan, R, note, by)`）

前提：R 的存储状态 ∈ {`claimed`, `running`, `review`}（评审在进行中），且 `R.loops` 非空；否则 `PlanError`。

一次打回做这些事（全在内存里，完了才 `commit`，任何一步抛错整次不写）：

1. 计算**循环段** S = 所有满足「在 R 的上游（含 R）且在某个 I 的下游（含 I）」的卡片
   （即 dep 图上 I → … → R 之间的所有路径上的卡片）。
2. 对每个 I：`rework += 1`，`feedback = note`，存储状态 → `pending`（有 owner 的会显示为已认领，回到原 owner 手上 —— 与 v1「打回回到原 owner」一致）。
3. S 中其余已 `done` 或在途的卡片（含 R 本身）→ `pending`。`cancelled` / `blocked` 的不动。
4. 每次状态变化都按 `set_status` 的写法追加 `runs`（`note` 前缀 `"loop: "`）、清 `exec`、写 log。
5. 若 R 的 `max_loops > 0` 且 I 的 `rework`（取 R.loops 里各 I 的最大值）≥ `max_loops` ⇒
   第 2–4 步照做，但 **R 的存储状态改为 `blocked`**，log 写「循环达到上限 N 次，等人拍板」。

v1 的「从 `review` 走出去且不是 done ⇒ 打回原 owner」**保留**（那是单张卡片的自审，不涉及 loop 边）。

### 5.3 与就绪判定的关系

`loops` 不参与 §4 的任何计算：I 打回后变 `pending`，它的 dep 前置都是 done ⇒ 显示 `claimed`/`ready`；
R 变 `pending` ⇒ 因为依赖 I 未 done 而显示 `waiting`。循环就这样自然重新跑一轮。

---

## 6. 改动的实现（`edits.py`）

所有函数只改 dict、返回结果、出错抛 `PlanError`；调用方 `store.commit()`。「→ log」= 写一条 `log_event`。

| 函数 | 做什么 | 关键规则 |
|---|---|---|
| `add_node(plan, *, parent="", index=None, title, kind, status, owner, doc, done_when, deps, pos=None, actor)` | 新建节点 | `parent` 是卡片 ⇒ 先按 §3.3 转成框；`status` 默认 `blocked`（CLI）/ 画布传 `pending`；→ log |
| `edit_node(plan, ref, **fields)` | 改字段（title/doc/input/output/command/kind/owner/done_when/max_loops） | 只改给了的键；没变化 ⇒ `PlanError("没有变化")`；→ log |
| `set_field(plan, ref, key, value)` | CLI `set_*` 的一处实现 | 同 v1 |
| `set_status(plan, ref, status, …)` | 改卡片状态 + 登记谁在做/线程 | v1 语义原样（`NODE_SET_FLAGS` = v1 `BLOCK_SET_FLAGS`）。对**框**：只接受 `cancelled` ⇒ 把所有后代中开着的卡片（`OPEN`）逐个置 `cancelled`；其他状态 ⇒ `PlanError`「框的状态是汇总出来的」 |
| `assign(plan, ref, who)` / `clear_exec(plan, ref)` | 同 v1 | |
| `move(plan, ref, *, parent, index=None, pos=None)` | 换父 / 同父内换序 | `parent=""` = 移到根；不能移进自己或自己的后代；目标是卡片 ⇒ §3.3 转框；原父失去最后一个子 ⇒ §3.3 变回卡片；移后 `guard_acyclic` + 祖先/后代 dep 检查（§3.5-4）；给了 `pos` 一并写；→ log |
| `group(plan, refs, *, title)` | 把若干**同父**节点包进一个新框 | 新框插在它们中第一个的位置；新框 `pos` = 它们包围盒外扩 24px；子节点 `pos` 换算成相对新框；→ log |
| `ungroup(plan, ref)` | 把框拆掉，子节点上移到框的父级 | 子节点 `pos` 换算回父级坐标；框的 `deps` **下放给每个直接子节点**（并入其 `deps`）以保持语义；指向这个框的 `deps` 改为指向它的每个直接子节点；框被删除；→ log |
| `remove(plan, ref, *, force=False)` | 删节点（框 = 连同整棵子树） | 有其他节点的 `deps`/`loops` 引用被删集合里的 id ⇒ 不给 `force` 就 `PlanError`（列出引用者）；`force` ⇒ 一并把这些引用删掉。返回**快照**（§6.1）；→ log |
| `bypass(plan, ref)` | 摘掉一个中间卡片，前后接起来 | 同 v1（去掉 `review_of` 分支）；指向它的 `loops` ⇒ 改指向它的前置中最后一个，没有前置 ⇒ `PlanError` |
| `insert(plan, anchor, *, before, title, …)` | 在 dep 链上 anchor 之前/之后插一张卡片 | 同 v1 语义：before ⇒ 新卡继承 anchor 的 `deps`、anchor 改为等新卡；after ⇒ 等 anchor 的那些改为等新卡、新卡等 anchor；新卡与 anchor 同父、紧挨着 |
| `set_deps(plan, ref, *, deps=None, add=(), rm=())` | 改 dep 边 | 存在性 + §3.5-4 + 查环；→ log |
| `set_loops(plan, ref, *, add=(), rm=())` | 改 loop 边 | §5.1；→ log |
| `reject(plan, ref, *, note, by)` | 打回 | §5.2 |
| `layout(plan, items)` | 批量写坐标：`items = [{"id", "pos": [x,y,w,h]}]` | 只改 `pos`；取整；`w,h ≥ 40`；id 不存在的**静默跳过**（并发下节点可能刚被删）；**不写 log、不写 runs**（Q11） |
| `restore(plan, snapshot)` | 把 `remove` 返回的快照放回去 | id 已存在 ⇒ `PlanError`；父节点不在了 ⇒ 放到根；被删时一并清掉的引用一起恢复（目标还在的才恢复）；→ log |

### 6.1 删除快照（给撤销用）

`remove` 返回：`{"nodes": {id: 节点完整 dict, …}, "parent": 原父 id, "index": 在原父 children 里的位置, "refs": [{"id": 引用者, "field": "deps"|"loops", "target": 被删 id}]}`。

---

## 7. HTTP 服务 v2（`serve.py`，给桌面应用用）

### 7.1 启动与握手

```
loomerto --plan <文件> serve [--port 0]
```

- 只绑 `127.0.0.1`；`--port 0` = 系统挑空闲端口。
- 启动成功后向 **stdout 打印恰好一行 JSON** 然后不再往 stdout 写：
  `{"ok": true, "port": 51234, "token": "<32 位随机 hex>", "file": "<绝对路径>", "pid": 12345}`。
  失败 ⇒ stderr 写原因、退出码 2。
- **鉴权**：除 `OPTIONS` 外的每个请求都必须带 `X-Loomerto-Token: <token>` 头，否则 403。
  （防本机网页对 127.0.0.1 做跨站请求；SSE 用 `fetch` 流式读取带头，不用 `EventSource`。）
  CORS：允许 `Origin: http://localhost:*`（Vite 开发服务）与 Electron 的打包 origin；`Access-Control-Allow-Headers: X-Loomerto-Token, Content-Type`。
- **生命周期**：服务读 stdin，**stdin 读到 EOF 就退出**（Electron 主进程持有管道，应用崩了服务也跟着走）。
  开发时手跑可加 `--no-stdin-watch`。
- `loomerto open` 与 `assets/canvas.html` **删除**（不留别名）。

### 7.2 读

`GET /api/plan` → `200 {"ok": true, "view": View}`

```ts
type View = {
  name: string; file: string; title: string; goal: string; status: string;
  rev: string; progress: [number, number];
  roots: string[];
  nodes: Record<string, NodeView>;
  edges: {from: string; to: string; kind: "dep" | "loop"}[];
  participant_options: {id: string; label: string; kind: string; registered: boolean}[];
  enums: {node_status: string[]; kinds: string[]; set_flags: Record<string, string[]>};
};
type NodeView = {
  id: string; title: string; kind: string;
  is_frame: boolean; parent: string;            // 派生
  children: string[];
  status: string;                               // 显示状态：卡片 = effective，框 = rollup
  raw_status: string;                           // 存储状态（框为 ""）
  owner: string; doc: string; input: string; output: string; command: string;
  done_when: string[]; artifacts: string[];
  deps: string[]; loops: string[]; effective_deps: string[];
  max_loops: number; rework: number; feedback: string;
  exec: Record<string, unknown>; claimed_by: string; claimed_at: string;
  runs: {at: string; by: string; from: string; to: string; note: string}[];   // 最近 10 条
  progress: [number, number] | null;            // 只有框有
  pos: [number, number, number, number] | null;
};
```

### 7.3 写

`POST /api/op`，body `{"op": "...", "rev": "<读到的 rev>", "by": "<actor，默认 canvas>", ...}`：

- `200 {"ok": true, "msg": "<中文一句话>", "ref": "<这次动到的主节点 id 或 ''>", "undo": <反向 op 或 null>, "view": View}`
- `400 {"ok": false, "error": "<中文原因>"}` —— 参数或模型层拒绝；**磁盘不变**
- `409 {"ok": false, "error": "这份 plan 在别处被改过了 —— 已刷新，请重做"}` —— `rev` 过期
- 一次请求 = 一次 `commit()`。**不做自动合并、前端不自动重试**（同 v1）。

| op | 参数 | 映射 | `undo`（服务端算好的反向 op） |
|---|---|---|---|
| `create` | `parent?` `index?` `title` `kind?` `pos?` | `add_node(status="pending")` | `{"op":"delete","id":新 id,"force":true}` |
| `edit` | `id` + 任意字段 + `status?` `doing?` `note?` | `edit_node` 后视需要 `set_status`（R-24「单一保存」语义原样） | `edit` 回旧值（含旧 status；status 回滚会再写一条 run —— 历史如实） |
| `status` | `id` `status` `doing?` `note?` | `set_status` | `status` 回旧值 |
| `reject` | `id` `note` | `reject` | `null`（打回不可撤销，界面上提示） |
| `move` | `id` `parent` `index?` `pos?` | `move` | `move` 回原父/原位置/原 pos |
| `group` | `ids[]` `title` | `group` | `{"op":"ungroup","id":新框}` |
| `ungroup` | `id` | `ungroup` | `null` |
| `delete` | `id` `force?` | `remove` | `{"op":"restore","snapshot":…}` |
| `restore` | `snapshot` | `restore` | `delete` |
| `edge_add` | `from` `to` `kind` | dep：`set_deps(to, add=[from])`；loop：`set_loops(from, add=[to])` | `edge_rm` 同参 |
| `edge_rm` | `from` `to` `kind` | 对称 | `edge_add` 同参 |
| `layout` | `items[]` | `layout` | `layout` 回旧 pos |

- **`layout` 免 `rev` 检查**：它只写 `pos`，按 id 合并到当前最新的 plan 上（最后写者赢），不会盖掉别人的字段改动。
  其余 op 一律检查 `rev`。
- 没有 `ref` 的 op 返回 `ref: ""`。

### 7.4 变更推送

`GET /api/events`（带 token 头）→ `text/event-stream`：

- 服务端每 250ms 看一次数据文件的 `(mtime_ns, size)`；变了就读出 `rev`，与上次推送的不同 ⇒ 推
  `event: changed\ndata: {"rev": "..."}\n\n`。
- 每 15s 推一行注释 `: ping` 保活。
- 文件被删/改成非法 json ⇒ 推 `event: error\ndata: {"error": "..."}`，继续轮询（agent 写一半不会被看到，因为是原子写；这里只是兜底）。
- 前端收到 `changed` 且 `rev` ≠ 手里的 ⇒ `GET /api/plan` 重取（§8.6）。自己 POST 造成的变化也会推过来，`rev` 相同则忽略。

---

## 8. 桌面应用（`desktop/`）

### 8.1 技术栈（Q7/Q14）

| 层 | 选型 | 为什么 |
|---|---|---|
| 外壳 | **Electron**（Electron Forge，`vite-typescript` 模板） | 自带 Chromium，各平台渲染一致；资料最多；主进程是 TS，不用学 Rust |
| UI | **React 18 + TypeScript** | React Flow 要求 React |
| 画布 | **`@xyflow/react`（React Flow v12，MIT）** | 原生支持节点/边/子流（`parentId`）、缩放平移、框选、连线手柄；自定义节点类型 = 卡片与框 |
| 自动排版 | **`elkjs`** | 分层布局、支持嵌套 |
| 状态 | **`zustand`** | 轻；React Flow 自己也用它 |
| 测试 | **Vitest**（逻辑）+ **Playwright**（浏览器 & `_electron`） | |

版本：实现时取各包**当前稳定版并在 `package.json` 写死精确版本**（不用 `^`）。Node 用当前 LTS，写进 `desktop/.nvmrc`。

### 8.2 进程结构

```
Electron 主进程（desktop/src/main/）
  ├─ 窗口管理：一窗一 plan；同一文件已开 ⇒ 聚焦那一窗
  ├─ 每个窗口 spawn 一个 `loomerto --plan <文件> serve --port 0`，读 stdout 首行拿 {port, token}
  ├─ 握手前子进程就退出（如 v1 文件被拒读、json 坏了）⇒ 窗口里原样显示它的 stderr + 「从侧栏移除」按钮，不弹崩溃框
  ├─ 运行中子进程意外退出 ⇒ 画布变只读 + 顶部红条「服务已退出：<stderr 末行>」+「重启」按钮
  ├─ 窗口关闭 ⇒ 关掉子进程 stdin（服务自退），2s 没退 ⇒ kill
  └─ 本地状态：userData/state.json（最近打开、各 plan 视口、窗口位置、折叠的框）
preload（desktop/src/preload/）—— contextBridge 只暴露：
  getServer(): {port, token, file}
  recent.list() / recent.remove(file)
  view.load(file) / view.save(file, {viewport, collapsed})
  openDialog() / newPlanDialog() / openFile(file)
渲染进程（desktop/src/renderer/）—— React 应用
  直接用 fetch 打 http://127.0.0.1:<port>/api/*（带 token 头）
```

**`loomerto` 在哪**：macOS 从 Dock 启动的 GUI 应用**不继承 shell 的 PATH**（这是常见坑）。查找顺序：
设置里的显式路径 → `~/.local/bin/loomerto`（`uv tool install` 的默认落点）→ 用登录 shell 解析
`$SHELL -lc 'command -v loomerto'` → 都没有 ⇒ 窗口里显示「找不到 loomerto，请 `uv tool install --editable <repo>` 或在设置里指定路径」。
v2 **不把 Python 打进安装包**。

### 8.3 窗口与侧栏（Q28）

- 应用**只打开 json 文件**，不浏览文件夹。
- 菜单：**文件 → 打开…**（文件对话框，只显示 `*.json`）/ **新建 plan…**（保存对话框选路径与文件名 ⇒ 主进程跑
  `loomerto --plan <路径> plan new --title <文件名>` ⇒ 打开它）/ **最近打开**（子菜单）/ 关闭窗口。
- **侧栏**（每个窗口左侧，可折叠）：最近打开过的 plan 列表（标题 + 文件名 + 上次打开时间，最新在上，最多 30 条）。
  点击 ⇒ 打开（新窗口）或聚焦已开的窗口；文件已不存在 ⇒ 灰显 + 「移除」按钮。当前窗口的 plan 高亮。
- **可多窗**；每窗标题 = plan 的 `title` — 文件名。
- 记住：每个 plan 的视口 `{x, y, zoom}`、折叠的框 id 集合、窗口位置大小。下次打开恢复。
- 启动时：恢复上次关闭时还开着的那些窗口；一个都没有 ⇒ 打开一个只有侧栏 + 「打开 / 新建」按钮的空窗口。

### 8.4 画布：显示

- **卡片**（React Flow 自定义节点 `card`）：标题；状态胶囊（颜色用主题变量 `var(--<状态>)`）；`@owner`（有在做的人时 `@owner→@exec.by`）；
  类型小标；判据计数；`rework > 0` 时显示 `↻N`；有线程登记时一个小点（悬停看 delegation/transcript）。
- **框**（自定义节点 `frame`，React Flow 子流的父节点）：标题栏（标题 + 汇总状态胶囊 + 进度 `done/total` + `@owner`）+ 内容区。
  双击标题栏左侧的折叠箭头 = 折叠成一张「摘要卡」（只是本地视图状态，不写 json）。
- 嵌套深度不限：子节点 `parentId` = 父框 id，`position` = json 里的相对坐标。
- **边**：`dep` 实线箭头（前置 → 后继）；`loop` 虚线、另一种颜色、弧形回指。框级 dep 就连在框上。
- 主题：v1 的 `loomerto/assets/theme.css` 搬到 `desktop/src/renderer/theme.css`，**颜色只来自它**（v1 的「前端不许自带颜色」约定延续）；补一套深色变量。
- 小地图 + 缩放控件（React Flow 自带）。

### 8.5 画布：操作（Q27）

| 操作 | 交互 | 发的 op |
|---|---|---|
| 新建卡片 | 双击空白（在根 / 在某个框内部）⇒ 原地出现卡片并进入标题编辑；回车确认、Esc 取消 | `create`（`parent` = 双击处最深的框，`pos` = 点击处，相对该框） |
| 改字段 / 状态 | 选中 ⇒ 右侧详情面板；**一个「保存」**把改动的字段 + 状态一次提交（R-24 语义原样） | `edit` |
| 打回 | 详情面板上，`loops` 非空且状态在途时出现「打回」按钮 + 意见输入框 | `reject` |
| 拖动 | 松手时才发；多选拖动 = 一次 `layout` 带多项 | `layout` |
| 换父 | 松手时指针下**最深的框**（排除自己与后代）≠ 原父 ⇒ 换父；拖到某张**卡片**上停留 600ms 高亮后松手 ⇒ 目标卡片变框 | `move`（带新的相对 `pos`） |
| 改框尺寸 | 框右下角手柄（`NodeResizer`），松手时发 | `layout` |
| 框自动长大 | 子节点被放到框外沿之外 ⇒ 框扩到能包住它（+24px 边距），与子节点的 `layout` 合成一次请求 | `layout` |
| 打包 | 多选同父节点 ⇒ ⌘G | `group` |
| 拆框 | 选中框 ⇒ ⇧⌘G | `ungroup` |
| 画 dep 边 | 从节点右侧手柄拖到另一节点 ⇒ 「目标等源」 | `edge_add kind=dep` |
| 画 loop 边 | 按住 ⌥ 拖，或工具栏切到「loop」模式 | `edge_add kind=loop`（源 = 评审者，目标 = 被打回者） |
| 删边 | 选中边 ⇒ ⌫ | `edge_rm` |
| 删节点 | 选中 ⇒ ⌫ ⇒ 确认框：卡片「删除 X？」；框「删除框和里面的 N 个节点 / 只拆框（保留里面的）/ 取消」；被引用 ⇒ 确认框里列出引用者并说明会一并断开 | `delete force=true` 或 `ungroup` |
| 撤销 / 重做 | ⌘Z / ⇧⌘Z | 发服务端给的 `undo` op（§8.7） |

服务端 400 ⇒ 右下角 toast 显示 `error` 原文（例如成环路径），画布回滚到服务端返回前的样子（不做乐观更新残留）。

### 8.6 实时刷新（Q6）

- 渲染进程订阅 `/api/events`；收到新 `rev` ⇒ 重取 `/api/plan` ⇒ 用 id 对齐地更新 React Flow 的 nodes/edges（**不重建**），保持视口与选中。
- 选中的节点被别处删了 ⇒ 取消选中并 toast「它在别处被删了」。
- 详情面板有未保存改动、而该节点在别处被改了 ⇒ 面板顶部黄条「这个节点在别处被改了：[载入新值] [继续编辑]」；继续编辑后保存会拿到 409 ⇒ 提示重做。
- 拖动进行中收到刷新 ⇒ 等松手后再应用。

### 8.7 撤销（Q27：撤销 = 再提交一次）

- 每个窗口一个本地撤销栈（不持久化，关窗即清）。每次成功的 op 把服务端返回的 `undo` 压栈（`null` 的不压，并清空重做栈）。
- ⌘Z ⇒ 弹出栈顶，**作为一次普通 op 提交**（带当前 `rev`，进 log，`actor` = `canvas`，`note` 前缀 `"undo: "`）；成功后它返回的 `undo` 压进重做栈。
- 撤销提交失败（400/409，例如目标已被别人删）⇒ toast 原因，丢弃这一项。
- 不可撤销的 op（`reject` / `ungroup`）执行前在确认框里写明「不可撤销」。

### 8.8 自动排版（Q12）

- 时机：每次拿到 view 后，找出 `pos == null` 的节点。
- 按父分组：某个父（或根）下**所有**子都没坐标 ⇒ 对这一组跑 ELK `layered`（方向左→右，按组内 dep 边），框的尺寸由内向外算（先排子、再定框大小）。
- 父下**部分**子有坐标 ⇒ 每个未放置节点：放在它组内最右的一个 dep 前置的右边 48px；没有组内前置 ⇒ 放在已放置兄弟包围盒的下方 48px，从左往右排。
- 算完一次性发一个 `layout` op 写回。**已有坐标的节点永不被自动挪动。**
- 工具栏另有「整理这个框」按钮（对选中框的直接子节点跑 ELK 并写回，可撤销）。

### 8.9 安全（Electron 默认加固）

`contextIsolation: true`、`nodeIntegration: false`、`sandbox: true`；禁止窗口内导航与新开窗口（`will-navigate` / `setWindowOpenHandler` 一律拒）；
CSP：`default-src 'self'; connect-src 'self' http://127.0.0.1:*; style-src 'self' 'unsafe-inline'`；
token 只经 preload 交给渲染进程，不写日志。Forge fuses：关 `RunAsNode`、关 `EnableNodeOptionsEnvironmentVariable`、开 `OnlyLoadAppFromAsar`。

### 8.10 打包（M5）

macOS arm64 优先：Forge `maker-dmg` + `maker-zip`，**不签名、不公证**（本机自用；首次打开需右键「打开」，README 写明）。Windows 保留可能性、不测试。

---

## 9. CLI v2（agent 的前端）

**无别名，`task` / `block` 两组整组删除。**一级命令：

| 命令 | 说明 |
|---|---|
| `plan new [--title] [--goal] [--owner id=kind:label]…` | 新建（必须 `--plan <x>.json` 或 `--plans-root <d>` + 名字） |
| `list` / `current` / `check` / `workers` / `digest` / `note <text>` | 同 v1，改为遍历节点；`current` 列 `ready`/`claimed`/在途的卡片；`check` 跑 §3.5 全部不变式 + v1 的图质量检查 |
| `render` | 只重写 `<名字>.md` |
| `serve [--port] [--no-stdin-watch]` | §7 |
| `node <动作>` | 见下表 |

`node` 组（`<ref>` = id 或唯一前缀）：

| 动作 | 用法 |
|---|---|
| `add` | `node add --title T [--parent <ref>] [--index N] [--kind] [--doc] [--done-when X]… [--owner] [--status] [--deps a,b]` ⇒ 打印新 id |
| `show` | `node show <ref>`（只读；框会列出子节点树与汇总状态） |
| `tree` | `node tree [<ref>]`：缩进树，一行一个节点：`id 状态 @owner 标题` |
| `remove` | `node remove <ref> [--force]` |
| `move` | `node move <ref> --parent <ref|root> [--index N]` |
| `group` / `ungroup` | `node group <ref> <ref>… --title T` / `node ungroup <ref>` |
| `insert` | `node insert <anchor> --before|--after --title T …` |
| `bypass` | `node bypass <ref>` |
| `deps` | `node deps <ref> [--add a,b] [--rm a] [--set a,b]` |
| `loops` | `node loops <ref> [--add a] [--rm a] [--max N]` |
| `reject` | `node reject <ref> --note "…" [--by who]` |
| `assign` | `node assign <ref> <who>` |
| `set_status` | 同 v1 `block set_status`（旗标表 `NODE_SET_FLAGS`）；对框只接受 `cancelled` |
| `set_title` / `set_doc` / `set_type` / `set_input` / `set_output` / `set_command` / `set_audit` | 一条属性一条命令，同 v1 |

- 删除的 v1 命令（不留别名、不给提示）：`task *`、`block *`、`block expand` / `compress` / `reorder`、`open`。
- 退出码、stderr 中文、stdout 给结果的约定同 v1。
- **agent 不写坐标**：CLI 没有任何写 `pos` 的入口（`layout` 只开给 HTTP）。

---

## 10. `<名字>.md`（`md.py`，原 `render.py` 只留这部分）

供 agent 与人读的文本视图，内容（顺序固定）：

1. `# 标题`、目标、进度 `done/total`、`rev`、更新时间、数据文件名。
2. **现在能动的**：所有 `ready` / `claimed` / 在途卡片（id、标题、@owner、状态）。
3. **结构树**：缩进列表，一行一个节点：`- [状态] 标题 · id · @owner`；卡片下缩进列出 `deps`（写成「等：标题（id）」）、`loops`（「打回到：…」）、判据。
4. **Mermaid 图**：`flowchart LR`，框 = `subgraph`（可嵌套），dep = `-->`，loop = `-.->|loop|`。节点超过 80 个时省略这一节并写一行说明。
5. 最近 20 条 log。

**不含**坐标、`pos`、视口等空间信息。

---

## 11. 删除清单（M1 结束时仓库里不应再有）

- `loomerto/assets/plan.html`、`loomerto/assets/canvas.html`；`theme.css` 搬去 `desktop/`（M3 前先放 `docs/theme-v1.css` 留档也可，由 M3 工单取走）
- `render.py` 里的 `render_html` / `render_canvas` / `depth_map` / `theme_css`（文件改名 `md.py`）；`store.TEMPLATE` 与 `$LOOMERTO_TEMPLATE`
- `model` 里的 `all_blocks` / `blocks_of` / `owner_of` / `next_block_id` / `block_deps` / `edge_deps` / `task_status` / `parse_step` / `BLOCK_HISTORY_FIELDS`
- `edits` 里的 `TASK_STATUS` / `TASK_SET_FLAGS` / `add_task` / `add_block` / `insert_block` / `move_block` / `reorder_blocks` / `bypass_block`（换成 §6 的新名）
- `serve.py` 里的 v1 `view_model` 与六个 v1 op；`docs/canvas-sync.md`（换成 `docs/desktop-api.md`）
- `pyproject.toml` 的 `package-data`（不再有资产）
- 示例 `examples/Loomerto-Example-1/`（Obsidian 草稿，用户确认不需要；未入库，删前问一句）

---

## 12. 验收（每个里程碑都要过的闸）

1. **单元测试**：`python3 -m unittest discover -s tests -v`（stdlib `unittest`，不引入 pytest）全绿。
2. **名字自检**：`python3 -m pyflakes` 不可用时用 `python3 -m py_compile loomerto/*.py` + 工单里给的 grep。
3. **端到端脚本**：`tests/e2e_cli.sh` 在临时目录里从零建一份 plan、走完 §13 的场景，最后 `loomerto check` 退 0。
4. **install 回读**：`uv tool install --editable .` 后从任意目录跑 `loomerto --plan /tmp/x/PlanA.json node tree`。
5. **前端**：`cd desktop && npm run typecheck && npm test && npm run e2e`。画布改动必须在真浏览器 / 真 Electron 里**真拖一次**（v1 第 5 道闸延续）。

## 13. 标准场景（e2e 与手测共用）

1. `plan new` 建 `PlanA.json`；同目录再建 `PlanB.json` —— 两份 `.md` 互不覆盖。
2. 建框「实现」，内有卡片 A「写代码」、B「评审」；B deps A；B loops A，`max_loops 2`。
3. A：`pending` → `running`（--by agent1）→ `done`；B 显示 `ready`。
4. B `running` → `node reject B --note 改命名`：A 回到 `pending`（显示 `claimed`，owner 不变）、`rework=1`、`feedback` 写入；B 显示 `waiting`。
5. 再走一轮、再打回：A `rework=2` ⇒ B 被置 `blocked`（达到上限）。
6. 尝试 `node deps A --add B` ⇒ 成环被拒，文件不变（比对 sha256）。
7. `node move A --parent root` ⇒ id 不变；框「实现」只剩 B。
8. 再 move B 到根 ⇒ 「实现」变回卡片，状态 `pending`。
9. 桌面端：打开 PlanA，拖动 A，`git diff` 只有 `pos` 一行 + `rev`/`updated_at`；另一个终端 `node set_title A 新名字` ⇒ 画布 1 秒内更新、视口不动。

# Tickets · 画布：参与方选择器 + 单一「保存」

> 父文档：[`canvas-participant-picker-and-single-save.md`](canvas-participant-picker-and-single-save.md)（spec，含实测证据与全部决定）。
> 每个 ticket 是一条**纵向切片**（schema → 服务 → 页面 → 验证），做完即可单独验证。
> `Blocked by` = 必须先完成的那张票。

---

## T1 · 参与方候选由服务端算好发下来

**Blocked by**: 无（可立即开始）

**Deliverable**：`GET /api/plan` 的 `view` 里多一份**参与方候选**，候选 = `plan.json` 的
`participants` **∪** 计划里实际出现的块 `owner` **∪** 实际出现的 `exec.by`，按 id 去重、
顺序稳定（建议：已注册的在前，按 `participants` 原序；未注册的按 id 排序在后）。
每项带 `id` / `label`（没有 label 就退回 id）/ `kind` / 一个「是否已注册」标记。

**不要动** `view.participants`（原样透传，避免破坏已有前端/别的消费者）。

**验证**：起服务，对一份 `participants` 只含 A、但块 owner 是 B 的 plan，
`curl /api/plan` 的候选里**同时有 A 和 B**；对 `dream-waveA`（三个都在名单里）候选仍是那 3 个、不重复。

**证据要求**：贴出两份 plan 的 `curl` 输出对比。

---

## T2 · 三个字段都挂上真能弹的下拉

**Blocked by**: T1

**Deliverable**：编辑器的 `认领人` / `在做（--by）`，以及**新建块** / **新建任务**表单的 `认领人`，
四处都能下拉出 T1 那份候选。修掉「新建表单里 `list="people"` 但 DOM 里没有那个 datalist」的悬空引用
（倾向：把 datalist 提到页面级，一次渲染、四处复用）。候选的显示文本用 `label`，`value` 仍用 `id`。
仍是可自由输入的 `<input list=…>`（白名单只作建议）。

**验证**：在真浏览器里（CDP 读 DOM）逐个字段断言 `input.list !== null` 且 `list.options.length > 0`；
新建块 / 新建任务表单**各自**都要过（这两处现在都是 `resolves:false`）。
对 `skillweave` 那份（44 个块 owner=`default`、participants 只有 `you`）断言：候选里**含 `default`**，
且 `#f_owner` 的当前值能在候选里找到。

**证据要求**：四个字段的 `list.options` 实测输出 + 一张真浏览器截图（下拉展开）。

---

## T3 · 合并成一个「保存」：字段 + 状态一次写

**Blocked by**: T2（T2 定下了候选怎么来；T3 依赖同一份 DOM 结构）

**Deliverable**：右栏删掉 `#setst`（「改状态」）按钮，只留一个「保存」；它把这一屏里改过的
**所有东西**（标题 / 做什么 / 判据 / 认领人 / 类型 / 状态）**一次提交、一次 `commit()`**。
「打回重做」保持独立按钮、行为不变。

**硬约束**：
- 字段写入仍走 `edits.edit_block`（或 `edits.set_field`），状态写入仍走 `edits.set_status` ——
  **不许在 `serve.py` 里重写这两段语义**。
- `--by`（在做）只在目标状态收它时才发（`edits.BLOCK_SET_FLAGS` 是唯一那张表），
  撞上不收的状态要给人话错误，不许无脑塞。
- 分清 **actor（谁在改，API 的 `by`）** 与 **`--by`（谁在做）** —— 现在前端 `fields()` 把两者混成一个值，顺手修掉。
- 只发**真的变了**的字段；无变化时提示「没有变化」，不发无效写入。
- 任一步 `PlanError` ⇒ **不 commit**，回 400，磁盘一个字节不变。

**验证**（这是 bug ② 的核心回归）：一次保存同时改**标题 + 状态** ⇒ 回读 `plan.json`：
标题变了、状态变了、`runs` 里多了一条；**标题不许丢**（这是现在 `op=status` 的隐性缺陷）。
再验：非法状态 ⇒ 400 且 `plan.json` 的 sha 不变；什么都没改就保存 ⇒ 提示「没有变化」；
点「打回重做」⇒ 状态回 `claimed`、note 进 `feedback`、`owner` 不变。

**证据要求**：前后 `plan.json` 的 diff + 那几条 HTTP 状态码。

---

## T4 · 协议文档与需求对齐

**Blocked by**: T3（要等实现定下来才能写准）

**Deliverable**：
- `docs/canvas-sync.md`：op 表里 `edit` 的「还要给」加上可选 `status`；
  写清「一次请求仍是一次 `commit()`」「字段与状态一起提交时的语义」「失败不写」。
- `REQUIREMENTS.md`：把这两条并入（新增 R-24，或挂到 R-02 的现状列 —— 按仓库纪律，改「现状」列、
  不新开清单）。需求条目号与「已落地 <sha>」照仓库惯例写。
- `specs/` 是新增目录 ⇒ 在 `README.md` / `REQUIREMENTS.md` §0 文档地图里加一行引用
  （否则 `.skillweave` 结构体检会把这份 spec 报成「到不了根」）。
- 若本轮改了代码，`docs/cli-reference.md` 顶部的**代码基线 sha** 按仓库纪律更新（若该文件受影响）。

**验证**：`docs/canvas-sync.md` 的 op 表与 `serve.py` 的 `_apply` **逐条对得上**；
文档地图里的链接**都能打开**（本地 `ls` 路径存在）。

**证据要求**：文档 diff。

---

## T5 · 全量回归与验收（交付前）

**Blocked by**: T1–T4

**Deliverable**：按 `docs/architecture.md` 的**四道闸**与 `maintain-the-loomerto-package` 的验收矩阵跑一遍：

1. **语法/导入**：`python3 -m py_compile` 全部改动文件；`python3 -c "import loomerto"`。
2. **CLI 不回归**：对真实 plans 库跑 `list` / `current` / `check` / `workers`，
   与改动前**逐字节对拉**（本改动不该动 CLI 一个字节 —— 差异就是回归）。
   ⚠ 跑另一份 checkout 时注意 `sys.path` 陷阱（见 `docs/architecture.md` §闸2）。
3. **真数据只读回归**：装好的 `loomerto` 对真实 plans 跑只读命令。
4. **真浏览器**：起服务、真读 DOM（T2/T3 的断言），改 `serve.py` 要**重启**服务、改资产不用。
5. **文件模式 + 库模式**两条路都跑（画布用 `--plan` 文件模式）。

**验证**：四道闸全绿；CLI 对拉**零差异**。

**证据要求**：每条闸的实测输出（命令 + 结果）。

---

## 提交纪律（仓库既有约定）

- **新特性分支**（如 `feat/canvas-participant-picker`），**不在 main 上直接改**。
- **两个提交**：代码一个（`feat(canvas): …`），文档/需求一个（`docs:` 且信息里写明代码基线 sha）。
- **不推远端**（用户没说要推）。
- 提交前 `git status --short` 确认没有别人的在途改动。

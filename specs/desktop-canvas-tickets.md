# 工单：Loomerto v2 —— 统一节点模型 + Electron 桌面画布

> 设计依据：[`desktop-canvas.md`](desktop-canvas.md)（下文「§N」都指它）。**工单与 spec 冲突时以 spec 为准**；
> spec 本身有问题 ⇒ 停下来在工单里写「待澄清：…」交回，不要自己拍板。

## 给执行者的通用规矩（每张工单都适用）

1. **先读**：spec 里本工单引用的那几节 + 本工单「要改的文件」的当前内容。不读 spec 不许动手。
2. **只动本工单列出的文件**。需要改别的文件 ⇒ 在交付说明里写明为什么，别顺手大改。
3. **Python 核心零依赖**：只用 stdlib，`>=3.9`（不用 `match`、不用 `X | Y` 类型写法以外的 3.10+ 语法 —— 文件头已有 `from __future__ import annotations` 的可以写注解里的 `X | None`）。
4. **核心层不 print、不 `sys.exit`**，错误只抛 `model.PlanError(msg, code)`；用户可见文字一律中文。
5. **写盘只走 `store.commit()`**；改动只在 `edits.py` 里实现一份。
6. 测试用 stdlib `unittest`，放 `tests/`，文件名 `test_<模块>.py`；跑法 `python3 -m unittest discover -s tests -v`。
7. 每张工单交付时附：改了哪些文件、验收命令的**实际输出**（贴原文，不要转述）、没做到的地方。
8. 在 git worktree 或独立分支上做；一张工单一个提交，提交信息 `feat(v2): <工单号> <一句话>`。
9. 「并行」列写着可并行的工单可以同时派给不同的人；其余按依赖顺序。

## 总览

| 工单 | 里程碑 | 一句话 | 依赖 | 可并行 |
|---|---|---|---|---|
| M1-01 | M1 核心 | 节点形状、id、树的辅助函数、不变式 | — | |
| M1-02 | M1 | 派生：有效依赖、显示状态、框汇总、环检测、loop 合法性 | M1-01 | |
| M1-03 | M1 | store：命名、定位、v1 拒读、紧凑 json、commit | M1-01 | 与 M1-02 并行 |
| M1-04 | M1 | `md.py`：`<名字>.md` 渲染；删 html/canvas 渲染 | M1-02, M1-03 | |
| M1-05 | M1 | edits A：增 / 改 / 状态 / 指派 | M1-02 | 与 M1-04 并行 |
| M1-06 | M1 | edits B：move / group / ungroup / remove / restore / layout | M1-05 | |
| M1-07 | M1 | edits C：deps / loops / reject / bypass / insert | M1-05 | 与 M1-06 并行 |
| M1-08 | M1 | CLI：全局旗标 + `plan` / `node` 组 | M1-04, M1-06, M1-07 | |
| M1-09 | M1 | CLI：`list` / `current` / `check` / `workers` / `digest` / `note` / `render` | M1-08 | |
| M1-10 | M1 | 端到端脚本 `tests/e2e_cli.sh`（§13 场景 1–8） | M1-09 | |
| M1-11 | M1 | 文档：cli-reference / architecture / REQUIREMENTS / README | M1-09 | 与 M1-10 并行 |
| M1-12 | M1 | skill：`loomerto-plan` 正文与 references 重写 | M1-09 | 与 M1-10/11 并行 |
| M1-13 | M1 | skill：remind / intake / check-commands / check-temps（含两个脚本） | M1-09 | 与 M1-12 并行 |
| M1-14 | M1 | skill：maintain-the-loomerto-package | M1-11 | |
| M2-01 | M2 服务 | `serve`：握手、token、stdin 退出、`GET /api/plan` | M1-09 | |
| M2-02 | M2 | `POST /api/op`：全部 op + undo + 409/400 | M2-01 | |
| M2-03 | M2 | `GET /api/events`（SSE） | M2-01 | 与 M2-02 并行 |
| M2-04 | M2 | `docs/desktop-api.md`，删 `canvas-sync.md` | M2-02, M2-03 | |
| M3-01 | M3 只读画布 | `desktop/` 脚手架 + 浏览器内开发模式 | M2-01 | |
| M3-02 | M3 | 类型化 API 客户端 + 事件订阅 + 状态仓库 | M3-01, M2-04 | |
| M3-03 | M3 | 卡片 / 框组件、嵌套、主题 | M3-02 | |
| M3-04 | M3 | dep / loop 边的绘制 | M3-03 | |
| M3-05 | M3 | 自动排版 + 写回 | M3-03 | 与 M3-04 并行 |
| M3-06 | M3 | 实时刷新对齐（保视口、保选中） | M3-03 | 与 M3-04/05 并行 |
| M4-01 | M4 编辑 | 拖动 / 改尺寸 / 换父 / 框自动长大 | M3-05 | |
| M4-02 | M4 | 新建 / 打包 / 拆框 / 删除对话框 | M3-06 | 与 M4-01 并行 |
| M4-03 | M4 | 画边 / 删边（dep、loop 两种模式） | M3-04 | 与 M4-01/02 并行 |
| M4-04 | M4 | 详情面板：字段 + 单一保存 + 打回 + 历史 | M3-06 | 与 M4-01/02/03 并行 |
| M4-05 | M4 | 撤销 / 重做 | M4-01..04 | |
| M4-06 | M4 | 冲突与失败呈现（409 黄条、400 toast、回滚） | M4-04 | 与 M4-05 并行 |
| M4-07 | M4 | 浏览器 e2e（Playwright） | M4-05, M4-06 | |
| M5-01 | M5 桌面 | 主进程：找 loomerto、每窗 spawn serve、生命周期 | M3-01 | 可与 M4 并行 |
| M5-02 | M5 | 多窗口：一窗一文件、聚焦已开、窗口位置 | M5-01 | |
| M5-03 | M5 | 侧栏「最近打开」+ 视口持久化 + 打开/新建/菜单 | M5-02 | |
| M5-04 | M5 | 安全加固（CSP / sandbox / 导航 / fuses） | M5-01 | 与 M5-02/03 并行 |
| M5-05 | M5 | 打包（dmg/zip，arm64，不签名）+ README | M5-03, M5-04 | |
| M5-06 | M5 | Electron e2e（Playwright `_electron`）+ §13 场景 9 | M5-05, M4-07 | |

---

# M1 · Python 核心重写

> M1 期间 `loomerto` 包会有一段时间不可用（模块换血）。**M1-01 到 M1-09 在同一条分支上串起来做**，合进 main 时整体合。

### M1-01 · 节点形状、id、树、不变式

- **读**：§3（全部）。
- **改**：`loomerto/model.py`（重写前半部分）；新建 `tests/test_model.py`、`tests/__init__.py`（空）。
- **做**：
  1. 保留 `PlanError`、`now`/`parse_ts`/`hours_since`、`log_event`、`STATUS_OWNER`/`owner_rule`、`KINDS`、`ACTIVE`/`OPEN`。
  2. `NODE_STATUS`（= v1 `BLOCK_STATUS` 同值同序）、`SCHEMA = "loomerto/plan@2"`。
  3. `NODE_FIELDS`：按 §3.2 表格的**键顺序**与默认值（可变默认值写成工厂 `list`/`dict`；`pos` 默认 `None`）。
  4. `new_id(plan) -> str`：§3.4（`secrets`，字母表常量 `ID_ALPHABET`，撞了重抽）。
  5. `new_node(plan, *, title, kind="impl", status="blocked", owner="", doc="", done_when=(), deps=(), pos=None) -> dict`：唯一字面量；校验标题非空、kind/status 合法；`pos` 给了就取整。
  6. `normalize_node(node) -> list[str]`、`normalize_plan(plan) -> dict`：只补缺键。
  7. 树辅助：`node(plan, id)`、`is_frame(n)`、`parent_index(plan) -> dict[id, parent_id_or_""]`、`siblings_list(plan, parent_id) -> list`（返回 `roots` 或父的 `children` **本体**，便于原地改）、`ancestors(plan, id) -> list`（近到远）、`descendants(plan, id) -> list`（先序，不含自己）、`leaves(plan, id) -> list`（后代卡片；对卡片返回 `[id]`）、`all_nodes(plan)`（先序遍历整棵树）。
  8. `resolve(plan, ref) -> str`：§3.4 的前缀解析与报错文案。
  9. `validate(plan) -> None`：§3.5 的第 1、2、3、4、7 条（第 5、6 条留给 M1-02，先写好调用位置并 `# M1-02` 注释）。
  10. 删除 §11 列出的 model 旧函数。
- **验收**：
  ```bash
  python3 -m unittest tests.test_model -v
  grep -nE "all_blocks|blocks_of|owner_of|next_block_id|block_deps|edge_deps|task_status|BLOCK_HISTORY|review_of|expanded_from" loomerto/model.py   # 应无输出
  ```
  测试至少覆盖：id 格式正则 `^n-[0-9a-hjkmnp-tv-z]{8}$`；前缀唯一/歧义/过短/找不到四种；`validate` 对孤儿、重复挂载、包含成环、悬空 dep、dep 祖先、框有 status 各报错。
- **不做**：派生状态、磁盘。

### M1-02 · 派生与图

- **读**：§4、§5.1、§5.3。
- **改**：`loomerto/model.py`（后半部分）；`tests/test_model.py` 追加。
- **做**：`effective_deps`、`effective`、`rollup`、`progress(plan, frame_id="")`、`stale_nodes(plan, hours)`（v1 `stale_blocks` 改遍历卡片）、`dep_graph`、`cycle`（照搬）、`guard_acyclic`、`upstream(plan, id) -> set`（沿有效依赖可达的卡片）、`check_loop(plan, src, dst)`（§5.1 三条，第 3 条返回 ⚠ 字符串而不是抛）、`edges(plan)`（§4.5）、`exec_brief` / `claim_of` 照搬、`participant_options` 改遍历节点。把 §3.5 的第 5、6 条接进 `validate`。
- **验收**：`python3 -m unittest tests.test_model -v`。测试至少覆盖：
  - 框级 dep：框 F deps X ⇒ F 里每张卡片的有效依赖含 X 的所有非取消后代卡片；
  - 等一个框 = 等它所有后代卡片，取消的不算；
  - `rollup` 的六条规则各一个用例；
  - loop 边不进环检测：A→B dep + B loops A ⇒ 无环；A deps B + B deps A ⇒ 抛错且消息含两个 id；
  - `check_loop`：目标不在上游 ⇒ 抛；目标是框 ⇒ 抛。

### M1-03 · store

- **读**：§2（全部）、§11。
- **改**：`loomerto/store.py`；新建 `tests/test_store.py`。
- **做**：
  1. 定位：`plan_path(name="")` 按 §2.2；`plans_root()`、`plan_file()` 保留；删掉「当前目录 plan.json」那条与 `plan_dir`、`TEMPLATE`。
  2. `md_path(path) -> Path` = 同目录 `<stem>.md`。
  3. `load(name)`：读 → §2.3 拒读 → `normalize_plan` → 返回。
  4. `dumps(plan) -> str`：§2.4 的紧凑序列化（自己写递归 encoder，别引第三方）；`save` 用它。
  5. `commit(name, plan, a=None)`：`model.validate(plan)` → 换 `rev`（照 v1：`uuid4().hex[:12]`）→ `updated_at` → `save` → 除非 `--no-render`，写 `<stem>.md`（调 `md.render_md`；M1-04 之前先 `try: from . import md` 失败就跳过，并在代码里标 `# M1-04`）。
  6. `list_plans(root) -> list[Path]`：§2.2 最后一段。
- **验收**：`python3 -m unittest tests.test_store -v`，至少覆盖：同目录 `PlanA.json`/`PlanB.json` 各自出 `PlanA.md`/`PlanB.md`；给目录给 `--plan` ⇒ `PlanError`；v1 文件（含 `tasks`）⇒ `PlanError` 且消息含「PLAN.md」；`dumps` 后 `pos` 在一行、`json.loads(dumps(p)) == p`；`validate` 失败时文件 sha256 不变。

### M1-04 · `md.py`

- **读**：§10、§11。
- **改**：`git mv loomerto/render.py loomerto/md.py` 后重写；删 `loomerto/assets/plan.html`、`loomerto/assets/canvas.html`；把 `loomerto/assets/theme.css` `git mv` 到 `docs/theme-v1.css`（留给 M3-03）；`pyproject.toml` 删 `package-data` 段；去掉 store 里的 `# M1-04` 兜底。新建 `tests/test_md.py`。
- **做**：`render_md(plan, *, name) -> str`，内容按 §10 五节、顺序固定；`STATUS_ZH` 保留（显示状态中文名，补齐 `waiting`/`ready`）。
- **验收**：`python3 -m unittest tests.test_md -v`（用一份手搭的三层嵌套 plan：断言五个小标题都在、loop 行用 `-.->|loop|`、输出里没有 `pos`）；`ls loomerto/assets 2>/dev/null` 无输出。

### M1-05 · edits A：增 / 改 / 状态 / 指派

- **读**：§3.3、§6 表中 `add_node` `edit_node` `set_field` `set_status` `assign` `clear_exec` 六行；当前 `edits.py` 的 `set_status`（语义要原样保留）。
- **改**：`loomerto/edits.py`；新建 `tests/test_edits.py`。
- **做**：六个函数；`NODE_SET_FLAGS` = v1 `BLOCK_SET_FLAGS`；`check_set_flags(status, given)`；私有 `_to_frame(plan, id)` / `_to_card(plan, id)` 实现 §3.3（M1-06 也会用）。框 `set_status cancelled` 的递归。所有函数第一步 `id = model.resolve(plan, ref)`。
- **验收**：`python3 -m unittest tests.test_edits -v`，覆盖：往 `running` 的卡片里 `add_node` ⇒ 抛；往 `pending` 卡片里加 ⇒ 它变框、`status==""`、`runs` 还在；框 `set_status running` ⇒ 抛；框 `cancelled` ⇒ 后代开着的卡片都 `cancelled`、`done` 的不动；v1 已有的状态语义（换人清线程、review 打回写 feedback、done 收 artifact）各一例。

### M1-06 · edits B：结构

- **读**：§3.3、§6 表中 `move` `group` `ungroup` `remove` `restore` `layout` 六行、§6.1。
- **改**：`loomerto/edits.py`；`tests/test_edits.py` 追加。
- **做**：六个函数。坐标换算：子节点相对父框 ⇒ 绝对坐标 = 沿祖先链累加 `pos[0..1]`；`pos` 为 `None` 的节点换算时按 `None` 保留（交给画布排版）。
- **验收**：单测覆盖 —— move 后 id 不变；移进自己的后代 ⇒ 抛；移空一个框 ⇒ 它变回 `pending` 卡片；move 造成「dep 祖先」⇒ 抛且 plan 未被改（深拷贝比对）；`group` 后子节点绝对坐标不变（换算前后比对）；`ungroup` 后原来 deps 框的节点改为 deps 它的每个直接子；`remove` 被引用无 force ⇒ 抛并列出引用者；`remove`→`restore` 往返后 `plan` 与删前 `==`（除 `log`）；`layout` 不增加 `log` 长度、给不存在的 id 不报错。

### M1-07 · edits C：边与循环

- **读**：§5（全部）、§6 表中 `set_deps` `set_loops` `reject` `bypass` `insert` 五行；当前 `edits.py` 的 `bypass_block` / `insert_block`。
- **改**：`loomerto/edits.py`；`tests/test_edits.py` 追加。
- **验收**：单测覆盖 §13 场景 2–6 的模型层版本（不经 CLI）：打回后 A `pending`+`rework==1`+`feedback`、B `pending` 且 `effective==waiting`；第二次打回后 B `blocked`；中间卡片（A→M→B，B loops A）打回后 M 也回 `pending`；不在循环段上的 done 卡片不动；`set_deps` 成环 ⇒ 抛且未改；`bypass` 一个被 loop 指向的卡片 ⇒ loop 改指它的前置。

### M1-08 · CLI：全局旗标 + `plan` / `node`

- **读**：§2.2、§9；当前 `cli.py`（保留它的输出风格与退出码约定）。
- **改**：`loomerto/cli.py`（大改：删掉 `task`、`block`、`open` 组）。
- **做**：§9 的 `plan new` 与 `node` 组全部动作。`--plan` 模式下不写名字、`--plans-root` 模式第一个位置参数是名字（沿用 v1 的 `shift` 机制）。`node add` 成功时 stdout 打印 `✓ n-xxxxxxxx 标题`。`node tree` 的格式按 §9。
- **验收**（在临时目录）：
  ```bash
  T=$(mktemp -d); L="python3 -m loomerto --plan $T/PlanA.json"
  $L plan new --title 测试 && $L node add --title 框 && $L node tree
  $L node add --title 子 --parent "$($L node tree | awk 'NR==1{print $1}')" && $L node tree
  $L task add x; echo "exit=$?"     # 期望 exit=2（argparse 不认识 task）
  python3 -m loomerto --help | grep -E "^\s+(task|block|open)\b"; echo "grep=$?"   # 期望 grep=1
  ```

### M1-09 · CLI：看与出

- **读**：§9 第一张表；当前 `cli.py` 里对应命令与 `workers.py`。
- **改**：`loomerto/cli.py`、`loomerto/workers.py`。
- **做**：`list` / `current` / `check`（含 `validate` 全部不变式 + v1 已有的图质量规则里仍适用的那些；不适用的删掉并在交付说明里列出）/ `workers`（遍历卡片的 `exec`）/ `digest` / `note` / `render`。
- **验收**：`grep -nE "tasks|blocks|T-0|#B-" loomerto/*.py` 无输出；`python3 -m unittest discover -s tests -v` 全绿。

### M1-10 · 端到端脚本

- **读**：§12、§13 场景 1–8。
- **改**：新建 `tests/e2e_cli.sh`（bash，`set -euo pipefail`，只用 `python3 -m loomerto` 与 coreutils）。
- **做**：逐条实现场景 1–8，每步断言（`grep` 输出或 `python3 -c` 读 json 断言）；场景 6 用 `shasum -a 256` 比对前后。脚本最后打印 `E2E OK`。
- **验收**：`bash tests/e2e_cli.sh` 输出末行 `E2E OK`、退出 0。

### M1-11 · 文档

- **改**：`docs/cli-reference.md`（按 §9 重写）、`docs/architecture.md`（模块表：`md.py` 替 `render.py`、删 assets、加 `desktop/` 一行、依赖方向更新）、`REQUIREMENTS.md`（§1 硬约束 1 改为「`<名字>.json` 唯一真相，派生只有 `<名字>.md`」；R-02 等受影响条目在「现状」列标注 v2；文档地图加本 spec）、`README.md`（文档地图与安装段落里的 `plan.json` / `open` / `task` / `block` 字样）。
- **验收**：`grep -rnE "plan\.html|plan\.canvas|loomerto open|block set_status|task add|T-00[0-9]#B" README.md REQUIREMENTS.md docs/` 只剩明确标注「v1（历史）」的段落。

### M1-12 · skill：`loomerto-plan`

- **改**：`skills/loomerto/references/loomerto/loomerto-plan/` 下 `SKILL.md` 与 `references/*.md`（`scripts/plan.py` 薄壳只改帮助文案里的路径说明）。
- **做**：所有命令示例换成 `node …`；ref 写法换成 id/前缀；「任务=泳道 / 块」的概念段改写成「节点：框与卡片」；新增 loop / `node reject` 一节；删掉 expand/compress/open/plan.html 相关段落；`plan-hard-rules.md` 的「改 plan 只走 loomerto 命令」保留，并加一条「agent 不写坐标」。顺手处理 `TODO.md` 第 3 条（SKILL.md 超限）：把概念部分拆进 references。
- **验收**：`grep -rnE "block |task |T-0|#B-|expand|compress|loomerto open|plan\.html" skills/loomerto/references/loomerto/loomerto-plan` 无输出；SKILL.md 正文 ≤ 500 行。

### M1-13 · skill：remind / intake / check-commands / check-temps

- **改**：这四个 skill 目录下的 `SKILL.md`、references、以及 `check_plan_node_commands.py`、`check_plan_temp_hygiene.py`。
- **做**：两个脚本的 plan 定位改成「给 `.json` 文件路径，或 `--plans-root` + 名字」，遍历改成 `nodes` 里的卡片（脚本是独立脚本，**不 import loomerto**，自己写 20 行遍历即可）；输出里的块 id 换成节点 id。顺手处理 `TODO.md` 第 1、2 条（脚本 > 20000 B）。
- **验收**：用 M1-10 生成的 `PlanA.json` 跑两个脚本各一次，退出码符合各自 SKILL.md 的约定；同上的 grep 无输出。

### M1-14 · skill：maintain-the-loomerto-package

- **改**：该 skill 目录全部 md。
- **做**：按新的模块表（M1-11 的 architecture.md）与五道闸（§12）重写；处理 `TODO.md` 第 4、9、10 条。
- **验收**：同上 grep 无输出；正文 ≤ 5000 tok（用 `wc -w` 粗估 ≤ 3500 词/字）。

---

# M2 · HTTP 服务 v2

### M2-01 · serve 骨架 + 读

- **读**：§7.1、§7.2。
- **改**：`loomerto/serve.py`（重写）、`loomerto/cli.py`（加 `serve` 命令）；新建 `tests/test_serve.py`。
- **做**：`http.server.ThreadingHTTPServer`；握手行；token 校验（403）；CORS；stdin EOF 退出（后台线程读 `sys.stdin`）；`--no-stdin-watch`；`view(plan, name, file) -> dict`（全部派生调 `model`，字段严格按 §7.2 的 `View` 类型）。
- **验收**：`python3 -m unittest tests.test_serve -v`（测试里 `subprocess.Popen` 起服务、读首行、带/不带 token 各请求一次、关 stdin 后 2s 内进程退出）。

### M2-02 · 写

- **读**：§7.3、§6。
- **改**：`loomerto/serve.py`；`tests/test_serve.py` 追加。
- **做**：`POST /api/op` 的 12 个 op；每个 op 返回 `undo`；`layout` 免 rev；出错 400 且磁盘不变；rev 过期 409。
- **验收**：单测对每个 op 各一条「成功 + 执行返回的 undo 后回到原状（比对 `nodes`）」；`reject` 与 `ungroup` 的 `undo` 为 `null`；过期 rev 的 `edit` ⇒ 409 且 sha256 不变；过期 rev 的 `layout` ⇒ 200。

### M2-03 · 事件

- **读**：§7.4。
- **改**：`loomerto/serve.py`；`tests/test_serve.py` 追加。
- **验收**：单测：连上 `/api/events`，另起子进程跑 `loomerto --plan … node set_title …`，1s 内收到 `event: changed` 且 `rev` 与文件一致；无 token ⇒ 403。

### M2-04 · 协议文档

- **改**：新建 `docs/desktop-api.md`（§7 的协议 + 每个 op 的请求/响应示例，示例要从 M2-02 测试里真实跑出来的贴）；`git rm docs/canvas-sync.md`；`docs/architecture.md` 与 `REQUIREMENTS.md` 里指向 canvas-sync 的链接改过来。
- **验收**：`grep -rn "canvas-sync" --include=*.md .` 无输出。

---

# M3 · 只读画布（先在浏览器里跑）

> 前端约定：TypeScript `strict: true`；组件一个文件一个；颜色只写 `var(--…)`；不写 `any`（确需时 `unknown` + 收窄）。

### M3-01 · 脚手架

- **读**：§8.1、§8.2。
- **改**：新建 `desktop/`。
- **做**：
  1. `npx create-electron-app@latest desktop --template=vite-typescript`；加 React、`@xyflow/react`、`zustand`、`elkjs`、`vitest`、`@playwright/test`，全部**精确版本**；`desktop/.nvmrc` 写当前 Node LTS。
  2. 目录：`src/main/`、`src/preload/`、`src/renderer/`（React 入口）、`src/shared/`（类型）。
  3. **浏览器开发模式**：`npm run dev:web` 只起 Vite 渲染进程，地址栏带 `?port=<p>&token=<t>` 时直接连那个 serve（不经 Electron）；没有参数显示「请从 Electron 打开或带 port/token」。
  4. 脚本：`dev`（Electron）、`dev:web`、`typecheck`（`tsc --noEmit`）、`test`（vitest）、`e2e`（playwright）、`make`。
  5. 根 `.gitignore` 加 `desktop/node_modules`、`desktop/out`、`desktop/.vite`。
- **验收**：`cd desktop && npm ci && npm run typecheck && npm test`；`npm run dev:web` 后浏览器打开显示提示文案。

### M3-02 · API 客户端 + 订阅 + 仓库

- **读**：§7.2–§7.4、`docs/desktop-api.md`。
- **改**：`desktop/src/shared/api-types.ts`（照 §7.2 / §7.3 手写类型）、`desktop/src/renderer/api.ts`、`desktop/src/renderer/store.ts`。
- **做**：`getPlan()`、`op(body)`（返回判别联合 `{ok:true,…}|{ok:false,status:400|409,error}`）、`subscribe(onChanged)`（`fetch` + `ReadableStream` 解析 SSE、断线 1s/2s/4s… 最长 10s 退避重连）；zustand 仓库持有 `view`、`rev`、`selection`、`undo/redo` 栈（栈先留空实现，M4-05 填）。
- **验收**：`npm test`（vitest，对 SSE 解析器喂分片的字节流；对 `op` 的三种响应做 mock）。

### M3-03 · 卡片与框

- **读**：§8.4。
- **改**：`desktop/src/renderer/canvas/CardNode.tsx`、`FrameNode.tsx`、`Canvas.tsx`、`toFlow.ts`（View → React Flow nodes/edges 的纯函数）、`theme.css`（从 `docs/theme-v1.css` 取走并 `git rm` 原件，补深色变量）。
- **做**：嵌套用 `parentId`；子节点要排在父之后（React Flow 要求）；框折叠是本地状态（先存内存，M5-03 持久化）。
- **验收**：`npm test` 覆盖 `toFlow`（三层嵌套的顺序、`parentId`、相对坐标原样传递、`pos==null` 的节点被标记为待排版）；手测：对 M1-10 生成的 plan 起 serve，`dev:web` 打开能看到框套卡片，截图附在交付说明。

### M3-04 · 边

- **读**：§4.5、§8.4 边那一条。
- **改**：`desktop/src/renderer/canvas/edges/DepEdge.tsx`、`LoopEdge.tsx`、`toFlow.ts`。
- **验收**：`npm test`（`toFlow` 产出的边数 = `view.edges` 数、kind 正确）；手测截图：一条实线 dep、一条虚线弧形 loop。

### M3-05 · 自动排版

- **读**：§8.8、§3.2 默认尺寸。
- **改**：`desktop/src/renderer/canvas/autoLayout.ts`。
- **做**：纯函数 `placeUnplaced(view) -> {id, pos}[]`（两种情形都要实现）；拿到非空结果发一次 `layout`。
- **验收**：vitest：全未放置 ⇒ 所有节点有 pos 且同组卡片互不重叠；部分放置 ⇒ 已放置节点的 pos **完全不变**、新节点在其前置右侧 48px；框尺寸能包住所有子节点。

### M3-06 · 实时刷新

- **读**：§8.6。
- **改**：`desktop/src/renderer/store.ts`、`Canvas.tsx`。
- **验收**：手测脚本写进交付说明：开着画布，另一终端 `node set_title` / `node add` / `node remove`，画布 1s 内更新、视口不动、选中保持（被删的除外并出 toast）。

---

# M4 · 编辑

### M4-01 · 拖动 / 尺寸 / 换父

- **读**：§8.5 表中「拖动」「换父」「改框尺寸」「框自动长大」四行。
- **改**：`Canvas.tsx`、新建 `desktop/src/renderer/canvas/dropTarget.ts`（纯函数：给指针绝对坐标与所有框的绝对矩形 ⇒ 最深的合法框）。
- **验收**：vitest 覆盖 `dropTarget`（排除自己与后代；重叠时取最深）；手测：拖一张卡片后 `git diff` 只有 `pos`/`rev`/`updated_at`（§13 场景 9 前半）。

### M4-02 · 新建 / 打包 / 拆框 / 删除

- **读**：§8.5 对应行、§6.1。
- **改**：`Canvas.tsx`、新建 `desktop/src/renderer/dialogs/ConfirmDelete.tsx`。
- **验收**：手测清单（每项截图或录屏）：根上双击建卡；框内双击建子卡；⌘G 打包三张卡；⇧⌘G 拆框；删一个被引用的卡片时确认框列出引用者。

### M4-03 · 画边 / 删边

- **读**：§8.5「画 dep 边」「画 loop 边」「删边」三行、§5.1。
- **改**：`Canvas.tsx`、工具栏组件。
- **验收**：手测：画一条会成环的 dep ⇒ toast 显示服务端给的环路、边消失；⌥ 拖出 loop 边到非上游节点 ⇒ toast 拒绝。

### M4-04 · 详情面板

- **读**：§8.5「改字段 / 状态」「打回」两行；`specs/canvas-participant-picker-and-single-save.md`（v1 的 R-24 单一保存语义，要原样继承）。
- **改**：新建 `desktop/src/renderer/panel/DetailPanel.tsx` 及子组件。
- **做**：字段：标题、做什么、输入、输出、命令、判据（增删行）、类型、认领人（`participant_options` 做 datalist）、状态 + 在做的人（`doing`，按 `enums.set_flags` 决定是否可填）、`max_loops`；只提交改过的键；打回按钮；`runs` 历史只读列表；线程登记只读。框：只显示标题、做什么、认领人、汇总状态与进度，以及「取消整个框」。
- **验收**：vitest 覆盖「只提交改过的键」的 diff 函数；手测：改标题+状态一次保存，`log` 里两条、`runs` 一条。

### M4-05 · 撤销 / 重做

- **读**：§8.7。
- **改**：`store.ts`、键盘处理。
- **验收**：vitest（对 mock API）：每类 op 撤销后再重做，发出的请求序列正确；`undo:null` 的 op 清空重做栈；撤销失败时栈项被丢弃。

### M4-06 · 冲突与失败

- **读**：§7.3 409/400、§8.5 最后一段、§8.6。
- **改**：`store.ts`、新建 `Toast.tsx`、`DetailPanel.tsx`（黄条）。
- **验收**：手测：面板改到一半，另一终端改同一节点 ⇒ 出黄条；坚持保存 ⇒ 409 提示。

### M4-07 · 浏览器 e2e

- **改**：新建 `desktop/e2e/web.spec.ts`、`desktop/playwright.config.ts`。
- **做**：测试里用 `child_process` 起 `python3 -m loomerto --plan <临时>/PlanA.json serve --port 0 --no-stdin-watch`，读握手行，开 `dev:web?port&token`；用 Playwright 的真鼠标（`page.mouse.down/move/up`）拖卡片、拖进框、画一条边、⌘Z；每步后读 json 断言。
- **验收**：`cd desktop && npm run e2e` 全绿。

---

# M5 · Electron 外壳

### M5-01 · 主进程 + serve 生命周期

- **读**：§8.2（含「`loomerto` 在哪」）。
- **改**：`desktop/src/main/serve.ts`、`desktop/src/main/locate.ts`、`desktop/src/preload/index.ts`。
- **验收**：vitest 覆盖 `locate`（mock fs 与 shell）；手测：`npm run dev` 打开 M1-10 的 plan 能正常显示；关窗后 `pgrep -f "loomerto.*serve"` 无残留。

### M5-02 · 多窗口

- **读**：§8.3。
- **改**：`desktop/src/main/windows.ts`。
- **验收**：手测：同一文件打开两次只有一个窗口（第二次聚焦）；两个不同文件两个窗口，各自 serve 进程独立。

### M5-03 · 侧栏 + 持久化 + 菜单

- **读**：§8.3 全部。
- **改**：`desktop/src/main/state.ts`（读写 `userData/state.json`，原子写）、`desktop/src/main/menu.ts`、`desktop/src/renderer/sidebar/Sidebar.tsx`。
- **验收**：手测：打开两个 plan、各自缩放到不同位置、退出重开 ⇒ 两个窗口恢复且视口一致；删掉其中一个文件 ⇒ 侧栏灰显 + 移除；「新建 plan…」产出 `<名字>.json` 与 `<名字>.md`。

### M5-04 · 安全加固

- **读**：§8.9；`desktop-security-electron` skill（若可用）。
- **改**：`desktop/src/main/index.ts`、`forge.config.ts`、`index.html`（CSP meta）。
- **验收**：手测：渲染进程控制台里 `require` 未定义、`window.open('https://example.com')` 被拒；`npx @electron/fuses read --app out/…` 显示三个 fuse 的期望值。

### M5-05 · 打包

- **读**：§8.10。
- **改**：`forge.config.ts`、新建 `desktop/README.md`（装 loomerto、`npm ci`、`npm run make`、首次打开右键「打开」）。
- **验收**：`npm run make` 产出 `out/make/*.dmg` 与 `*.zip`；装上后从 Dock 启动（不是从终端）能找到 `~/.local/bin/loomerto` 并打开 plan。

### M5-06 · Electron e2e

- **改**：新建 `desktop/e2e/electron.spec.ts`。
- **做**：Playwright `_electron.launch` 启动打包前的应用，打开临时 plan，跑 §13 场景 9（拖动后 diff 只改 `pos`；另一进程改标题后画布 1s 内更新）。
- **验收**：`npm run e2e` 全绿。

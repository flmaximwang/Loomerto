## 坑

- **新分支从哪起，先看「装好的 CLI 跑的是哪一支」**：包是 editable 装的，跑的就是主 checkout 当前所在的分支
  （`git worktree list` 第三个字段）。这个仓库常态是：`main` 落后于一条特性分支（如 B 档「块的形状收口」在
  `feat/block-schema`、`model` 里有 `BLOCK_FIELDS` / `new_block`，`main` 上**没有**）—— 从 `main` 起新 worktree
  会直接 `ImportError: cannot import name 'new_block'`。要「改完立刻能在装有工具的会话里试」，就**从当前装在跑的那支起**
  （`git worktree add -b feat/<名> <路径> feat/block-schema`），并在提交信息/文档里写清它依赖哪一支、还没合进 main。
- **这个包没有 `tests/` 目录**：验收按 `docs/architecture.md` §7 的「五道闸」**现场跑**（① 名字自检：静态查用到的名字能不能解析
  ② 对拉：与 `git show <旧 sha>:<路径>` 比 stdout/stderr/退出码与产出文件 ③ 真实库只读回归 ④ install + 薄壳回读
  ⑤ 前端改动要真浏览器拖一次），**不落成测试文件**（别自创 `tests/`）；跑验收的脚本放 `$BH_AGENT_WORKSPACE` / 临时目录即可。
- **派生状态的规则不止一处实现**：`model.block_effective`（包）+ 仓库 `loomerto` 包里 `assets/` 下那份看板模板的内联 `eff()`
  （看板内联 JS 自己算一份）+ `canvas.html` 里解释派生原因的那段。动「派生状态」的定义必须三处一起改，
  否则看板与画布会各显示一套。**改完还要重渲真实 plan 的派生视图**（`loomerto --plans-root <库> render <slug>`）：
  `plan.html` 里嵌着它自己那份 JS，不重渲的话旧文件会一直按旧规则显示（`plan.json` 一个字节都不会变 ——
  重渲只写 PLAN.md / plan.html / plan.canvas，可以拿渲染前后 `plan.json` 的 sha 证明这点）。
- **「状态 × 负责人」的单一来源 = `model.STATUS_OWNER`**（三档 never/required/may）：`待认领` 不能有负责人
  （定义就是「还没人接」）—— 所以 `pending` + 依赖就绪时，**有 `owner` 派生为 `claimed`（已认领）**、
  无 `owner` 才是 `ready`；`等前置`/`待批准` 可以有 owner（先派活、写「等谁点头」）。改这条要有心理准备：
  真实库里凡是「早就派好 owner、依赖又已就绪」的块，显示状态会整批从 待认领 变 已认领 —— 先拿真数据
  逐份对拉（旧 vs 新 `current` / `check` / `workers`）再落，别拿一块自造的 plan 就宣布改好了。
- **「在途」要按存储状态判，不能按派生状态判**：派生出来的「已认领」没有开工时刻（`status_since` 还是
  当初置 `pending` 那一刻）、也没登记线程 —— 拿它进 `stale_blocks` / `workers` 只会误报。这两处改回
  `b["status"] in ACTIVE`（存储）之后，真实库的告警面就与改前逐字节相同。
- **命令面按对象分组是这个包的定局**：一级名字只留 `plan` / `task` / `block`（分组）+ `note` / `digest` /
  `render` / `check` / `current` / `workers` / `list` / `open`。**新加一个动作时先问它作用在谁身上**：
  块的事进 `block`（`move` 就是这么从顶层挪进来的）。顶层再长出一个平铺动作，下一轮就得重排一遍。
- **另一个会话可能同时在 main 上推提交**：动手前 `git log --oneline -1 main` 记下基线；收工前再看一眼。
  真撞上了就在分支上 `git merge main` 解冲突（**别 rebase** —— 那会改写代码提交的 sha，而文档里「代码基线」
  引的就是那个 sha）。冲突处通常是：parser 段、`edits` 里新函数的位置、`REQUIREMENTS.md` 的编号
  （**新条目要接着 main 的最大号往下排**，我曾把 R-16 用重了）。
- **跑「另一份 checkout 的代码」用 `python3 -m` 必须看 cwd**：`-m` 把**当前目录**放在 `sys.path` 最前，
  在 main 的目录里给 `PYTHONPATH=<worktree>` 跑的还是 main 那份 —— 对拉会变成自己跟自己比。要么 `cd` 进
  那份 checkout，要么在没有 `loomerto/` 目录的地方跑；**先证明跑的是哪一份**（两份 `--help` 的子命令表必须不同）。
- **删 / 改一个旗标前先搜谁在用**：除了这个 clone，还要扫别的 profile 的 skill 正文与 cron
  （`search_files pattern="<旗标>" file_glob="*.md" path=<profiles 根>`）。**不要 `grep -r` 扫整个 profile 目录**：
  里面躺着几十 MB 的单行 JSON（curator 账本 / 缓存索引），实测挂到 180 s 超时还什么都拿不到。
- **`save()` 要写回「解析后的数据文件路径」**，不是 `<plan_dir>/plan.json`：数据文件叫什么名都行
  （`mine.json`），视图与它同目录；只改定位、忘了改写入，那份自定义名字的数据就会与它的三视图分家。
- **`shift` 为空的子命令（`current` / `check` / `workers` / `render` / `digest` / `list` / `task` / `block`）
  不要顺手清 slug**：那会让「还按老写法写 slug」的调用静默丢参数。
- **`python3 -m loomerto` 只在仓库根（或 `PYTHONPATH` 指过去）可用**，任何目录可用的是装好的 CLI
  （`loomerto` / `plan` 两个入口名）。换目录后 `No module named loomerto` 是正常的，不是包坏了。
- **块的形状只有一处声明 = `model.BLOCK_FIELDS`**（键 → 默认值/工厂）：加一个块级键就改它一处 ——
  `model.new_block()` 是建块的**唯一**字面量（`edits.add_block` / `edits.insert_block` / `cli` 的 `expand`
  步骤与 `compress` 合并块都调它）。别在命令层再手写块字典：以前四处各抄一遍，实库里 9 份 plan 的
  158 个块没有 `exec` 就是这么来的。只有 `compress` 写的 `folded_from` 归 `BLOCK_HISTORY_FIELDS`，
  **不进那张表**（不是每个块都有）。
- **老数据的缺键是「读时补齐、写时落盘」**：`store.load()` 调 `model.normalize_block()`（只补不改、不删），
  只读命令跑完文件一个字节不变；**写操作**才把这些键材料化进 `plan.json`。所以对拉改动前后时，
  老 plan 的文件里可能多出 `exec: {}`（语义 no-op）—— 别当成行为差异，要报就先说清是读出来的还是写出来的。
- **只动模型/内部实现时，`--help` 的子命令表两份一模一样**：对拉「跑的是哪一份代码」不能拿它当判据，
  要印这次真动过的记号（例：`python3 -c "from loomerto import model; print('BLOCK_FIELDS' in dir(model))"`）。
- **给用户的结论要先给现象与命令**：他问「支持吗 / 入口在哪」时，先给「存在 / 不存在 + 入口命令」+
  一条真跑过的输出，再讲机制与落点；他会直接拿着命令去试。
- **别把「只读的核查」变成待决项问他**：能自己跑掉的就跑掉（现状清单、回归、diff），
  留给他的只有「动哪份数据 / 定哪个口径」。

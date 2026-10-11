---
name: maintain-the-loomerto-package
description: "Use when 要改 loomerto 这个包（CLI 旗标、子命令、文档、需求）时用；含定位规则与验收矩阵。"
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [loomerto, package-development, cli, cross-harness, docs-discipline]
    category: loomerto
    related_skills: [loomerto-plan, loomerto-check-commands, author-a-skill-in-a-pack-repo]
---

# 改 loomerto 这个包（代码 / CLI / 文档 / 需求）

## When to Use

- 用户说「loomerto 支持这个功能吗」「给一个 cli 入口」「加一个子命令」「这个旗标是干嘛的」——要**改包**或
  按包的现状回答。
- 要动 `/Users/maxim/Repositories/Repo/Loomerto` 里的 `loomerto/*.py`、`docs/`、`REQUIREMENTS.md`、`skills/`。
- **不属于**：用这个工具去维护一份 plan（改状态 / 派活 / 发提醒 / 看图）→ `loomerto-plan` 那一族
  （族里另有 `loomerto-remind` / `loomerto-check-commands` / `loomerto-check-temps` / `loomerto-intake`）；
  把包仓库当「发 skill 的包」来写 skill 的通用流程 → `author-a-skill-in-a-pack-repo`（本 skill 只补它没有的
  包本体约定）。

## 硬规则（用户拍板，不许自行回退）

1. **一份 plan 在哪，永远由调用方说清**（包里不猜、也没有默认 harness 目录）：
   `--plan <plan 数据文件>`（= `$LOOMERTO_PLAN_FILE`；给目录也行）→ `--plans-root <目录>` + `slug`
   （= `$LOOMERTO_PLANS_ROOT`）→ 当前目录的 `plan.json`（存在才认）→ 都没有就**退 2 并把该给什么打印出来**。
   两个旗标都给时按 `--plan` 算（打一行 ⚠）。`new` 必须显式说落点。
2. **包里不许出现 profile / hermes / 任何 harness 的目录假设** —— 用户的原话是「不要支持 profile，
   我从来没有定义过 loomerto 就是给 hermes 用的」。所以：没有 `--profile`、没有 `$LOOMERTO_PROFILE`、
   没有 `plans_root()` 里推断 `~/.hermes/...`、没有按 profile 拼转录路径的助手（`hermes_home()` /
   `live_root()` / `transcript_path()` 这类一律不进包）。Hermes 侧的路径约定（plans 库位置、
   `cache/delegation/live/<deleg>/task-<n>.log`）写在 **Hermes 侧的 skill / 薄壳**里，由它显式传给包
   （例：`exec … --transcript <路径>`）。**这条是定位级约定**：只要包里留下一条 harness 假设，下一条就会跟进去。
3. 纯 stdlib、零依赖、`requires-python >= 3.9`；`plan.json` 是唯一真相，`PLAN.md` / `plan.html` /
   `plan.canvas` 永远是派生物（原子落盘）；**所有写入过 `store.commit()`** 一处。
4. 分层不许串：`model`（纯函数、不 print 不 exit、抛 `PlanError`）/ `store`（磁盘 + 唯一写入漏斗）/
   `render`（只返回字符串）/ `workers`（只读文件）/ `cli`（**唯一的 print 与退出码**）。
5. 文件模式（给了 `--plan`）下 **slug 可以省**：命令的第一个位置参数本来是 slug，此时位置参数**整体左移一位**
   （`block set_status <ref> <状态>` / `block set_<属性> <ref> <值>` / `task set_status <ref> <状态>` /
   `block show|remove|insert|bypass|move|expand|compress|deps <ref>` / `task show|remove <ref>` / `note <text>`）；
   多写一个 slug 退 2 并点明。**允许留空的格子**要在 `set_defaults(..., optional=[…])` 里登记：
   `block set_status … --unset` 不带状态、`block set_*` 只给 ref 不给值（= 清空那一格）——
   `_apply_file_mode` 的「少写了」检查会跳过 `optional` 里的 dest；不登记就会把合法的「留空」判成缺参数。
   **`nargs="*"` 的格子（如 `--deps` / `--add` / `set_audit` 的判据）不能放进 `shift`**：定长左移靠
   `getattr(a, d) in (None, "")` 判「这格空没空」，而列表**永远不等于 `None`** ⇒ 合法的「只给 ref、
   值留空」会被判成缺参数。这类命令**不走 `shift`**（在 `_SET_ATTRS` 里把键记进 `_NO_SHIFT`），
   在 `_apply_file_mode` 里单独给一段：把 `slug` 还成 `ref`、把 `ref` 并回 `value`，再清 `slug`
   （argparse 对 「`?` + `?` + `*`」是**贪心**的 —— 位置参数 ≥3 时第一格落 `slug`、第二格落 `ref`、
   其余全进 `value`，所以并回去的顺序别搞反）。命令仍要声明 `slug` 那一格，`value` 的默认值
   给 `[]` 而**不是** `None`（`cmd_*` 里 `a.value or []` 才稳）。
6. **`set_status` 一个入口管「状态 + 身份」，参数按状态卡**：`edits.BLOCK_SET_FLAGS` / `TASK_SET_FLAGS` 是唯一那张表
   （在途状态才收 `--by`/`--delegation`/`--task-index`/`--transcript`，`--artifact` 只在 `done`），
   命令层 `_apply_set` 先调 `check_set_flags` 再落 `set_status` —— 加状态或加旗标时**只改那张表**，
   命令与画布（`serve.py` → `set_status`）同受约束。状态本身不合法时 `check_set_flags` 要**直接返回**
   （让 `set_status` 报「状态只能是 […]」），不然 `table[status]` 会 KeyError。
   **其余属性一条命令一个**（`block set_title|set_doc|set_type|set_input|set_output|set_command|set_audit`）：
   形状统一 `block set_<属性> <ref> <值>`，全部走 `edits.set_field` 一处。加一条新属性 = ①
   `model.BLOCK_FIELDS` 加键（若还没有）② `cli._SET_ATTRS` 加一行（命令名那截 → 键 + 说明）③ 在
   `_parser()` 的 block 组里按想要的顺序加一句 `_add_set_attr(g, "<名字>")` —— 别再照抄一份 parser。
   值**整组替换**、留空 = 清空（标题除外），**没有变化 ⇒ 退 2 且一个字不写**（与 `set_deps` 同一纪律）。
7. 退出码：参数错 / 找不到对象 → **2**；`check` 有图错误 → **1**；`workers` 见 ⚠/❌ → **1**；其余 → 0。

## 改包的固定动作

1. **认门**：`cd /Users/maxim/Repositories/Repo/Loomerto && git status --short` 看有没有别人的在途改动
   （这套仓库常有别的会话在写）；包是 **editable 装**的（`uv tool install --editable <repo>` → 
   `~/.local/bin/{loomerto,plan}`），所以**改完代码立刻生效，不用重装**。
2. **改代码**：按上面第 4 条分层放。新的定位/派生逻辑先想清楚它属于哪一层（路径解析归 `store`，
   图/派生状态归 `model`，只有参数解析与打印归 `cli`）。
3. **真跑验收**（不许只看 diff）：文件模式与库模式**两条都要跑**，文件模式下**逐条**跑完 **34 个叶子命令**
   （= 31 条真命令 + `info`×2 / `threads` 三个别名；`plan new` / `task add|set_status|remove|show` /
   `block add|remove|insert|bypass|expand|compress|show|deps|assign|move|set_type|set_title|set_doc|set_status|set_input|set_output|set_command|set_audit` /
   `note|digest|render|check|current|workers|list|open`），再对**真实数据**做只读回归（`--plans-root <真实库> list / current / check / workers`）。
   改过名字的**每一组**都要扫旧名退 2（`block` 五个 + `task` 三个），别只扫刚动手的那一组。
   矩阵与边界用例、以及跑矩阵的三个现成形状（取旧版 / 断言器 / 子命令自检）见
   `references/loomerto-cli-location-and-cli-conventions.md` §3 与 §3.1。
   - **闸① 的子命令自检读 `parser._defaults["f"]`**：`set_defaults(f=…)` 落在 `_defaults` 里、**不是**一条
     action —— 遍历 `p._actions` 找 `f` 会把每个叶子都判成「没挂处理函数」（一片假红，白查一轮）。
     顺带**断言叶子总数**：不写这个数，改了命令面之后自检会假装通过。
   - **闸④/⑤ 起画布服务要 `PYTHONUNBUFFERED=1`**：`print` 到重定向文件是**块缓冲**的，后台起服务后那行 URL
     还压在缓冲区里、从日志里抽不出来，看着像「服务起不来」。`PYTHONUNBUFFERED=1 loomerto --plan <P> open --port 0 --no-open >log 2>&1 &`
     → `sleep 2` → 从 log 取 URL → `curl` 回读页面。**改页面资产不用重启服务**（每次 `GET /` 从磁盘重读），
     改 `serve.py` 才要重启；纯文字改动也要回读一次页面并数一遍反引号是否成对 —— 提示文字就嵌在模板字面量里。
4. **文档对齐**：`docs/cli-reference.md` 是**从 `argparse` 取出来的现状清单** —— 改了命令就必须改它，
   并把顶部那行**代码基线 sha** 换成这次代码提交的 sha；`docs/architecture.md` 里的跨 harness 约定
   （定位顺序、薄壳契约、验收四道闸）同步改。
5. **需求**：`REQUIREMENTS.md` 是**需求单一入口** —— 条目**只在用户明确说了或拍板之后**才增删，
   实现状态变了只改「现状」列、不新开一份清单。
6. **skill 与副本**：仓库里的 `skills/**/SKILL.md` 与 `skills/loomerto/references/loomerto/loomerto-plan/scripts/plan.py` 薄壳是发布源；改完把
   `default` profile 的**已装副本**也拉平，并 `diff -rq <repo 目录> <profile 副本目录>`（除 `.DS_Store`
   外应为空）。薄壳里「调用方自己给过旗标就不补」的那段判断要跟着新旗标改。
   **副本可能比仓库源「新」**（别的会话直接把段落写进 profile 副本，仓库那份还没回移植）：动副本前先
   `diff <repo>/SKILL.md <profile 副本>/SKILL.md` **看方向**，别整份 `cp`（那是把别人写进副本的段落抹掉）——
   把同一段改动**分别打在两份上**，再把「副本先行、diff 不为空是预期的、拿什么判据代替」写进
   `REQUIREMENTS.md` §4 的拉平记录。判据用「两份各出现几处新标识」（如 `grep -c "block deps"`），不是 diff 为空。
   **回移植前先核这条差异是不是「分支限定」的**：装好的 CLI 跑的是主 checkout 当前那一支，副本里可能写着
   只活在**别的特性分支**上的命令 —— 拿**装着的 CLI** 直接探一次那条命令（例：`loomerto block deps …`
   回 `error: invalid choice: 'deps'`）就能判；探不到就别把 B 支的文档搬进当前这支（那是把「别人分支上的
   功能」写成当前分支的现状）。
7. **提交分两个**：代码一个（`feat(...)`），文档 / 需求 / skill 一个（`docs:` 且信息里写明代码基线 sha）——
   基线 sha 只有在代码提交之后才存在。**不推远端，除非用户说**。
8. **用户说「把现在几个 feature 合并到 main 再推」时**：先给分支分档 —— `git rev-list --left-right --count main...<分支>`，
   两边都是 0 = 同一支；左 N 右 0 = main 领先（祖先，什么都不用做）；左 0 右 N = 可**快进**（`git merge --ff-only`）；
   两边都不为 0 = 真合并（`git merge --no-edit`，冲突集中在 `REQUIREMENTS.md` 编号 / README / docs / skill 正文）。
   合并前在每个 worktree 里 `git status --short` 确认没人在写。合完 **`main` 就是「装好的 CLI 现在跑的那份」**
   —— 那些「只活在特性分支上的命令」（如 `block deps`）要到合进 main 才对真机可用，profile 副本里的对应文档
   也才不再是空头；接着按第 6 条恢复拉平，再按用户指示推 `origin/main`。

## 改**命令面**（改名 / 重排 / 加一条命令） → 见 `references/plan-extra-改-命令面-改名-重排-加一条命令-.md`
## 出版面：改 skill 名 / 改 description → 见 `references/plan-extra-出版面-改-skill-名-改-descript.md`
## 「按对象拆 module / 重构包结构」这一类请求 → 见 `references/plan-extra--按对象拆-module-重构包结构-这一类请求.md`
## 坑 → 见 `references/plan-pitfalls.md`
## Support files

| 文件 | 承担什么 |
|---|---|
| `references/loomerto-cli-location-and-cli-conventions.md` | 定位解析顺序表、文件模式位置参数左移的实现要点（含 `optional` 格子）、改旗标时的验收矩阵与跑矩阵的现成形状（§3.1：取旧版 / 断言器 / 子命令自检）、改完拉平副本的命令 |
| `references/object-definition-and-operation-map.md` | plan / task / block 各自的定义在哪、操作落哪一层、按对象切不动的地方、与硬规则 4 的关系（回答「每个对象如何定义」「按对象分 module」时先看它） |

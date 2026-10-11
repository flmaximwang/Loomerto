# 披露链 · wt

## 术语表

- **文档 / 节点**：一份文件（`SKILL.md`、`references/*.md`、`scripts/*.py` 各算一份）。
- **根**：起点文档 = 一个 skill 的 `SKILL.md`。**根有几个 = 形状**（1 个 = 单入口，K 个 = 多入口）。
- **披露**：一份文档正文里真的写出了另一份文档的路径。**不算**披露：URL、代码围栏里的示例路径、没有目录的裸文件名。
- **L1/L2/…**：离根几步。根 = L1，根披露的 = L2，依次往下。
- **fanout**：这份文档披露了几份别的文档（某一层「摆出几项」= 该层各文档 fanout 之和）。
- **到不了根**：从任何根顺着披露都走不到它（没有任何文档点到它）。
- **超限**：超过它那类的上限（正文 ≥5000 tok 或 >500 行 · description >1024 字符 · 资源文件 >20000 B）。
- **悬空**：写了像本 pack 内的路径，但那个文件不存在。
- **external**：正文里提到的本 pack 之外的文件（不算披露、也不算错）。
- **近重复**：同层两份文档太像，Agent 会选错。

口径：**层 = 披露深度**（L1 = 根文档 `SKILL.md`；L2 = L1 正文直接点到的文档；Ln+1 由 Ln 披露）。不是「按文件类型分三级」那套。

- 根：**1 个**（每个 skill 的 `SKILL.md` 是一个根；1 个 = 单入口形状）
- 文档节点：**18 份** · 层数：**4**
- 超限：**4** 份 · 到不了根：**4** 份 · 悬空引用：**2** 条 · 外部引用（不算披露）：**1** 条
- 根选择层近重复（Jaccard≥0.5·靠 description 选根才算）：**0** 对
- **待办 10 条 · 已豁免 0 条** → 见同目录 [`TODO.md`](TODO.md)（豁免规则 0 条，来自包里 `.skillweave/waivers.txt`）

## L1（1 份）

| 文档 | fanout（披露几项） | 体积 | 判定 |
|---|---|---|---|
| `skills/loomerto/SKILL.md` | 1 | 204 tok / 7 行 | 根 |

## L2（1 份）

| 文档 | fanout（披露几项） | 体积 | 判定 |
|---|---|---|---|
| `skills/loomerto/references/loomerto/INDEX.md` | 6 | 1507 B | ✅ |

## L3（6 份）

| 文档 | fanout（披露几项） | 体积 | 判定 |
|---|---|---|---|
| `skills/loomerto/references/loomerto/loomerto-check-commands/SKILL.md` | 1 | 2474 tok / 115 行 | ✅ |
| `skills/loomerto/references/loomerto/loomerto-check-temps/SKILL.md` | 1 | 2511 tok / 109 行 | ✅ |
| `skills/loomerto/references/loomerto/loomerto-intake/SKILL.md` | 1 | 2048 tok / 87 行 | ✅ |
| `skills/loomerto/references/loomerto/loomerto-plan/SKILL.md` | 3 | 19664 tok / 636 行 | ⚠️ 超限 正文 19664 tok > 5000; 正文 636 行 > 500 |
| `skills/loomerto/references/loomerto/loomerto-remind/SKILL.md` | 0 | 1446 tok / 77 行 | ✅ |
| `skills/loomerto/references/loomerto/maintain-the-loomerto-package/SKILL.md` | 7 | 8036 tok / 236 行 | ⚠️ 超限 正文 8036 tok > 5000 · ❌ 有悬空引用 |

## L4（6 份）

| 文档 | fanout（披露几项） | 体积 | 判定 |
|---|---|---|---|
| `skills/loomerto/references/loomerto/loomerto-check-commands/scripts/check_plan_node_commands.py` | 0 | 22367 B | ⚠️ 超限 22367 B > 20000 B |
| `skills/loomerto/references/loomerto/loomerto-check-temps/scripts/check_plan_temp_hygiene.py` | 0 | 22401 B | ⚠️ 超限 22401 B > 20000 B |
| `skills/loomerto/references/loomerto/loomerto-intake/references/read-only-evidence-recipes.md` | 0 | 2950 B | ✅ |
| `skills/loomerto/references/loomerto/loomerto-plan/scripts/plan.py` | 0 | 3486 B | ✅ |
| `skills/loomerto/references/loomerto/maintain-the-loomerto-package/references/loomerto-cli-location-and-cli-conventions.md` | 0 | 12331 B | ❌ 有悬空引用 |
| `skills/loomerto/references/loomerto/maintain-the-loomerto-package/references/object-definition-and-operation-map.md` | 0 | 5340 B | ✅ |

## 到不了的文档（没有任何根披露到它们）

| 文档 | fanout（披露几项） | 体积 | 判定 |
|---|---|---|---|
| `skills/loomerto/references/loomerto/loomerto-check-commands/test-prompts.json` | 0 | 2517 B | ⚠️ 孤儿 |
| `skills/loomerto/references/loomerto/loomerto-check-commands/test-results.md` | 1 | 1415 B | ⚠️ 孤儿 |
| `skills/loomerto/references/loomerto/loomerto-check-temps/test-prompts.json` | 0 | 2586 B | ⚠️ 孤儿 |
| `skills/loomerto/references/loomerto/loomerto-check-temps/test-results.md` | 1 | 1373 B | ⚠️ 孤儿 |

## 根选择层（L0）

根不是被谁披露出来的，而是**靠 description 选出来的** —— 所以它单独一层，混进 Ln 会让两个形状的层数不可比。

| 根 | description 字符 | 常驻 tok | 同层近重复 |
|---|---|---|---|
| `skills/loomerto/SKILL.md` | 136 | 109 | — |

### 悬空引用

- `skills/loomerto/references/loomerto/maintain-the-loomerto-package/SKILL.md:93` → `scripts/plan.py`（不存在）
- `skills/loomerto/references/loomerto/maintain-the-loomerto-package/references/loomerto-cli-location-and-cli-conventions.md:158` → `scripts/plan.py`（不存在）

### 超限清单（带超多少）

- `skills/loomerto/references/loomerto/loomerto-check-commands/scripts/check_plan_node_commands.py` — 22367 B > 20000 B
- `skills/loomerto/references/loomerto/loomerto-check-temps/scripts/check_plan_temp_hygiene.py` — 22401 B > 20000 B
- `skills/loomerto/references/loomerto/loomerto-plan/SKILL.md` — 正文 19664 tok > 5000; 正文 636 行 > 500
- `skills/loomerto/references/loomerto/maintain-the-loomerto-package/SKILL.md` — 正文 8036 tok > 5000

## 口径

- 披露 = 正文里真实点到的另一份文档（先按该文件所在目录、再按 pack 根解析）；URL、围栏里的路径、没有目录部分的裸文件名都不算。
- caps（出处 `docs/contract.md` §4）：正文 <5000 tok / ≤500 行 · description ≤1024 字符 · 单份文件 ≤20000 B。
- 每个文档在表里恰好出现一次（由命令自己核对：节点数 == 表行数）。

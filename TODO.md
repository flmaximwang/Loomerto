# 待办 · wt

由 `weave.py disclosure` 生成。术语见 `DISCLOSURE.md` 头部术语表。

- **待办 10 条**（机械可判 0 · 待模型 10）
- **已豁免 0 条**（按包里 `.skillweave/waivers.txt`，共 0 条规则）

## 待办（照着做）

| # | 文档 | 问题 | 数字 | 建议动作 | 谁判 |
|---|---|---|---|---|---|
| 1 | `skills/loomerto/references/loomerto/loomerto-check-commands/scripts/check_plan_node_commands.py` | 超限 | 22367 B > 20000 B | 拆成多份（拆分方案交模型判断，按主题切不按标题切） | 待模型 |
| 2 | `skills/loomerto/references/loomerto/loomerto-check-temps/scripts/check_plan_temp_hygiene.py` | 超限 | 22401 B > 20000 B | 拆成多份（拆分方案交模型判断，按主题切不按标题切） | 待模型 |
| 3 | `skills/loomerto/references/loomerto/loomerto-plan/SKILL.md` | 超限 | 正文 19664 tok > 5000; 正文 636 行 > 500 | 拆成多份（拆分方案交模型判断，按主题切不按标题切） | 待模型 |
| 4 | `skills/loomerto/references/loomerto/maintain-the-loomerto-package/SKILL.md` | 超限 | 正文 8036 tok > 5000 | 拆成多份（拆分方案交模型判断，按主题切不按标题切） | 待模型 |
| 5 | `skills/loomerto/references/loomerto/loomerto-check-commands/test-prompts.json` | 到不了根 | 2517 B | 判：该被引用（加一条引用）· 该删 · 还是刻意的（免检） | 待模型 |
| 6 | `skills/loomerto/references/loomerto/loomerto-check-commands/test-results.md` | 到不了根 | 1415 B | 判：该被引用（加一条引用）· 该删 · 还是刻意的（免检） | 待模型 |
| 7 | `skills/loomerto/references/loomerto/loomerto-check-temps/test-prompts.json` | 到不了根 | 2586 B | 判：该被引用（加一条引用）· 该删 · 还是刻意的（免检） | 待模型 |
| 8 | `skills/loomerto/references/loomerto/loomerto-check-temps/test-results.md` | 到不了根 | 1373 B | 判：该被引用（加一条引用）· 该删 · 还是刻意的（免检） | 待模型 |
| 9 | `skills/loomerto/references/loomerto/maintain-the-loomerto-package/SKILL.md` | 悬空 | 1 条（93: `scripts/plan.py`） | **先判**：包内路径写错（改路径）· 还是在说包外文件（免检）→ 再动手 | 待模型 |
| 10 | `skills/loomerto/references/loomerto/maintain-the-loomerto-package/references/loomerto-cli-location-and-cli-conventions.md` | 悬空 | 1 条（158: `scripts/plan.py`） | **先判**：包内路径写错（改路径）· 还是在说包外文件（免检）→ 再动手 | 待模型 |

## 已豁免（匹配了 waivers.txt，不动它）

| 文档 | 问题 | 数字 | 理由 |
|---|---|---|---|
| — | — | — | 没有命中豁免规则 |

## 免检表长什么样（`<pack>/.skillweave/waivers.txt`）

```
# 一行一条：<glob>  两个以上空格  理由（# 开头是注释）
*/assets/*.png             给人看的图，Agent 不读
*/test-prompts.json        测试留档，本来就不该被披露
```

豁免是**包主的判断**，写在包里、工具只匹配；没写规则的按上面的建议动作办。

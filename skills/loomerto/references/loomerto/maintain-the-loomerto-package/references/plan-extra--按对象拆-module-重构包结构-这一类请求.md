## 「按对象拆 module / 重构包结构」这一类请求

用户说「把 plan / task / block 的定义与操作分 module 保存」「能不能整理成一个 class」「让整套代码更容易查看」时，
先查清楚**定义与操作现在在哪**再谈拆法，别顺手就搬：

- **只有 block 的形状有声明**（`model.BLOCK_FIELDS`）。**plan 的键只在 `cli.cmd_new` 的字典字面量里、
  task 的键只在 `edits.add_task` 的字面量里** —— 问「plan / task 是怎么定义的」，答案只能从建对象的那个
  函数里读出来，`model` 里找不到。回答这类问题时先给这两处的 `文件:行`。
- 对象清单（每个对象的定义在哪、操作落哪一层、以及**跨对象、按对象切不动的地方**）见
  `references/object-definition-and-operation-map.md`。函数名是主键、行号会漂：引行号前重跑
  `grep -n "^def " loomerto/cli.py` 与 `grep -n "^[A-Z_]* = " loomerto/*.py`。
- **「拆成对象模块」会动硬规则 4**（`model` 纯函数 / `edits` 唯一改动实现 要并进同一个对象模块，纯函数区与
  改动区同文件）—— 这条是用户拍板过的，**只能由用户重新拍板，不许自己放宽**；把它作为代价写进选项里，
  同时点明「`cli` 是唯一 print / 退出码」这一半保持不变。
- 这一族的选项空间就三档，别自造第四档：**拆模块**（两个大文件搬迁 + 全套验收重跑）/ **只给 plan、task 补形状
  声明**（照 `BLOCK_FIELDS` 的样子加 `PLAN_FIELDS` / `TASK_FIELDS` + `new_plan()` / `new_task()`，最小、不碰任何
  硬规则）/ **只出文档**（`docs/objects/<对象>.md`，代码不动）。
- **批量重组先获批准**（用户的规矩）：把现状表 + 三档代价问出去、拿到答复再动手；问之前先把只读的现状清单跑掉。

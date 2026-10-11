# loomerto

- **loomerto-check-commands** — Use when 要看一份 plan 每个节点有没有能直接跑的命令、可替换的变量有没有定义。缺命令或变量没定义就不批准（exit 1）。
  `loomerto-check-commands/SKILL.md`
- **loomerto-check-temps** — Use when 要看一份 plan 会不会留下没人清的临时文件。逐块扫生产证据 / 声明 / 收尾节点，给 ✅⚠️❌➖ 判定，并给出要补的 loomerto 命令。
  `loomerto-check-temps/SKILL.md`
- **loomerto-intake** — Use when 用户把你拉进别人正在跑的协作（come and help us）。先用秒级只读证据把状态写成记录。
  `loomerto-intake/SKILL.md`
- **loomerto-plan** — Use when 与人/其他 agent 协同时要维护一份共享 plan。用 loomerto CLI（loomerto --plans-root <库> <子命令> <slug>）改状态、渲染三视图、发提醒；plan.json 是唯一真相，只经 CLI 改。上下文变长（跨 3+ 回合 / 要向接手的人解释背景 / 多条线并行）时先把后续规划、子代理分配与执行落到 plan 上。
  `loomerto-plan/SKILL.md`
- **loomerto-remind** — Use when 要提醒人或别的 agent 按 plan 推进。先算清该谁动、只推一件事、带上 plan 路径；包含静默与去重纪律。
  `loomerto-remind/SKILL.md`
- **maintain-the-loomerto-package** — Use when 要改 loomerto 这个包（CLI 旗标、子命令、文档、需求）时用；含定位规则与验收矩阵。
  `maintain-the-loomerto-package/SKILL.md`

## 坑

- **新接上的 bot 头一两分钟会对所有频道返回 403 `Missing Access`（权限传播延迟）**，看起来非常像
  「私有 thread 没邀请它」或「服务器权限配错了」。先等 1–2 分钟重试，**不要**立刻去改服务器权限。
  判据：同一个 token 读一个它显然看得见的频道 `/channels/<id>/messages?limit=1`，从 403 变 200。
- 传播期内 `PUT /channels/<thread>/thread-members/@me` 也会 403，**它单独不能证明 thread 是私有的**。
- 判断「提醒通道真的通了」的唯一判据不是 `hermes send` 回显 `sent`，而是**回读那条消息的 author.id**
  等于本 profile bot 自己的 user id（`/users/@me`）。否则可能发成了别的 profile 的 bot。
- **`block set_status` / `task set_status` 不接受 `ready`/`waiting`（派生状态）；写 `pending` 让依赖去决定。**
- **`note --kind note` 不存在**：合法 kind = `summary|decision|reminder|created|task|block|status|insert|assign|deps|remove|move|bypass|expand|collapse|compress`。写 `note` 会退 2 并把这串候选列出来；记「这一段发生了什么」用 `--kind summary`。同族提醒：**任何旗标被拒时，先读它自己打印的候选清单再重试** —— 换一个近义词继续猜（note→summary）会白多烧一轮。
- **`block set_status … done` 的 `--artifact` 收多条：写完读回条数**（`block show <ref>`），只看到「✓ done」看不出少登记了哪条证据。
- **「待认领」= 依赖就绪 且没人接**（2026-10-08 起）：一个块只要有负责人，依赖一就绪就显示成**已认领** ——
  别指望「先派活、还显示待认领」。想让它回到待认领（谁都有空谁接）就 `block assign <ref> --unset`。
  反过来 `等前置` / `待批准` 照样可以有 owner（先派活、写「等谁点头」）。
  「已认领 / 进行中 / 待评审 却没有负责人」会被 `check` 报 ⚠（谁接的没说清）。
  连带：泳道摘要（`task show` 的 `[已认领…]`）会把「只有已派活未开工的块」算成进行中；悬置超时与
  `workers` 只数**存储**状态在途的块，派生出来的已认领不进这两份报告。
- **`block set_status` 会自动把 `--by` 写进 `exec.by`**（没给 `--by` 就落到 owner），所以「谁在做」不用另起一道仪式；
  **换人（`--by` 与原来不同）会连带清掉旧的 `delegation`/`transcript`** —— 旧线程不再代表这一块，这是故意的。
- **`workers` 是 `check` 的姊妹**：`check` 查图（环 / 悬空依赖 / 已认领却没写负责人），`workers` 查「干活的那个人」。
  只把 `⚠ 线程已结束` 与 `❌ 号记错` 当硬信号（退 1）；`❓ 看不到` 与 `➖` 是提示 —— 但它们出现时别默认「没事」，
  要说清是「看不到」还是「没在跑」。
- **线程号短命，别把它当档案号**：live 转录 7 天回收、跨机器的路径在这边根本看不到。要让后人知道
  「这块是谁做的、证据在哪」，写进 `artifacts` 与 run 的 `note`；线程登记只保证**现在**能查在动没在动。
- **改一条已经建好的接线走 `block deps`**：`--add` / `--rm` / `--deps`（整组替换，给空即清空），
  三道闸都在一处 —— 依赖必须**已经存在**（悬空依赖 `check` 会一直报）、引用当场规整成规范 id 并去重、
  改完**查环**（成环退 2 且一个字不写）。它是独立命令而不是给 `set_status` / `set_doc` 加旗标，因为「补一条依赖」
  改的是**接线**（前后顺序），不是字段。**`--rm` 一条本来就不等的 = 什么都没变 ⇒ 退 2**，别把它当成功
  （说明你写的 ref 或对象不对）。改完它会打一行 ⚠：所属任务的**任务级**依赖也算它的前置（`block show` 里
  标「任务级」的那几条）。换地方（换泳道）仍走 `block move`，换粒度走 `expand` / `compress`（顺手重接）。
- **一块要等齐 N 条上游时，`kind=review` 会永久留一条 ⚠**（`是评审块但没写 review_of`）：`--review-of` 只收**一条** ref ⇒ 等齐 N 条的那种校验/收尾步建 `kind=impl`，前置写成一个给全的 `--deps A B C`。别为了消掉告警硬给一个上游，那等于声明它只审那一条。
- **让 N 个下游块间接等一个共享前置：把前置挂到链上中间那一块**（`block add <中间块> --deps <共享前置>`，下游只等中间块），比逐个给下游挂 `deps` 干净（逐个挂要用 `block deps <下游块> --add <共享前置>`，各写一条接线，多一片线）。
- **`--deps` 是 `nargs='*'`：重复写多个 `--deps` 只有最后一个生效。** 一个块要多条前置，写成
  `--deps T-004#B-001 T-005#B-001 T-006#B-001`（一个旗标跟一串），**不要** `--deps A --deps B` ——
  后者静默只留 B，图上看起来「有依赖」，实际只等一条（本 skill 踩过：总校验块本该等齐 13 条归档块，
  结果只挂了最后一条，而 `check` 不会有任何意见）。同理 `--review-of` 也只看最后一次。
  同一族的 `--artifact`（只在 `done` 收，用来登记证据路径）也不要假设重复旗标会累积：**要登记多条证据，
  就一个旗标跟一串（`--artifact a b c`），或分几条 `block set_status … done --artifact <一个>`（每条追一条 run）；
  写完 `block show <ref>` 读回 artifact 的条数** —— 只回显「✓ done」看不出少了哪几条。
  **批量建块 / 批量写证据后的读回断言要查这些字段**，别只查「doc / done_when 非空」—— 接线与证据正是那一档漏掉的。
- **修接线错首选 `block deps <块> --deps A B C`（或 `--add` / `--rm`）**；**只是要摘掉一个中间块**（让它前面
  直接接后面）用 `block bypass <块>` —— 它顺手把引用改对，比删了再重建小得多。确实要重建整块时才用下面这招 ——
  **倒着 rm 再重建**（块被下游引用时正向删会被拒）：`block remove <收口块>` →
  `block remove <校验块>` → 用一次给全的 `--deps` 重建校验块（`--no-render`）→ 再重建收口块并 `--deps`
  指回校验块 → `render` + `check`。两块都还没开工（`blocked`）时无损；有 runs 的块要按
  `block set_status … <状态> --by <原 actor> --at <原 ISO> --note "<原 note>"` 把 run 补回。
- **`task` 只建不改：没有改名、没有重排命令。** 改泳道标题或调整泳道顺序只能改 `plan.json`：
  先 `cp plan.json plan.json.bak-<时间戳>`，只动那几个 `title` 字段 / `tasks` 列表顺序，再 `check` →
  `render`，然后**读回一条**确认（标题不是状态，不为它破坏「状态只经 CLI 改」，但改完必须 check）。
  任务顺序 = 看板泳道顺序 = 流程顺序；用户说「按 X 排序／我们先做最简单的」时，就是重排 `tasks` 列表，
  **块的 id 不动**（id 是位置不是身份），所以重排后要把「新顺序 ↔ 旧 T id」的对照念给用户。
- **`block compress --into` 落进一个有任务级依赖的任务会继承它的全部块**：`block_deps` 会把落点任务的
  任务级前置展开成「那些任务的每一个块」，所以压出来的块可能凭空多等一批块、甚至成环。
  报错里会点名是哪个任务级依赖；换个落点或用 `--keep-task` 即可。
- **`new --owner "you=…"` 传了也没用**：`cmd_new` 在循环之后无条件再 `add_participant(plan, "you=human:本人")`，
  把你刚填的 label/channel 覆盖掉（同名 id 先删后加）。要让人类参与方带上 Discord 身份，就**另起一个 id**
  （如 `stronghold=human:Stronghold3369@discord:<thread>`），块上仍用 `you` 当 owner（digest `--to you` 会显示「你」）。
- **别把「涉及删除」自动升级成用户审批。** 用户的规矩是「删/清理前逐项证明内容已在别处存在」——
  举证满足即可由执行方完成。把这条写进块的 `done_when`（如「逐个 cmp 证明目标处内容逐字节相同才删」），
  而不是把块置 `blocked` 等用户点头：挡得过头会把「其实没事」做成待决项，用户会反问「为什么要我审核」。
  **他明确批准、但仍要前置时**（原话「我直接批准，但是必须要前置完成才能执行」）：建删除块用
  `--deps <检查块> --status claimed --owner <执行方>`，【前置】写进 `doc`，**不要置 `blocked`** ——
  那等于把已经给出的批准又收回去。谁动手由他定（他自己删 / 让 agent 起后台进程删），删完回读
  「路径不存在 + 体积回收」再收工。
- **block 的 `doc` 写错了要改原文**（`block set_doc <块ref> "<新 doc>"` —— 整段替换），不要把更正只留在日志里：人和 agent 读的是
  html / PLAN.md 里的 doc 原文，日志里的「纠正」救不了他 —— 他会拿着错前提来问你。
- **改错时连带把过期的证据路径改掉**：`--artifact` 还指着被纠正前的旧路径，看板就挂一条假证据（路径已不存在）。已 `done` 的块再发一次 `block set_status <ref> done --artifact <新路径> --note "<为什么改>"` 即可（已 done 再收一次会追加一条 run，不报错）。
- **用户纠正「形状」时把它落成约定，别只改这一次**：路径层级 / 命名这类形状被纠正后，`note --kind decision` 记一条（「层级 = …」）并在块的 `doc` 里引用，否则下一批同类块还会照错的形状建。
- **用户报了几块就建几块；范围重叠用「显式排除」解决，不要提议合并**：两个删除块若按目录跑会互相吞（后一块的范围里包含前一块要处理的文件）。
  做法是在后一块的 `doc` 里写「本块删除范围 = … 减去 `<文件>`（由 `T-00N#B-00N` 单列处理）」并把它写进判据；
  把「要么合成一块」这种选项提出来会被否（用户原话「2 个 block，不是一个 block」）。同理，两块有先后就老实给 `--deps`
>   （事后要加用 `block deps <后一块> --add <前一块>`）。
- **`blocked` 在 html/canvas 里就显示为「待批准」**，所以「某块需要人点头」的表达方式就是把它置
  `blocked` 并把 owner 改成那个人；不要另外造一个「待批准」状态。
- **改状态前先看这个块是不是已经完成**：执行方与记录员同时动手会撞出「标记 blocked / 实际已 done」的
  矛盾（曾 15 秒内撞车）。落状态前先读最新 run 与 PLAN.md 的更新时间，再决定动不动它。
- **同一份 plan 上可能有人（另一个 profile / 另一个会话）在并行加块**：任务图会在你眼皮底下变（实测块数 46 → 56，
  多出两条新任务且已有块在 `running`）。所以 ① 汇报进度用**刚读到的实时数**，别用几轮前的；② 建块前先读一遍，避免重复建；
  ③ 不是自己建的块**只报告、不接管** —— 除非用户明确说「直接认领并执行即可」，那就是接手（哪怕它已被别人 `claimed` / 挂在 `running`）：
  `block set_status <slug> <ref> running --by <你>` → 干完按它自己的判据 `done --artifact … --note …`，**不换 owner**（除非用户要换）。
- **没有 `status` 子命令**：每轮汇报用的 `<已完成>/<总数>` 用 `loomerto list`（自带进度）拿，或**只读** `plan.json` 数一遍状态；
  只读计数可以读文件，**写入一律走 CLI**。
- 任务级依赖不写进块里，但**会被算进开工条件**：块看起来"没人挡着"却动不了时，查它所属任务的 `deps`。
- `plan.html` 用 `file://` 打开即可，**不需要服务器**（不要为它起 http 服务）；依赖 Chrome/Safari 的
  现代 JS（无构建步骤）。
- 要发给用户/群的**静态图**（**只在对方明确索取时才做** —— 默认交付是看板 URL，见上节）：本机
  Chrome headless 会挂住不退出（`--screenshot` 其实已经写出了 png），
  必须用 `perl` 的 alarm 兜住，否则命令永远不返回：

  ```bash
  perl -e 'alarm shift; exec @ARGV' 45 \
    env -u HTTP_PROXY -u HTTPS_PROXY "$CHROME" --headless --disable-gpu --no-sandbox \
    --no-proxy-server --hide-scrollbars --user-data-dir=/tmp/cr-shot \
    --window-size=1560,1200 --virtual-time-budget=4000 \
    --screenshot=<plan 目录>/plan.png "file://<plan 目录>/plan.html"
  ```

  跑完 `pkill -9 -f cr-shot` 收尾。截图前先跑 `loomerto render`。
## 出版面：改 skill 名 / 改 description

- **改 skill 名（slug）是一次「必须推远端」的批量重组**，不是 `mv` 个目录：`hermes skills install` / `update`
  按标识符（`<owner>/<repo>/skills/<类目>/<名>`）**回远端取件**，本地 `git mv` 出来的新名字装不回来
  （`install` 只吃 hub 标识符或**单个 SKILL.md 的 URL**，没有本地目录入口 ⇒ 带 `scripts/` / `references/`
  的 skill 靠 URL 也装不全）。把代价链按这个顺序摆给用户，拿到答复再动手：① 仓库 `git mv` + frontmatter
  `name:` + README / REQUIREMENTS / docs / 兄弟引用（几十处，机械改）；② **推远端**（= 把当前分支所有在途
  提交一起发出去，要用户点头 —— 这仓库常态就是「本地领先远端二十来个提交」）；③ 卸载旧名 + 装新名
  （`--category` 只在安装时读），再 `diff -rq` 回读。「只在仓库里改好、副本留旧名」= 仓库与 profile 不一致，
  除非用户明确要这样，否则别默认。提改名方案的动因通常是**可发现性**（agent 想到「用 <包名>」时，按包名
  前缀能在上百条 skill 里一次找齐同族）—— 把这条理由和「现名 → 新名」的对照表一起给。
- **往 description 尾部追加钩子不动路由窗口，因此不用跑盲测**：判据是**前 57 字符逐字不变** —— 拿新旧
  两条做字符串切片比一次（别眼看）；要动窗口里面（哪怕挪一个词）才按 `skill-routing-blind-test` 跑一轮。
- **指令性内容落 SKILL.md 正文的具名小节，描述位只放一行钩子**：描述会被截断、也不该承载流程。同族 skill
  用一行「见 `<核心 skill>` 的「<小节名>」」指过去，别把同一段在这族里抄五遍。
- **这一族现在的名字（引用一律用新名）**：`loomerto-plan`（旧 `maintain-a-shared-plan`）· `loomerto-remind`
  （旧 `remind-collaborators`）· `loomerto-check-commands`（旧 `check-plan-node-commands`）· `loomerto-check-temps`
  （旧 `check-plan-temp-hygiene`）· `loomerto-intake`（旧 `intake-a-running-collaboration`）。改名后**立刻扫一遍
  别处指向旧名的引用**（`grep -rl <旧名> …` 排除 `blind-tests/` 与 `.git/` 应全为空）——本 skill 自己就在
  When to Use 与 `related_skills` 里指过旧名；执行配方（git mv → 引用改写 → 卸载/重装 → 回读）见
  `author-a-skill-in-a-pack-repo` 的「改名一条已发布的 skill」。

## 可编辑的画布（`loomerto open`）

只读看板是 `plan.html`（`file://` 打开，**不起服务、不发附件**）；**要动手改**就用 `open`：

```bash
loomerto --plan <plan 数据文件> open      # 等价：loomerto open <plan 数据文件>
```

- 起一个**只绑 `127.0.0.1`** 的本地服务（纯 stdlib `http.server`），默认自动挑空闲端口并打开浏览器；`Ctrl-C` 停。
- 画布上能改：块的 **标题 / 做什么 / 判据 / 认领人 / 类型 / 状态**（认领 · 开干 · 送审 · 打回 · 收工）、
  **新建任务**（顶栏与泳道下方各一个入口）、**新建块**、**拖动卡片改同一条泳道里的先后**、
  **拖到别的泳道 = 换任务**（拖到某张卡上＝插在它前面，拖到泳道空白处＝追加到末尾）。
- **跨泳道拖 = `block move`，不是「改个字段」**：块的 id 是 `T-00N#B-00N`（**位置即身份**），所以服务端
  要换 id（目标任务里取最小空位）+ 把引用旧 id 的 `deps` / `review_of` / `expanded_from.block` 一次重接，
  **成环就拒改**（400、一个字不写，界面把原因显示在顶栏）。源任务被搬空**不删任务**（空泳道留着）。
  这套演算只有一份（`edits.move_block`），命令 `loomerto block move <ref> --task T-00N [--index N]` 与画布共用。
- **观感与只读看板同源**：两页共用 `loomerto/assets/theme.css`（颜色/字体/状态胶囊/按钮/分隔线/进度条/图例）；
  右侧详情栏是常驻栏，**拖那条分隔线调宽度**（双击复位，宽度记在浏览器里；`Esc` / 「清空」只清内容）。
- 每次改动走 `edits`（改动的唯一实现）→ `store.commit()`：数据文件与 `PLAN.md` / `plan.html` / `plan.canvas`
  **同时**更新；前端带 `rev`（= `updated_at`），对不上回 **409** —— 让人先「刷新」再改（**不做自动合并**）。
- **不做**：删块 / 删任务、直接改 `deps` / `review_of` —— 那些会一脚踩坏判据或接线，
  走 `block remove` / `block move` / `block expand` / `block compress` / `block add` 更安全。
- 要接第二个前端（别的 web 服务 / 别的画布）就读协议：`docs/canvas-sync.md`。

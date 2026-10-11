- **三视图必须原子落盘**（`atomic_write`：同目录 tmp + fsync + `os.replace`）。旧写法 `write_text` 是「先截断再写」，而 `plan.html` 每次改状态都重写；读者（用户浏览器）只要正好落在那一瞬，就会看到**空白页**。用户报「你发的 file 链接是空的」时先怀疑这个，别去查浏览器。
- 排查用另存一份**死文件** `plan-snapshot.html`（不随重渲更新），用来区分「文件问题」还是「打开方式问题」；给用户的日常 URL 永远是会自动重渲的 `plan.html`。
- **节点宽度与「一行几块」是按 `#graph` 的宽度算出来的，不是常量**（2026-10-07 用户要求：「这么大的空间，
  节点却要换行」）：`graph()` 先用 `WMIN=216` / `GXMIN=56` 估出这一屏放得下几列（列数不超过「块最多的那条
  泳道」，多出来的列本来也是空的），再把剩余宽度摊到每列；被 `WMAX=380` / `WMIN` 夹住时剩余空间摊到列间距
  （`GAPCAP=120`）。节点宽度写进行内样式，窗口 resize 用 rAF 合帧重排。**改布局时别再写死宽度**——
  原先的 `PER=4` + `W=216` 在宽屏上会让块无故折到第二行、右边空一大片（用户截图就是这个症状）。
- 一条任务的块数超过这一屏的列数时仍会折行（如 11 块 / 每行 9 列），这是宽度上限内的正常行为；
  `overflow:auto` 保证不丢数据。
- **详情栏是常驻的右侧固定栏**（2026-10-07 用户定，两轮：① 不用拖来拖去、贴右占满整页高、画布为它让出一段
  宽度、一条分隔线同时改两边的宽；② **常驻，不要「点开才跳出」**——点节点只换内容，页面布局一次都不许跳）：
  `--detail-w` 一个变量同时驱动 `#detailpanel` 的宽度和 `body{padding-right}`，所以画布 `#graph` 的
  `clientWidth` 跟着变、`graph()` 自动重排。**只有「宽度真的变了」才重排**（拖分隔线、窗口 resize、
  隐藏已完成）；点节点 / 点卡片 / 清空 **绝不重排**。清空 = 把 `#detail` 写回载入时那段提示
  （`DETAIL_HINT` 直接从 DOM 里取，别在 JS 里另抄一份文案）+ 取消高亮，入口两个：详情栏标题行的
  「清空」按钮与 Esc；**没有关闭边栏的入口**（`body{padding-right}` 常驻，载入时布局就已经是最终
  样子 —— 用户要的就是这个）。拖完把宽度写进 localStorage（`plan.detailw.<slug>`），
  窗口 resize 时先夹进窗口再重排。
  **这段初始化代码必须排在第一个 `view('graph', graph)` 之前**：否则首帧按默认宽度排一遍、再按记住的
  宽度排第二遍，用户就会看见跳一下（本 skill 踩过）。**别退回「浮窗 + 拖标题栏」**：浮窗会盖住节点，
  用户明确否过。
- 详情栏的几何判据（真 Chrome，把测试脚本追加进真 `<plan>.html` 里跑 `--dump-dom`）：载入即
  `#detailpanel` 的 `top==0 && bottom==innerHeight && right==innerWidth`、`#splitter.right ≈ panel.left`、
  `body` 的 `padding-right == panel.width`、节点最右缘 ≤ `panel.left`；**点节点前后取「所有节点
  `style.cssText` + `#graph.clientWidth` 的签名，必须逐字符相同**（这条就是用户要的「布局不跳」）；
  点节点后按 Esc / 点「清空」按钮：`#detail` 应回到载入时那段提示、`.sel` 清零、签名仍不变，
  空态再按 Esc 无副作用；
  拖到 560 后等一帧 `gcw` 应变小且最右缘 ≤ `panel.left`；拖到低于 240 夹回 240；同一 profile 重新载入
  沿用记住的宽度。
- 截图验详情栏之前**先把页面滚回顶部**：点节点会触发 `scrollIntoView`，而 headless Chrome 在「已经滚动过」
  的那一态会把固定定位元素画错位、并留一片未绘制的空白带（看着像布局塌了；旧版同样复现 ⇒ headless 伪影，
  不是产物缺陷）。这一态只信几何数字：`#detailpanel` 满足 `top==0 && bottom==innerHeight && right==innerWidth`、
  `#splitter.right ≈ panel.left`、`body` 的 `padding-right == panel.width`、且节点最右缘 ≤ `panel.left`。
- 改模板（随包发布的 `plan.html`）后的自检（2026-10-07 实测）：`loomerto render <slug>` 后拿 headless Chrome
  `--dump-dom`（同样要 `perl -e 'alarm shift; exec @ARGV' 12` 兜住不退出）读 `#graph` 的 `clientWidth`
  与每个节点的 `style="left/top/width"`，判据是「各泳道行数 == ceil(块数 / 每行块数)」+「最右缘 ≈
  `#graph` 宽 − PAD」+「块数 == plan.json 里的块数」。想验「按容器宽度重排」不必真改窗口：
  `function graph(){}` 是顶层函数声明（在 `window` 上），测试脚本里改 `#graph` 的 `style.width` 之后
  直接调 `window.graph()` 再量一次即可；resize 事件那条路（rAF 合帧）用 `dispatchEvent(new Event('resize'))`
  + 两层 `requestAnimationFrame` 量。
- 渲染看板时若所有块都 done，泳道会折叠成空图 —— 这是正常现象（去掉「隐藏已完成」即可）。
- 改模板（随包的 `plan.html`）后不用开浏览器验证布局：用 node 打桩跑一遍内联脚本，能直接拿到每个节点的
  坐标并暴露渲染异常（本 skill 就是这么发现"隐藏已完成后节点被推到屏幕外"的）。
- **改画布（`canvas.html` / `serve.py`）必须在真浏览器里真拖一次**，别只跑 CLI 与 curl（2026-10-07 跨泳道
  拖动这轮）：CLI 全绿、服务端没有一行报错，而页面上 `drop` **静默什么都不做** —— 前端把「目标泳道 id」
  当块 id 去查表，`moveTo` 直接 return 了。派发合成事件的写法：
  `const dt=new DataTransfer(); src.dispatchEvent(new DragEvent('dragstart',{bubbles:true,cancelable:true,dataTransfer:dt})); dst.dispatchEvent(new DragEvent('drop',{bubbles:true,cancelable:true,dataTransfer:dt}));`
  然后在页面里读回泳道→卡片 id 的顺序、`#msg` 文本、选中卡片；**再回读磁盘上的 `plan.json`**（页面对了
  而文件没变 = 写回那条路断了）。
- **`serve.py` 是已加载的模块，改它必须重启服务；`canvas.html` 不用**（`GET /` 每次都从磁盘重读资产）。
  不重启就会拿旧代码验收 —— 会得到「改了却没生效」或「本该好的地方报 KeyError」这类假结果（两个都踩过）。
- **`_apply` 的返回值是 `(msg, ref)` 二元组**（`ref` = 这次动到的块的当前 id，跨泳道换 id 时前端靠它保住
  选中）：加/改一个 op 时**所有分支都要给全**，前端没传的可选键一律 `p.get(...)` —— 曾经 `reorder` 用了
  `p["ref"]`，同泳道一拖就 `KeyError`（而跨泳道那条路是好的）。
- **块的 id 就是位置**：跨泳道移动会换成目标任务里的**最小空位**，可能正好拿回刚腾出来的老号（`T-002#B-001`
  换泳道后又变回 `T-002#B-001`，但已经是另一块了）。历史字段（`runs[].block` / `folded_from`）留着原样 ——
  那是记录不是引用；要追溯就用 `log` 里那条 `move`（它同时写了旧 id 与新 id）。
- **跑「另一份 checkout 的代码」要小心 `python3 -m` 的 cwd 优先**：`-m` 把**当前目录**放在 `sys.path` 最前，
  所以在 main 的目录里写 `PYTHONPATH=<worktree> python3 -m loomerto` 跑的其实是 **main 那份** ——
  「改动前 vs 改动后」的对拉会变成自己跟自己比（本 skill 踩过：11 份 plan 报「0 处差异」是假的）。
  要跑另一份代码，先 `cd` 到那份 checkout 的根，或者在**没有 `loomerto/` 目录**的地方显式给 `PYTHONPATH`；
  并且**先证明跑的是哪一份**。证明办法要挑**这次改动真的动了的东西**：拿 `--help` 的子命令表当判据只在
  「改了命令面」时成立 —— 只动模型/内部实现时两份 `--help` **一模一样**（这轮 B 档就是这样），
  这时印一个代码里的记号才对，例如
  `python3 -c "from loomerto import model; print('BLOCK_FIELDS' in dir(model))"`。
- **块的形状只有一处声明 = `model.BLOCK_FIELDS`**（键 → 默认值/工厂）。**加一个块级键就改它一处**：
  `model.new_block()` 是建块的唯一字面量，`edits.add_block` / `edits.insert_block` / `cli` 的 `expand` 步骤
  与 `compress` 合并块都调它 —— 别再手写块字典（以前四处各抄一遍，实库里 9 份 plan 的 158 个块没有
  `exec` 就是这么来的）。只有 `block compress` 会写的 `folded_from` 属于 `BLOCK_HISTORY_FIELDS`，
  **不进那张表**（不是每个块都有），`normalize_block()` 也不许删它。
- **老 plan 的缺键是「读时补齐、写时落盘」**：`store.load()` 按 `BLOCK_FIELDS` 只补不改地补齐每个块，
  但**不因此落盘** —— 只读命令（`block show` / `check` / `current` / `workers` / `digest`）跑完，
  `plan.json` 一个字节不变；对它做**任何写操作**才会让这些键材料化进文件（`exec: {}` 就是「没登记线程」，
  语义不变；`commit()` 里顺手写掉）。所以「老 plan 文件里突然多出 `exec`」不是数据被动过，
  也不是新 bug；要拿它当证据时得先说清是读出来的还是写出来的。

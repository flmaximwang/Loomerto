## 会话一长，先把它落到 plan 上

**上下文变长时不要继续在聊天里硬推** —— 把接下来的事写进 plan，用 plan 做进一步的规划、派子代理、按块执行。
「长」的判据不是 token 数，而是这三件里的任一件：一件事已经跨了 3 个以上回合、你开始要向接手的人解释背景、
或者有多条互不相干的线同时在推。到了这一步，聊天里的结论已经开始丢，plan 是唯一不会被压缩掉的那份记录。

固定动作（缺一步就等于没落到 plan 上）：

1. **先读现状**：`loomerto current <slug>`；还没有 plan 就先
   `loomerto plan new <slug> --title "…" --goal "…"`。
2. **把剩下的活写成任务与块**：一个对象 / 一条工作线 = 一条泳道（`task`）；块要带 `doc`（做什么）与
   `done_when`（**别人能重跑**的判据）—— 判据写不成可核验的，说明这一块还没想清楚，别建。
3. **按块派子代理**：一块一个子代理，派完立刻登记线程
   `loomerto block set_status <slug> <块ref> running --by <谁> --delegation <deleg_id> --task-index N --transcript <转录路径>`。
   **子代理禁止起后台进程**（子代理退出时它的后台进程会接管 thread）；长活按目录 / 前缀分片、每条命令自带超时。
4. **按块收工**：`loomerto workers <slug>` 看在途线程还在不在动、`loomerto check <slug>` 看图质量；
   做完 `loomerto block set_status <slug> <块ref> done --artifact <产物绝对路径> --note "<做了什么>"`。
5. **回到聊天只留三样**：结论 + 该谁动（块 id）+ 那行 `file://…/plan.html`。

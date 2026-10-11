## 模型（借自 PlanWeave）

| 概念 | 是什么 | 约束 |
|---|---|---|
| plan | 一个协作目标 | 一个 slug 一个目录 |
| task（节点） | 一条工作线 | 可带任务级 `deps`（别的任务） |
| block（文档） | 一份可独立认领、可评审的工作 | 有 `doc`（做什么）和 `done_when`（判据），**没有判据的块不许建**；另三个可选属性 `input`（吃什么）/ `output`（吐什么）/ `command`（具体跑什么）—— 各由 `block set_input` / `set_output` / `set_command` 一条属性一条命令地写 |
| run | 一次执行记录 | 改状态时自动追加，带 `by` 和 `note` |
| exec | **谁在做 + 那条子代理线程**（认领之外的第二个身份） | 只记现在时：`by` / `delegation` / `task_index` / `transcript` / `started`；由 `block set_status <ref> <在途状态> --by … --delegation …` 写，收工自动清掉 |

**派生状态，不要手填**：block 存 `pending/claimed/running/review/done/blocked/cancelled`；`ready` 与
`waiting` 由依赖**与认领**算出来（依赖全 done **且没人认领** ⇒ `ready` 待认领；依赖全 done **且已经有
`owner`** ⇒ `claimed` 已认领；依赖没全 done ⇒ `waiting` 等前置）。想写 `ready` 会被拒绝是**故意的**——
两处真相就是这个系统要消灭的东西。

任务的开工条件 = 它所有任务级前置任务的**全部**块都 done（`block_deps`）；图上的边只从那些任务
的收尾块引出（`edge_deps`），免得一片线。`review_of` 同时是一条依赖边。块的 `deps` 事后要改走 `block deps`
（`--add` / `--rm` / `--deps` 整组替换；查存在性与环）。

## 谁认领了 / 谁在做 / 线程在哪

一个块上站着**两个人**：**认领人**（接了这块的人/agent，`owner`）与**在做的人**（此刻真正动手的那个，
`exec.by`）—— 记录员认领、子代理动手时两者不是同一个人，所以必须分开写。第三个问题是**那条子代理线程在哪**，
这是「检查子代理是否正常工作」唯一的入口。

| 问 | 看哪 | 怎么写 |
|---|---|---|
| 谁认领了 | 块的 `owner` + runs 里最后一条进入 `claimed` 的记录 | `loomerto block set_status <slug> <ref> claimed --by <谁> [--owner <谁>]`；只换人不改状态用 `loomerto block assign <slug> <ref> --to <谁>`（`--unset` 清掉） |
| 谁在做 | `exec.by` | `loomerto block set_status … running --by <谁>` 自动写上；换人时旧线程登记会被清掉 |
| 线程在哪 | `exec.delegation` + `exec.task_index` + `exec.transcript` | `loomerto block set_status <slug> <ref> running --by <谁> --delegation deleg_xxxxxxxx [--task-index N]` |

```bash
$LT block set_status <slug> T-002#B-001 running --by default --delegation deleg_05e3c787 --task-index 0 \
        --transcript <转录文件绝对路径> --note "前半段：写脚本"
                                    # 转录在哪由你给（Hermes 侧 = <hermes home>/cache/delegation/live/<deleg>/task-<n>.log）
                                    # 不给 --transcript 就只记线程号，workers 只能报「❓ 看不到」
$LT block set_status <slug> T-002#B-001 --unset   # 线程收工 / 交回别人（状态不动，<状态> 这时可以省）
$LT workers <slug>                   # ← 检查：每个在途块登记的线程还在动吗
$LT block show <slug> T-002#B-001    # ← 一次读全：状态 · 认领人(含认领时刻) · 在做+线程+转录 · 判据 · run（只读）
```

- **线程号从哪来**：`delegate_task` 返回的 `delegation_id`（`deleg_xxxxxxxx`）与它在该批次里的
  `task_index`（本文档一律写成 `deleg_05e3c787#0` 这种形式）。**转录路径由调用方登记**
  （`block set_status … --transcript <路径>`）—— 包不猜 harness 的目录；Hermes 侧的对应写法是 hermes home 下的
  `cache/delegation/live/<delegation_id>/task-<n>.log`（default profile 的 home 是 `~/.hermes`，
  其余是 `~/.hermes/profiles/<名字>/`）。不给 `--transcript` 时只记线程号，`workers` 会报「❓ 看不到」。
- **`workers` 七种结论，一个都不许合并**：`✅ 在动`（转录最近还在写）/ `⏳ 静默`（超 `--stale-min`
  —— 默认 30 分钟没写一行，可能卡住或已死）/ `⚠ 线程已结束`（manifest 说这条线程已 completed/failed，
  而块还挂在 running ⇒ **该对账**）/ `❌ 号记错`（delegation 目录在，但没有这个 task 的转录）/
  `❓ 看不到`（连目录都不在：过了 7 天保留期 / 在别的机器上 / 号记错）/ `➖ 无线程`（人在做，或谁在做
  都没登记）/ `➖ 已无意义`（块已不在途，登记还挂着）。**只有 `⚠` 与 `❌` 退 1**（先对账再往下走）；
  `⏳ ❓ ➖` 只提示、退 0。`--json` 给 agent 读。
- **`❓ 看不到` ≠「子代理没在跑」**：live 转录 7 天就被回收，跨机器也看不到。说得出「看不到」，
  说不出「没在跑」。要更硬的判活，去**线程所在的那个 profile** 里用 `delegate_task action='list'`
  （那里比 pid + 进程启动时间指纹），别在这边把「读不到」写成结论。
- **它只看文件、不读库**：`workers` 只读转录与同目录的 `manifest.json`（纯 stdlib、跨 profile、跨机器
  都能跑）。所以它能回答「还在写吗 / 结束了吗」，回答不了「进程还活着吗」—— 后者走上面那条升级路径。
- **线程登记是现在时，不是简历**：收工（`done`/`cancelled`）或退回（`pending`）时 CLI 会自动清掉它；
  「谁做过的」留在该块的 `runs`（每条带 `by`）与日志里。
- **线程登记短命，产物才长命**：live 转录 7 天后回收，所以线程结束前要把真正要留的证据写进块的
  `artifacts` 或 run 的 `note`（`loomerto block set_status … --artifact <路径>`），别指望以后还能回读转录。

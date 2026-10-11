## 命令速查

```bash
# 唯一入口 = 装好的 loomerto 命令。plans 根由调用方交代（包不认任何 harness 的目录）：
LT="loomerto --plans-root <本 profile 的 workspace/plans>"
#   换成 --plan <plan 数据文件> 则只认这一份，此后命令里**不写 slug**（位置参数整体左移一位）
#   例：loomerto --plan ./plan.json block set_status T-001#B-002 done
# 一级命令 = 对象/全局动作，动作在二级：plan / task / block 是分组，其余是单层动作
$LT list                                  # 所有 plan + 进度
$LT plan new <slug> --title "…" --goal "…" --owner "you=human:本人@discord:<ch>" \
   --owner "rdm-assistance=agent:RdmAsst3813"
$LT task add <slug> --title "…" [--deps T-001] [--owner x]
$LT task set_status <slug> <任务ref> <状态> [--by x] [--note "…"]        # 任务状态；线程登记只在 running 收
$LT task show <slug> <任务ref> [--json]                           # 一条任务的详情（只读）
$LT task remove <slug> <任务ref> [--force]                            # 真删任务（连带它的块）
$LT block add <slug> --task T-001 --title "…" --kind impl|review|decision|research \
   --doc "做什么" --done-when "可核验的判据" [--deps T-001#B-002] [--review-of T-001#B-002] \
   [--owner x] [--status 状态]      # --status 默认 blocked（待批准）；无需审批才显式给 pending
$LT block insert <slug> <锚块ref> [--before|--after] --title "…" [--doc "…"] [--done-when "…"] \
   [--owner x] [--status 状态] [--dry-run]     # 插到某块之前/之后，接线一次改对
$LT block set_status <slug> <ref> <状态> [--by x] [--note "…"] [--artifact <路径>] [--at ISO] \
      [--delegation deleg_xxxxxxxx] [--task-index N] [--transcript <路径>] [--unset] \
      [--doc "…"] [--done-when "…"]   # 状态 + 谁在做 + 子代理线程一个入口；参数按状态卡
$LT block set_title|set_doc|set_type|set_input|set_output|set_command|set_audit <slug> <块ref> "<值>"
                                      # 一条属性一条命令：值整组替换、留空 = 清空（标题除外），
                                      # 没有变化退 2 一个字不写。只改属性 —— 状态与身份走 set_status、
                                      # 认领人走 assign、接线走 deps。set_type ∈ impl|review|decision|research；
                                      # set_audit 的判据多条用 ; 分隔
$LT block assign <slug> <块ref> [--to <参与方 id> | --unset] [--note "为什么"]
                                      # 指派/取消指派认领人（不动状态）
$LT block deps <slug> <块ref> [--deps <一串ref> | --add <一串> | --rm <一串>] [--note "为什么"]
                                      # 接线事后改：存在性/自指/环三道闸，成环退 2 一个字不写；
                                      # --deps 整组替换（给空 = 清空），不能与 --add/--rm 同给
$LT workers <slug> [--stale-min 30] [--json]
                                         # 在途块的线程还在动吗（已结束/号记错 ⇒ exit 1）
$LT block expand <slug> <块ref> [--title "…"] [--step "标题 :: 做什么 :: 判据1;判据2"] [--dry-run]
                                         # 一个块 → 一个任务（块成为第一步）
$LT block compress <slug> <任务ref> [--into <块ref|任务ref>] [--keep-task] [--force] [--dry-run]
                                         # 一个任务 → 一个块（默认回展开前的位置）
$LT block move <slug> <块ref> --task T-00N [--index N] [--note "…"]
                                         # 一个块 → 另一条任务（泳道）：换块 id + 重接 deps/review_of；
                                         # 成环则拒改（一个字不写）；源任务空了不删。画布上的跨泳道拖走这条
$LT block remove <slug> <块ref> [--force]     # 真删一个块（被引用时默认拒删）
$LT block bypass <slug> <块ref> [--note "…"]  # 把一个中间块从链上摘掉：它在等的前置直接接给原来等它的块
                                      # （A→B→C ⇒ A→C）；替你把那几处引用改对再删，图上不留悬空
$LT note <slug> "…" --kind summary|decision|reminder [--ref T-001#B-001]
                                         # --kind note 非法（退 2，会列出全部候选）；「这段发生了什么」用 summary
$LT current <slug>                        # 现在该谁动
$LT block show <slug> <块ref> [--json] [--runs N]   # 一个块的详情（只读；--runs 0 = 全列 run）
$LT check <slug>                          # 图质量（有错误 exit 1）
$LT digest <slug> [--to <参与方>] [--stale-hours 24]
$LT render <slug>                         # 手动刷新三个视图
$LT open <plan 数据文件> [--port N]        # 起本地服务，把这份 plan 开成**可编辑的画布**（只绑 127.0.0.1）
```

`ref` 可以写 `T-002`（任务）或 `T-002#B-001`（块），块也可以只写 `B-001`。

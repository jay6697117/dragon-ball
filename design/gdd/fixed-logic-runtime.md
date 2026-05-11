# 固定逻辑步进与战斗运行时

> **Status**: Revised — second Design Review NEEDS REVISION blockers addressed; Pending Re-review
> **Author**: SteveZhang + Claude Code Game Studios
> **Last Updated**: 2026-05-11
> **Implements Pillar**: 读招定胜负；短连招，高回合；气槽辅助，不接管战斗；第一回合必须好玩；热血能量必须可读
> **System ID**: fixed-logic-runtime
> **Priority**: MVP
> **Layer**: Foundation
> **Creative Director Review (CD-GDD-ALIGN)**: APPROVED 2026-05-11 — 固定逻辑运行时清楚保护“我可以练会”的公平时间基础，并未扩大 MVP 范围；trace/replay 语义继续限定为 QA/debug。
> **Design Review**: NEEDS REVISION 2026-05-11 twice; revised same day to address recovery-pause semantics, transition snapshots, catch-up guards, command/AI schemas, event ordering, UI/audio delivery, trace bounds, and AC triage; pending fresh re-review.

## Overview

固定逻辑步进与战斗运行时是《星核斗魂》所有 MVP 战斗系统的时间基础。它定义战斗逻辑以固定 tick 更新，规定输入、移动、状态机、hitbox/hurtbox、命中停顿、硬直、伤害、气槽、对局流程和战斗事件的统一处理顺序，确保同一组输入和数据在浏览器中得到可重复、可验证的结果。玩家不会直接操作这个系统，但会通过稳定的防御时机、可信的反击窗口、不会漂移的短连招和准确同步的 HUD/VFX/音效感受到它；如果本系统不成立，“读招定胜负”就会变成不可预测的帧率运气。

## Player Fantasy

玩家不会直接看到或操作固定逻辑步进与战斗运行时，但应该始终感觉战斗背后有一套公平、稳定、可信的时间规则。成功防住起手、抓到硬直、完成短连时，玩家会觉得“这是我判断对了”；失败被反打时，也应该能理解为“我慢了、贪了、读错了”，而不是怀疑游戏判定漂移、输入丢失或帧率影响结果。

这个系统服务的玩家幻想是“我可以练会”。《星核斗魂》的战斗由短回合组成：试探、出手、防御、抓硬直、打出 2–4 段短连，然后重新回到立回。固定运行时必须让同样的输入、同样的时机、同样的数据产生同样可信的结果，让玩家把注意力放在读招和决策上，而不是担心浏览器帧率、特效数量或画面波动改变胜负。

它主要支撑三条支柱：

- **读招定胜负**：玩家相信命中、防御、打空和硬直窗口是稳定规则的结果。
- **短连招，高回合**：玩家能通过练习掌握短连节奏，而不是依赖运气碰上正确帧。
- **热血能量必须可读**：即使爆气、命中停顿、HUD、VFX 和音效同时发生，战斗结果仍然清楚、有因果、能被理解。

## Detailed Rules

### Core Rules

1. **固定 combat tick 是唯一战斗权威时间**  
   战斗逻辑必须以固定 combat tick 推进。MVP 默认目标为每秒 60 个 combat tick；所有前摇、活跃帧、收招、移动锁定、受击硬直、格挡硬直、hitstop、输入缓冲、气槽收益、回合计时和胜负检查都以 tick 计量。渲染帧率、动画播放帧率、VFX 数量或浏览器刷新率不得改变战斗结果。

2. **本 GDD 的 tick 术语必须统一**  
   GDD 中提到的“帧”默认指 combat tick，除非明确写为“渲染帧”或“动画帧”。本 GDD 使用以下固定术语：

   - `combat tick`：固定逻辑时间单位。
   - `committed runtime tick`：已经提交 snapshot 的 runtime tick；`running` 和 `hitstop` 都会提交 runtime tick 并递增 `committed_tick_index`。
   - `running tick`：以 `tick_execution_state = running` 处理的 committed runtime tick；会推进动作、移动、判定、硬直、timer、projectile 和 gameplay 状态。
   - `hitstop tick`：以 `tick_execution_state = hitstop` 处理的 committed runtime tick；只推进 hitstop countdown、输入采集、表现/debug，不推进 running gameplay。
   - `tick_execution_state`：本 committed runtime tick 实际用哪种状态规则处理；用于区分“本 tick 如何执行”。
   - `post_commit_runtime_state`：本 tick 提交后 runtime 将处于的状态；用于区分“本 tick 造成了什么状态变化”。
   - `runtime_state`：非 tick 内部语境下的当前运行状态；snapshot 中必须同时暴露 `tick_execution_state` 和 `post_commit_runtime_state`，避免 transition tick 歧义。
   - `non-advancing state`：`inactive`、`countdown`、`paused`、`focus_suspended`、`round_ended`；这些状态不提交 runtime tick，也不递增 `committed_tick_index`。
   - `recovery_pause`：`backlog_classification` 和 `state_reason` / `pause_reason`，不是独立 `runtime_state`；运行时表现为 `runtime_state = paused`。

3. **逻辑时间与表现时间分离**  
   committed runtime tick 提交战斗事实；渲染、HUD、VFX、音效、相机和调试显示只读取最近提交的战斗事实并表现它。表现层可以延迟、插值、闪烁、震动或播放特效，但不得修改战斗状态。

4. **combat tick 不允许静默跳过，catch-up 必须公平优先**  
   如果浏览器短暂掉帧，运行时最多可以按顺序补跑 `catch_up_max_ticks` 内的少量积压 tick，确保逻辑时间连续。补跑同时受 tick 数、`catch_up_wall_clock_budget_ms`、blocking state 和 fairness-stop event 约束；任一 guard 触发时，guard 优先于剩余 backlog。若补跑 tick 产生玩家必须看见的关键事件，运行时可以提交该触发 tick，但必须在继续推进下一枚 combat tick 前进入 `runtime_state = paused`、`state_reason = recovery_pause` 的安全恢复流程。运行时不得丢弃 tick、一次性快进大量 tick，或让战斗在不可见状态下继续推进。本 GDD 默认 `catch_up_max_ticks = 6`，`catch_up_wall_clock_budget_ms = 4.0`；具体 accumulator 精度和 host-frame 集成由 Runtime ADR 验证。

5. **所有战斗状态变化只在 tick 边界提交**  
   输入、移动、状态切换、hitbox/hurtbox、命中、格挡、伤害、硬直、气槽、KO 和 round end 都必须在 tick 的固定结算流程中提交。任何系统不得在 tick 中途直接修改已提交的战斗结果。

6. **running tick 使用固定结算顺序**  
   每个 running tick 必须按以下顺序结算：

   1. 处理运行时请求：开始、暂停、恢复、快速重开、退出。
   2. 读取目标 tick 的玩家输入快照、CPU 命令和训练木桩命令。
   3. 战斗状态机判断命令是否可执行。
   4. 推进角色动作状态、移动、动作计时器和可行动窗口。
   5. 根据当前动作 tick 生成 hitbox/hurtbox 事实。
   6. 按确定性顺序解析碰撞、命中、格挡、打空。
   7. 应用伤害、受击硬直、格挡硬直、hitstop、气槽和 combo 变化。
   8. 检查 KO、时间结束和 round end。
   9. 提交本 tick 战斗快照。
   10. 发布本 tick 的有序战斗事件批次。

   如果命令在 running tick `T` 的命令阶段被接受，并启动新动作，则动作开始事件在 tick `T` 提交，动作的 `action_tick_index = 1` 也从 tick `T` 的动作推进/判定生成阶段计起。后续 startup/active/recovery 窗口必须使用这一 convention，避免 off-by-one。

   | Scenario | Tick | Expected action timing |
   |---|---:|---|
   | Command accepted and new action starts | `T` | `action_tick_index = 1`; startup/active/recovery checks for tick `T` read index `1`。 |
   | Next running tick with no hitstop/pause/focus | `T + 1` | Same action advances to `action_tick_index = 2` before hitbox/hurtbox generation。 |
   | Hitstop begins after hit on tick `T + K` | `T + K + 1 ...` | `action_tick_index` freezes during hitstop ticks and resumes on the next running tick。 |
   | Action ends during recovery | final running action tick | End/recovery-complete event uses the same index that made the action complete; no hidden extra tick is inserted。 |

7. **hitstop tick 使用独立结算顺序**  
   造成 hitstop 的 running tick `T` 正常提交；tick `T` 本身不消耗第 1 个 hitstop tick。tick `T` 的 snapshot 必须表达 `tick_execution_state = running`，并在 `post_commit_runtime_state = hitstop` 或 `post_commit_runtime_state = round_ended` 中表达结果状态，不能只用一个 `runtime_state` 混淆执行状态和提交后状态。若该命中/格挡请求 `N > 0` hitstop，则后续第一个 hitstop tick 是 committed runtime tick `T + 1`，并递减一次 hitstop countdown。每个 hitstop tick 必须按以下顺序结算：

   1. 处理更高优先级请求：round end、focus suspended、pause、restart、exit。
   2. 采集允许的输入快照并交给输入缓冲，但不验证或执行普通 combat action。
   3. 不推进动作帧、移动、hitbox/hurtbox、hitstun、blockstun、recovery、projectile 或 round timer。
   4. 递减 hitstop 自身剩余 tick。
   5. 提交 hitstop snapshot，递增 `committed_tick_index`。
   6. 发布 hitstop/state/debug 相关事件或空事件批次。

   | Case | committed_tick_index | tick_execution_state | post_commit_runtime_state | elapsed_running_ticks |
   |---|---:|---|---|---:|
   | Hit-causing tick with `N = 2` hitstop | `T` | `running` | `hitstop` unless `round_ended` wins | +1 |
   | First hitstop countdown tick | `T + 1` | `hitstop` | `hitstop` if one remains | +0 |
   | Final hitstop countdown tick | `T + 2` | `hitstop` | `running` unless higher-priority state wins | +0 |
   | Earliest resumed running tick | `T + 3` | `running` | current result of tick `T + 3` | +1 |

8. **同 tick 冲突必须有稳定规则**  
   同一 tick 内发生多个战斗事实时，不得由 Godot 节点顺序、渲染顺序或数组偶然顺序决定结果。MVP 规则如下：

   - 双方攻击在同一 tick 同时命中时，按同时命中处理。
   - 命中与防御输入同 tick 出现时，防御是否成立只看本 tick 输入系统交付给状态机的防御命令和角色当前可防御状态；不能在命中后补防。
   - 同 tick 出现多个 hitstop 请求时，MVP 取最大 hitstop 值，不累加。
   - 同 tick 出现 KO 与时间结束时，先结算伤害/KO，再检查时间结束；最终胜负细则由“血量、伤害、计时与胜负”GDD 定义。

9. **hitstop 是战斗冻结反馈，不是暂停菜单**  
   hitstop 表示命中或格挡瞬间的短冻结，用来强化打击感和可读性。hitstop 期间可以继续采集输入并记录输入缓冲，但普通行动、移动、攻击推进和硬直倒计时不得继续执行。

   hitstop 期间冻结：

   - 角色动作帧推进。
   - 角色移动位移。
   - hitbox/hurtbox 活跃变化。
   - 受击硬直、格挡硬直、收招硬直倒计时。
   - 对局计时器。
   - 已存在攻击判定或 projectile 的战斗推进。

   hitstop 期间允许继续：

   - 采集玩家输入。
   - 写入输入缓冲。
   - 播放已经触发的 VFX/音效。
   - 更新 HUD 表现。
   - 更新相机震动表现。
   - 更新调试显示。
   - 递减 hitstop 自身剩余 tick。

10. **hitstop 与 hitstun/blockstun 必须分离**  
    hitstop 是双方或局部战斗的冻结反馈；hitstun/blockstun 是角色不能行动的硬直时间。hitstun 和 blockstun 不得在 hitstop 期间减少，避免浏览器帧率或特效负载改变确反窗口。

11. **pause、focus lost、hitstop、round end 不能混用**  
    pause 是玩家或系统请求的暂停；focus lost 是浏览器标签页失焦、窗口隐藏、canvas/input 焦点丢失或 Godot UI 焦点转移导致战斗输入不安全；hitstop 是命中反馈；round end 是对局结束流程。四者必须使用不同运行状态，不能用一个“暂停”概念混在一起。

12. **浏览器失焦必须停止 combat tick，恢复必须安全倒计时**  
    Web MVP 中，页面隐藏、浏览器/窗口 blur、canvas 或键盘输入焦点丢失、fullscreen/audio unlock gate 等 unsafe focus loss 必须进入 `focus_suspended`，停止 combat tick，不累计补跑，不接受新的战斗输入。普通 UI hover、鼠标焦点变化或暂停菜单内部控件焦点移动不等于 unsafe focus loss，不得自动覆盖 `paused` 为 `focus_suspended`。恢复焦点后，运行时必须显示原因、清理或重新确认 held input，确认输入由 UI/menu context 消费，随后进入短 resume countdown / Ready-Go reacquisition，之后才恢复 combat command flow，避免玩家回到页面时自动移动、攻击、防御、气弹或爆气。

13. **玩家、训练木桩和简单 CPU 走同一命令入口**  
    玩家输入、训练木桩脚本和简单 CPU 都必须转化为同一种战斗命令格式，再交给战斗状态机判断。CPU 或木桩不得绕过输入缓冲、状态机、硬直、防御、气槽或 hitbox/hurtbox 规则。

    | Command field | Required values / type | Rule |
    |---|---|---|
    | `round_instance_id` | current round id | 必须匹配当前 round；旧 round command 拒绝。 |
    | `command_source` | `player` / `training_dummy` / `simple_cpu` / `qa_fixture` | 不允许隐式来源。 |
    | `source_actor_id` | stable actor id | 必须匹配可行动 actor。 |
    | `command_id` | unique id within source + round | 重复 command id 幂等拒绝或去重。 |
    | `command_sequence_key` | `(round_instance_id, target_committed_tick_index, source_actor_id, source_priority, command_priority, command_id)` | 同 tick 多命令按此稳定排序，不依赖 Godot signal/node/array 顺序。 |
    | `target_committed_tick_index` | integer `>= current_committed_tick_index + 1` unless queued by input-buffer rules | 不允许修改当前已开始或已提交 tick。 |
    | `input_snapshot_id` | player-only input snapshot id | 仅玩家输入可用；CPU/木桩不能用它替代观察快照。 |
    | `based_on_committed_tick_index` | non-player required integer `< target_committed_tick_index` | CPU/木桩必须基于已提交观察；MVP 默认 `target_committed_tick_index = based_on_committed_tick_index + 1`，除非后续 AI GDD 定义延迟队列。 |
    | `command_type` | `movement` / `guard` / `light_attack` / `heavy_attack` / `projectile` / `burst` / `menu_runtime_request` / `qa_fixture` | 非法 enum 拒绝。 |
    | `pressed_or_held` | `pressed` / `held` / `released` / `axis` / `scripted` | 输入语义必须显式。 |
    | `runtime_state_at_capture` | runtime state enum | 不符合当前 input policy 的 command 拒绝。 |

    最小 rejected-command payload 必须表达：`command_id`、`command_source`、`source_actor_id`、`target_committed_tick_index`、`runtime_state_at_capture`、`command_acceptance_result`、`rejection_reason`、`dedupe_or_supersede_reason`。CPU 和木桩只能基于 `AIObservationSnapshot` 生成命令，不能读取同 tick 尚未提交的玩家输入、输入缓冲、事件总线、debug trace、UI payload、live Godot node 或半更新状态。

    `AIObservationSnapshot` 是 CPU/木桩唯一可见战斗事实，最小字段为：`round_instance_id`、`committed_tick_index`、`runtime_state`、双方 actor 的 stable id / position / facing / visible action state / visible action phase、HUD 可见的 health / energy / timer、粗粒度 projectile facts、可见 round state。它不得包含玩家 raw input、input buffer 内容、command-source metadata、debug-only trace、UI prompt state 或 mutable runtime internals。CPU/木桩 command trace 必须记录 `observation_snapshot_id` 或 hash、`based_on_committed_tick_index`、`decision_age_ticks`、`script_or_config_id`、accept/reject result。

14. **战斗事件是已提交事实，不是请求**  
    每个 committed runtime tick 结束后，运行时发布一个有序事件批次。事件只描述已经发生并提交的战斗事实，例如 `attack_started`、`startup_started`、`active_frame_started`、`recovery_started`、`punish_window_opened`、`punish_window_closed`、`hit_landed`、`blocked`、`whiffed`、`hitstop_started`、`hitstun_started`、`energy_meter_changed`、`burst_started`、`round_ended`。HUD、VFX、音效、相机和调试显示只能消费事件，不能通过事件反向请求改变本 tick 或过去 tick 的战斗结果。

    Event ordering 必须使用带 namespace 的 key：`runtime_event_sequence_key = (round_instance_sequence, committed_tick_index, phase_order_namespace, tick_phase_order, within_phase_event_index)`。`phase_order_namespace` 至少包含 `running_tick_order`、`hitstop_tick_order`、`runtime_state_transition_order`、`debug_trace_order`。`within_phase_event_index` 必须由稳定 tie-breaker 生成：event type priority、source actor id、target actor id、move/projectile id、stable spawn/order id；不得依赖 Godot node order、signal order、render order 或容器插入顺序。

    最小 event cardinality：`attack_started` 每次 accepted action 一次；`hit_landed` / `blocked` 每个 resolved hit instance 一次；`whiffed` 每个 whiffed attack window 一次，不是每个空 active tick 一次；`hitstop_started` 每次进入 hitstop 一次；`round_ended` 每个 round instance 一次。Impact/block 音效默认由造成命中/格挡的 running tick 的 `hit_landed` / `blocked` 触发一次；hitstop countdown tick 不得重复触发 impact audio。

15. **snapshot、event、trace 和 UI state 必须有最小可测合同**  
    当前 HUD 数值的权威来源是 committed snapshot summary；event batch 只用于一次性反馈、VFX、音效、HUD pulse、debug log 和因果解释。如果 snapshot 与 event-derived UI 状态冲突，snapshot wins。所有 snapshot、event、trace、UI payload 都必须是 value-copy、stable id、primitive value、不可变记录或只读数据引用；不得把 live Godot `Node`、mutable authority `Resource`、可被消费者改写的 `Array` / `Dictionary` 作为权威对象交给表现、UI、音频或 debug。

    最小 snapshot summary 必须表达：`round_instance_id`、`round_instance_sequence`、`committed_tick_index`、`tick_execution_state`、`post_commit_runtime_state`、`elapsed_running_ticks`、`round_timer_remaining_ticks`、`hitstop_remaining_ticks`、双方 actor 的 stable id / position / facing / current health / max health / current energy / max energy / action state / action phase / action_tick_index、combo summary、KO/round result summary、当前 command-source metadata 引用。  
    最小 event 必须表达：`runtime_event_sequence_key`、`event_type`、`source_actor_id`、`target_actor_id`、`move_id` 或 `cause`、`result_type`、`position_context`、`idempotency_key`、`delivery_context_id`。  
    最小 event batch delivery context 必须表达：`delivery_context_id`、`delivery_mode` (`normal` / `catch_up` / `recovery_resume` / `stale_discarded`)、`host_delivery_sequence`、`batch_index_in_delivery`、`batch_count_in_delivery`、`catch_up_tick_count`、`round_instance_id`。重复 delivery 不得重复播放同一 `idempotency_key` 的 HUD pulse、VFX、audio one-shot 或 debug one-shot。  
    最小 UI runtime state payload 必须表达：`runtime_state`、`state_reason`、`previous_runtime_state` 或 `resume_target_state`、`runtime_state_sequence_key`、`requires_player_confirm`、`combat_input_policy`、`held_input_cleanup_required`、`held_input_cleanup_state`、`countdown_phase`、`backlog_classification`、`round_instance_id`。

    `combat_input_policy` 允许值为：`combat_execute_allowed`、`capture_only`、`direction_pre_read_only`、`menu_only`、`resume_confirm_only`、`blocked_all`。`countdown_phase` 允许值为：`none`、`ready`、`three`、`two`、`one`、`go`、`resume_ready`、`resume_go`。`runtime_state_sequence_key` 用于 pause/focus/resume/restart 等非 combat state transition 的排序和幂等，不得伪装成 combat event。

16. **Godot 2D physics 不能作为格斗判定或权威位置来源**  
    Godot 2D physics 可用于场景边界、地面辅助、宽泛碰撞或调试可视化，但 MVP 格斗命中必须由数据驱动 hitbox/hurtbox 在 combat tick 内结算。不得依赖 sprite 边缘、动画可见像素、physics contact callback 或渲染节点顺序决定命中。physics 辅助结果也不得直接写入权威战斗位置；必须先进入 fixed runtime 的移动/边界阶段，再由 committed snapshot 暴露。

17. **运行时必须支持 bounded QA trace**  
    MVP 运行时必须能记录每 tick 输入快照、关键状态变化和事件批次，用于 QA 复现输入缓冲、状态切换、命中/格挡、硬直、气槽和胜负问题。trace 必须同时受 `trace_window_ticks`、`trace_max_events_per_tick`、`trace_max_payload_bytes_per_entry`、`trace_max_objects_per_tick` 和 `trace_warning_throttle_per_second` 限制，避免 Web 内存和 GC 风险。超限时可以淘汰旧 trace 或 throttle warning，但不得改变权威 combat state 或 event ordering。该 trace 是开发与 QA 工具，不是正式玩家 replay；MVP 不要求 rollback、联网同步、跨版本回放兼容或观战功能。

    MVP 默认 trace bounds：`trace_window_ticks = 180`、`trace_max_events_per_tick = 32`、`trace_max_payload_bytes_per_entry = 4096`、`trace_max_objects_per_tick = 64`、`trace_warning_throttle_per_second = 10`。这些是 Web-safe 起点；Runtime ADR 可在浏览器实测后调整，但必须保留同名配置、trace diagnostics 和 QA 覆盖。

18. **实现方式留给 ADR，不写进本 GDD**  
    本 GDD 只规定战斗时间、结算规则和最小跨系统合同，不规定 Godot 节点结构、Autoload/signal/event bus 选型、`_physics_process` 使用方式、accumulator 实现、日志格式、浮点/定点选择或渲染插值方案。这些属于后续架构决策。

### States and Transitions

| Runtime State | 含义 | committed runtime tick 是否提交 | 输入处理 | 允许进入 | 允许离开 |
|---|---|---:|---|---|---|
| `inactive` | 战斗运行时未开始或已卸载 | 否 | 不处理战斗输入 | 初始状态、退出对局 | `countdown` |
| `countdown` | 回合开始前倒计时/Ready-Go 表现 | 否 | 可预读方向输入；攻击/气弹/爆气不执行 | `inactive`、快速重开 | `running`、`paused` |
| `running` | 正常战斗结算 | 是；递增 `committed_tick_index` 和 `elapsed_running_ticks` | 采集并执行合法命令 | `countdown`、`hitstop` 结束、`paused` 恢复 | `hitstop`、`paused`、`focus_suspended`、`round_ended` |
| `hitstop` | 命中/格挡冻结反馈 | 是；递增 `committed_tick_index`，不递增 `elapsed_running_ticks` | 采集输入并写入缓冲，不执行普通行动 | `running` 中命中或格挡触发 | `running`、`paused`、`focus_suspended`、`round_ended` |
| `paused` | 玩家暂停、菜单暂停或 `state_reason = recovery_pause` 的安全恢复暂停 | 否 | 不接受战斗行动；菜单/恢复确认由 UI/menu context 处理 | `countdown`、`running`、`hitstop`、catch-up guard | 返回 resume countdown、原安全状态或 `inactive` |
| `focus_suspended` | 浏览器失焦、页面隐藏、canvas/input 焦点丢失或 Godot UI 焦点转移导致战斗输入不安全 | 否 | 不接受新战斗输入；恢复时清理/确认 held input | `running`、`hitstop`、`paused`、`countdown` | `paused` 或恢复到安全继续状态 |
| `round_ended` | KO、时间结束或系统判定回合结束 | 否 | 不接受战斗行动；只接受重开/退出/菜单确认 | `running`、`hitstop` | `countdown`、`inactive` |

`recovery_pause` 不得出现在 `runtime_state` 列表中；它只能作为 `backlog_classification`、`state_reason` 或 `pause_reason` 出现，并表现为 `runtime_state = paused`。

状态优先级从高到低为：

1. `round_ended`
2. `focus_suspended`
3. `paused`
4. `hitstop`
5. `running`
6. `countdown`
7. `inactive`

如果同一 tick 触发多个状态请求，必须按上述优先级决定最终运行状态。例如：命中触发 hitstop 的同一 tick 造成 KO，则进入 `round_ended`，而不是继续停留在 `hitstop`。浏览器失焦优先于普通 pause；如果失焦前已经处于 `paused`，恢复焦点后仍回到 `paused`，不得自动恢复到 `running`。

### Interactions with Other Systems

| System | Runtime 输入 | Runtime 输出 | 边界规则 |
|---|---|---|---|
| 输入映射与输入缓冲 | 每 tick 输入快照、buffered command、held input 状态 | 当前 runtime state、是否允许采集/执行输入、tick id | 输入系统负责采集和缓冲；运行时决定何时读取、执行、冻结或清理。 |
| 角色数据与招式数据 | 招式 tick 数据、伤害、硬直、气槽收益、取消窗口、命中属性 | 当前 tick 可执行命令、动作推进结果、事件上下文 | 数据系统提供合法数据；运行时只消费已验证数据，不在运行中修正非法招式。 |
| 动画帧数据与资产元数据 | 动作帧对应的 hitbox/hurtbox、pivot、VFX spawn point | 当前动作 tick、事件触发点 | 动画元数据可定义判定出现在哪些 combat tick；动画播放本身不能决定判定。 |
| 移动与距离控制 | 移动命令、速度、加速度、位置约束 | 提交后的角色位置、朝向、移动事件 | 移动只在 combat tick 中结算；渲染插值不得改变真实位置。 |
| 战斗状态机 | 当前状态、命令、硬直、取消窗口、资源状态 | 状态切换结果、动作开始/结束事件 | 状态机决定角色能不能行动；运行时负责调用顺序和 tick 边界提交。 |
| Hitbox / Hurtbox 判定 | 本 tick 的 hitbox/hurtbox 事实、角色位置、朝向 | `hit_landed`、`blocked`、`whiffed` 等事件 | 判定系统只在运行时指定阶段结算；不能由 physics callback 或动画信号抢先结算。 |
| 血量、伤害、计时与胜负 | 命中/格挡结果、伤害值、round timer tick | 血量变化、KO、timeout、round end 事件 | 胜负检查在伤害和计时结算后执行；最终胜负细则由该系统 GDD 定义。 |
| 防御、格挡与受击反馈 | 防御命令、命中属性、朝向/距离、可防御状态 | blockstun、hitstun、guard feedback 事件 | 运行时提供稳定 tick 顺序；防御系统定义防御是否成立。 |
| 命中停顿与硬直窗口 | 命中/格挡结果、动作数据中的 hitstop/hitstun/blockstun | hitstop 状态、硬直倒计时、可反击窗口 | hitstop 冻结硬直倒计时；硬直窗口必须以 combat tick 为准。 |
| 短连招与取消规则 | hit confirm、动作阶段、取消窗口、combo 状态 | combo counter、cancel accepted/rejected、回合重置事件 | combo 和 cancel 只在运行时固定顺序中提交；不得由动画信号或 VFX timing 触发。 |
| 气槽与爆气反杀 | 命中、格挡、受击、爆气命令、气槽数据 | `energy_meter_changed`、`burst_ready`、`burst_started` | 气槽变化必须由已提交战斗事实触发；HUD 不能直接修改气槽。 |
| 气弹 / 能量攻击 | projectile spawn 数据、移动 tick、命中属性 | projectile 位置、命中/格挡/消失事件 | MVP 气弹是可读牵制，不做对波、反弹、多弹种或复杂 projectile 模拟。 |
| 对局流程与快速重开 | start round、restart、exit、round config | state transition、round start/end tick、重置快照 | 对局流程可以请求状态变化；不能跳过运行时重置规则。 |
| 训练木桩 | 木桩脚本命令、受击/格挡配置 | 木桩状态事件、debug 状态 | 木桩必须走同一状态机和判定规则，不得作为特殊命中目标绕过系统。 |
| 简单脚本 CPU | CPU 每 tick 命令 | CPU 行动事件、状态变化 | CPU 不直接改状态，只提交命令；MVP 不做完整 AI。 |
| HUD 与战斗信息反馈 | committed snapshot summary、ordered event batches、UI runtime state payload | 血条/气槽/计时/combo 显示、一次性反馈 pulse | HUD 当前数值以 snapshot summary 为权威；event batch 只触发一次性反馈；冲突时 snapshot wins。 |
| VFX 可读性系统 | tick 事件批次、位置上下文、结果类型、catch-up delivery metadata | 命中特效、格挡火花、爆气特效 | VFX 只表现事实；不得遮挡读招或改变判定。 |
| 动画播放与 sprite 表现 | 当前动作状态、动作 tick、事件 | sprite 帧、动画状态 | 动画播放必须跟随 combat tick 状态；动画帧不能驱动 combat tick。 |
| 音效反馈 | tick 事件批次、delivery context、idempotency key | 命中、格挡、爆气、倒计时等音效 | 音效只消费事件；音频延迟不能影响战斗结果。 |
| Web 平台壳与浏览器焦点 | focus/blur、page visibility、canvas/input focus、fullscreen、audio unlock 状态 | `focus_suspended` 请求、恢复请求、focus reason | Web 平台壳负责告知运行时失焦原因；运行时负责停止/恢复 combat tick 并隔离恢复确认输入。 |
| 暂停与基础菜单 | pause/resume/restart/exit 请求、UI confirm input | runtime state、state reason、requires_player_confirm、combat input policy | 菜单可暂停战斗；恢复确认输入由 UI/menu context 消费，不能进入 combat command flow。 |
| 调参与调试显示 | bounded tick trace、状态快照、事件批次、rejected-request warnings | 可视化判定框、状态文本、日志 | 调试系统只读；不得成为 gameplay 逻辑入口；trace/log 输出必须有窗口和 payload 上限。 |
| 美术资产生产管线 | 动画帧、sprite sheet、VFX anchor、metadata | 数据校验错误、缺失引用报告 | 运行时依赖已验证 metadata；资产生成工具不能直接定义运行时规则。 |
| 基础可访问性与按键提示 | runtime state、输入上下文、事件 | 按键提示、状态提示、可读反馈 | 可访问性反馈只能展示或辅助理解，不能改变战斗判定。 |

本节刻意不定义本地双人、训练课题、多角色差异、能量波对轰、联网同步、rollback 或正式 replay；这些都不是 MVP 的固定逻辑运行时范围。

## Formulas

### Formula Ownership

本系统只拥有“固定逻辑时间如何换算、推进、排序、冻结和记录”的公式。它不拥有伤害、气槽收益、连段缩放、移动速度、气弹速度、AI 反应时间或动画/VFX timing 的平衡公式。

### Combat Tick Duration

The `combat_tick_duration_seconds` formula is defined as:

`combat_tick_duration_seconds = 1 / combat_ticks_per_second`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Combat ticks per second | `combat_ticks_per_second` | int | `30–120` valid range；MVP locked `60` | 战斗权威 tick rate。非 `60` 值只允许 approved test config 或后续 ADR 调整。 |
| Combat tick duration seconds | `combat_tick_duration_seconds` | float | `> 0` | 一个 combat tick 代表的真实时间长度。 |

**Output Range:** `0.008333...–0.033333...` 秒；MVP 默认 `60` ticks/sec 时，输出约为 `0.0166667` 秒。  
**Example:** `combat_tick_duration_seconds = 1 / 60 = 0.0166667 seconds`

### Seconds To Combat Ticks

The `seconds_to_combat_ticks` formula is defined as:

`duration_ticks = max(0, ceil(duration_seconds * combat_ticks_per_second))`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Duration seconds | `duration_seconds` | float | finite number；negative clamps to `0` | 其他系统以秒为单位提供的持续时间；负数按 `0` 处理。 |
| Combat ticks per second | `combat_ticks_per_second` | int | `30–120` valid range；MVP locked `60` | 战斗权威 tick rate。非 `60` 值只允许 approved test config 或后续 ADR 调整。 |
| Duration ticks | `duration_ticks` | int | `>= 0` | 转换后的整 tick 数。 |

**Output Range:** 非负整数。使用 `ceil`，保证正持续时间不会因为换算被缩短。  
**Example:** `duration_ticks = max(0, ceil(0.25 * 60)) = 15 ticks`

如果下游 GDD 已直接用 tick 编写数值，则不需要使用本换算公式。

### Next Committed Tick Index

The `next_committed_tick_index` formula is defined as:

`next_committed_tick_index = current_committed_tick_index + 1`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Current committed tick index | `current_committed_tick_index` | int | `>= 0` | 最近一次已提交的 runtime tick index。 |
| Next committed tick index | `next_committed_tick_index` | int | `>= 1` | 下一次已提交 runtime tick 的 index。 |

**Output Range:** 非负递增整数。tick `0` 是 initial snapshot / trace baseline；第一枚真实 committed runtime tick 是 `1`。只在 `running` 和 `hitstop` 实际提交 runtime tick 时递增；`inactive`、`countdown`、`paused`、`focus_suspended`、`round_ended` 中不递增。  
**Example:** 如果 `current_committed_tick_index = 1842`，下一个 running 或 hitstop tick 提交后，`next_committed_tick_index = 1843`。

### Elapsed Running Ticks

The `elapsed_running_ticks` formula is defined as:

`elapsed_running_ticks = sum(running_tick_flag_i for i = 1 to committed_tick_count)`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Running tick flag | `running_tick_flag_i` | int | `{0, 1}` | 第 `i` 个 committed runtime tick 是否处于 `running` 状态；是则为 `1`，hitstop tick 或其他非 running 记录为 `0`。 |
| Committed tick count | `committed_tick_count` | int | `>= 0` | 被统计的 committed runtime tick 数，包含 running tick 和 hitstop tick。 |
| Elapsed running ticks | `elapsed_running_ticks` | int | `0–committed_tick_count` | 真正计入 active round time 的 running tick 数。 |

**Output Range:** `0` 到 `committed_tick_count`。不计入 `hitstop`、`paused`、`focus_suspended`、`countdown`、`inactive` 和 `round_ended`。  
**Example:** 如果 tick `0` 是 initial snapshot，之后本回合已提交 `970` 个 runtime ticks，其中 `930` ticks 为 `running`、`40` ticks 为 `hitstop`，则 `elapsed_running_ticks = 930`。`paused` 不会产生 committed runtime tick。

### Elapsed Running Time Seconds

The `elapsed_running_time_seconds` formula is defined as:

`elapsed_running_time_seconds = elapsed_running_ticks / combat_ticks_per_second`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Elapsed running ticks | `elapsed_running_ticks` | int | `>= 0` | 已经过的 active running tick 数。 |
| Combat ticks per second | `combat_ticks_per_second` | int | `30–120` valid range；MVP locked `60` | 战斗权威 tick rate。非 `60` 值只允许 approved test config 或后续 ADR 调整。 |
| Elapsed running time seconds | `elapsed_running_time_seconds` | float | `>= 0` | 换算成秒的 active round time。 |

**Output Range:** 非负秒数；因为来源是 `elapsed_running_ticks`，所以不包含 hitstop、pause 或 focus suspension。  
**Example:** `elapsed_running_time_seconds = 930 / 60 = 15.5 seconds`

### Round Timer Remaining Ticks

The `round_timer_remaining_ticks` formula is defined as:

`round_timer_remaining_ticks = max(0, round_duration_ticks - elapsed_running_ticks)`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Round duration ticks | `round_duration_ticks` | int | `>= 0` | 回合总时长对应的 tick 数。具体回合长度由“血量、伤害、计时与胜负”GDD 拥有。 |
| Elapsed running ticks | `elapsed_running_ticks` | int | `>= 0` | 已经过的 active running tick 数。 |
| Round timer remaining ticks | `round_timer_remaining_ticks` | int | `0–round_duration_ticks` | 当前剩余 active round time。 |

**Output Range:** `0` 到 `round_duration_ticks`。hitstop、pause、focus suspension、countdown、inactive 和 round end 后都不减少。  
**Example:** 如果 90 秒回合在 60 ticks/sec 下为 `round_duration_ticks = 90 * 60 = 5400`，且 `elapsed_running_ticks = 930`，则 `round_timer_remaining_ticks = max(0, 5400 - 930) = 4470 ticks`。

### Hitstop Countdown

The `hitstop_remaining_ticks_after_tick` formula is defined as:

`hitstop_remaining_ticks_after_tick = max(0, hitstop_remaining_ticks_before_tick - hitstop_tick_flag)`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Hitstop remaining before tick | `hitstop_remaining_ticks_before_tick` | int | `>= 0` | 当前 tick 处理前剩余的 hitstop tick 数。 |
| Hitstop tick flag | `hitstop_tick_flag` | int | `{0, 1}` | 当前已提交 tick 是否为 hitstop tick；是则为 `1`，否则为 `0`。 |
| Hitstop remaining after tick | `hitstop_remaining_ticks_after_tick` | int | `>= 0` | 当前 tick 处理后剩余的 hitstop tick 数。 |

**Output Range:** 非负整数。只在 `hitstop` 中减少；`paused` 和 `focus_suspended` 中不减少。  
**Example:** 如果 hitstop 剩余 `8` ticks，当前 tick 是 hitstop tick，则 `hitstop_remaining_ticks_after_tick = max(0, 8 - 1) = 7`。如果当前处于 pause，则 `hitstop_tick_flag = 0`，剩余仍为 `8`。

### Runtime Event Sequence Key

The `runtime_event_sequence_key` formula is defined as:

`runtime_event_sequence_key = (round_instance_sequence, committed_tick_index, phase_order_namespace, tick_phase_order, within_phase_event_index)`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Round instance sequence | `round_instance_sequence` | int | `>= 1` | 当前回合实例的单调递增序号；用于排序。`round_instance_id` 可继续作为 equality/idempotency 分区，但不能混用 int/string 参与排序。 |
| Committed tick index | `committed_tick_index` | int | `>= 1` for runtime events; `0` for initial snapshot/reference events | 事件事实关联的 runtime tick。非 advancing state transition 使用 `runtime_state_sequence_key`，不伪装成 combat event。 |
| Phase order namespace | `phase_order_namespace` | enum | `running_tick_order` / `hitstop_tick_order` / `runtime_state_transition_order` / `debug_trace_order` | phase order 属于哪个顺序空间。 |
| Tick phase order | `tick_phase_order` | int | namespace-defined | 对应 namespace 内的阶段编号。 |
| Within-phase event index | `within_phase_event_index` | int | `>= 0` | 同一 tick、同一 namespace、同一阶段内事件的确定性顺序。 |
| Runtime event sequence key | `runtime_event_sequence_key` | tuple | `(int, int, enum, int, int)` | 用于事件排序和去重的字典序 key。 |

**Output Range:** 五元组。先按 round sequence 排序，再按 committed tick 排序，再按 namespace 和 phase 排序，最后按阶段内事件顺序排序。  
**Example:** Event A 的 key 为 `(3, 1200, running_tick_order, 6, 2)`，Event B 的 key 为 `(3, 1200, running_tick_order, 7, 0)`；A 先于 B，因为阶段 `6` 早于阶段 `7`。如果 hitstop debug event 的 key 为 `(3, 1201, hitstop_tick_order, 6, 0)`，它不会误用 running tick phase。快速重开后 `round_instance_sequence = 4` 的事件不会与旧 round 冲突。

### Runtime State Sequence Key

The `runtime_state_sequence_key` formula is defined as:

`runtime_state_sequence_key = (round_instance_sequence, state_transition_index)`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Round instance sequence | `round_instance_sequence` | int | `>= 1` | 当前 round 的单调递增序号。 |
| State transition index | `state_transition_index` | int | `>= 0` | 本 round 内非 combat state transition 的单调递增序号。 |
| Runtime state sequence key | `runtime_state_sequence_key` | tuple | `(int, int)` | pause/focus/recovery/resume/restart 等 UI/runtime state 记录的排序和幂等 key。 |

**Output Range:** 二元组。它排序 runtime state transition records，不用于 combat gameplay event ordering。  
**Example:** `(3, 12)` 表示 round sequence `3` 的第 12 个 state transition。重复收到同一 key 的 pause/focus/resume presentation record 不得重复播放 UI/audio one-shot。

### Trace Window Bounds

The `trace_window_bounds` formula is defined as:

`trace_window_bounds = [max(0, latest_committed_tick_index - trace_window_ticks + 1), latest_committed_tick_index]`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Latest committed tick index | `latest_committed_tick_index` | int | `>= 0` | 最近一次已提交 runtime tick。 |
| Trace window ticks | `trace_window_ticks` | int | `>= 1` | QA/debug trace 保留的最近 tick 数。 |
| Trace window bounds | `trace_window_bounds` | int pair | `[0–latest_committed_tick_index]` | trace 保留的闭区间 tick 范围。 |

**Output Range:** 闭区间 `[start_tick, latest_committed_tick_index]`，起点最小为 `0`。本公式只定义开发/QA trace 窗口，不定义正式玩家 replay。  
**Example:** 如果 `latest_committed_tick_index = 2400`，`trace_window_ticks = 180`，则 `trace_window_bounds = [max(0, 2400 - 180 + 1), 2400] = [2221, 2400]`。

### Backlog Tick Count

The `backlog_tick_count` formula is defined as:

`backlog_tick_count = max(0, floor(max(0, delayed_real_seconds) / combat_tick_duration_seconds))`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Delayed real seconds | `delayed_real_seconds` | float | finite number；negative clamps to `0` | 运行时未能按预期处理 combat tick 的真实时间延迟；平台时钟异常产生负数时按 `0` 处理。 |
| Combat tick duration seconds | `combat_tick_duration_seconds` | float | `> 0` | 一个 combat tick 的真实时间长度。 |
| Backlog tick count | `backlog_tick_count` | int | `>= 0` | 延迟对应的积压 tick 数。 |

**Output Range:** 非负整数。小于一个 tick 的延迟输出 `0`；负延迟也输出 `0`。本公式只识别积压大小，不规定底层 accumulator 实现。  
**Example:** 在 60 ticks/sec 下，`combat_tick_duration_seconds = 1 / 60 = 0.0166667`。如果延迟为 `0.09` 秒，则 `backlog_tick_count = max(0, floor(max(0, 0.09) / 0.0166667)) = 5 ticks`。如果延迟为 `-0.01` 秒，则输出 `0 ticks`。

### Backlog Classification

The `backlog_classification` formula is defined as:

```text
backlog_classification =
  focus_suspended, if focus_lost = true
  none, if focus_lost = false and backlog_tick_count = 0
  in_order_catch_up, if focus_lost = false and 1 <= backlog_tick_count <= catch_up_max_ticks
  recovery_pause, if focus_lost = false and backlog_tick_count > catch_up_max_ticks
```

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Focus lost | `focus_lost` | bool | `{true, false}` | 浏览器/窗口/输入焦点是否丢失。 |
| Backlog tick count | `backlog_tick_count` | int | `>= 0` | 积压 combat tick 数。 |
| Catch-up max ticks | `catch_up_max_ticks` | int | MVP safe `2–8`；debug/stress may use `0` | 允许按顺序补跑的最大积压 tick 数；MVP 默认 `6`，`0` 只允许显式 debug/stress config。 |
| Backlog classification | `backlog_classification` | enum | `{focus_suspended, none, in_order_catch_up, recovery_pause}` | 运行时对积压情况的处理类别。 |

**Output Range:** 四种明确类别之一。focus lost 优先级最高；大积压不得静默丢 tick 或无提示快进。`recovery_pause` 是进入 `paused` 的原因，不是独立 runtime state。  
**Example:** 如果 `focus_lost = false`、`backlog_tick_count = 5`、`catch_up_max_ticks = 6`，则 `backlog_classification = in_order_catch_up`。如果 `focus_lost = true`，则无论积压多少，`backlog_classification = focus_suspended`。

### Catch-up Wall-clock Guard

The `catch_up_wall_clock_guard_exceeded` formula is defined as:

`catch_up_wall_clock_guard_exceeded = measured_catch_up_wall_clock_ms >= catch_up_wall_clock_budget_ms`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Measured catch-up wall-clock ms | `measured_catch_up_wall_clock_ms` | float | `>= 0` | 本 host update 内 catch-up runtime work 已消耗的实际 wall-clock 时间，包含 simulation、snapshot commit、event batch creation、trace instrumentation 和 runtime delivery metadata。 |
| Catch-up wall-clock budget ms | `catch_up_wall_clock_budget_ms` | float | MVP default `4.0`; safe range `2.0–6.0` | 单次 host update 允许 catch-up runtime work 使用的 wall-clock 上限。 |
| Catch-up wall-clock guard exceeded | `catch_up_wall_clock_guard_exceeded` | bool | `{true, false}` | 是否必须停止本次 silent catch-up。 |

**Output Range:** boolean。此 guard 优先于 `catch_up_max_ticks`；即使 backlog tick 数仍在安全范围内，只要 wall-clock guard 触发，就必须在安全 tick boundary 停止并进入 `recovery_pause` 或显式恢复流程。  
**Example:** 如果 `measured_catch_up_wall_clock_ms = 4.3`、`catch_up_wall_clock_budget_ms = 4.0`，则 guard exceeded 为 `true`。

### Catch-up Stop Reason

The `catch_up_stop_reason` output is defined as one of:

`none | completed_backlog | blocking_state | wall_clock_guard | fairness_guard | focus_suspended | invalid_backlog`

**Output Range:** 上述 enum 之一。每次 catch-up 尝试必须 trace：`backlog_tick_count`、`catch_up_ticks_committed`、`remaining_backlog_ticks`、`catch_up_stop_reason`，以及 fairness guard 触发时的 `blocked_transition_type` 和 tick id。

### Explicit Non-Ownership

以下公式和值不属于本 GDD，必须由对应系统定义：

| Formula / Value | 不属于本系统的原因 | Owner GDD |
|---|---|---|
| 伤害数值、chip damage、guard damage | 运行时只提交伤害事实，不决定伤害平衡。 | 血量、伤害、计时与胜负；防御、格挡与受击反馈 |
| 气槽获得、消耗、再生 | 运行时只提交气槽事件，不设计资源经济。 | 气槽与爆气反杀 |
| hitstun/blockstun 具体时长 | 运行时负责倒计时，不拥有每招硬直平衡。 | 命中停顿与硬直窗口；角色数据与招式数据 |
| combo scaling、取消窗口宽容度 | 属于连段和招式平衡。 | 短连招与取消规则 |
| 移动速度、跳跃速度、冲刺距离 | 属于移动手感和平衡。 | 移动与距离控制 |
| 气弹速度、生命周期、轨迹 | 属于 projectile 玩法。 | 气弹 / 能量攻击 |
| 输入缓冲窗口 | 属于输入系统的手感设计。 | 输入映射与输入缓冲 |
| CPU 反应延迟和决策频率 | 属于 AI/脚本行为。 | 简单脚本 CPU |
| 动画插值、相机平滑、VFX timing | 属于表现层。 | 动画播放与 sprite 表现；VFX 可读性系统；星核拳馆场景与战斗相机 |
| 默认回合长度 | 运行时负责倒计时，回合规则负责长度。 | 血量、伤害、计时与胜负 |
| `catch_up_max_ticks` 的最终实现策略 | 本 GDD 定义 MVP 默认值、安全范围和分类规则；具体 accumulator / recovery-pause 实现依赖 Web 性能策略。 | Runtime ADR / Technical Design |

### Registry Requirements

本系统的跨系统常量和公式必须登记到 `design/registry/entities.yaml`。已登记或本次修订必须补登记的项目如下：

| Registry Item | Type | Value / Expression | Why |
|---|---|---|---|
| `combat_ticks_per_second` | constant | `60` ticks/second | 所有战斗、输入、移动、HUD、debug trace 和测试必须共享同一 tick rate。 |
| `catch_up_max_ticks` | constant | `6` ticks | runtime、QA 和浏览器恢复行为需要共享阈值。 |
| `trace_window_ticks` | constant | `180` ticks | QA/debug trace 需要统一保留窗口。 |
| `combat_tick_duration_seconds` | formula | `1 / combat_ticks_per_second` | 下游系统需要从 tick rate 推导真实秒长。 |
| `seconds_to_combat_ticks` | formula | `max(0, ceil(duration_seconds * combat_ticks_per_second))` | 下游系统若提供秒数，必须统一向上换算。 |
| `next_committed_tick_index` | formula | `current_committed_tick_index + 1` | running/hitstop committed runtime tick 必须统一递增。 |
| `elapsed_running_ticks` | formula | `sum(running_tick_flag_i)` | timer、HUD 和 QA trace 需要区分 running tick 与 hitstop tick。 |
| `elapsed_running_time_seconds` | formula | `elapsed_running_ticks / combat_ticks_per_second` | UI/QA 需要统一 active running time。 |
| `round_timer_remaining_ticks` | formula | `max(0, round_duration_ticks - elapsed_running_ticks)` | 回合 timer 需要排除 hitstop、pause、focus。 |
| `hitstop_remaining_ticks_after_tick` | formula | `max(0, hitstop_remaining_ticks_before_tick - hitstop_tick_flag)` | hitstop countdown 需要跨系统一致。 |
| `runtime_event_sequence_key` | formula | `(round_instance_id, committed_tick_index, tick_phase_order, within_phase_event_index)` | HUD/VFX/音效/debug 都依赖同一排序和去重 key。 |
| `trace_window_bounds` | formula | `[max(0, latest_committed_tick_index - trace_window_ticks + 1), latest_committed_tick_index]` | QA/debug trace 需要统一保留窗口。 |
| `backlog_tick_count` | formula | `max(0, floor(max(0, delayed_real_seconds) / combat_tick_duration_seconds))` | Web backlog 分类必须统一且 clamp negative delay。 |
| `backlog_classification` | formula | focus/backlog enum classification | pause/focus/recovery 行为必须跨 runtime、UI、QA 一致。 |

`runtime_tick_phase_order` 是 deterministic invariant，不作为 tuning knob；后续如果 registry 支持 enum/list invariant，可再登记。

## Edge Cases

### Tick Scheduling, Backlog, and Browser Timing

- **If `delayed_real_seconds < combat_tick_duration_seconds`**: 不提交新的 runtime tick；表现层只能重绘最近一次已提交 snapshot。
- **If `backlog_tick_count = 0`**: 不执行 catch-up；运行时等待累计到足够真实时间后再提交下一 tick。
- **If `1 <= backlog_tick_count <= catch_up_max_ticks`**: 运行时最多按顺序补跑 `backlog_tick_count` 个 ticks，每个 tick 单独提交 snapshot 和 event batch；如果途中触发 blocking state、wall-clock guard 或 fairness guard，则提前停止，并记录 `catch_up_stop_reason`。
- **If `backlog_tick_count > catch_up_max_ticks`**: 运行时进入 `runtime_state = paused`，`state_reason = recovery_pause`；不静默跳 tick，不一次性快进，不在隐藏状态下推进战斗。
- **If `catch_up_max_ticks = 0` and `backlog_tick_count > 0`**: 任何积压都进入 `recovery_pause`；`0` 只允许显式 debug/stress config，不是 MVP 正常配置。
- **If focus is lost while backlog exists**: `focus_suspended` 优先于 catch-up；运行时停止 combat advancement，清理/忽略积压，不在恢复焦点后补跑失焦期间错过的 ticks。
- **If delayed time is negative due to clock/platform anomaly**: 将 delay 当作 `0`；不提交反向 tick，并记录 debug/trace warning。
- **If catch-up processing reaches a tick that enters `hitstop`, `paused`, `focus_suspended`, or `round_ended`**: 在提交该 transition tick 后停止本次 catch-up；不得继续补跑后续 ticks。
- **If catch-up processing reaches or exceeds `catch_up_wall_clock_budget_ms`**: 在最近一个安全 tick boundary 停止补跑并进入 `runtime_state = paused`、`state_reason = recovery_pause`；不得为了追赶时间制造更长主线程卡顿。
- **If catch-up tick emits a fairness-stop event**: 允许提交触发该事件的 tick，但必须在继续推进下一枚 combat tick 前进入 `runtime_state = paused`、`state_reason = recovery_pause`，并通过恢复 UI/倒计时让玩家重新接管。MVP fairness-stop events 至少包括 `startup_started`、`active_frame_started`、`recovery_started`、`punish_window_opened`、`burst_started`、`hit_landed`、`blocked`、`ko_started`、`round_ended`。
- **If catch-up needs an input snapshot for a delayed target tick**: 只能使用该目标 tick 已排队/已采集的 input snapshot；不得 retroactively 读取当前 host frame 的按键状态来填补过去 tick。
- **If CPU/dummy decision generation would observe an unpresented fairness-stop transition during catch-up**: 停止 catch-up 并进入 `recovery_pause`；CPU/木桩不得在玩家尚未看到的关键 transition 后继续生成下一 tick 反应。
- **If runtime enters `paused` because of `recovery_pause`**: backlog/accumulator 不得继续增长；恢复确认后从新的安全 timing baseline 继续，不能 replay 暂停期间真实时间。
- **If multiple catch-up ticks run inside one host/render frame**: 每个 tick 仍必须执行完整固定 tick order，并各自产生独立 event batch；逻辑不得合并 tick。event batch 必须带 catch-up delivery metadata，供 HUD/VFX/audio 压缩表现。

### Atomic Tick Boundaries

- **If a runtime request, input, physics callback, animation signal, UI event, or debug command arrives mid-tick**: 它不能修改当前 tick；只能排队到下一个合法 tick boundary，或在无效时被拒绝。
- **If a tick begins processing**: 它必须作为一个完整 snapshot + event batch 原子提交；如果 fatal runtime error 在提交前中止，则该 tick 不得半提交。
- **If a state change is calculated during a tick**: 该变化只在 tick commit boundary 后可见；其他系统不得读取半更新状态。
- **If a command becomes invalid before the state-machine phase resolves it**: 该 command 在本 tick 被拒绝，不启动 partial action。
- **If presentation, HUD, audio, VFX, camera, debug UI, animation playback, or Godot physics attempts to change combat state**: 该变化必须被忽略/拒绝，并记录 runtime violation。

### State Priority and Transitions

- **If multiple runtime states are requested at the same tick boundary**: 使用优先级 `round_ended > focus_suspended > paused > hitstop > running > countdown > inactive` 决定最终状态。
- **If `round_ended` is reached in the same tick as a hitstop request**: 进入 `round_ended`；不进入或继续 `hitstop`。
- **If pause is requested while in `running`**: 在下一个 combat-action phase 前停止 combat advancement；paused 期间不执行移动、攻击、命中、伤害、timer、气槽或 projectile 逻辑。
- **If pause is requested while in `hitstop`**: 冻结当前 `hitstop_remaining_ticks`；resume 后先执行安全恢复确认/倒计时，再回到 hitstop，并保留原剩余值，除非更高优先级状态已经替换它。
- **If focus is lost while in `running`, `hitstop`, `paused`, or `countdown`**: 进入 `focus_suspended`；不接受新战斗输入，不推进 combat tick，不累计 catch-up。
- **If focus is restored after `focus_suspended`**: 运行时必须显示恢复原因，清理或重新确认 held input；确认输入由 UI/menu context 消费，随后进入 `resume_ready` / `resume_go` 倒计时或等价 reacquisition 流程；攻击、防御、气弹、爆气和移动不得因为恢复焦点或确认键自动触发。
- **If focus was lost while already `paused`**: 恢复焦点后回到 `paused`，不得自动回到 `running`；pause menu 内部焦点移动不得被错误升级为 `focus_suspended`。
- **If `round_ended` is active**: 忽略所有战斗 command；只允许 restart、exit 或菜单确认类 request。
- **If an invalid state transition is requested, such as `inactive -> hitstop` or `round_ended -> running` without restart/countdown**: 拒绝该 transition，保持当前合法状态，并记录 debug/trace warning。
- **If quick restart is requested during `running`, `hitstop`, `paused`, `focus_suspended`, or `round_ended`**: 必须先清空旧 round 的 hitstop、input buffers、pending runtime requests、actors、projectiles、timers、snapshots 和 event batches，再进入新 round flow。
- **If quick restart creates a new round instance**: 必须生成新的 `round_instance_id`；旧 round event key 不得影响新 round。

### Countdown Behavior

- **If `countdown` is active**: combat tick 不推进，round timer 不减少，攻击、气弹、爆气不能执行。
- **If direction input is held during `countdown`**: 只能作为允许的 pre-read input 被记录；不得在 `running` 前产生移动或 combat state advancement。
- **If attack, projectile, or burst is pressed during `countdown`**: 不得在 countdown 中执行，也不得在第一枚 running tick 自动释放，除非后续输入 GDD 明确允许该类 buffer。
- **If countdown is paused or focus-suspended**: countdown 表现停止/等待；不得隐藏 catch-up 到 round start。

### Hitstop

- **If a running tick creates a valid hitstop request of `N > 0`**: 造成命中/格挡的 running tick 正常提交，然后运行时进入 `hitstop`，剩余 hitstop ticks 为 `N`；造成 hitstop 的 running tick 本身不消耗 1 个 hitstop tick。
- **If a valid hitstop request has `N = 0`**: 不进入 `hitstop`；下一 tick 继续正常 running。
- **If multiple hitstop requests occur in the same tick**: 使用最大 hitstop 值，不累加。
- **If runtime is already in `hitstop`**: 动作帧、移动、hitbox/hurtbox 变化、hitstun/blockstun/recovery 倒计时、round timer 和 projectile combat advancement 全部冻结。
- **If hitstop reaches `0` after a hitstop tick**: 在下一个 tick boundary 回到 `running`；不得在同一枚 hitstop tick 内再执行 normal combat advancement。
- **If input is captured during `hitstop`**: 可以进入输入缓冲，但不得在恢复 `running` 前验证或执行为 combat action。输入系统必须同时获得 `committed_tick_index` 与 running/action-time context；hitstop-captured inputs 至少在 hitstop 后第一枚合法 running tick 仍可被评估，除非输入 GDD 用明确规则拒绝它们。输入缓冲不得仅按 committed runtime tick 在 hitstop 中自然过期。
- **If a projectile would move, expire, collide, or spawn another combat effect during `hitstop`**: 该 projectile combat advancement 冻结到 hitstop 结束。
- **If any system attempts to create a new hit/block/collision result during `hitstop`**: 拒绝该结果；hitbox/hurtbox collision authority 在 hitstop 冻结期间不运行。
- **If a future system asks for local/partial hitstop**: MVP 固定运行时不支持；MVP hitstop 是 global combat freeze。

### Timer and Round End

- **If round timer reaches `0` during a running tick**: 先完成固定 tick order 中的伤害/KO 结算，再解析 timeout/round end。
- **If KO and timeout occur in the same tick**: damage/KO 先结算；timeout 后检查。
- **If `hitstop`, `paused`, `focus_suspended`, `countdown`, `inactive`, or `round_ended` is active**: round timer 不减少。
- **If `round_timer_remaining_ticks` is already `0` before a would-be running tick starts**: 不处理新的 combat command；直接进入 timeout/round-end resolution。
- **If `round_ended` is committed**: 本 round 后续不得再产生 hit、block、movement、timer、energy、projectile 或 combo facts。

### Events, Snapshots, and Trace

- **If a tick commits with no combat events**: 仍必须提交 snapshot；event batch 可以为空，但必须与该 tick 关联。
- **If several events occur in the same tick**: 必须按 `runtime_event_sequence_key` 发布；不得使用 Godot node order、render order、signal order 或偶然数组顺序。
- **If a tick produces hit, block, damage, KO, or round-end facts**: 所有已提交事实必须进入同一个有序 event batch；消费者不得从表现层自行推断缺失战斗事实。
- **If an event consumer processes the same event batch more than once due to redraw, pause/resume, or browser repaint**: gameplay 不得改变；event sequence key / idempotency key 用于让消费者去重或幂等处理；HUD pulse、VFX、audio one-shot 和 debug one-shot 也不得重复播放。
- **If catch-up delivers multiple event batches in one host frame**: delivery context 必须标明 `delivery_mode = catch_up`、批次数量和批次索引；表现层可压缩非权威 pulse/VFX/audio，但不得删除或重排 authoritative facts。
- **If a runtime state transition occurs outside committed runtime tick advancement**: 使用 `runtime_state_sequence_key` 发布 read-only state transition record；不得伪造 committed combat event。
- **If an old event batch from a previous round remains after quick restart**: 它必须被 invalidated；不得影响新 round 的 HUD、VFX、audio、debug state 或 gameplay。
- **If trace history exceeds `trace_window_ticks`**: 可以淘汰更老 trace entries，但当前 combat state 和 event ordering 不得改变。
- **If no gameplay tick has been committed for the current round**: trace 必须报告 empty history 或 tick `0` initial snapshot；不得伪造 running tick。
- **If a non-combat state transition occurs outside gameplay tick advancement, such as pause/focus/restart request**: 可以记录 trace/state-transition record，但不得递增 combat tick index，也不得发布 combat gameplay event batch。

### Authority Boundaries

- **If Godot 2D physics reports a collision but data-driven hitbox/hurtbox logic does not**: 不发生 combat hit/block。
- **If data-driven hitbox/hurtbox logic reports a combat collision but Godot physics does not**: combat collision 仍然成立。
- **If sprite pixels, animation frame timing, VFX timing, audio latency, camera timing, or HUD timing disagree with combat tick data**: combat tick data wins。
- **If hitbox/hurtbox metadata is missing for an action frame**: 运行时不得从 sprite visuals 推断 hitbox；必须使用 empty/invalid combat facts 并记录 data/debug error，或在 match 前由数据验证失败阻止进入战斗。

### Deferred Edge Cases

以下边界情况不在本 GDD 内解决，只要求它们必须通过固定 tick 和 runtime state 规则进入系统：

| Deferred Case | Owner GDD |
|---|---|
| 同一输入快照中左右、上下、攻击/防御、多按钮冲突如何解析 | 输入映射与输入缓冲 |
| pause/focus 恢复后的 held input 是持续、清空还是要求重按 | 输入映射与输入缓冲 |
| 输入缓冲是否在 hitstop、pause、countdown、focus suspension 中过期 | 输入映射与输入缓冲 |
| move cancel、chain、whiff-cancel、hit-confirm、combo reset | 短连招与取消规则；角色数据与招式数据 |
| 防御方向、cross-up、projectile guard、unblockable tag | 防御、格挡与受击反馈 |
| 每招 hitstun、blockstun、hitstop、recovery 数值 | 命中停顿与硬直窗口；角色数据与招式数据 |
| damage、chip damage、health clamp、KO tie-breaker、timeout winner、draw rules | 血量、伤害、计时与胜负 |
| 气槽获得、消耗、refund、burst 可用性、burst invulnerability | 气槽与爆气反杀 |
| projectile speed、lifetime、owner collision、反弹、对波、多弹种 | 气弹 / 能量攻击 |
| wall collision、corner push、facing flip、overlap correction、jump/dash movement | 移动与距离控制 |
| CPU reaction delay、decision frequency、script priority、dummy behavior | 简单脚本 CPU；训练木桩 |
| catch-up 后 HUD/VFX/audio 如何压缩或展示多个 event batches | HUD 与战斗信息反馈；VFX 可读性系统；音效反馈 |
| rollback、networking、本地双人、正式 replay、beam clash、多角色差异 | 非 MVP；对应后续系统 GDD |

## Dependencies

### Upstream Dependencies

本系统是 MVP Foundation 层的第一个系统，**没有其他 GDD 系统作为上游硬依赖**。它可以被设计和实现为战斗运行时的底层合同。

| Dependency | Type | Required? | Notes |
|---|---|---:|---|
| Game Concept | Product/design constraint | Yes | 必须服务“读招定胜负”“短连招，高回合”“热血能量必须可读”。 |
| Systems Index | Production/design order constraint | Yes | 本系统是 design order #1，其他 MVP combat systems 依赖它。 |
| Technical Preferences | Technical constraint | Yes | Godot 4.6.2、GDScript、Godot 2D Canvas、Web/Browser、60fps target、custom data-driven hitbox/hurtbox。 |
| Engine Reference Docs | Technical risk constraint | Yes | Godot 4.6.2 为 HIGH risk；实现前必须用 ADR/engine-reference 验证 Godot API。 |
| Art Bible | Presentation constraint | Soft | 运行时本身不产生美术，但其事件合同必须支持可读 HUD/VFX/音效表现。 |

### Direct Downstream Dependents

以下系统在 `systems-index.md` 中直接依赖固定逻辑运行时，后续 GDD 必须反向引用本系统：

| Dependent System | Dependency Type | What This Runtime Provides | What The Dependent Must Not Do |
|---|---|---|---|
| 输入映射与输入缓冲 | Hard | tick id、runtime state、何时采集/执行/冻结/清理输入 | 不得直接绕过 runtime 执行动作。 |
| 移动与距离控制 | Hard | 固定 tick、提交边界、running/hitstop/pause/focus 状态 | 不得用渲染帧或动画 timing 改变真实位置。 |
| 战斗状态机 | Hard | tick 顺序、状态提交边界、runtime state priority | 不得在 tick 中途暴露半更新状态。 |
| Hitbox / Hurtbox 判定 | Hard | 判定阶段、tick authority、event batch 输出 | 不得依赖 Godot physics callback、sprite 像素或动画信号决定 combat hit。 |
| 血量、伤害、计时与胜负 | Hard | running tick timebase、round timer 减少规则、KO/timeout 顺序 | 不得在 hitstop、pause、focus、countdown 中减少 round timer。 |
| 命中停顿与硬直窗口 | Hard | hitstop 状态、hitstop countdown、硬直冻结规则 | 不得让 hitstun/blockstun 在 hitstop 中减少。 |
| 对局流程与快速重开 | Hard | runtime state transitions、round instance、restart reset boundary | 不得复用旧 round event batches 或旧 runtime state。 |
| Web 平台壳与浏览器焦点 | Hard | `focus_suspended` 合同、backlog/focus 恢复规则 | 不得让浏览器失焦期间隐藏推进战斗。 |
| 暂停与基础菜单 | Hard | `paused` 状态、pause reason、resume boundary | 不得在 paused 中执行 combat command。 |
| 调参与调试显示 | Hard | tick trace、snapshot、event sequence key | 调试显示只读，不得成为 gameplay authority。 |
| 本地双人对战 | Later-scope hard | 同一 runtime tick authority 和 event ordering | Alpha 前不得推动 rollback、netcode 或正式 replay 需求进入 MVP。 |

### Indirect Downstream Dependents

以下系统未必在 systems-index 中直接写依赖本系统，但会通过状态机、判定、伤害、防御、硬直、气槽或对局流程间接依赖它：

| Indirect Dependent | Runtime Contract It Relies On |
|---|---|
| 防御、格挡与受击反馈 | 同 tick guard/hit 结算顺序、blockstun/hitstun 不在 hitstop 中减少。 |
| 短连招与取消规则 | tick 边界、cancel window 解释、事件顺序和短回合重置。 |
| 气槽与爆气反杀 | energy event 是已提交事实；HUD/VFX 不能修改气槽。 |
| 气弹 / 能量攻击 | projectile combat advancement 受 running/hitstop/pause/focus 状态控制。 |
| 训练木桩 | 木桩命令必须走同一 command/state-machine/runtime 路径。 |
| 简单脚本 CPU | CPU 命令必须走同一 command/state-machine/runtime 路径。 |
| HUD 与战斗信息反馈 | 只消费 snapshot 和 ordered event batches。 |
| VFX 可读性系统 | 只消费 committed combat events，不驱动判定。 |
| 动画播放与 sprite 表现 | 动画帧跟随 runtime state，不驱动 combat result。 |
| 音效反馈 | 只消费事件；音频延迟不影响战斗。 |
| 基础可访问性与按键提示 | 展示 runtime state 和输入上下文，不修改战斗规则。 |

### External Non-Dependencies

以下内容不是本系统依赖，不能成为 MVP fixed runtime 的前置条件：

| Non-Dependency | Reason |
|---|---|
| rollback netcode | MVP 不做联网、回滚或同步预测。 |
| formal player replay | MVP trace 只服务 QA/debug，不服务玩家回放。 |
| local versus | Alpha later scope，不应扩大 MVP runtime。 |
| multi-character framework | Alpha later scope；MVP 只需支持单一均衡武斗家及镜像/木桩/CPU 复用。 |
| beam clash | Full Vision optional，不得污染 MVP projectile/runtime 合同。 |
| advanced physics simulation | 格斗命中权威来自数据驱动 hitbox/hurtbox，不来自复杂 physics。 |

### Bidirectional Consistency Requirements

- 本 GDD 必须在下游 GDD 中被引用为 tick authority、runtime state authority、event ordering authority。
- 任何下游 GDD 如果定义 timing、window、duration、event、pause、focus、trace、round restart，都必须说明它如何接入本系统。
- 如果下游 GDD 需要改变本系统定义的 tick rate、state priority、event sequence key、hitstop freeze scope 或 backlog behavior，必须先回到本 GDD 修改，而不是在下游 GDD 中局部覆盖。
- 当前尚未有下游系统 GDD 完成，因此不存在已写文档的双向冲突；后续每完成一个依赖本系统的 GDD，都必须检查是否反向引用 `fixed-logic-runtime`。

## Tuning Knobs

本系统只拥有固定运行时本身的调参项：tick rate、短暂掉帧补跑、QA trace 窗口、countdown 输入预读策略、浏览器失焦恢复策略。伤害、硬直、招式帧数据、输入缓冲、移动速度、气弹速度、CPU 延迟、VFX/音效时长和回合长度都不属于本系统。

| Knob | MVP 默认 / 建议 | 安全范围 | 影响什么 | 过低 / 关闭会坏什么 | 过高 / 开启过度会坏什么 | Owner / Source of Truth |
|---|---:|---:|---|---|---|---|
| `combat_ticks_per_second` | `60` ticks/sec | `30–120`；MVP 锁定 `60` | 所有战斗时间粒度、帧数据解释、timer、trace、自动化测试。对应公式：`combat_tick_duration_seconds`、`seconds_to_combat_ticks`。 | 战斗 timing 变粗；短前摇、短确反、短连段窗口难以表达；玩家会感觉判定不精确。 | 浏览器 CPU 压力、事件量和 trace 量上升；所有下游 tick 数据都要重写；测试维护成本变高。 | `fixed-logic-runtime` GDD；跨系统常量，后续应登记到 registry。 |
| `catch_up_max_ticks` | `6` ticks，约 `0.10s` at 60 ticks/sec | `2–8`；`0` 只适合 debug/stress 模式 | 浏览器短暂卡顿后允许按顺序补跑多少 combat ticks。对应公式：`backlog_classification`。 | 轻微卡顿也频繁进入 `recovery_pause`，战斗被打断，玩家感觉游戏“太敏感”。 | 一次补跑太多 tick，可能造成主线程长帧、输入/表现跳跃，甚至 spiral-of-death。 | GDD 定义语义；最终数值由 Runtime ADR / Web 性能验证锁定。 |
| `trace_window_ticks` | `180` ticks，约 `3s` at 60 ticks/sec | `120–600`；更长只用于 dev/QA 临时抓取 | QA/debug 能回看最近多少 tick 的输入、状态变化、snapshot 和 event batch。对应公式：`trace_window_bounds`。 | 很容易丢失 bug 起因，例如输入、hitstop、pause/focus、hit/block 前的上下文。 | trace 噪音和内存/日志体积变大，Web 调试可能变慢，QA 更难定位重点。 | `fixed-logic-runtime` debug/QA config；不是玩家 balance。 |
| `countdown_direction_pre_read_enabled` | `true`，仅方向输入 | `{true, false}` | round start 前是否允许记录方向 held input；不允许攻击、气弹、爆气在第一枚 running tick 自动释放。 | 玩家在 countdown 期间按住方向无效，开局第一 tick 可能感觉迟钝，需要重新按键。 | 如果扩展到攻击/气弹/爆气，会造成开局自动出招；如果方向预读规则不清，会产生 round-start option select。 | Runtime 决定 countdown 状态允许读取什么；输入 GDD 负责方向冲突和 held input 细则。 |
| `focus_restore_requires_explicit_confirm` | `true` | `{true, false}`；`false` 只适合 debug 或已验证安全的特殊模式 | 浏览器失焦恢复后是否要求玩家确认再继续；防止 stale held input 和玩家未准备好时继续战斗。 | 恢复焦点时可能直接继续战斗，玩家可能因旧按键、窗口切换或页面恢复而误移动/误防御/误出招。 | 频繁 focus 抖动时会增加确认负担，打断节奏。 | Runtime 拥有 focus 恢复安全策略；菜单/UI 只负责确认提示表现；输入 GDD 负责 held input 清理/重按规则。 |

### Deferred or Non-Runtime Knobs

| Candidate | Decision | Correct Owner |
|---|---|---|
| `max_event_batches_exposed_per_presentation_frame` | MVP 不放入本 GDD。Runtime 必须发布完整、有序、不可丢失的 event batches；表现层如果要压缩展示，由 HUD/VFX/音效策略决定。 | HUD、VFX、Audio、Presentation ADR |
| `input_buffer_window_ticks` | 不属于 fixed runtime。Runtime 只提供 tick id、state 和冻结规则。 | 输入映射与输入缓冲 |
| `resume_input_flush_ticks` / held input 重按窗口 | 不作为本系统 tuning knob。Runtime 只规定“不得自动触发 held input”；具体清理、重按、fresh edge 规则由输入系统定义。 | 输入映射与输入缓冲 |
| `round_duration_ticks` | Runtime 只倒计时，不决定回合长度。 | 血量、伤害、计时与胜负 |
| `hitstop_ticks`、`hitstun_ticks`、`blockstun_ticks` | Runtime 负责冻结和倒计时，不拥有每招时长。 | 命中停顿与硬直窗口；角色数据与招式数据 |
| damage、chip damage、guard damage | Runtime 只提交事件，不做伤害平衡。 | 血量、伤害、计时与胜负；防御、格挡与受击反馈 |
| move startup/active/recovery、cancel window | 属于招式和连段设计。 | 角色数据与招式数据；短连招与取消规则 |
| movement speed、projectile speed、CPU delay | 属于对应玩法系统。 | 移动与距离控制；气弹 / 能量攻击；简单脚本 CPU |
| VFX duration、audio timing、camera smoothing | 表现层可读性问题，不得影响 combat tick。 | VFX、音效、相机 / Presentation GDD |
| `runtime_tick_phase_order`、state priority、event sequence key | 这些是确定性合同，不是 tuning knobs。 | `fixed-logic-runtime` invariant |

### Registry Summary

以下 tuning constants 必须与 `design/registry/entities.yaml` 保持一致：

| Constant | Value | Unit | Why |
|---|---:|---|---|
| `combat_ticks_per_second` | `60` | ticks/second | 所有战斗、输入、移动、HUD、debug trace 和测试必须共享同一 tick rate。 |
| `catch_up_max_ticks` | `6` | ticks | runtime、QA 和浏览器恢复行为需要共享阈值。 |
| `trace_window_ticks` | `180` | ticks | QA/debug trace 需要统一保留窗口。 |

`countdown_direction_pre_read_enabled` 和 `focus_restore_requires_explicit_confirm` 是跨系统策略；如果项目决定把 boolean policy constants 也登记进 registry，后续再加入。

## Acceptance Criteria

### AC-FLR-01 — Combat ticks are the authoritative simulation unit

**Given** a round is in `running` state with `combat_ticks_per_second = 60`, **When** the runtime advances combat for one second of valid running time, **Then** exactly 60 committed combat ticks are produced, tick IDs increase by exactly 1 per committed running tick, and all gameplay state changes are visible only in committed tick snapshots.

### AC-FLR-02 — “Frame” means combat tick unless otherwise specified

**Given** a move, timer, stun duration, recovery duration, or combat rule is described in frames, **When** the value is consumed by the fixed-logic runtime, **Then** the value is interpreted as combat ticks, and presentation frame rate changes do not alter the number of combat ticks required.

### AC-FLR-03 — Tick 0 is the initial snapshot and gameplay begins at tick 1

**Given** a new round instance has been created, **When** the initial combat snapshot is emitted, **Then** the snapshot uses `committed_tick_index = 0`, and no gameplay command, attack, projectile, damage, KO, timeout, or timer decrement has been processed yet.

**Given** the round enters its first valid `running` tick, **When** the first gameplay tick commits, **Then** the committed tick index is `1`.

### AC-FLR-04 — Non-running states do not increment committed gameplay ticks

**Given** the runtime is in `inactive`, `countdown`, `paused`, `focus_suspended`, or `round_ended`, **When** real time passes, **Then** no new committed gameplay tick index is produced, `elapsed_running_ticks` does not increase, and the round timer does not decrease.

### AC-FLR-05 — Running ticks execute in the required deterministic order

**Given** a round is in `running` state, **When** a combat tick is processed, **Then** the observable results match this phase order:

1. runtime requests
2. player, dummy, and CPU commands
3. combat state machine
4. action frames, movement, and timers
5. hitbox and hurtbox facts
6. collision, hit, block, and whiff resolution
7. damage, hitstun, blockstun, hitstop, energy, and combo updates
8. KO, timeout, and round-end checks
9. committed snapshot
10. ordered event batch

**And** events generated in the tick are ordered consistently with the same phase order.

### AC-FLR-06 — State changes commit only at tick boundaries

**Given** an input, CPU command, pause request, hit result, KO result, or other combat-affecting request occurs during a tick, **When** QA observes snapshots and event batches, **Then** no partially applied state is visible inside the same tick, and the committed result appears only in a tick-boundary snapshot or a rejected-request trace warning.

### AC-FLR-07 — Mid-tick external requests are queued or rejected deterministically

**Given** an external combat-affecting request arrives after a tick has started, **When** the current tick commits, **Then** the request does not alter the already-running tick, and the request is either applied at the next valid tick boundary or rejected with an observable trace warning containing the tick ID and reason.

### AC-FLR-08 — Combat tick duration formula is correct

**Given** `combat_ticks_per_second` is configured, **When** QA reads the runtime configuration or trace summary, **Then** `combat_tick_duration_seconds = 1 / combat_ticks_per_second`.

| combat_ticks_per_second | expected combat_tick_duration_seconds |
|---:|---:|
| 60 | 0.016666... |
| 30 | 0.033333... |
| 120 | 0.008333... |

### AC-FLR-09 — Duration conversion uses ceiling and clamps negative values to zero

**Given** a duration in seconds is converted to ticks, **When** the runtime calculates `duration_ticks`, **Then** the result equals `max(0, ceil(duration_seconds * combat_ticks_per_second))`.

| duration_seconds at 60 ticks/sec | expected duration_ticks |
|---:|---:|
| -1.0 | 0 |
| 0.0 | 0 |
| 0.001 | 1 |
| 0.5 | 30 |
| 1.0 | 60 |
| 1.016 | 61 |

### AC-FLR-10 — Next committed tick index formula is correct

**Given** the latest committed runtime tick is `current_committed_tick_index`, **When** one valid `running` tick commits, **Then** `next_committed_tick_index = current_committed_tick_index + 1`, and `elapsed_running_ticks` also increases by 1.

**Given** the latest committed runtime tick is `current_committed_tick_index`, **When** one valid `hitstop` tick commits, **Then** `next_committed_tick_index = current_committed_tick_index + 1`, and `elapsed_running_ticks` does not increase.

**Given** the runtime is in `inactive`, `countdown`, `paused`, `focus_suspended`, or `round_ended`, **When** real time passes, **Then** the committed runtime tick index does not increase.

### AC-FLR-11 — Elapsed running ticks count only running ticks

**Given** a round includes time spent in countdown, running, hitstop, paused, focus suspended, and round ended states, **When** QA checks `elapsed_running_ticks`, **Then** it equals the number of committed `running` ticks only, and it excludes countdown, hitstop, pause, focus suspension, inactive, and round-ended time.

### AC-FLR-12 — Elapsed running time formula is correct

**Given** `elapsed_running_ticks = 90` and `combat_ticks_per_second = 60`, **When** elapsed running time is reported, **Then** `elapsed_running_time_seconds = elapsed_running_ticks / combat_ticks_per_second = 1.5 seconds`.

### AC-FLR-13 — Round timer formula is correct

**Given** a round has `round_duration_ticks` and `elapsed_running_ticks`, **When** the runtime reports remaining round time, **Then** `round_timer_remaining_ticks = max(0, round_duration_ticks - elapsed_running_ticks)`, and the timer never becomes negative.

### AC-FLR-14 — Round timer decreases only during running ticks

**Given** a round timer has remaining time, **When** the runtime spends time in `hitstop`, `paused`, `focus_suspended`, `countdown`, `inactive`, or `round_ended`, **Then** `round_timer_remaining_ticks` does not decrease.

**Given** the runtime commits one valid `running` tick, **When** the tick completes, **Then** `round_timer_remaining_ticks` decreases by exactly 1 unless it is already 0.

### AC-FLR-15 — Runtime state priority is enforced

**Given** multiple runtime state requests are valid in the same observation window, **When** the runtime resolves the active state, **Then** the selected state follows `round_ended > focus_suspended > paused > hitstop > running > countdown > inactive`, and lower-priority states do not continue advancing combat while a higher-priority state is active.

### AC-FLR-16 — Countdown does not execute combat actions

**Given** the runtime is in `countdown`, **When** player, dummy, or CPU attack, projectile, burst, or guard commands are submitted, **Then** attacks, projectiles, burst actions, hitboxes, damage, stun, hitstop, and combo updates do not execute.

**Given** `countdown_direction_pre_read_enabled = true`, **When** direction input is held during countdown, **Then** directional input may be read for facing or readiness only, without producing combat effects.

### AC-FLR-17 — Same-tick simultaneous hits are resolved as simultaneous

**Given** two opposing hitboxes become valid against opposing hurtboxes on the same committed running tick, **When** collision and hit resolution is processed, **Then** both valid hits are resolved as same-tick simultaneous hits, neither hit is removed only because the other hit occurred in the same tick, and the committed snapshot and event batch show both hit facts for that tick.

### AC-FLR-18 — Guard cannot be applied retroactively within the same tick

**Given** an incoming hit is resolved during a committed tick, **When** a guard command becomes available too late to be part of that tick’s command phase, **Then** the hit is not converted into a block retroactively, and the guard command may only affect a later valid tick if still applicable.

### AC-FLR-19 — Multiple same-tick hitstop requests use the maximum value

**Given** multiple hitstop requests are generated in the same committed tick, **When** hitstop is resolved, **Then** the applied hitstop duration equals the maximum requested hitstop duration, and lower hitstop requests do not stack additively.

### AC-FLR-20 — Zero-length hitstop does not enter hitstop state

**Given** a hit or block result requests `0` hitstop ticks, **When** the tick commits, **Then** the runtime does not enter `hitstop` state, and the next valid running tick may proceed normally unless another higher-priority state applies.

### AC-FLR-21 — Hit-causing tick does not consume the first hitstop tick

**Given** a hit occurs on committed running tick `T` and applies `N` hitstop ticks where `N > 0`, **When** tick `T` commits, **Then** the hit result is visible in tick `T`’s snapshot and event batch, `T` itself is not counted as one of the `N` hitstop ticks, and the hitstop countdown begins after tick `T` has committed.

### AC-FLR-22 — Hitstop freezes combat advancement globally

**Given** the runtime is in `hitstop`, **When** real time advances by one or more hitstop ticks, **Then** action frames do not advance, movement does not advance, hitbox and hurtbox facts do not change, hitstun, blockstun, and recovery countdowns do not decrease, the round timer does not decrease, and projectile combat advancement does not proceed.

### AC-FLR-23 — Hitstop still allows input capture and non-authoritative presentation

**Given** the runtime is in `hitstop`, **When** player input, presentation, debug, HUD, VFX, audio, or camera updates occur, **Then** input may be captured for buffering according to input-buffer rules, presentation/debug consumers may update, and no presentation, HUD, VFX, audio, camera, sprite, or animation update changes authoritative combat state.

### AC-FLR-24 — Hitstop remaining formula is correct

**Given** `hitstop_remaining_ticks_before_tick` is greater than 0, **When** one hitstop countdown tick is consumed, **Then** `hitstop_remaining_ticks_after_tick = max(0, hitstop_remaining_ticks_before_tick - hitstop_tick_flag)`.

**Given** `hitstop_remaining_ticks_after_tick = 0`, **When** no higher-priority state applies, **Then** the runtime may return to `running` on the next valid combat advancement.

### AC-FLR-25 — Sub-tick real-time delay commits no gameplay tick

**Given** the accumulated real-time delay is less than `combat_tick_duration_seconds`, **When** the runtime update is processed, **Then** no committed gameplay tick is produced, and no combat state, timer, or event batch changes.

### AC-FLR-26 — Backlog tick count formula clamps negative delay

**Given** delayed real time has accumulated while the runtime is allowed to advance, **When** backlog is classified, **Then** `backlog_tick_count = max(0, floor(max(0, delayed_real_seconds) / combat_tick_duration_seconds))`.

| delayed_real_seconds at 60 ticks/sec | expected backlog_tick_count |
|---:|---:|
| -0.01 | 0 |
| 0.0 | 0 |
| 0.01 | 0 |
| 0.09 | 5 |

### AC-FLR-27 — Backlog classification is correct

**Given** backlog is evaluated, **When** focus is lost, **Then** backlog classification is `focus_suspended`.

**Given** focus is not lost and `backlog_tick_count = 0`, **When** backlog is evaluated, **Then** backlog classification is `none`.

**Given** focus is not lost and `backlog_tick_count` is between `1` and `catch_up_max_ticks` inclusive, **When** backlog is evaluated, **Then** backlog classification is `in_order_catch_up`.

**Given** focus is not lost and `backlog_tick_count > catch_up_max_ticks`, **When** backlog is evaluated, **Then** backlog classification is `recovery_pause`.

### AC-FLR-28 — Small backlog catches up at most to the safe limit

**Given** the runtime has a backlog classified as `in_order_catch_up`, **When** catch-up is processed, **Then** the runtime commits at most `backlog_tick_count` delayed ticks, each committed tick is processed one at a time in increasing tick order, no committed tick ID is skipped, and each committed catch-up tick produces the same observable snapshot and event ordering rules as a normal runtime tick.

**Given** catch-up is processing delayed ticks, **When** a blocking state, CPU wall-clock guard, or fairness guard is reached, **Then** catch-up stops at the nearest safe tick boundary instead of forcing the remaining backlog through silently.

### AC-FLR-29 — Catch-up stops when a blocking state is reached

**Given** catch-up processing is committing delayed running ticks, **When** a committed tick causes `hitstop`, `paused`, `focus_suspended`, or `round_ended`, **Then** catch-up stops immediately after that tick commits, and no later backlog tick is processed while that blocking state is active.

### AC-FLR-30 — Large or unsafe backlog enters recovery pause instead of skipping ticks

**Given** focus is not lost and `backlog_tick_count > catch_up_max_ticks`, **When** backlog is classified, **Then** the runtime enters `recovery_pause` or equivalent paused recovery behavior, no missed combat ticks are replayed silently, no gameplay tick IDs are skipped, and the trace records the backlog size and recovery decision.

**Given** focus is not lost and `backlog_tick_count <= catch_up_max_ticks`, **When** catch-up would exceed its CPU wall-clock guard or hide a player-visible startup/active/recovery/punish/burst/KO/round-end transition, **Then** the runtime stops silent catch-up and enters `recovery_pause` or an explicit recovery flow.

### AC-FLR-31 — Focus loss wins over backlog catch-up

**Given** focus is lost while real-time delay or backlog exists, **When** the runtime classifies advancement, **Then** the runtime enters `focus_suspended`, missed time is not replayed, and no catch-up ticks are committed for the focus-lost interval.

### AC-FLR-32 — Focus suspension blocks combat ticks and new combat input

**Given** the runtime is in `focus_suspended`, **When** real time passes or new combat input is attempted, **Then** no combat ticks are committed, no new combat input is accepted into authoritative command processing, and the round timer, action frames, movement, projectiles, hitstun, blockstun, recovery, and combo state do not advance.

### AC-FLR-33 — Focus restore confirmation cannot leak into combat input

**Given** `focus_restore_requires_explicit_confirm = true` and the player was holding one or more combat inputs when focus was lost, **When** focus is restored, **Then** those held inputs are not automatically treated as newly pressed combat commands, and combat input resumes only after the required cleanup or explicit reconfirmation is observed.

**Given** the player presses a confirm/resume input while focus recovery UI is active, **When** that input is consumed, **Then** the input is consumed by the UI/menu context, does not enter the combat command entry path, and cannot trigger movement, guard, attack, projectile, burst, or dummy/CPU command effects on the resume tick.

### AC-FLR-34 — Pause blocks combat advancement

**Given** the runtime is in `paused`, **When** real time passes, **Then** no committed gameplay ticks are produced, and no round timer, action frame, movement, projectile, hitstun, blockstun, recovery, combo, damage, KO, or timeout advancement occurs.

### AC-FLR-35 — KO resolves before timeout on the same tick

**Given** a hit reduces a fighter to KO state on the same committed tick that the round timer reaches 0, **When** KO and timeout/end resolution is processed, **Then** KO resolution takes priority over timeout resolution, and the committed event batch orders the KO-related event before any timeout/end event for that tick.

### AC-FLR-36 — Round end prevents later combat advancement

**Given** the runtime has entered `round_ended`, **When** real time passes or commands are submitted, **Then** no additional combat ticks are committed for that round instance, and no further damage, movement, projectiles, guard, burst, combo, timeout, or KO changes occur for that ended round.

### AC-FLR-37 — Event sequence keys are deterministic and unique within a round

**Given** events are emitted by the fixed-logic runtime, **When** QA inspects each event, **Then** each event includes `round_instance_id`, `committed_tick_index`, `tick_phase_order`, and `within_phase_event_index`, and sorting events by `(round_instance_id, committed_tick_index, tick_phase_order, within_phase_event_index)` reproduces the committed event order.

### AC-FLR-38 — Event batches are committed facts, not requests

**Given** an ordered event batch is emitted for a committed tick, **When** a presentation, HUD, VFX, audio, camera, or debug consumer receives the batch, **Then** each event represents a fact that already occurred in authoritative combat state, and consumers cannot convert the event into a different authoritative combat result.

### AC-FLR-39 — Event batches are idempotent for consumers

**Given** the same committed event batch is delivered to a consumer more than once, **When** the consumer processes the duplicate delivery, **Then** authoritative combat state does not change, and duplicate delivery does not create additional authoritative hits, damage, energy, KO, timeout, or combo changes.

### AC-FLR-40 — Quick restart invalidates old event batches

**Given** a round is restarted quickly, **When** the new round begins, **Then** the new round uses a new `round_instance_id`.

**Given** an old event batch from the previous round instance is received after restart, **When** consumers inspect the event metadata, **Then** the old batch is distinguishable from the current round by `round_instance_id`, and it cannot be mistaken for a current-round authoritative event.

### AC-FLR-41 — Trace window bounds formula is correct

**Given** `latest_committed_tick_index = 200` and `trace_window_ticks = 180`, **When** the runtime reports the available debug trace window, **Then** `trace_window_bounds = [max(0, latest_committed_tick_index - trace_window_ticks + 1), latest_committed_tick_index] = [21, 200]`.

### AC-FLR-42 — Minimal QA/debug trace contains required runtime facts

**Given** QA enables the minimal fixed-runtime trace, **When** combat ticks, backlog decisions, state transitions, rejected requests, snapshots, or event batches occur, **Then** the trace exposes these fields or references: `round_instance_id`, `committed_tick_index`, `runtime_state`, `previous_runtime_state`, `state_reason`, `elapsed_running_ticks`, `round_timer_remaining_ticks`, `hitstop_remaining_ticks`, `backlog_tick_count`, `backlog_classification`, `pause_reason`, `focus_restore_gate_state`, `command_source`, `command_id`, `tick_phase_order`, `within_phase_event_index`, `event_type`, `rejection_reason`, `snapshot_summary`, and `trace_window_bounds`.

### AC-FLR-43 — Invalid runtime transitions are rejected with trace warnings

**Given** a runtime state transition request is invalid for the current state, **When** the request is processed, **Then** the runtime rejects the transition, authoritative state does not change because of that request, and the trace records a warning with the current state, requested state, tick context, and rejection reason.

### AC-FLR-44 — Tick commits are atomic

**Given** a tick is being processed, **When** QA observes snapshots and event batches, **Then** QA can observe either the previous committed tick or the next committed tick, and QA cannot observe a half-committed mixture of old and new combat state.

### AC-FLR-45 — Godot 2D physics is not authoritative for fighting hits or combat position

**Given** Godot 2D physics collision, sprite overlap, animation position, or visual bounds differ from data-driven hitbox and hurtbox facts, **When** combat hit, block, whiff, damage, stun, hitstop, combo, or KO resolution occurs, **Then** the authoritative result follows the fixed tick data-driven combat facts, not the Godot 2D physics body result, sprite bounds, animation frame, VFX, camera, HUD, or audio state.

**Given** Godot 2D physics reports a boundary or ground-assist result, **When** the result would affect fighter position, **Then** that result must enter the fixed runtime movement/boundary phase before it can appear in an authoritative committed snapshot.

### AC-FLR-46 — Presentation systems are consumers only

**Given** HUD, VFX, audio, camera, animation, sprite, or debug systems receive snapshots or event batches, **When** those systems update, **Then** they may display, play, animate, log, or visualize the committed facts, and they cannot override authoritative combat state, timer state, hit results, command results, or runtime state.

### AC-FLR-47 — Snapshot summary is the current HUD truth source

**Given** HUD displays health, energy, round timer, combo count, runtime state, or actor state, **When** a committed snapshot summary and an event-derived display state disagree, **Then** the HUD uses the committed snapshot summary as the current truth, while event batches may only trigger one-shot feedback such as pulses, hit sparks, sounds, or debug log entries.

### AC-FLR-48 — Player, dummy, and CPU use the same command entry path

**Given** a player command, training dummy command, and simple CPU command represent the same legal action from the same legal combat state, **When** each command is submitted through the runtime command entry, **Then** each command is evaluated by the same tick-boundary command rules, and each command entry exposes `round_instance_id`, `command_source`, `source_actor_id`, `command_id`, `target_committed_tick_index`, `input_snapshot_id` or `based_on_committed_tick_index`, `command_type`, `pressed_or_held`, and `runtime_state_at_capture`.

### AC-FLR-49 — CPU commands do not bypass runtime rules or read same-tick hidden state

**Given** the simple CPU script chooses an attack, guard, movement, projectile, or burst command, **When** the command is submitted, **Then** the command is processed in the same command phase as player and dummy commands, cannot bypass countdown, pause, focus suspension, hitstop freeze rules, command timing, state-machine legality, or tick-boundary commitment, and is based only on a previously committed snapshot rather than same-tick player input or half-updated state.

### AC-FLR-50 — Training dummy commands do not bypass runtime rules

**Given** the training dummy is configured to perform a command, **When** the command is submitted, **Then** the command is processed through the same command entry path as player and CPU commands, and it cannot directly create hits, blocks, damage, stun, hitstop, energy, combo changes, KO, or timeout events outside the fixed tick order.

### AC-FLR-51 — MVP tuning defaults are observable and enforced

**Given** the MVP fixed-logic runtime is initialized, **When** QA inspects runtime configuration or startup trace, **Then** the default values are `combat_ticks_per_second = 60`, `catch_up_max_ticks = 6`, `trace_window_ticks = 180`, `countdown_direction_pre_read_enabled = true`, and `focus_restore_requires_explicit_confirm = true`.

**And** for MVP builds, `combat_ticks_per_second` remains locked to `60` unless an approved test configuration explicitly overrides it.

### AC-FLR-52 — Safe tuning ranges reject invalid values

**Given** runtime tuning values are loaded, **When** a value falls outside its safe range, **Then** the runtime rejects the invalid value or falls back to an approved safe/default value, and the trace records the invalid value and the effective value.

| Tuning knob | MVP safe range | Explicit debug/stress override |
|---|---:|---:|
| `combat_ticks_per_second` | 30–120; MVP locked 60 | none without ADR |
| `catch_up_max_ticks` | 2–8 | `0` allowed only in explicit debug/stress config |
| `trace_window_ticks` | 120–600 | larger only in temporary QA/dev config |

### AC-FLR-53 — Deterministic QA re-simulation of the same test inputs produces the same observable results

**Given** the same initial round state, same tuning values, same player commands, same dummy commands, same CPU script outputs, and same runtime requests, **When** the fixed-logic runtime is executed twice in a deterministic QA test harness, **Then** both runs produce the same committed tick IDs, snapshots, runtime state transitions, and ordered event sequence keys, and no presentation frame rate, HUD, VFX, audio, camera, or Godot physics timing difference changes the authoritative combat result.

This criterion defines a QA/test re-simulation expectation only; it does not create a formal player replay, rollback, spectator, networking, or cross-version replay requirement.

### AC-FLR-54 — UI runtime state payload supports pause and focus recovery

**Given** the runtime enters `paused`, `focus_suspended`, `recovery_pause`, `countdown`, or `round_ended`, **When** UI/menu/HUD/debug consumers read the runtime state payload, **Then** the payload exposes `runtime_state`, `state_reason`, `previous_runtime_state` or `resume_target_state`, `requires_player_confirm`, `combat_input_policy`, `held_input_cleanup_required`, `countdown_phase`, `backlog_classification`, and `round_instance_id`.

### AC-FLR-55 — Trace and instrumentation are bounded for Web

**Given** trace, rejected-request warnings, snapshot summaries, or event batches are emitted, **When** their count or payload size exceeds the configured QA/debug bounds, **Then** old bounded trace data may be evicted or warnings may be throttled, while authoritative combat state and event ordering remain unchanged.

### AC-FLR-56 — Burst and energy commands use the runtime command and event path

**Given** a burst, energy, or projectile command is submitted by player, dummy, or CPU, **When** the command is accepted or rejected, **Then** the decision occurs through the same command entry path and state-machine legality checks as other combat commands, and committed results appear only through snapshots and ordered events such as `energy_meter_changed`, `burst_ready`, `burst_started`, or rejected-command trace warnings.

### AC-FLR-57 — Critical readability phase events are observable

**Given** a move transitions through startup, active frames, recovery, or punishable windows, **When** those phase changes are committed, **Then** ordered event batches or snapshot summaries expose enough information for QA and presentation systems to observe `startup_started`, `active_frame_started`, `recovery_started`, `punish_window_opened`, and `punish_window_closed` when those concepts are defined by downstream move/state-machine data.

### Instrumentation Note

The following criteria require explicit QA-facing instrumentation if it does not already exist: ordered event sequence keys, committed snapshot summaries, runtime state transition trace, backlog classification trace, rejected-request warnings, trace window bounds, command-source metadata for player/dummy/CPU command entry, UI runtime state payload, catch-up CPU/fairness guard decisions, and bounded trace payload diagnostics.

## Visual/Audio Requirements

本系统不直接拥有视觉资产、音效资产、动画资产或相机表现；它只定义这些表现系统可以信任和消费的 runtime facts。

- Runtime 必须为 HUD、VFX、音效、相机和调试显示提供 committed snapshot summaries、ordered event batches、runtime state payload、tick id、round instance、position context、result type 和 idempotency key。
- `startup_started`、`active_frame_started`、`recovery_started`、`punish_window_opened`、`punish_window_closed`、`hit_landed`、`blocked`、`whiffed`、`hitstop_started`、`energy_meter_changed`、`burst_started`、`round_ended` 等事件必须足够明确，让表现层不需要自行推断 combat result。
- VFX、音效、动画、相机震动和 HUD 闪烁可以延迟、压缩或去重表现，但不得改变、补写、删除或重排 authoritative combat facts。
- hitstop 中允许表现层继续播放已触发的 VFX/音效和 HUD 动画，但不得推进 combat action frames、判定、硬直、timer 或 projectile combat advancement。
- catch-up 期间 runtime 仍输出完整事件顺序；事件批次必须带有 delivery/catch-up metadata，表现层如何压缩展示由 HUD/VFX/音效 GDD 决定。
- 表现层如果重复收到同一 event batch，必须用 event sequence key / idempotency key 去重；不得重复造成权威 gameplay 变化。
- 本系统不触发 asset-spec 生产；后续 `combat-hud-feedback`、`readable-combat-vfx`、`combat-audio-feedback` 和 `sprite-animation-presentation` GDD 会定义具体资产和表现规格。

## UI Requirements

本系统不拥有正式玩家 UI 屏幕，但必须向下游 UI、菜单和调试系统暴露稳定状态。

- HUD 必须能读取当前 `round_instance_id`、`committed_tick_index`、runtime state、round timer remaining、ordered event batches 和必要 snapshot summary。
- HUD 当前数值必须以 committed snapshot summary 为权威；event batch 只触发一次性反馈、pulse、音效/VFX 请求或 debug log。若二者冲突，snapshot wins。
- 暂停菜单和 Web focus recovery UI 必须通过 runtime request 进入/离开 `paused` 或 `focus_suspended`，不得直接修改 combat state。
- UI runtime state payload 必须表达 `runtime_state`、`state_reason`、`previous_runtime_state` 或 `resume_target_state`、`requires_player_confirm`、`combat_input_policy`、`held_input_cleanup_required`、`countdown_phase`、`backlog_classification`、`round_instance_id`。
- 当 `focus_restore_requires_explicit_confirm = true` 时，UI 必须能表现“恢复焦点后需要确认继续”的状态；确认输入由 UI/menu context 消费，不得作为 combat command 进入移动、防御、攻击、气弹或爆气流程。
- 调试显示必须能读取 tick id、runtime state、backlog classification、trace window bounds、最近 event sequence keys、rejected request warnings、catch-up guard decisions 和 bounded trace diagnostics。
- UI/HUD/Debug 只读 combat facts；除明确 runtime request（pause、resume、restart、exit）外，不得写入血量、气槽、硬直、判定、timer、combo 或胜负状态。

## Open Questions

| Question | Owner | Target Resolution | Notes |
|---|---|---|---|
| Godot 4.6.2 中 fixed tick scheduling、accumulator、pause/focus handling 的具体实现方式是什么？ | Runtime ADR / technical-director / godot-specialist | Before runtime implementation starts | GDD 只定义行为；实现必须查 `docs/engine-reference/godot/`，不能靠记忆写 Godot API。 |
| Runtime architecture、fixed tick scheduling、input snapshot、event batch/signal delivery、snapshot immutability、trace/instrumentation、Godot 2D physics boundary、numeric determinism、presentation interpolation、Web audio unlock、testing strategy 是否都已有 ADR？ | Runtime ADR set / technical-director / godot-specialist | Before runtime implementation starts | design-review 指出这些是 implementation gate；GDD 冻结最小合同，ADR 决定 Godot 实现方式。 |
| `catch_up_max_ticks = 6` 是否在 Web 导出实测中合适？ | Runtime ADR / performance-analyst | During first browser prototype week | GDD 给出推荐值和安全范围；最终值需要 Web 性能验证。 |
| `trace_window_ticks = 180` 是否足够 QA 复现输入、hitstop、focus 和 event ordering 问题？ | QA Lead / Runtime ADR | Before QA instrumentation task | 如 QA 发现 3 秒窗口不足，可在安全范围内提高。 |
| Event batch 的最终字段名、序列化格式和存储位置是什么？ | Runtime ADR / lead-programmer | Before event bus implementation | GDD 只规定必须包含的信息类别和排序 key。 |
| 输入缓冲在 hitstop、pause、countdown、focus restore 中如何过期或清理？ | 输入映射与输入缓冲 GDD | During input-buffering GDD | 本系统只规定 runtime state 和“不得自动触发 held input”。 |
| HUD/VFX/音效如何表现 catch-up 后连续多个 event batches？ | HUD/VFX/Audio GDDs | During presentation GDDs | Runtime 保证顺序和完整性；表现层决定是否压缩展示。 |

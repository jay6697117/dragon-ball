# 固定逻辑步进与战斗运行时

> **Status**: Revised — eighth fresh Design Review MAJOR REVISION blockers addressed; Pending Fresh Re-review
> **Author**: SteveZhang + Claude Code Game Studios
> **Last Updated**: 2026-05-12
> **Implements Pillar**: 读招定胜负；短连招，高回合；气槽辅助，不接管战斗；第一回合必须好玩；热血能量必须可读
> **System ID**: fixed-logic-runtime
> **Priority**: MVP
> **Layer**: Foundation
> **Creative Director Review (CD-GDD-ALIGN)**: APPROVED 2026-05-11 — 固定逻辑运行时清楚保护“我可以练会”的公平时间基础，并未扩大 MVP 范围；trace/replay 语义继续限定为 QA/debug。
> **Design Review**: MAJOR REVISION NEEDED 2026-05-12 on eighth fresh re-review; revised same day to mark visible recovery pauses as compromised exchanges, narrow must-show gating to critical hidden causality, require combat-layer visual proof for spatial facts, split combat-critical ack deadlines from non-combat UI deadlines, reframe AI reaction age as a technical floor, add cold first-round profiling, and keep ADR gates explicit without creating ADR files; pending fresh re-review.

## Overview

固定逻辑步进与战斗运行时是《星核斗魂》所有 MVP 战斗系统的时间基础。它定义战斗逻辑以固定 tick 更新，规定输入、移动、状态机、hitbox/hurtbox、命中停顿、硬直、伤害、气槽、对局流程和战斗事件的统一处理顺序，确保同一组输入和数据在浏览器中得到可重复、可验证的结果。玩家不会直接操作这个系统，但会通过稳定的防御时机、可信的反击窗口、不会漂移的短连招和准确同步的 HUD/VFX/音效感受到它；如果本系统不成立，“读招定胜负”就会变成不可预测的帧率运气。

## Player Fantasy

玩家不会直接看到或操作固定逻辑步进与战斗运行时，但应该始终感觉战斗背后有一套公平、稳定、可信的时间规则。成功防住起手、抓到硬直、完成短连时，玩家会觉得“这是我判断对了”；失败被反打时，也应该能理解为“我慢了、贪了、读错了”，而不是怀疑游戏判定漂移、输入丢失或帧率影响结果。

这个系统服务的玩家幻想是“我可以练会”。《星核斗魂》的战斗由短回合组成：试探、出手、防御、抓硬直、打出 2–4 段短连，然后重新回到立回。固定运行时必须让同样的输入、同样的时机、同样的数据产生同样可信的结果，让玩家把注意力放在读招和决策上，而不是担心浏览器帧率、特效数量或画面波动改变胜负。正常第一回合的目标是 **零可见 runtime recovery pause**；如果浏览器恢复、显示水位或 catch-up safety 迫使系统在关键交换中暂停，该交换必须被标记为 compromised exchange，并通过安全中立重取节奏或训练模式重放处理，不能伪装成未被系统干预的公平实时读招。

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
   - `presentation_ack_wait`：`state_reason` / `pause_reason` 和 delivery gate，表示 runtime 已提交含关键可读事实的 packet，但正在等待 display watermark 或明确视觉降级；它不是独立 `runtime_state`，运行时表现为 `runtime_state = paused`。

3. **逻辑时间与表现时间分离**  
   committed runtime tick 提交战斗事实；渲染、HUD、VFX、音效、相机和调试显示只读取最近提交的战斗事实并表现它。表现层可以延迟、插值、闪烁、震动或播放特效，但不得修改战斗状态。

4. **combat tick 不允许静默跳过，catch-up 必须以玩家已看到的关键因果为边界**
   如果浏览器短暂掉帧，运行时最多可以按顺序补跑 `catch_up_max_ticks` 内的少量积压 tick，但只能补跑不会隐藏关键读招因果的 tick。MVP 默认采用小步追赶：`catch_up_max_ticks = 2`，`catch_up_wall_clock_budget_ms = 3.0`，且 catch-up wall-clock 预算包含 preflight descriptor 构建、preflight 分类、模拟、snapshot/event packet 构建、trace metadata 和 delivery metadata。每个 delayed tick 在正式提交前必须先生成只读 `catch_up_preflight_descriptor`，再由 `catch_up_preflight` 分类：descriptor 只能由上一枚已提交 snapshot、目标 tick 已采集/排队的 input snapshot、确定性招式 phase schedule、projectile schedule、已排队 runtime request、staged command set 和只读 runtime metadata 派生；最小字段为 `target_committed_tick_index`、`target_running_tick_index_if_committed`、`candidate_phase_transitions`、`candidate_projectile_threat_transitions`、`candidate_result_transitions`、`candidate_runtime_state_after_commit`、`uses_rng = false`、`descriptor_complete` 和 `descriptor_hash`。preflight 不得执行第二套战斗模拟、不得修改 runtime state、不得生成 combat event、不得消耗或查询 RNG、不得读取 live Godot node 或表现层状态。若 descriptor 缺失、不完整、hash 不稳定或数据不足以证明安全，必须 fail closed 为 `fairness_stop_required`。运行时不得丢弃 tick、一次性快进大量 tick，或把命中/防御/确反窗口/气弹威胁这类结果藏在不可见补跑里。

   Catch-up fairness 使用三层策略；`fairness_stop_required` 是 **pre-commit stop classification**，不是已经 emitted 的 combat event tier：

   | Tier | Rule | Minimum events / facts |
   |---|---|---|
   | `fairness_stop_required` | **暂停在结果前并标记 compromised exchange**：如果 delayed tick 的 preflight descriptor 显示该 tick 会在玩家未看到原因前进入 active threat、projectile threat、punish opportunity、hit/block/KO/round end 或改变可防御/可反击结果，则不得提交该 delayed tick；runtime 停在上一枚已提交 tick，进入 `runtime_state = paused`、`state_reason = recovery_pause`，并记录 `exchange_integrity = compromised_by_runtime_recovery`。正常对战恢复后必须先回到安全中立/重新取得节奏；训练模式可以选择重放或延长 telegraph，但 UI 不得泄露比无中断实时战斗更多的战术信息。 | `blocked_transition_type` 至少包括 `active_threat_entered_precommit`、`projectile_threat_entered_precommit`、`punish_window_opened_precommit`、`hit_or_block_result_precommit`、`burst_or_ko_or_round_end_precommit`。这些是未提交 tick 的 guard reason；被 guard 的 event 还没有 emitted。 |
   | `presentation_must_show` | **只 gate 已提交但未被玩家显示水位覆盖的关键因果**：当前 delayed tick 可以提交，但仅当该 fact 由 catch-up/recovery/resume/stale-delivery 风险产生，且会影响玩家理解“为什么被打、为什么挡住、为什么打空、为什么可惩罚、为什么气弹有威胁、为什么控制权返回、为什么 KO/回合结束”时，packet 才生成 `presentation_ack_requirement`。普通实时 startup/recovery/control transition 必须正常显示，但不得频繁阻塞 runtime。依赖该 fact 的 AI/木桩决策、自动 resume 和新的 silent catch-up tick 必须等待 display watermark、明确视觉降级或 terminal failure policy。audio 不能单独满足 must-show ack。 | 已提交且处于隐藏/压缩/恢复风险中的 `active_frame_started`、`hit_landed`、`blocked`、`whiffed`、`hitstop_started`、`recovery_started`、`punish_window_opened`、`punish_window_closed`、`control_resumed`、`projectile_spawned`、`projectile_threat_changed`、`burst_started`、`ko_started`、`round_ended`。`startup_started` 仅当 catch-up/recovery 会隐藏其起手因果时进入 must-show。 |
   | `compressible_phase` | 可以继续 catch-up；表现层只能压缩非读招关键反馈，事件仍完整保留，且不得被压缩成会改变玩家理解的假因果。 | 非威胁 spacing/facing 微调、重复 idle/hold 状态、debug-only event；不得包含会改变当前交换理解的 hit/block/whiff/result、projectile threat、punish window 或 control-return 事实。 |

   `presentation_must_show` 事实必须同时携带 `must_show_fact_id`、`minimum_visual_cue_type`、`minimum_visual_cue_requirement`、`required_visual_consumer_policy` 和 `hud_only_allowed`，否则不能生成 satisfied ack。MVP 最小 cue map 为：

   | must-show fact | Minimum visual cue requirement |
   |---|---|
   | `startup_started` when hidden by catch-up/recovery | actor silhouette or pose change must be committed by an `actor_sprite` consumer at a host-frame boundary; HUD marker may reinforce but cannot satisfy normal-combat spatial proof alone. |
   | `active_frame_started` / `projectile_threat_changed` | attack limb/weapon shape, projectile core, or combat VFX direction must express threat direction and guard/punish meaning through `actor_sprite`, `projectile_vfx`, or `combat_vfx`; HUD-only is allowed only in training/debug/accessibility assist modes. |
   | `hit_landed` / `blocked` / `whiffed` / `hitstop_started` | result must be visible in the combat layer through contact spark, guard spark, whiff spacing, frozen pose, or hitstop pose contrast; HUD/audio may supplement but cannot satisfy result proof alone. |
   | `recovery_started` / `punish_window_opened` | recovery or punishable state must be visible through actor pose/silhouette or combat-layer cue; if preflight finds it would be hidden before commit, use `fairness_stop_required` instead. |
   | `punish_window_closed` / `control_resumed` | must express that danger/punish window ended or control returned without implying the window remains open; HUD can supplement, but normal combat proof requires actor/control-state visual change. |
   | `projectile_spawned` / `burst_started` / `ko_started` / `round_ended` | projectile position/threat, burst state, KO, or round end must be visible through projectile/combat-layer VFX or round-state UI with a non-color backup; audio-only is never sufficient. |

   Runtime 必须记录 `recovery_pause_count_per_round`、`focus_suspend_count_per_round`、`presentation_ack_wait_count_per_round`、`compromised_exchange_count_per_round` 和 `combat_critical_ack_failure_count_per_round`。MVP QA 默认阈值为：正常第一回合阻塞浏览器 profile 的目标是 `recovery_pause_count_per_round = 0` 且 `compromised_exchange_count_per_round = 0`；任一 live combat exchange 中出现 compromised exchange 都标记 first-round experience failure。非第一回合单回合 `recovery_pause_count_per_round > 1` 触发 warning，`> 2` 视为 Web 体验失败，需要 Runtime ADR 或性能调参处理。combat-critical must-show cue 必须在 next host-frame boundary 或 `combat_critical_ack_deadline_ms = 50` 内 ack/degrade/fail；`presentation_ack_deadline_ms = 250` 只适用于非战斗 UI 或恢复类提示。恢复类玩家文案不得暴露技术 enum，但必须区分原因类别：`browser_recovering`、`showing_key_moment`、`pause_continue`、`round_restart_ready`。确认输入只由 UI/menu context 消费，不得进入 combat command flow。

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
    Web MVP 中，页面隐藏、浏览器/窗口 blur、canvas 或键盘输入焦点丢失、fullscreen gate 等 unsafe focus loss 必须进入 `focus_suspended`，停止 combat tick，不累计补跑，不接受新的战斗输入。单独的 audio unlock、muted tab、browser autoplay policy 或 audio device failure 只产生 `audio_degradation_reason` 和 telemetry，不能阻塞 combat tick，也不能满足 display watermark；只有当音频流程同时造成 canvas/keyboard focus loss 时，才按对应 focus loss reason 进入 `focus_suspended`。普通 UI hover、鼠标焦点变化或暂停菜单内部控件焦点移动不等于 unsafe focus loss，不得自动覆盖 `paused` 为 `focus_suspended`。恢复焦点后，运行时必须显示原因、清理或重新确认 held input，确认输入由 UI/menu context 消费，随后进入短 resume countdown / Ready-Go reacquisition，之后才恢复 combat command flow，避免玩家回到页面时自动移动、攻击、防御、气弹或爆气。

13. **玩家、训练木桩和简单 CPU 走同一战斗命令入口**
    玩家输入、训练木桩脚本和简单 CPU 都必须转化为同一种战斗命令格式，再交给战斗状态机判断。CPU 或木桩不得绕过输入缓冲、状态机、硬直、防御、气槽或 hitbox/hurtbox 规则。pause、resume、restart、exit、focus restore confirm 属于 runtime request，必须在 running tick phase 1 或非推进状态 transition flow 处理，不得伪装成 combat command。

    | Command field | Required values / type | Rule |
    |---|---|---|
    | `round_instance_id` | current round id | 必须匹配当前 round；旧 round command 拒绝。 |
    | `round_instance_sequence` | monotonic int `>= 1` | 用于排序和 trace；不得用 string/id 参与字典序排序。 |
    | `command_source` | `player` / `training_dummy` / `simple_cpu` / `qa_fixture` | 不允许隐式来源；`qa_fixture` 只允许 test/dev build。 |
    | `command_source_ordinal` | `0` / `10` / `20` / `90` | 显式排序值；不得从字符串、Godot node order 或 signal order 推导。 |
    | `source_actor_id` | stable actor id | 必须匹配可行动 actor。 |
    | `command_id` | unique id within source + round | 重复 command id 幂等拒绝或去重。 |
    | `command_type` | `movement` / `guard` / `light_attack` / `heavy_attack` / `projectile` / `burst` / `qa_fixture` | 非法 enum 拒绝；runtime request 不属于本 enum。 |
    | `command_type_ordinal` | explicit int ordinal | MVP 默认见下文；同 tick 多命令不得依赖 enum 字符串排序。 |
    | `command_sequence_key` | `(round_instance_sequence, target_committed_tick_index, command_source_ordinal, source_actor_id, command_type_ordinal, command_id)` | 同 tick 多命令按此稳定排序；`round_instance_id` 只用于 equality/idempotency 分区，不参与排序。 |
    | `target_committed_tick_index` | integer `>= current_committed_tick_index + 1` unless queued by input-buffer rules | 不允许修改当前已开始或已提交 tick。 |
    | `target_presented_running_tick_index` | runtime-derived integer for non-player commands | CPU/木桩命令目标对应的已呈现 running-time index；runtime 必须由 `target_committed_tick_index` 和 presented-running queue 推导，不信任 command payload 自报值。hitstop、pause、focus 和未 ack 的 catch-up tick 不计入。 |
    | `input_snapshot_id` | player-only input snapshot id | 仅玩家输入可用；CPU/木桩不能用它替代观察快照。 |
    | `based_on_committed_tick_index` | non-player required committed tick id | CPU/木桩观察来源的 committed tick；用于定位 snapshot，不单独证明反应延迟。 |
    | `based_on_presented_running_tick_index` | non-player required integer `<= target_presented_running_tick_index - min_ai_decision_age_ticks` | CPU/木桩最小技术反应延迟只按玩家已看到的 running ticks 计算；`min_ai_decision_age_ticks = 6` 是 runtime 公平下限，不是 first-round/simple CPU 玩法默认。 |
    | `observation_snapshot_id` / `observation_snapshot_hash` | non-player required | 必须匹配 retained visible-only observation snapshot。 |
    | `visibility_ack_generation_id` | non-player required generation id | 只证明 observation 属于哪个 presentation generation；不能单独证明 must-show facts 已进入显示水位。 |
    | `visibility_ack_watermark` | non-player required tuple | `(presentation_generation_id, acknowledged_presented_running_tick_index, acknowledged_must_show_fact_ids_hash)`；证明 observation 之前的 must-show facts 已达到 display watermark 或明确视觉降级。 |
    | `pressed_or_held` | `pressed` / `held` / `released` / `axis` / `scripted` | 输入语义必须显式。 |
    | `runtime_state_at_capture` | runtime state enum | 不符合当前 input policy 的 command 拒绝。 |

    最小 rejected-command payload 必须表达：`command_id`、`round_instance_id`、`round_instance_sequence`、`command_source`、`command_source_ordinal`、`source_actor_id`、`target_committed_tick_index`、`target_presented_running_tick_index`、`runtime_state_at_capture`、`command_acceptance_result`、`rejection_reason`、`dedupe_or_supersede_reason`、`visibility_ack_generation_id` 和 `visibility_ack_watermark` when applicable。`rejection_reason` 最小 enum 为：`stale_round`、`duplicate_command_id`、`target_tick_already_processing`、`invalid_presented_tick_mapping`、`invalid_source_actor`、`invalid_runtime_state`、`missing_observation_snapshot`、`observation_tick_mismatch`、`observation_hash_mismatch`、`duplicate_observation_snapshot_id`、`future_observation`、`ack_watermark_missing`、`ack_watermark_stale`、`ack_watermark_hash_mismatch`、`decision_age_too_young`、`stale_due_to_runtime_interruption`、`hidden_state_source`、`conflicting_same_actor_command`、`invalid_enum`、`disallowed_by_input_policy`、`qa_fixture_disallowed_in_build`、`state_machine_rejected`。如果多个条件同时成立，rejection precedence 必须按上述 enum 顺序选择首个适用原因，保证同一输入在重复 QA 中得到同一 `rejection_reason`。CPU 和木桩只能基于 `AIObservationSnapshot` 生成命令，不能读取同 tick 尚未提交的玩家输入、输入缓冲、事件总线、debug trace、UI payload、live Godot node 或半更新状态。

    Runtime command ordering 必须先 batch-stage 同一 target tick 的全部命令，再按 `command_sequence_key` 验证和提交 accepted set；验证过程中不得因为某个 command 较早处理就暴露半更新状态给后续 command。MVP 默认 `command_source_ordinal` 为：`player = 0`、`training_dummy = 10`、`simple_cpu = 20`、`qa_fixture = 90`。MVP 默认 `command_type_ordinal` 为：`burst = 10`、`guard = 20`、`movement = 30`、`light_attack = 40`、`heavy_attack = 50`、`projectile = 60`、`qa_fixture = 90`。这些 ordinal 只用于稳定命令入口和同 actor 冲突处理，不得让玩家、CPU 或木桩在同 tick combat resolution 中获得隐藏胜负偏置；命中/格挡/同时命中仍由固定 hit resolution 规则决定。同一 actor 同一 target tick 默认最多接受一个非 movement combat action；多个冲突 action 必须全部拒绝并记录 `conflicting_same_actor_command`，除非下游输入 GDD 明确为同一 input snapshot 定义 supersede 规则。

    CPU/木桩决策边界必须是纯 value-copy 接口：`AICommand[] = decide(AIObservationSnapshot value_copy, AIConfig value_copy, AIRngStream deterministic_stream)`。`AIObservationSnapshot` 必须由 runtime 的 visible-observation builder 从 committed snapshot + acknowledged/degraded presentation watermark 派生，是 CPU 和公平反应木桩唯一可见战斗事实。最小字段为：`round_instance_id`、`round_instance_sequence`、`committed_tick_index`、`presented_running_tick_index`、`visibility_ack_generation_id`、`visibility_ack_watermark`、`interruption_epoch`、`tick_execution_state`、`post_commit_runtime_state`、`runtime_state`、双方 actor 的 stable id / position band / facing / visible action state / visible action phase bucket / coarse phase age bucket、HUD 可见的 health / energy / timer、粗粒度 projectile facts（projectile stable id、owner、visible position band、visible threat state、threat_entered_presented_running_tick_index 或 `unknown`）、可见 round state、`observation_snapshot_id`、`observation_snapshot_hash`。MVP bucket 枚举必须固定：`position_band` 使用 `{left_corner, left_mid, center, right_mid, right_corner, airborne_left, airborne_center, airborne_right, unknown}`；`visible_action_state` 使用 `{idle, moving, guarding, startup, active_visible, recovery, punishable_recovery, hitstun, blockstun, knockdown, burst, round_locked, unknown}`；`visible_action_phase_bucket` 使用 `{none, startup_early, startup_late, active_visible, recovery_early, recovery_late, punishable, control_ready, unknown}`；`coarse_phase_age_bucket` 使用 `{age_0, age_1_2, age_3_5, age_6_11, age_12_plus, unknown}` presented running ticks；`visible_threat_state` 使用 `{not_visible, visible_non_threat, threat_entering, active_threat, threat_receding, unknown}`。字段级 projection 规则是 fail-closed：如果 runtime 不能证明某字段已由 committed snapshot 和 display watermark 覆盖，就必须写入 `unknown` 或省略该可选字段，不得回退读取 hidden state。它不得包含玩家 raw input、input buffer 内容、精确 action tick range、exact pixel/subpixel position、exact phase tick count、command-source metadata、debug-only trace、UI prompt state、live Godot node、mutable runtime internals、unacked must-show facts 或同 tick 尚未提交的 collision/result facts。CPU/木桩 command trace 必须记录 `observation_snapshot_id`、`observation_snapshot_hash`、`based_on_committed_tick_index`、`based_on_presented_running_tick_index`、runtime-derived `target_presented_running_tick_index`、`decision_age_ticks_raw`、accepted-command `decision_age_ticks` when valid、`visibility_ack_generation_id`、`visibility_ack_watermark`、`interruption_epoch_at_observation`、`interruption_epoch_at_validation`、`script_or_config_id`、`script_or_config_hash`、deterministic RNG seed、RNG stream id、RNG call counter before/after if randomness is used、primary accept/reject result 和 `all_failed_validation_flags`。Runtime 必须拒绝 round id 不匹配、tick 不匹配、hash 不匹配、missing snapshot、duplicate snapshot id、future snapshot、invalid presented tick mapping、ack watermark missing/stale/hash mismatch、decision age too young、interruption epoch mismatch、stale due to pause/focus/recovery/presentation-ack interruption、QA fixture in gameplay/export build 或来自 hidden state 的 command。`min_ai_decision_age_ticks = 6` 只是 exported gameplay 的技术下限；first-round/simple CPU profile 必须由下游 CPU GDD 使用 `>= 12` presented running ticks 或更慢的脚本节奏，不能用 runtime 下限伪装成新手友好默认。

    训练木桩必须声明模式：`passive_dummy` 不生成 combat command；`scheduled_script_dummy` 只按脚本/config 预先排定的 target committed tick 提交命令，脚本不得读取玩家 same-tick 输入、collision、display watermark 或当前动作精确 tick；`fair_reactive_dummy` 与 simple CPU 一样必须使用 runtime-built `AIObservationSnapshot`、`visibility_ack_watermark` 和 `decision_age_ticks >= min_ai_decision_age_ticks`；`qa_fixture_dummy` 只允许 test/dev build，并必须使用 `command_source = qa_fixture`，导出 gameplay build 必须拒绝。`scheduled_script_dummy` 用于录制固定练习节奏，`fair_reactive_dummy` 用于可见事实反应练习，二者不得共享同一 decision generator。

14. **战斗事件是已提交事实，不是请求**  
    每个 committed runtime tick 结束后，运行时发布一个有序事件批次。事件只描述已经发生并提交的战斗事实，例如 `attack_started`、`startup_started`、`active_frame_started`、`recovery_started`、`recovery_ended`、`punish_window_opened`、`punish_window_closed`、`control_resumed`、`projectile_spawned`、`projectile_threat_entered`、`projectile_threat_changed`、`hit_landed`、`blocked`、`whiffed`、`hitstop_started`、`hitstun_started`、`energy_meter_changed`、`burst_started`、`round_ended`。HUD、VFX、音效、相机和调试显示只能消费事件，不能通过事件反向请求改变本 tick 或过去 tick 的战斗结果。

    Event ordering 必须使用带 namespace ordinal 的 key：`runtime_event_sequence_key = (round_instance_sequence, committed_tick_index, phase_order_namespace_ordinal, tick_phase_order, within_phase_event_index)`。`phase_order_namespace_ordinal` 使用固定 ordinal：`running_tick_order = 10`、`hitstop_tick_order = 20`、`debug_trace_order = 90`。非 combat state transition 不得使用 combat event key；它们只使用 `runtime_state_sequence_key`。`tick_phase_order` 必须使用本 GDD 定义的 running/hitstop phase order 数字；`within_phase_event_index` 必须由稳定 tie-breaker 生成：event type ordinal、source actor stable numeric/order id、target actor stable numeric/order id、move id、action instance id、hit instance id、projectile stable id、projectile instance id、stable spawn/order id；不得依赖 Godot node order、signal order、render order 或容器插入顺序。

    MVP 默认 event type ordinal 为：`attack_started = 100`、`startup_started = 110`、`active_frame_started = 120`、`recovery_started = 130`、`recovery_ended = 131`、`punish_window_opened = 140`、`punish_window_closed = 141`、`control_resumed = 150`、`projectile_spawned = 160`、`projectile_threat_entered = 161`、`projectile_threat_changed = 162`、`projectile_expired = 163`、`whiffed = 200`、`hit_landed = 210`、`blocked = 220`、`hitstop_started = 230`、`hitstun_started = 240`、`blockstun_started = 250`、`energy_meter_changed = 300`、`burst_ready = 310`、`burst_started = 320`、`ko_started = 400`、`round_ended = 410`。后续 GDD 可追加 ordinal，但不得复用已有 ordinal。

    最小 event cardinality：`attack_started` 每次 accepted action 一次；`recovery_started`、`recovery_ended`、`punish_window_opened`、`punish_window_closed` 和 `control_resumed` 每个 action phase/window transition 最多一次；`projectile_spawned` 每个 projectile instance 一次；`projectile_threat_entered` 每个 projectile instance 每次进入可反应威胁带一次；`projectile_threat_changed` 每次 threat state 变化一次；`hit_landed` / `blocked` 每个 resolved hit instance 一次；`whiffed` 每个 whiffed attack window 一次，不是每个空 active tick 一次；`hitstop_started` 每次进入 hitstop 一次；`round_ended` 每个 round instance 一次。Impact/block 音效默认由造成命中/格挡的 running tick 的 `hit_landed` / `blocked` 触发一次；hitstop countdown tick 不得重复触发 impact audio。

15. **snapshot、event、trace 和 UI state 必须有最小可测合同**  
    当前 HUD 数值的权威来源是 committed snapshot summary；event batch 只用于一次性反馈、VFX、音效、HUD pulse、debug log 和因果解释。如果 snapshot 与 event-derived UI 状态冲突，snapshot wins。所有 snapshot、event、trace、UI payload 都必须是 value-copy、stable id、primitive value、不可变记录或只读数据引用；不得把 live Godot `Node`、mutable authority `Resource`、可被消费者改写的 `Array` / `Dictionary` 作为权威对象交给表现、UI、音频或 debug。

    最小 snapshot summary 必须表达：`round_instance_id`、`round_instance_sequence`、`committed_tick_index`、`presented_running_tick_index`、`tick_execution_state`、`post_commit_runtime_state`、`elapsed_running_ticks`、`round_timer_remaining_ticks`、`hitstop_remaining_ticks`、双方 actor 的 stable id / position / facing / current health / max health / current energy / max energy / action state / action phase / action_tick_index、combo summary、KO/round result summary、当前 command-source metadata 引用、可见 projectile summary。
    最小 event 必须表达：`runtime_event_sequence_key`、`event_type`、`event_type_ordinal`、`source_actor_id`、`target_actor_id`、`move_id` 或 `cause`、`action_instance_id` when applicable、`hit_instance_id` when applicable、`projectile_instance_id` when applicable、`result_type`、`position_context`、`idempotency_key`、`delivery_context_id`、`presentation_tier` (`presentation_must_show` / `compressible_phase`)、`hud_delta` 或 `hud_pulse_hint` when applicable。`fairness_stop_required` 只允许出现在 pre-commit descriptor、catch-up stop reason 和 delivery context 的 `contains_fairness_stop_preflight`，不得作为已提交 combat event tier。`idempotency_key` 作用域为 `(presentation_generation_id, round_instance_id, runtime_event_sequence_key or runtime_state_sequence_key, event_type, source_actor_id, target_actor_id, move_id_or_cause, action_instance_id, hit_instance_id, projectile_instance_id)`；同一 key 的 HUD pulse、VFX one-shot、audio one-shot、UI one-shot、debug one-shot 在同一 generation 内最多播放一次。
    最小 event batch delivery context 必须逐批表达，字段为：`delivery_context_id`、`delivery_mode` (`normal` / `catch_up` / `recovery_resume` / `presentation_ack_wait` / `stale_discarded`)、`host_delivery_sequence`、`batch_index_in_delivery`、`batch_count_in_delivery`、`delivery_tick_start`、`delivery_tick_end`、`final_snapshot_committed_tick_index`、`catch_up_tick_count`、`round_instance_id`、`presentation_generation_id`、`contains_must_show`、`contains_fairness_stop_preflight`、`presentation_ack_required`、`presentation_ack_id`、`presentation_priority_reason`、`display_watermark_target`。重复 delivery 不得重复播放同一 `idempotency_key` 的 HUD pulse、VFX、audio one-shot、UI one-shot 或 debug one-shot。
    最小 UI runtime state payload 必须表达：`runtime_state`、`state_reason`、`pause_reason` when applicable、`focus_loss_reason` when applicable、`audio_degradation_reason` when applicable、`previous_runtime_state` 或 `resume_target_state`、`runtime_state_sequence_key`、`requires_player_confirm`、`combat_input_policy`、`held_input_cleanup_required`、`held_input_cleanup_state`、`countdown_phase`、`countdown_visual_ack_id` when applicable、`player_facing_recovery_label`、`safe_resume_countdown_id`、`resume_confirm_consumed_by_ui`、`exchange_integrity` when applicable、`backlog_classification`、`catch_up_stop_reason` when applicable、`presentation_ack_required`、`presentation_ack_id`、`presentation_ack_wait_reason` when applicable、`display_watermark_target` when applicable、`ack_deadline_ms` when applicable、`ack_wait_duration_ms` when applicable、`round_instance_id`、`round_instance_sequence`。

    `combat_input_policy` 允许值为：`combat_execute_allowed`、`capture_only`、`direction_pre_read_only`、`menu_only`、`resume_confirm_only`、`blocked_all`。`countdown_phase` 允许值为：`none`、`ready`、`three`、`two`、`one`、`go`、`resume_ready`、`resume_go`；`go` / `resume_go` 只有在 `combat_input_policy` 即将变为 `combat_execute_allowed` 且没有 pending blocking ack 时才能显示。`held_input_cleanup_state` 允许值为：`not_required`、`pending_release`、`released`、`reconfirmed`、`failed_timeout`。`player_facing_recovery_label` MVP 只允许：`none`、`browser_recovering`、`showing_key_moment`、`pause_continue`、`round_restart_ready`；这些是玩家文案 key，不得直接显示技术 enum。`state_reason` / `pause_reason` 最小 enum 为：`player_pause`、`recovery_pause`、`focus_suspended`、`presentation_ack_wait`、`round_end`、`restart_requested`、`exit_requested`。`exchange_integrity` 最小 enum 为：`uninterrupted`、`compromised_by_runtime_recovery`、`training_assisted_replay`、`invalidated_by_restart`。`focus_loss_reason` 最小 enum 为：`page_hidden`、`browser_window_blur`、`canvas_blur`、`keyboard_focus_lost`、`fullscreen_gate`、`internal_menu_focus`、`hover_change`；只有前五项是 unsafe focus loss，`internal_menu_focus` 和 `hover_change` 不得触发 `focus_suspended`。单独的 audio unlock、muted tab、browser autoplay policy 或 audio device failure 不属于 unsafe focus loss；它们必须写入 `audio_degradation_reason`，允许值为：`none`、`audio_unlock_required`、`muted_tab`、`browser_autoplay_blocked`、`audio_device_failure`。`stale_policy_result` 允许值为：`accepted_current_generation`、`discarded_stale_generation`、`discarded_stale_round`、`discarded_superseded_ack`、`discarded_superseded_ui_action`、`discarded_superseded_audio_request`。`runtime_state_sequence_key` 用于 pause/focus/resume/restart 等非 combat state transition 的排序和幂等，不得伪装成 combat event。

    Runtime 必须向表现层交付一个原子 `RuntimePresentationPacket` 或等价只读包，而不是让 HUD/VFX/audio/debug 分别猜测流顺序。最小 packet 字段为：`presentation_generation_id`、`packet_sequence_key`、`round_instance_id`、`round_instance_sequence`、`final_snapshot_committed_tick_index`、`delivery_tick_start`、`delivery_tick_end`、`snapshot_summary`、`ordered_event_batches`、`event_batch_delivery_contexts`、`ui_runtime_state_payload`、`runtime_state_transition_records`、`presentation_ack_requirements`、`packet_size_bytes`、`stale_policy_result`。`packet_sequence_key = (presentation_generation_id, host_delivery_sequence)`；同一 generation 内严格递增。`event_batch_delivery_contexts` 必须与 `ordered_event_batches` 一一对应，不得用单数 context 描述多个 batch。`packet_size_bytes` 使用 Runtime Data Contract ADR 批准的近似序列化口径；超过 `snapshot_summary_max_bytes` 或 `event_batch_max_bytes` 时必须产生 bound diagnostic，且不得截断权威 combat facts。

    `presentation_ack_requirements` 的最小字段为：`presentation_ack_id`、`presentation_generation_id`、`ack_owner`、`required_visual_consumer_set`、`required_visual_consumer_policy` (`any_of` / `all_of` / `specific_consumer_required`)、`optional_audio_consumer_set`、`must_show_fact_ids`、`must_show_fact_ids_hash`、`ack_requirement_items`、`display_watermark_target`、`ack_deadline_policy`、`ack_deadline_ms`、`ack_state`、`ack_result`、`ack_degraded_reason`、`ack_wait_started_at_host_ms`、`ack_wait_duration_ms` 和 `ack_response_record_ids`。每个 `ack_requirement_item` 必须包含 `must_show_fact_id`、`minimum_visual_cue_type`、`minimum_visual_cue_requirement`、`allowed_consumer_types`、`hud_only_allowed`、`spatial_combat_fact`、`non_color_backup_required` 和 `minimum_visible_duration_ms`。`ack_deadline_policy` MVP 允许：`wait_until_ack`、`degrade_after_deadline`、`fail_terminal_after_deadline`、`discard_if_stale_generation`；combat-critical must-show facts 默认使用 `degrade_after_deadline`，并使用 `ack_deadline_ms = combat_critical_ack_deadline_ms = 50` 或 next host-frame boundary 中较早可测结果；非战斗 UI/recovery 提示可使用 `presentation_ack_deadline_ms = 250`。deadline 到期后必须显示等价可读视觉降级提示并记录 `ack_result = visually_degraded`；如果降级也失败，必须记录 `ack_result = failed_terminal`，阻止自动 resume、AI/木桩决策或新的 silent catch-up 继续依赖该 fact。Display watermark 成立条件：符合 policy 的 visual consumer 确认关键事实已提交到一个可呈现的 host-frame boundary，并且对应 `ack_requirement_item` 的 cue 合同满足；normal-combat spatial facts 必须由 `actor_sprite`、`projectile_vfx` 或 `combat_vfx` 满足，HUD/UI 只能辅助，除非 mode 明确为 training/debug/accessibility assist。该 ack 只证明显示水位和最小 cue 合同达成，不证明玩家真的看见或理解。audio 可以附加 ack，但 audio unlock、静音或浏览器策略导致的 audio-only delivery 不能单独满足 must-show ack。任何依赖 must-show fact 的 CPU/木桩决策、后续 silent catch-up tick 或自动 resume，都必须等待该 ack、降级记录或 terminal failure policy。

    Presentation ack response 必须是独立只读记录，不得预先塞进 immutable packet；packet 只能引用已创建的 `ack_response_record_ids`。最小字段为：`presentation_ack_id`、`presentation_generation_id`、`round_instance_id`、`round_instance_sequence`、`consumer_id`、`consumer_type` (`actor_sprite` / `projectile_vfx` / `combat_vfx` / `hud` / `ui_overlay` / `audio_optional` / `debug_optional`)、`must_show_fact_ids_hash`、`ack_requirement_item_results`、`acknowledged_presented_running_tick_index`、`ack_response_result` (`acknowledged` / `visually_degraded` / `failed_terminal` / `discarded_stale_generation` / `discarded_stale_round` / `discarded_superseded_ack`)、`host_frame_index`、`host_time_ms`、`visual_cue_type`、`minimum_visual_cue_met`、`non_color_backup_met`、`stale_policy_result`。Runtime 只接受当前 round、当前 `presentation_generation_id`、匹配 `presentation_ack_id`、匹配 `must_show_fact_ids_hash`、满足 per-fact cue policy、且 `acknowledged_presented_running_tick_index >= display_watermark_target` 的 response；watermark 实际推进仍必须按下一枚未呈现 running tick 顺序推进，不能因为后续 tick 先 ack 而跳过前序 tick。旧 generation、旧 round、被 supersede 的 ack、错误 target index 和 HUD-only normal-combat spatial ack 必须丢弃并记录 stale/invalid diagnostics。

    如果 packet 中的 snapshot、event batch、UI state、transition record、presentation ack、outgoing UI action、audio one-shot、audio loop request 或 debug one-shot 不属于当前 `presentation_generation_id`，消费者必须丢弃；quick restart 必须递增 `presentation_generation_id` 并停止/清空旧 HUD pulse、VFX one-shot、audio one-shot/loop group、countdown overlay、pending presentation ack、outgoing UI action 和 debug one-shot。

16. **Godot 2D physics 不能作为格斗判定或权威位置来源**  
    Godot 2D physics 可用于场景边界、地面辅助、宽泛碰撞或调试可视化，但 MVP 格斗命中必须由数据驱动 hitbox/hurtbox 在 combat tick 内结算。不得依赖 sprite 边缘、动画可见像素、physics contact callback 或渲染节点顺序决定命中。physics 辅助结果也不得直接写入权威战斗位置；必须先进入 fixed runtime 的移动/边界阶段，再由 committed snapshot 暴露。

17. **运行时必须支持 bounded QA trace**  
    MVP 运行时必须能记录每 tick 输入快照、关键状态变化和事件批次，用于 QA 复现输入缓冲、状态切换、命中/格挡、硬直、气槽和胜负问题。trace 必须同时受 `trace_window_ticks`、`trace_max_events_per_tick`、`trace_max_payload_bytes_per_entry`、`trace_max_objects_per_tick` 和 `trace_warning_throttle_per_second` 限制，避免 Web 内存和 GC 风险。超限时可以淘汰旧 trace 或 throttle warning，但不得改变权威 combat state 或 event ordering。该 trace 是开发与 QA 工具，不是正式玩家 replay；MVP 不要求 rollback、联网同步、跨版本回放兼容或观战功能。

    MVP 默认 diagnostic trace bounds：`trace_window_ticks = 180`、`trace_max_events_per_tick = 32`、`trace_max_payload_bytes_per_entry = 4096`、`trace_max_objects_per_tick = 64`、`trace_warning_throttle_per_second = 10`、`trace_total_memory_budget_bytes = 1048576`。这些只限制 debug/QA trace retention；超限可以 evict/throttle diagnostic data，但不得截断 authoritative snapshot、event、command rejection 或 AI visibility facts。

    MVP 默认 authoritative data-contract budgets：`snapshot_summary_max_bytes = 2048`、`event_batch_max_bytes = 8192`、`ai_observation_snapshot_max_bytes = 2048`。超过这些预算是 data-contract failure，需要减少 schema 内容或 ADR-approved budget change；不得为了过 byte cap 删除权威 combat facts。

    MVP 默认 Web performance / memory budgets：`host_frame_p95_budget_ms = 16.67`、`host_frame_p99_budget_ms = 25.0`、`runtime_normal_tick_budget_ms = 1.0`、`catch_up_wall_clock_budget_ms = 3.0`、`max_single_frame_stall_ms = 50.0`、`profiling_heap_growth_fail_bytes = 4194304` over 60 seconds、`profiling_gc_pause_fail_ms = 8.0` when browser/runtime exposes GC pause data、`combat_critical_ack_deadline_ms = 50`、`presentation_ack_deadline_ms = 250`。`combat_critical_ack_deadline_ms` 适用于命中/格挡/打空、active threat、projectile threat、punish/control、burst/KO/round-end 等战斗关键 cue；必须在 next host-frame boundary 或 50ms 内 ack/degrade/fail。`presentation_ack_deadline_ms` 只作为非战斗 UI、恢复提示和调试 overlay 的有界等待上限。`catch_up_attempt_budget_ms` 不再是独立 tuning knob；若 registry 中保留该旧名，只能作为 deprecated alias 指向 `catch_up_wall_clock_budget_ms`。这些预算用于 exported Web profiling 的 QA 起点，不代表最终优化目标；Runtime Profiling ADR 可基于真实浏览器实测调整，但必须解释调整原因并保留自动化或手动证据。Pre-round readiness 是 first-round 阻塞条件：countdown 进入 `go` 前，HUD、actor sprite、projectile/combat VFX、audio unlock/degradation state 和 debug-minimal trace 必须处于 ready 或 explicit degraded-ready；第一下 hit/block、第一枚 projectile、第一次 burst、第一次 HUD pulse 或第一次 quick restart 不得成为 shader/asset/audio 初始化卡顿点。

18. **实现方式留给 ADR，但 ADR 是 implementation gate**
    本 GDD 只规定战斗时间、结算规则和最小跨系统合同，不规定 Godot 节点结构、Autoload/signal/event bus 选型、`_physics_process` 使用方式、accumulator 实现、日志格式、浮点/定点选择或渲染插值方案。这些属于后续架构决策。本轮按用户决策不创建 ADR 文件；因此这些 ADR 仍是 implementation gate，而不是本 GDD 已经解决的实现细节。实现拆分前必须至少完成并通过以下 ADR gate：Runtime Scheduler ADR 必须定义 accumulator、normal tick vs catch-up、staged/double-buffer commit 和 request priority；Input Snapshot ADR 必须定义 capture timing、held-input cleanup、focus recovery 和 countdown pre-read；Presentation Delivery ADR 必须定义 packet lifecycle、ack lifecycle、display watermark、reentrancy/signal-order safety 和 consumer policy；Runtime Data Packet ADR 必须定义 GDScript value-copy / freeze / read-only strategy；Web Focus/Audio Unlock Shell ADR 必须映射 Godot 4.6.2 Web focus、visibility、fullscreen、canvas/keyboard focus 和 audio unlock/degradation events；Godot Pause/Time-scale ADR 必须说明是否避免或隔离 Godot pause/time-scale 以保护 combat authority；Physics Boundary ADR 必须定义 Godot 2D physics 如何只提供 broad collision/scene support；Profiling/QA Instrumentation ADR 必须定义 browser matrix、device class、warm-up、metric source、run aggregation、GC unavailable path 和 cold first-round scenario。每个 ADR 必须说明 Godot 4.6.2 API 依据、Web 导出限制、失败/降级路径、QA 证据入口，以及它如何满足本 GDD 的固定 tick、显示水印、AI 公平和不可变 packet 合同；若任一 ADR 未完成，runtime implementation task 不得开始。

### States and Transitions

| Runtime State | 含义 | committed runtime tick 是否提交 | 输入处理 | 允许进入 | 允许离开 |
|---|---|---:|---|---|---|
| `inactive` | 战斗运行时未开始或已卸载 | 否 | 不处理战斗输入 | 初始状态、退出对局 | `countdown` |
| `countdown` | 回合开始前倒计时/Ready-Go 表现 | 否 | 可预读方向输入；攻击/气弹/爆气不执行 | `inactive`、快速重开 | `running`、`paused` |
| `running` | 正常战斗结算 | 是；递增 `committed_tick_index` 和 `elapsed_running_ticks` | 采集并执行合法命令 | `countdown`、`hitstop` 结束、`paused` 恢复 | `hitstop`、`paused`、`focus_suspended`、`round_ended` |
| `hitstop` | 命中/格挡冻结反馈 | 是；递增 `committed_tick_index`，不递增 `elapsed_running_ticks` | 采集输入并写入缓冲，不执行普通行动 | `running` 中命中或格挡触发 | `running`、`paused`、`focus_suspended`、`round_ended` |
| `paused` | 玩家暂停、菜单暂停、`state_reason = recovery_pause` 的安全恢复暂停，或 `state_reason = presentation_ack_wait` 的显示水位等待 | 否 | 不接受战斗行动；菜单/恢复确认由 UI/menu context 处理；ack wait 期间 CPU/木桩和 silent catch-up blocked | `countdown`、`running`、`hitstop`、catch-up guard、presentation ack guard | 返回 resume countdown、原安全状态或 `inactive` |
| `focus_suspended` | 浏览器失焦、页面隐藏、canvas/input 焦点丢失或 Godot UI 焦点转移导致战斗输入不安全 | 否 | 不接受新战斗输入；恢复时清理/确认 held input | `running`、`hitstop`、`paused`、`countdown` | `paused` 或恢复到安全继续状态 |
| `round_ended` | KO、时间结束或系统判定回合结束 | 否 | 不接受战斗行动；只接受重开/退出/菜单确认 | `running`、`hitstop` | `countdown`、`inactive` |

`recovery_pause` 和 `presentation_ack_wait` 不得出现在 `runtime_state` 列表中；它们只能作为 `backlog_classification`、`state_reason`、`pause_reason` 或 delivery gate reason 出现，并表现为 `runtime_state = paused`。

状态优先级从高到低为：

1. `round_ended`
2. `focus_suspended`
3. `paused`
4. `hitstop`
5. `running`
6. `countdown`
7. `inactive`

如果同一 tick 触发多个状态请求，必须按上述优先级决定最终运行状态。例如：命中触发 hitstop 的同一 tick 造成 KO，则进入 `round_ended`，而不是继续停留在 `hitstop`。浏览器失焦优先于普通 pause；如果失焦前已经处于 `paused`，恢复焦点后仍回到 `paused`，不得自动恢复到 `running`。

最小 transition matrix：

| Current state | Request / trigger | Tick commit behavior | Result |
|---|---|---|---|
| `running` | valid hit/block requests hitstop only | Commit current running tick | `post_commit_runtime_state = hitstop`; first hitstop countdown starts next committed runtime tick. |
| `running` | KO or timeout reaches round end | Commit current running tick | `post_commit_runtime_state = round_ended`; no later combat tick for the round. |
| `running` | focus loss before next tick starts | No new tick | `runtime_state = focus_suspended`; backlog cleared, no catch-up for focus-lost interval. |
| `running` | pause request before combat action phase | No partial tick | `runtime_state = paused`; `pause_reason = player_pause` or system reason. |
| `hitstop` | normal hitstop countdown | Commit one hitstop tick | Decrement hitstop only; return to `running` on next boundary if remaining becomes 0 and no higher state applies. |
| `hitstop` | pause/focus/restart before hitstop tick commit | No hitstop tick committed | Remaining hitstop count is preserved unless restart/round end clears the round. |
| `hitstop` | round end becomes active from prior committed facts | No further hitstop tick | `runtime_state = round_ended`; final-hit presentation facts remain available in the last packet. |
| `paused` with `state_reason = recovery_pause` | resume confirmed | No combat tick during confirm | UI consumes confirm, then `countdown_phase = resume_ready -> resume_go`; combat resumes only after `resume_go` completes. |
| `paused` with `state_reason = presentation_ack_wait` | display watermark acknowledged or visually degraded | No combat tick during wait | Runtime may resume only the blocked dependency: later silent catch-up, automatic resume, or CPU/dummy decisions that depend on the shown fact. |
| `focus_suspended` | focus restored | No combat tick during restore | Requires focus reason display, held input cleanup/reconfirm, and resume countdown unless returning to existing `paused`. |
| any active round state | quick restart | No old round tick after request boundary | Clear old round runtime data, increment `round_instance_sequence` and `presentation_generation_id`, emit new tick 0 initial snapshot. |
| `round_ended` | restart | No old round tick | Enter new round `countdown` with new round instance. |

Recovery countdown minimum flow: `resume_ready` lasts until both conditions are true: at least 0.5 real seconds have elapsed and at least one visual countdown update has been acknowledged by UI/HUD; `resume_go` lasts until both conditions are true: at least 0.2 real seconds have elapsed and at least one visual `go` update has been acknowledged. Render-frame counts may be recorded for diagnostics, but they are not the pass/fail source of truth. During both phases, `combat_input_policy = resume_confirm_only` or `direction_pre_read_only`; attack, guard, projectile, burst, CPU command, and dummy command execution are blocked until the first post-countdown `running` tick. Direction pre-read may be recorded only if input-buffering later allows it; it cannot produce movement before combat resumes.

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

`runtime_event_sequence_key = (round_instance_sequence, committed_tick_index, phase_order_namespace_ordinal, tick_phase_order, within_phase_event_index)`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Round instance sequence | `round_instance_sequence` | int | `>= 1` | 当前回合实例的单调递增序号；用于排序。`round_instance_id` 可继续作为 equality/idempotency 分区，但不能混用 int/string 参与排序。 |
| Committed tick index | `committed_tick_index` | int | `>= 1` for runtime events; `0` for initial snapshot/reference events | 事件事实关联的 runtime tick。非 advancing state transition 使用 `runtime_state_sequence_key`，不伪装成 combat event。 |
| Phase order namespace ordinal | `phase_order_namespace_ordinal` | int | `10` / `20` / `90` | phase order 属于哪个顺序空间：`10 = running_tick_order`、`20 = hitstop_tick_order`、`90 = debug_trace_order`。非 combat state transition 使用 `runtime_state_sequence_key`，不进入本 key。 |
| Tick phase order | `tick_phase_order` | int | namespace-defined ordinal | 对应 namespace 内的阶段编号；running tick 使用 Detailed Rules 的 1–10，hitstop tick 使用 1–6，debug trace 使用 90+。 |
| Within-phase event index | `within_phase_event_index` | int | `>= 0` | 同一 tick、同一 namespace、同一阶段内事件的确定性顺序。 |
| Runtime event sequence key | `runtime_event_sequence_key` | tuple | `(int, int, enum, int, int)` | 用于事件排序和去重的字典序 key。 |

**Output Range:** 五元组。先按 round sequence 排序，再按 committed tick 排序，再按 namespace ordinal 和 phase ordinal 排序，最后按阶段内事件顺序排序。
**Example:** Event A 的 key 为 `(3, 1200, 10, 6, 2)`，Event B 的 key 为 `(3, 1200, 10, 7, 0)`；A 先于 B，因为阶段 `6` 早于阶段 `7`。如果 hitstop debug event 的 key 为 `(3, 1201, 20, 6, 0)`，它不会误用 running tick phase。快速重开后 `round_instance_sequence = 4` 的事件不会与旧 round 冲突。

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
| Catch-up max ticks | `catch_up_max_ticks` | int | MVP safe `1–4`；debug/stress may use `0` | 允许按顺序补跑的最大积压 tick 数；MVP 默认 `2`，`0` 只允许显式 debug/stress config。 |
| Backlog classification | `backlog_classification` | enum | `{focus_suspended, none, in_order_catch_up, recovery_pause}` | 运行时对积压情况的处理类别。 |

**Output Range:** 四种明确类别之一。focus lost 优先级最高；大积压不得静默丢 tick 或无提示快进。`recovery_pause` 是进入 `paused` 的原因，不是独立 runtime state。  
**Example:** 如果 `focus_lost = false`、`backlog_tick_count = 2`、`catch_up_max_ticks = 2`，则 `backlog_classification = in_order_catch_up`。如果 `backlog_tick_count = 3`，则 `backlog_classification = recovery_pause`。如果 `focus_lost = true`，则无论积压多少，`backlog_classification = focus_suspended`。

### Catch-up Wall-clock Guard

The `catch_up_wall_clock_guard_exceeded` formula is defined as:

`catch_up_wall_clock_guard_exceeded = measured_catch_up_wall_clock_ms >= catch_up_wall_clock_budget_ms`

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Measured catch-up wall-clock ms | `measured_catch_up_wall_clock_ms` | float | `>= 0` | 本 host update 内 catch-up runtime work 已消耗的实际 wall-clock 时间，包含 preflight、simulation、snapshot commit、event batch creation、trace instrumentation 和 runtime delivery metadata。 |
| Catch-up wall-clock budget ms | `catch_up_wall_clock_budget_ms` | float | MVP default `3.0`; safe range `1.0–4.0` | 单次 host update 允许 catch-up runtime work 使用的 wall-clock 上限。 |
| Catch-up wall-clock guard exceeded | `catch_up_wall_clock_guard_exceeded` | bool | `{true, false}` | 是否必须停止本次 silent catch-up。 |

**Output Range:** boolean。此 guard 限制单次 host update 内的 catch-up runtime work；如果 guard 在 backlog 完成前触发，就必须在安全 tick boundary 停止并进入 `recovery_pause` 或显式恢复流程。若 backlog 已完成，则 `completed_backlog` 优先于 wall-clock stop reason。
**Example:** 如果 `measured_catch_up_wall_clock_ms = 3.2`、`catch_up_wall_clock_budget_ms = 3.0`，则 guard exceeded 为 `true`。

### Catch-up Stop Reason

The `catch_up_stop_reason` formula is defined by precedence:

```text
catch_up_stop_reason =
  focus_suspended, if focus_lost = true
  invalid_backlog, if backlog_tick_count < 0 or backlog_tick_count > catch_up_max_ticks before catch-up starts
  none, if catch_up_attempted = false
  fairness_guard, if fairness_stop_required = true before the delayed tick commits
  blocking_state, if a committed catch-up tick ends in round_ended, focus_suspended, paused, or hitstop
  presentation_ack_guard, if presentation_ack_required = true after a committed catch-up tick
  completed_backlog, if remaining_backlog_ticks = 0
  wall_clock_guard, if catch_up_wall_clock_guard_exceeded = true and remaining_backlog_ticks > 0
```

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Catch-up attempted | `catch_up_attempted` | bool | `{true, false}` | 本 host update 是否实际尝试 catch-up；用于让 `none` 可测。 |
| Focus lost | `focus_lost` | bool | `{true, false}` | 是否发生 unsafe focus loss。 |
| Backlog tick count | `backlog_tick_count` | int | `>= 0` after validation | catch-up 开始前的积压 tick 数；负数归为 invalid/debug anomaly。 |
| Catch-up max ticks | `catch_up_max_ticks` | int | MVP `1–4`; debug may use `0` | 允许 silent catch-up 的最大 tick 数；MVP 默认 `2`。 |
| Blocking state reached | `blocking_state_reached` | bool | `{true, false}` | 已提交 catch-up tick 是否进入 `round_ended`、`focus_suspended`、`paused` 或 `hitstop`。 |
| Wall-clock guard exceeded | `catch_up_wall_clock_guard_exceeded` | bool | `{true, false}` | catch-up runtime work 是否达到 wall-clock budget。 |
| Fairness stop required | `fairness_stop_required` | bool | `{true, false}` | delayed tick 正式提交前的只读 preflight 是否发现会隐藏关键读招因果；true 时该 delayed tick 不提交。 |
| Presentation ack required | `presentation_ack_required` | bool | `{true, false}` | 已提交 catch-up packet 是否含有必须等待 display watermark 或视觉降级的 must-show fact。 |
| Remaining backlog ticks | `remaining_backlog_ticks` | int | `>= 0` | 本次 catch-up 停止时仍未处理的 backlog tick 数。 |

**Output Range:** `none | completed_backlog | wall_clock_guard | presentation_ack_guard | blocking_state | fairness_guard | focus_suspended | invalid_backlog`。每次 catch-up 尝试必须 trace：`catch_up_attempt_id`、`catch_up_attempted`、`backlog_tick_count`、`catch_up_ticks_committed`、`remaining_backlog_ticks`、`catch_up_stop_reason`、`blocked_transition_type`、`presentation_ack_id` when applicable、`display_watermark_target` when applicable、触发 tick id、以及是否进入 `recovery_pause`。`none` 只表示没有 catch-up 尝试且没有 focus/invalid backlog；`completed_backlog` 只表示已尝试 catch-up 且 backlog 清零。

**Combined-trigger rule:** 如果 catch-up preflight 发现 delayed tick 会在未展示前产生 hit/block、hitstop 或 KO，则该 delayed tick 不提交，`catch_up_stop_reason = fairness_guard`，runtime 停在上一枚已提交 tick 并进入 `runtime_state = paused`、`state_reason = recovery_pause`。如果一个已经安全提交的 catch-up tick 后续进入 `round_ended` 或 `hitstop`，则 `catch_up_stop_reason = blocking_state`。只有在没有更高优先级 blocking state 时，`presentation_ack_guard` 才把 runtime 转入 `runtime_state = paused`、`state_reason = presentation_ack_wait`；这不是独立 runtime state。`completed_backlog` 优先于 `wall_clock_guard`：如果 backlog 已清零，即使测得 wall-clock 超过预算，本次 stop reason 仍为 `completed_backlog`，另行记录 performance warning。

### Presented Running Tick Index

The `presented_running_tick_index` formula is defined as:

```text
presented_running_tick_index_next =
  presented_running_tick_index_current + 1,
    if next_unpresented_running_tick_index = presented_running_tick_index_current + 1
    and (
      (running_presentation_advance_policy = auto_advance
       and running_presentation_advance_result = auto_advanced)
      or
      (running_presentation_advance_policy = display_watermark_required
       and running_presentation_advance_result in {acknowledged, visually_degraded})
    )
  presented_running_tick_index_current,
    otherwise
```

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Current presented running tick index | `presented_running_tick_index_current` | int | `>= 0` | 玩家显示水位已经覆盖的最新 running-time index。tick `0` 表示 initial visible baseline。 |
| Next unpresented running tick index | `next_unpresented_running_tick_index` | int | `>= 1` | 下一枚尚未推进显示水位的 running tick。必须按顺序推进，不得跳过。 |
| Running presentation advance policy | `running_presentation_advance_policy` | enum | `{auto_advance, display_watermark_required, stale_discarded}` | 当前 running tick 如何推进显示水位。无关键可读事实的普通 tick 使用 `auto_advance`；含 `presentation_must_show` 事实的 tick 使用 `display_watermark_required`；旧 generation packet 使用 `stale_discarded`。 |
| Running presentation advance result | `running_presentation_advance_result` | enum | `{not_ready, auto_advanced, acknowledged, visually_degraded, stale_discarded, invalid_policy_result}` | `auto_advance` tick 在 packet 到达 primary snapshot consumer 后只能记为 `auto_advanced`；must-show tick 只有 display watermark 或明确视觉降级后才记为 `acknowledged` / `visually_degraded`；`stale_discarded` 和任何 policy/result 不匹配都不推进水位。 |
| Presented running tick index next | `presented_running_tick_index_next` | int | `>= presented_running_tick_index_current` | 下一次可用于 AI/木桩公平反应和 presentation visibility 的 running-time 水位。 |

**Output Range:** 非递减整数。只按 running ticks 推进；hitstop、pause、focus suspension、recovery pause、countdown、unacked must-show packet、stale packet 和 policy/result 不匹配的 ack response 不增加本 index。每个 committed running tick 必须在 commit 时写入 `committed_to_presented_running_map[committed_tick_index] = running_tick_ordinal`；hitstop 和非推进状态写入 `none`。多 tick catch-up packet 必须按 `ordered_event_batches` 的 running tick ordinal 顺序进入 display-watermark queue；即使后续 tick 的 consumer 先 ack，watermark 也只能从最小未呈现 running tick 顺序推进。无 must-show 的 normal realtime tick 只有在 packet 到达 primary snapshot consumer 且 host frame boundary 被记录后才可 `auto_advanced`，不能在构建 packet 时提前 auto-advance。每个 running tick 必须进入 display-watermark 队列或 auto-advance 队列，避免 normal tick 永久卡住水位。水位必须顺序推进：即使 tick `105` 已 ack，如果 tick `104` 还未 auto-advance、ack 或降级，`presented_running_tick_index` 仍停在 `103`。
**Example:** 当前 `presented_running_tick_index_current = 100`。tick `101` 没有关键可读事实，packet 到达 primary snapshot consumer 后 result = `auto_advanced`，输出 `101`。tick `102` 含 `startup_started`，必须等 display watermark 或视觉降级；若还未 ack，输出保持 `101`，CPU/木桩不得把 tick `102` 当作玩家已看到的反应基础。

### Decision Age Ticks

The `decision_age_ticks` formula is defined as:

```text
decision_age_ticks_raw = target_presented_running_tick_index - based_on_presented_running_tick_index

decision_age_ticks = decision_age_ticks_raw,
  if decision_age_ticks_raw >= min_ai_decision_age_ticks
  and the command is accepted as fair-reactive
```

`decision_age_ticks_raw` may be negative for invalid commands; accepted fair-reactive commands only expose `decision_age_ticks` after validation.

**Variables:**

| Variable | Symbol | Type | Range | Description |
|---|---:|---|---|---|
| Target presented running tick index | `target_presented_running_tick_index` | int | `>= min_ai_decision_age_ticks` for accepted fair-reactive non-player commands | CPU/木桩 command 目标 tick 对应的玩家已看到 running-time index；低于最小年龄时不得接受公平反应命令。 |
| Based-on presented running tick index | `based_on_presented_running_tick_index` | int | `>= 0` and `<= target_presented_running_tick_index - min_ai_decision_age_ticks` for accepted fair-reactive commands | CPU/木桩观察快照最后确认给玩家看到的 running-time index。 |
| Minimum AI decision age ticks | `min_ai_decision_age_ticks` | int | technical floor `6`; safe `6–18` | CPU/木桩最小可见反应延迟技术下限，只按已呈现 running ticks 计算；first-round/simple CPU 玩法默认不得低于 `12`，由下游 CPU GDD 选择更慢 profile。 |
| Decision age ticks raw | `decision_age_ticks_raw` | int | may be negative before validation | 目标 presented tick 与观察 presented tick 的原始差值；负数表示 observation 来自目标之后，必须拒绝。 |
| Decision age ticks | `decision_age_ticks` | int | `>= min_ai_decision_age_ticks` for accepted non-player commands | 决策从玩家已看到的观察到目标执行间隔了多少 presented running ticks；只对通过验证的 accepted fair-reactive command 成立。 |

**Output Range:** `decision_age_ticks_raw` 是有符号整数；若 `decision_age_ticks_raw < 0`，runtime 必须拒绝并记录 `rejection_reason = future_observation` 或 `invalid_presented_tick_mapping`。accepted simple CPU 或 `fair_reactive_dummy` command 的 `decision_age_ticks` 必须为非负且 `>= min_ai_decision_age_ticks`，否则拒绝并记录 `rejection_reason = decision_age_too_young`。`passive_dummy` 不生成命令；`scheduled_script_dummy` 使用脚本/config target tick 验证但不得读取玩家隐藏状态；`qa_fixture_dummy` 只允许 test/dev override。hitstop、pause、focus suspension、recovery pause、unacked catch-up presentation 不增加本公式的年龄。
**Example:** `target_presented_running_tick_index = 106`，`based_on_presented_running_tick_index = 100`，则 `decision_age_ticks = 6`，满足 MVP 默认最小反应延迟。如果 `target_presented_running_tick_index = 4` 且默认最小值为 `6`，公平反应命令不得被接受。

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
| `presented_running_tick_index` 的消费策略 | 本 GDD 拥有水位推进公式；HUD/VFX/UI/audio 各自如何达成 display watermark 属于表现层/ADR。 | Presentation Delivery ADR；HUD/VFX/Audio GDD |

### Registry Requirements

本系统的跨系统常量和公式必须登记到 `design/registry/entities.yaml`。已登记或本次修订必须补登记的项目如下：

| Registry Item | Type | Value / Expression | Why |
|---|---|---|---|
| `combat_ticks_per_second` | constant | `60` ticks/second | 所有战斗、输入、移动、HUD、debug trace 和测试必须共享同一 tick rate。 |
| `catch_up_max_ticks` | constant | `2` ticks | runtime、QA 和浏览器恢复行为需要共享阈值；MVP 选择小步追赶，避免 Web 长帧。 |
| `catch_up_wall_clock_budget_ms` | constant | `3.0` ms | catch-up wall-clock guard 需要可测默认预算，且包含 preflight 成本。 |
| `trace_window_ticks` | constant | `180` ticks | QA/debug trace 需要统一保留窗口。 |
| `trace_max_events_per_tick` | constant | `32` events | QA/debug trace 需要事件数量上限。 |
| `trace_max_payload_bytes_per_entry` | constant | `4096` bytes | QA/debug trace 需要 payload 上限。 |
| `trace_max_objects_per_tick` | constant | `64` objects | QA/debug trace 需要对象数量上限。 |
| `trace_warning_throttle_per_second` | constant | `10` warnings/sec | warning 输出需要节流。 |
| `min_ai_decision_age_ticks` | constant | `6` presented running ticks | CPU/木桩不能用 hitstop、pause、focus 或未展示 catch-up tick 伪装公平反应；这是技术下限，不是 first-round/simple CPU 玩法默认。 |
| `combat_critical_ack_deadline_ms` | constant | `50` ms | 战斗关键 cue 需要 next-host-frame / 短阈值内 ack、降级或失败，避免 250ms 微暂停破坏读招。 |
| `presentation_ack_deadline_ms` | constant | `250` ms | 非战斗 UI、恢复提示和调试 overlay 的 visual ack 需要有界等待和视觉降级上限。 |
| `host_frame_p95_budget_ms` | constant | `16.67` ms | Web exported profiling 需要明确 60fps p95 起点。 |
| `host_frame_p99_budget_ms` | constant | `25.0` ms | Web exported profiling 需要长帧告警起点。 |
| `runtime_normal_tick_budget_ms` | constant | `1.0` ms | 正常 runtime tick 需要保留渲染/UI/audio 预算，并让 2-tick catch-up 不吃满 Web frame。 |
| `max_single_frame_stall_ms` | constant | `50.0` ms | Web 体验需要限制可见卡顿尖峰。 |
| `profiling_heap_growth_fail_bytes` | constant | `4194304` bytes over 60s | Web profiling 需要稳态 heap growth 失败阈值。 |
| `profiling_gc_pause_fail_ms` | constant | `8.0` ms when available | Web profiling 需要 GC pause 指标可用时的失败阈值。 |
| `trace_total_memory_budget_bytes` | constant | `1048576` bytes | trace 需要总内存预算，不只按条数限制。 |
| `snapshot_summary_max_bytes` | constant | `2048` bytes | snapshot summary 需要可测 payload 上限。 |
| `event_batch_max_bytes` | constant | `8192` bytes | catch-up delivery 需要事件批次 payload 上限。 |
| `ai_observation_snapshot_max_bytes` | constant | `2048` bytes | CPU/木桩观察快照需要可测 payload 上限。 |
| `combat_tick_duration_seconds` | formula | `1 / combat_ticks_per_second` | 下游系统需要从 tick rate 推导真实秒长。 |
| `seconds_to_combat_ticks` | formula | `max(0, ceil(duration_seconds * combat_ticks_per_second))` | 下游系统若提供秒数，必须统一向上换算。 |
| `next_committed_tick_index` | formula | `current_committed_tick_index + 1` | running/hitstop committed runtime tick 必须统一递增。 |
| `elapsed_running_ticks` | formula | `sum(running_tick_flag_i)` | timer、HUD 和 QA trace 需要区分 running tick 与 hitstop tick。 |
| `elapsed_running_time_seconds` | formula | `elapsed_running_ticks / combat_ticks_per_second` | UI/QA 需要统一 active running time。 |
| `round_timer_remaining_ticks` | formula | `max(0, round_duration_ticks - elapsed_running_ticks)` | 回合 timer 需要排除 hitstop、pause、focus。 |
| `hitstop_remaining_ticks_after_tick` | formula | `max(0, hitstop_remaining_ticks_before_tick - hitstop_tick_flag)` | hitstop countdown 需要跨系统一致。 |
| `runtime_event_sequence_key` | formula | `(round_instance_sequence, committed_tick_index, phase_order_namespace_ordinal, tick_phase_order, within_phase_event_index)` | HUD/VFX/音效/debug 都依赖同一排序和去重 key，且 namespace 必须有明确 ordinal。 |
| `runtime_state_sequence_key` | formula | `(round_instance_sequence, state_transition_index)` | pause/focus/recovery/resume 等非 combat state transition 需要独立排序和幂等。 |
| `trace_window_bounds` | formula | `[max(0, latest_committed_tick_index - trace_window_ticks + 1), latest_committed_tick_index]` | QA/debug trace 需要统一保留窗口。 |
| `backlog_tick_count` | formula | `max(0, floor(max(0, delayed_real_seconds) / combat_tick_duration_seconds))` | Web backlog 分类必须统一且 clamp negative delay。 |
| `backlog_classification` | formula | focus/backlog enum classification | pause/focus/recovery 行为必须跨 runtime、UI、QA 一致。 |
| `catch_up_wall_clock_guard_exceeded` | formula | `measured_catch_up_wall_clock_ms >= catch_up_wall_clock_budget_ms` | Web catch-up 必须有可测 wall-clock guard。 |
| `catch_up_stop_reason` | formula | priority enum formula | QA 和恢复 UI 需要知道补跑停止原因、是否暂停在结果前、以及是否等待 presentation ack。 |
| `presented_running_tick_index` | formula | sequential auto-advance or display-watermark advancement | CPU/木桩公平反应、catch-up ack 和 HUD 可见水位必须共享同一 presented-running-time 定义，且 normal tick 不得永久卡住水位。 |
| `decision_age_ticks` | formula | `target_presented_running_tick_index - based_on_presented_running_tick_index` | CPU/木桩反应延迟必须按玩家已看到的 running ticks 可测，且不可读同 tick 隐藏状态。 |

`runtime_tick_phase_order` 是 deterministic invariant，不作为 tuning knob；后续如果 registry 支持 enum/list invariant，可再登记。

## Edge Cases

### Tick Scheduling, Backlog, and Browser Timing

- **If `delayed_real_seconds < combat_tick_duration_seconds`**: 不提交新的 runtime tick；表现层只能重绘最近一次已提交 snapshot。
- **If `backlog_tick_count = 0`**: 不执行 catch-up；运行时等待累计到足够真实时间后再提交下一 tick。
- **If `1 <= backlog_tick_count <= catch_up_max_ticks`**: 运行时最多按顺序补跑 `backlog_tick_count` 个 ticks；MVP 默认最多 `2` ticks。每个 delayed tick 正式提交前必须先通过只读 `catch_up_preflight`。如果 preflight、blocking state、presentation ack guard 或 wall-clock guard 触发，则提前停止，并记录 `catch_up_stop_reason`。
- **If `backlog_tick_count > catch_up_max_ticks`**: 运行时进入 `runtime_state = paused`，`state_reason = recovery_pause`；不静默跳 tick，不一次性快进，不在隐藏状态下推进战斗。
- **If `catch_up_max_ticks = 0` and `backlog_tick_count > 0`**: 任何积压都进入 `recovery_pause`；`0` 只允许显式 debug/stress config，不是 MVP 正常配置。
- **If focus is lost while backlog exists**: `focus_suspended` 优先于 catch-up；运行时停止 combat advancement，清理/忽略积压，不在恢复焦点后补跑失焦期间错过的 ticks。
- **If delayed time is negative due to clock/platform anomaly**: 将 delay 当作 `0`；不提交反向 tick，并记录 debug/trace warning。
- **If catch-up preflight descriptor is missing, incomplete, unstable, or finds a `fairness_stop_required` fact**: 不提交该 delayed tick；runtime 停在上一枚已提交 tick，进入 `runtime_state = paused`、`state_reason = recovery_pause`，trace 记录 `catch_up_stop_reason = fairness_guard`、`blocked_transition_type`、`blocked_tick_index`、`descriptor_complete`、`descriptor_hash` 和 `presentation_priority_reason`，并标记 `exchange_integrity = compromised_by_runtime_recovery`。MVP `fairness_stop_required` 至少包括将进入 active threat 的攻击或 projectile、`punish_window_opened`、`projectile_threat_entered`、以及可能在未展示前导致 `hit_landed`、`blocked`、`whiffed`、`burst_started`、`ko_started`、`round_ended` 或改变可反击/可防御结果的事实。
- **If a safely committed catch-up tick reaches `hitstop`, `paused`, `focus_suspended`, or `round_ended`**: 在提交该 transition tick 后停止本次 catch-up；不得继续补跑后续 ticks。
- **If catch-up processing reaches or exceeds `catch_up_wall_clock_budget_ms`**: 在最近一个安全 tick boundary 停止补跑并进入 `runtime_state = paused`、`state_reason = recovery_pause`；不得为了追赶时间制造更长主线程卡顿。若同一尝试也触发 fairness preflight 或 presentation ack guard，则优先记录玩家可见性原因。live combat exchange 内由 wall-clock guard 造成的恢复暂停同样标记 compromised exchange。
- **If catch-up tick emits `presentation_must_show` facts**: delivery context 必须标记 `presentation_ack_required = true`，并给出 `presentation_ack_id`、`display_watermark_target`、per-fact `ack_requirement_items` 与 `presentation_priority_reason`；每个 combat-spatial fact 必须由 combat-layer consumer 达到 display watermark、明确视觉降级或 terminal failure policy 后，后续依赖该事实的 AI/木桩决策、新的 silent catch-up tick 和自动 resume 才能继续。HUD-only 不能满足 normal-combat spatial fact。
- **If catch-up tick emits only `compressible_phase` facts**: 可以继续 catch-up；表现层可以压缩非威胁 spacing/facing、重复 idle/hold 状态和 debug-only 反馈，但不得压缩会改变当前交换理解的 hit/block/whiff/result、projectile threat、punish window 或 control-return 事实，不得删除 authoritative facts、改变 event ordering 或制造与真实 tick 顺序相反的反馈。
- **If catch-up needs an input snapshot for a delayed target tick**: 只能使用该目标 tick 已排队/已采集的 input snapshot；不得 retroactively 读取当前 host frame 的按键状态来填补过去 tick。
- **If CPU/dummy decision generation would observe an unpresented `fairness_stop_required` or `presentation_must_show` fact during catch-up**: 停止 catch-up 或冻结 CPU/木桩决策生成，直到对应 presentation packet 达到 display watermark 或明确视觉降级；CPU/木桩不得在玩家显示水位尚未覆盖的关键 transition、projectile threat、must-show readability phase 或 energy threshold 后继续生成下一 tick 反应。
- **If runtime enters `paused` because of `recovery_pause`**: backlog/accumulator 不得继续增长；恢复确认后从新的安全 timing baseline 继续，不能 replay 暂停期间真实时间。
- **If multiple catch-up ticks run inside one host/render frame**: 每个已提交 tick 仍必须执行完整固定 tick order，并各自产生独立 event batch；逻辑不得合并 tick。event batch 必须带 catch-up delivery metadata 和逐批 context，供 HUD/VFX/audio 压缩表现。

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
- **If quick restart creates a new round instance**: 必须生成新的 `round_instance_id`，递增 `round_instance_sequence` 和 `presentation_generation_id`，重置 `committed_tick_index = 0`，并拒绝旧 round command/event/snapshot/UI/audio/VFX/debug delivery。旧 round event key 不得影响新 round。

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
- **If catch-up delivers multiple event batches in one host frame**: `event_batch_delivery_contexts` 必须逐批标明 `delivery_mode = catch_up`、`delivery_tick_start`、`delivery_tick_end`、批次数量、批次索引、`final_snapshot_committed_tick_index`、`presentation_generation_id`、`contains_must_show`、`presentation_ack_required` 和 `display_watermark_target` when applicable；表现层可压缩非权威 pulse/VFX/audio，但不得删除或重排 authoritative facts。
- **If snapshot, event batch, UI state payload, or runtime state transition record is delivered to presentation**: 它们必须属于同一个 `RuntimePresentationPacket` 或等价原子 delivery；消费者不得把 tick `T` 的 event pulse 套到 tick `T+2` 的 snapshot 数值上，除非 packet 明确标记为 catch-up compression result。
- **If a runtime state transition occurs outside committed runtime tick advancement**: 使用 `runtime_state_sequence_key` 和 state-transition `idempotency_key` 发布 read-only state transition record；不得伪造 committed combat event。
- **If an old event batch from a previous round remains after quick restart**: 它必须返回 `stale_policy_result = discarded_stale_generation` 或 `discarded_stale_round`；不得影响新 round 的 HUD、VFX、audio、debug state 或 gameplay。
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
| 输入缓冲在 pause、countdown、focus suspension 中如何过期；hitstop 中不得仅按 committed runtime tick 自然过期，具体窗口由输入 GDD 定义 | 输入映射与输入缓冲 |
| move cancel、chain、whiff-cancel、hit-confirm、combo reset | 短连招与取消规则；角色数据与招式数据 |
| 防御方向、cross-up、projectile guard、unblockable tag | 防御、格挡与受击反馈 |
| 每招 hitstun、blockstun、hitstop、recovery 数值 | 命中停顿与硬直窗口；角色数据与招式数据 |
| damage、chip damage、health clamp、KO tie-breaker、timeout winner、draw rules | 血量、伤害、计时与胜负 |
| 气槽获得、消耗、refund、burst 可用性、burst invulnerability | 气槽与爆气反杀 |
| projectile speed、lifetime、owner collision、反弹、对波、多弹种 | 气弹 / 能量攻击 |
| wall collision、corner push、facing flip、overlap correction、jump/dash movement | 移动与距离控制 |
| CPU 具体行为表、decision frequency、script priority、dummy behavior；但最小公平反应延迟下限仍由本 GDD 定义 | 简单脚本 CPU；训练木桩 |
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
| 训练木桩 | Hard | command entry、AIObservationSnapshot、rejected-command trace | 木桩不得读取 same-tick collision/player input 或直接制造命中事实。 |
| 简单脚本 CPU | Hard | command entry、AIObservationSnapshot、decision-age trace、catch-up fairness guard | CPU 不得读取未呈现关键 transition、live node、debug trace 或 same-tick hidden state。 |
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

本系统只拥有固定运行时本身的调参项：tick rate、短暂掉帧补跑、QA trace 窗口、最小 CPU/木桩观察反应延迟、Web 性能/内存预算、countdown 输入预读策略、浏览器失焦恢复策略。伤害、硬直、招式帧数据、输入缓冲、移动速度、气弹速度、CPU 行为表、VFX/音效时长和回合长度都不属于本系统。

| Knob | MVP 默认 / 建议 | 安全范围 | 影响什么 | 过低 / 关闭会坏什么 | 过高 / 开启过度会坏什么 | Owner / Source of Truth |
|---|---:|---:|---|---|---|---|
| `combat_ticks_per_second` | `60` ticks/sec | `30–120`；MVP 锁定 `60` | 所有战斗时间粒度、帧数据解释、timer、trace、自动化测试。对应公式：`combat_tick_duration_seconds`、`seconds_to_combat_ticks`。 | 战斗 timing 变粗；短前摇、短确反、短连段窗口难以表达；玩家会感觉判定不精确。 | 浏览器 CPU 压力、事件量和 trace 量上升；所有下游 tick 数据都要重写；测试维护成本变高。 | `fixed-logic-runtime` GDD；跨系统常量，后续应登记到 registry。 |
| `catch_up_max_ticks` | `2` ticks，约 `0.033s` at 60 ticks/sec | `1–4`；`0` 只适合 debug/stress 模式 | 浏览器短暂卡顿后允许按顺序补跑多少 combat ticks。对应公式：`backlog_classification`。 | 轻微卡顿也频繁进入 `recovery_pause`，战斗被打断，玩家感觉游戏“太敏感”。 | 一次补跑太多 tick，可能造成主线程长帧、输入/表现跳跃，甚至 spiral-of-death；wall-clock guard 和 fairness guard 必须优先。 | GDD 定义语义；最终数值由 Runtime ADR / Web 性能验证锁定。 |
| `catch_up_wall_clock_budget_ms` | `3.0` ms per host update | `1.0–4.0` ms | 单次 host update 内 catch-up runtime work 的 wall-clock 上限，包含 preflight、模拟、snapshot/event packet、trace 和 delivery metadata。对应公式：`catch_up_wall_clock_guard_exceeded`。 | 太低会让轻微卡顿频繁进入 recovery pause。 | 太高会挤占渲染、UI、音频和浏览器预算，增加长帧与 spiral-of-death 风险。 | GDD 给出 MVP 起点；Runtime ADR / browser profiling 可调整。 |
| `trace_window_ticks` | `180` ticks，约 `3s` at 60 ticks/sec | `120–600`；更长只用于 dev/QA 临时抓取 | QA/debug 能回看最近多少 tick 的输入、状态变化、snapshot 和 event batch。对应公式：`trace_window_bounds`。 | 很容易丢失 bug 起因，例如输入、hitstop、pause/focus、hit/block 前的上下文。 | trace 噪音和内存/日志体积变大，Web 调试可能变慢，QA 更难定位重点。 | `fixed-logic-runtime` debug/QA config；不是玩家 balance。 |
| `trace_max_events_per_tick` | `32` events | `8–64` | 单 tick trace/event retention 的事件数量上限。 | 复杂同 tick bug 可能被截断，需要 warning。 | Web memory/GC 和 debug 噪音升高。 | `fixed-logic-runtime` debug/QA config。 |
| `trace_max_payload_bytes_per_entry` | `4096` bytes | `1024–8192` | 单 trace entry 的非权威诊断 payload 上限。 | 上下文过少，QA 难复现。 | 大 payload 造成 Web 内存和序列化压力。 | `fixed-logic-runtime` debug/QA config；不得截断权威事实。 |
| `trace_max_objects_per_tick` | `64` objects | `16–128` | 单 tick trace 可保留对象/记录数量上限。 | 复杂 tick 诊断信息不足。 | 对象分配和 GC 压力上升。 | `fixed-logic-runtime` debug/QA config。 |
| `trace_warning_throttle_per_second` | `10` warnings/sec | `1–30` | warning 输出节流，防止日志洪水。 | QA 可能看不到重复错误频率。 | 日志刷屏、Web 调试卡顿。 | `fixed-logic-runtime` debug/QA config。 |
| `min_ai_decision_age_ticks` | `6` ticks，约 `0.10s` at 60 ticks/sec | `6–18`；低于 `6` 禁止进入 exported gameplay | CPU/木桩从已提交观察到执行 command 的技术最小 tick 间隔。对应公式：`decision_age_ticks`。 | CPU/木桩会像读输入或读同 tick hidden state，破坏“我可以练会”。 | CPU/木桩显得迟钝；但 first-round/simple CPU 应由下游 GDD 使用 `>= 12` ticks 或脚本节奏保护新手公平感。 | Runtime 提供最小公平合同；具体 AI 行为表由 `simple-scripted-cpu` / `training-dummy` GDD 调整但不得低于已批准下限。 |
| `combat_critical_ack_deadline_ms` | `50` ms or next host-frame boundary | `16.67–100` ms | hit/block/whiff、active/projectile threat、punish/control、burst/KO/round-end 等战斗关键 cue 的 ack/degrade/fail 上限。 | 太低会把浏览器正常 host-frame jitter 误判为失败。 | 太高会制造玩家能感知的微暂停或晚提示，破坏读招。 | Presentation Delivery ADR 可调整；默认按玩家优先保守策略。 |
| `presentation_ack_deadline_ms` | `250` ms | `100–500` ms | 非战斗 UI、恢复提示和调试 overlay 的 visual ack 等待上限；到期必须显示等价视觉降级提示后才能继续依赖该 fact 的自动 resume、AI/木桩决策或 silent catch-up。 | 太低会频繁降级，削弱恢复/提示表现。 | 太高会让 ack wait 像卡死，破坏第一回合流畅度；不得用于 combat-critical cues。 | Presentation Delivery ADR 可调整；不能改成无上限等待。 |
| `host_frame_p95_budget_ms` | `16.67` ms | `<= 16.67` target | exported Web build 的 p95 host frame 时间。 | 预算过低会造成不必要警报。 | 预算过高会掩盖 60fps 失败。 | Runtime ADR / Web profiling。 |
| `host_frame_p99_budget_ms` | `25.0` ms | `16.67–33.33` ms | exported Web build 的 p99 长帧预算。 | 预算过低会把轻微浏览器 jitter 当失败。 | 预算过高会让明显卡顿通过。 | Runtime ADR / Web profiling。 |
| `runtime_normal_tick_budget_ms` | `1.0` ms | `0.5–2.0` ms | 单个正常 runtime tick 的模拟预算。 | 预算过低会阻碍调试实现。 | 预算过高会挤占渲染、UI、音频，并让 2-tick catch-up 接近或超过 Web frame 安全余量。 | Runtime ADR / Web profiling。 |
| `max_single_frame_stall_ms` | `50.0` ms | `33.33–100.0` ms | 单次可见卡顿尖峰上限。 | 预算过低会将浏览器偶发调度误判为失败。 | 预算过高会让玩家明显失控。 | Runtime ADR / Web profiling。 |
| `profiling_heap_growth_fail_bytes` | `4194304` bytes over 60s | `1048576–8388608` bytes | Web profiling 稳态 heap growth 失败阈值。 | 太低会把浏览器正常抖动误判为失败。 | 太高会掩盖 GDScript/packet/trace 泄漏。 | Profiling/QA Instrumentation ADR 可调整。 |
| `profiling_gc_pause_fail_ms` | `8.0` ms when available | `4.0–16.0` ms | 浏览器/runtime 暴露 GC pause 指标时的失败阈值；不可用时必须记录 unavailable 并使用 allocation proxy。 | 太低会造成不可复现失败。 | 太高会掩盖可感知卡顿。 | Profiling/QA Instrumentation ADR 可调整。 |
| `trace_total_memory_budget_bytes` | `1048576` bytes | `262144–4194304` bytes | trace ring buffer 总内存预算。 | QA 上下文不足。 | Web heap/GC 压力过高。 | Runtime debug/QA config。 |
| `snapshot_summary_max_bytes` | `2048` bytes | `1024–4096` bytes | 单个 snapshot summary 的数据合同预算。 | 预算过低会导致数据合同失败，而不是删除必需 HUD 字段。 | 每 tick copy/alloc 成本过高。 | Runtime data contract ADR。 |
| `event_batch_max_bytes` | `8192` bytes | `2048–16384` bytes | 单 tick event batch 的数据合同预算。 | 预算过低会导致数据合同失败或要求下游减少事件，而不是截断权威事件。 | catch-up delivery 和 debug 成本过高。 | Runtime data contract ADR。 |
| `ai_observation_snapshot_max_bytes` | `2048` bytes | `1024–4096` bytes | CPU/木桩观察快照的数据合同预算。 | 预算过低会导致 AI snapshot 合同失败，而不是补读隐藏事实。 | AI replay/hash 和 GC 成本过高。 | Runtime data contract ADR；AI GDD 消费。 |
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
| movement speed、projectile speed、CPU 行为表 / 决策频率 / 难度曲线 | 属于对应玩法系统；runtime 只拥有最小公平反应延迟下限。 | 移动与距离控制；气弹 / 能量攻击；简单脚本 CPU |
| VFX duration、audio timing、camera smoothing | 表现层可读性问题，不得影响 combat tick。 | VFX、音效、相机 / Presentation GDD |
| `runtime_tick_phase_order`、state priority、event sequence key | 这些是确定性合同，不是 tuning knobs。 | `fixed-logic-runtime` invariant |

### Registry Summary

以下 tuning constants 必须与 `design/registry/entities.yaml` 保持一致：

| Constant | Value | Unit | Why |
|---|---:|---|---|
| `combat_ticks_per_second` | `60` | ticks/second | 所有战斗、输入、移动、HUD、debug trace 和测试必须共享同一 tick rate。 |
| `catch_up_max_ticks` | `2` | ticks | runtime、QA 和浏览器恢复行为需要共享阈值；MVP 选择小步追赶。 |
| `catch_up_wall_clock_budget_ms` | `3.0` | ms | catch-up wall-clock guard 需要可测默认预算，且包含 preflight 成本。 |
| `trace_window_ticks` | `180` | ticks | QA/debug trace 需要统一保留窗口。 |
| `trace_max_events_per_tick` | `32` | events | QA/debug trace 需要事件数量上限。 |
| `trace_max_payload_bytes_per_entry` | `4096` | bytes | QA/debug trace 需要 payload 上限。 |
| `trace_max_objects_per_tick` | `64` | objects | QA/debug trace 需要对象数量上限。 |
| `trace_warning_throttle_per_second` | `10` | warnings/sec | warning 输出需要节流。 |
| `min_ai_decision_age_ticks` | `6` | presented running ticks | CPU/木桩最小观察反应延迟技术下限需要跨 runtime、AI、dummy 和 QA 一致；first-round/simple CPU 默认不得低于 `12`。 |
| `combat_critical_ack_deadline_ms` | `50` | ms | 战斗关键 cue 的 ack/degrade/fail 上限；避免 250ms 等待破坏读招。 |
| `presentation_ack_deadline_ms` | `250` | ms | 非战斗 UI、恢复提示和调试 overlay 的 visual ack 等待上限；超时必须走等价视觉降级提示。 |
| `host_frame_p95_budget_ms` | `16.67` | ms | Web profiling 需要 60fps p95 起点。 |
| `host_frame_p99_budget_ms` | `25.0` | ms | Web profiling 需要长帧告警起点。 |
| `runtime_normal_tick_budget_ms` | `1.0` | ms | 正常 tick runtime work 需要可测预算，并为 2-tick catch-up 保留 Web frame 余量。 |
| `max_single_frame_stall_ms` | `50.0` | ms | Web 体验需要可测卡顿上限。 |
| `profiling_heap_growth_fail_bytes` | `4194304` | bytes over 60s | Web profiling 稳态 heap growth 失败阈值。 |
| `profiling_gc_pause_fail_ms` | `8.0` | ms when available | Web profiling GC pause 指标可用时的失败阈值；不可用时必须记录 unavailable 并使用 allocation proxy。 |
| `trace_total_memory_budget_bytes` | `1048576` | bytes | trace ring buffer 需要总内存上限。 |
| `snapshot_summary_max_bytes` | `2048` | bytes | snapshot summary 需要 payload 上限。 |
| `event_batch_max_bytes` | `8192` | bytes | event batch 需要 payload 上限。 |
| `ai_observation_snapshot_max_bytes` | `2048` | bytes | AI observation snapshot 需要 payload 上限。 |

`countdown_direction_pre_read_enabled` 和 `focus_restore_requires_explicit_confirm` 是跨系统 boolean policy；它们保持在本 GDD 的 tuning knob 表中。后续如果 registry 支持 boolean policy/invariant，可登记它们；当前 registry 只同步数值 constants 和 formulas。

## Acceptance Criteria

### Acceptance Criteria Tiers

| Tier | Meaning | Criteria |
|---|---|---|
| MVP-blocking | Must pass before fixed runtime can be used by downstream MVP gameplay systems. Focus is fixed stepping, safe 2-tick catch-up, display-watermark fairness, minimum CPU/dummy fairness, Web smoke performance, immutable runtime delivery, ack validation, presented watermark advancement, audio degradation safety, cold first-round readiness, and player-facing causality trust. | AC-FLR-01–31, AC-FLR-37–42, AC-FLR-45–51, AC-FLR-53–55, AC-FLR-57–62, AC-FLR-65–73, AC-FLR-75–79 |
| ADR-gated | Must be accepted before implementation task breakdown; these are architecture decisions, not substitutes for behavior ACs. | Runtime Scheduler ADR; Input Snapshot ADR; Presentation Delivery ADR; Runtime Data Packet ADR; Web Focus/Audio Unlock Shell ADR; Godot Pause/Time-scale ADR; Physics Boundary ADR; Profiling/QA Instrumentation ADR |
| Hardening / downstream validation | Must be covered before content expansion or system-specific implementation depends on it, but does not block the first fixed-runtime vertical slice by itself. | AC-FLR-32–36, AC-FLR-43–44, AC-FLR-52, AC-FLR-56, AC-FLR-63–64, AC-FLR-74 |

Every behavior AC appears in exactly one primary tier. Required ADRs unblock implementation planning only; they do not count as passing runtime behavior tests. Because training dummy and simple CPU are MVP scope, their minimum fairness contracts remain MVP-blocking, while full behavior tables and advanced dummy lessons belong to downstream AI/dummy GDDs.

### AC-FLR-01 — Combat ticks are the authoritative simulation unit

**Given** a round is in `running` state with `combat_ticks_per_second = 60`, **When** the runtime advances combat for one second of valid running time, **Then** exactly 60 committed combat ticks are produced, tick IDs increase by exactly 1 per committed running tick, and all gameplay state changes are visible only in committed tick snapshots.

### AC-FLR-02 — “Frame” means combat tick unless otherwise specified

**Given** a move, timer, stun duration, recovery duration, or combat rule is described in frames, **When** the value is consumed by the fixed-logic runtime, **Then** the value is interpreted as combat ticks, and presentation frame rate changes do not alter the number of combat ticks required.

### AC-FLR-03 — Tick 0 is the initial snapshot and gameplay begins at tick 1

**Given** a new round instance has been created, **When** the initial combat snapshot is emitted, **Then** the snapshot uses `committed_tick_index = 0`, and no gameplay command, attack, projectile, damage, KO, timeout, or timer decrement has been processed yet.

**Given** the round enters its first valid `running` tick, **When** the first gameplay tick commits, **Then** the committed tick index is `1`.

### AC-FLR-04 — Non-advancing states do not increment committed runtime ticks

**Given** the runtime is in `inactive`, `countdown`, `paused`, `focus_suspended`, or `round_ended`, **When** real time passes, **Then** no new `committed_tick_index` is produced, `elapsed_running_ticks` does not increase, and the round timer does not decrease.

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

**Given** `hitstop_remaining_ticks_after_tick = 0`, **When** no higher-priority state applies, **Then** the runtime returns to `running` on the next valid combat advancement and records the resulting `post_commit_runtime_state`.

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

### AC-FLR-28 — Small backlog catches up only across fairness-safe ticks

**Given** the runtime has a backlog classified as `in_order_catch_up`, **When** catch-up is processed, **Then** each delayed tick is checked by read-only `catch_up_preflight` before commit, only preflight-safe delayed ticks are committed, committed delayed ticks are processed one at a time in increasing tick order, no committed tick ID is skipped, and each committed catch-up tick produces the same observable snapshot and event ordering rules as a normal runtime tick.

**Given** a QA test config forces `catch_up_wall_clock_budget_ms` to be exceeded after 1 preflight-safe catch-up tick, and `backlog_tick_count = 2`, **When** catch-up processing begins, **Then** exactly 1 delayed tick is committed, catch-up stops at the next safe tick boundary, runtime enters `runtime_state = paused` with `state_reason = recovery_pause`, and trace records `catch_up_stop_reason = wall_clock_guard`, `backlog_tick_count = 2`, `catch_up_ticks_committed = 1`, and `remaining_backlog_ticks = 1`.

**Given** delayed tick `T` has a `catch_up_preflight_descriptor` whose candidate transition fields include active threat, punish window, projectile threat, hit/block/whiff/KO/round end, or whose `descriptor_complete = false`, **When** `catch_up_preflight` evaluates tick `T`, **Then** tick `T` is not committed during silent catch-up, runtime remains at the previous committed tick, runtime enters `runtime_state = paused` with `state_reason = recovery_pause`, and trace records `catch_up_stop_reason = fairness_guard`, `blocked_tick_index = T`, `blocked_transition_type`, `descriptor_complete`, `descriptor_hash`, `presentation_priority_reason`, and `exchange_integrity = compromised_by_runtime_recovery`.

### AC-FLR-29 — Catch-up stops when a blocking state is reached

**Given** catch-up processing is committing delayed running ticks, **When** a committed tick causes `hitstop`, `paused`, `focus_suspended`, or `round_ended`, **Then** catch-up stops immediately after that tick commits, and no later backlog tick is processed while that blocking state is active.

### AC-FLR-30 — Large or unsafe backlog enters recovery pause instead of skipping ticks

**Given** focus is not lost and `backlog_tick_count > catch_up_max_ticks`, **When** backlog is classified, **Then** the runtime enters `runtime_state = paused`, `state_reason = recovery_pause`, no missed combat ticks are replayed silently, no committed runtime tick IDs are skipped, and the trace records `backlog_tick_count`, `catch_up_max_ticks`, `catch_up_attempted = false`, and `catch_up_stop_reason = invalid_backlog`.

**Given** focus is not lost and `backlog_tick_count <= catch_up_max_ticks`, **When** catch-up preflight detects a fairness-stop tick, catch-up exceeds `catch_up_wall_clock_budget_ms`, reaches a blocking state, or requires presentation ack, **Then** the runtime stops silent catch-up at the specified boundary, records the exact `catch_up_stop_reason`, exposes whether the stop occurred before or after a delayed tick commit, and does not continue accumulating backlog while paused or waiting for ack.

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

**Given** runtime events are emitted by the fixed-logic runtime, **When** QA inspects each event, **Then** each event includes `round_instance_sequence`, `committed_tick_index`, `phase_order_namespace_ordinal`, `tick_phase_order`, and `within_phase_event_index`, and sorting events by `(round_instance_sequence, committed_tick_index, phase_order_namespace_ordinal, tick_phase_order, within_phase_event_index)` reproduces the committed event order.

**Given** hitstop events, debug trace records, or state-transition presentation facts are emitted, **When** QA inspects their ordering metadata, **Then** combat/debug events use a valid `phase_order_namespace_ordinal`, while non-combat runtime state transitions use `runtime_state_sequence_key`; they do not reuse running-tick phase numbers without namespace and they do not mix state transition records into combat event ordering.

### AC-FLR-38 — Event batches are committed facts, not requests

**Given** an ordered event batch is emitted for a committed tick, **When** a presentation, HUD, VFX, audio, camera, or debug consumer receives the batch, **Then** each event represents a fact that already occurred in authoritative combat state, and consumers cannot convert the event into a different authoritative combat result.

### AC-FLR-39 — Event batches are idempotent for consumers

**Given** the same committed event batch is delivered to a consumer more than once, **When** the consumer processes the duplicate delivery, **Then** authoritative combat state does not change, and duplicate delivery does not create additional authoritative hits, damage, energy, KO, timeout, or combo changes.

### AC-FLR-40 — Quick restart invalidates old event batches

**Given** a round is restarted quickly, **When** the new round begins, **Then** the new round uses a new `round_instance_id`.

**Given** an old event batch from the previous round instance is received after restart, **When** consumers inspect the event metadata, **Then** the old batch is distinguishable from the current round by `round_instance_id`, and it cannot be mistaken for a current-round authoritative event.

### AC-FLR-41 — Trace window bounds formula is correct

**Given** `latest_committed_tick_index = 200` and `trace_window_ticks = 180`, **When** the runtime reports the available debug trace window, **Then** `trace_window_bounds = [max(0, latest_committed_tick_index - trace_window_ticks + 1), latest_committed_tick_index] = [21, 200]`.

### AC-FLR-42 — Minimal QA/debug trace exposes versioned schema-specific runtime facts

**Given** QA enables the minimal fixed-runtime trace, **When** any trace record is emitted, **Then** every record includes a shared header with `schema_version`, `round_instance_id`, `round_instance_sequence`, `record_type`, `committed_tick_index` when applicable, `runtime_state`, `tick_execution_state`, `post_commit_runtime_state`, and `trace_window_bounds`.

**Given** a `tick_snapshot` record is emitted, **When** QA validates the trace schema, **Then** it includes `elapsed_running_ticks`, `round_timer_remaining_ticks`, `hitstop_remaining_ticks`, `presented_running_tick_index`, and `snapshot_summary`.

**Given** an `event_batch` record is emitted, **When** QA validates the trace schema, **Then** it includes `phase_order_namespace_ordinal`, `tick_phase_order`, `within_phase_event_index`, `event_type`, `event_type_ordinal`, `runtime_event_sequence_key`, `presentation_tier`, `idempotency_key`, and `delivery_context_id` for each event.

**Given** a `runtime_state_transition` record is emitted, **When** QA validates the trace schema, **Then** it includes `previous_runtime_state`, `new_runtime_state`, `resume_target_state`, `state_reason`, `pause_reason`, `focus_loss_reason`, `runtime_state_sequence_key`, and transition `idempotency_key`.

**Given** a `catch_up_decision` record is emitted, **When** QA validates the trace schema, **Then** it includes `catch_up_attempt_id`, `catch_up_attempted`, `backlog_tick_count`, `backlog_classification`, `catch_up_ticks_committed`, `remaining_backlog_ticks`, `catch_up_stop_reason`, `blocked_transition_type`, `blocked_tick_index`, `presentation_ack_required`, `presentation_ack_id`, `display_watermark_target`, and `presentation_priority_reason`.

**Given** a `rejected_command` record is emitted, **When** QA validates the trace schema, **Then** it includes `command_source`, `command_source_ordinal`, `source_actor_id`, `command_id`, `command_sequence_key`, `target_committed_tick_index`, `target_presented_running_tick_index`, `input_snapshot_id`, `based_on_committed_tick_index`, `based_on_presented_running_tick_index`, `observation_snapshot_id`, `observation_snapshot_hash`, `decision_age_ticks`, `visibility_ack_generation_id`, `visibility_ack_watermark`, `script_or_config_id`, `script_or_config_hash`, `command_type`, `command_type_ordinal`, `pressed_or_held`, `runtime_state_at_capture`, `command_acceptance_result`, and `rejection_reason`.

**Given** a `presentation_delivery` or `bound_diagnostic` record is emitted, **When** QA validates the trace schema, **Then** delivery records include `presentation_generation_id`, `packet_sequence_key`, `delivery_context_id`, `delivery_mode`, `delivery_tick_start`, `delivery_tick_end`, `final_snapshot_committed_tick_index`, `presentation_ack_required`, `presentation_ack_id`, `display_watermark_target`, `ack_result`, and `stale_policy_result`; bound diagnostics include `bound_exceeded_type`, `configured_limit`, `observed_value`, and `entries_evicted_or_throttled`.

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

**Given** a player command, training dummy command, and simple CPU command represent the same legal action from the same legal combat state, **When** each command is submitted through the runtime command entry, **Then** each command is evaluated by the same tick-boundary command rules, and each command entry exposes `round_instance_id`, `round_instance_sequence`, `command_source`, `command_source_ordinal`, `source_actor_id`, `command_id`, `command_sequence_key`, `target_committed_tick_index`, `target_presented_running_tick_index` for non-player commands, `input_snapshot_id` for player commands or `based_on_committed_tick_index`, `based_on_presented_running_tick_index`, `observation_snapshot_id`, `observation_snapshot_hash`, and `visibility_ack_generation_id` for CPU/dummy commands, `command_type`, `command_type_ordinal`, `pressed_or_held`, `runtime_state_at_capture`, and `command_acceptance_result`.

**Given** multiple commands target the same committed tick, **When** the runtime builds the command batch, **Then** command order follows `command_sequence_key`, duplicate command IDs are idempotently deduped or rejected, and rejection trace records include a deterministic `rejection_reason`.

### AC-FLR-49 — CPU commands do not bypass runtime rules or read same-tick hidden state

**Given** the simple CPU script chooses an attack, guard, movement, projectile, or burst command for target committed tick `T`, **When** the command is submitted, **Then** the command is processed in the same batch-staged command phase as player and dummy commands, cannot bypass countdown, pause, focus suspension, hitstop freeze rules, command timing, state-machine legality, or tick-boundary commitment, and command-entry trace shows `command_source = simple_cpu`, `target_committed_tick_index = T`, `target_presented_running_tick_index`, `based_on_presented_running_tick_index`, `observation_snapshot_id`, `observation_snapshot_hash`, `visibility_ack_generation_id`, `interruption_epoch_at_observation`, `interruption_epoch_at_validation`, and `decision_age_ticks >= min_ai_decision_age_ticks`.

**Given** exported gameplay runtime minimum tuning is active, **When** a CPU command targets presented running tick index `P`, **Then** `min_ai_decision_age_ticks = 6`, so the command must be based on an observation whose `based_on_presented_running_tick_index <= P - 6`; hitstop ticks, pause time, focus-suspended time, recovery-pause time, and unacked catch-up presentation do not count toward those 6 ticks. This is only the technical floor; first-round/simple CPU profiles must target `P - 12` or slower in downstream CPU tuning.

**Given** a CPU command references missing, future, same-tick, stale-round, hidden, hash-mismatched, duplicate, too-young, stale-after-interruption, or disallowed observation data, **When** the command is evaluated, **Then** the runtime rejects it with a deterministic `rejection_reason` and no gameplay fact is created.

### AC-FLR-50 — Training dummy modes do not bypass runtime rules

**Given** the training dummy is configured as `passive_dummy`, **When** runtime command entry is processed, **Then** the dummy submits no combat command and cannot directly create hits, blocks, damage, stun, hitstop, energy, combo changes, KO, or timeout events.

**Given** the training dummy is configured as `scheduled_script_dummy`, **When** a scripted command is submitted, **Then** the command declares `pressed_or_held = scripted`, stable `script_or_config_id`, stable `script_or_config_hash`, target tick, and source metadata; it follows the same target tick/state-machine validation as other commands and cannot read same-tick collision, player input, or half-updated state.

**Given** the training dummy is configured as `fair_reactive_dummy`, **When** a reactive command is submitted, **Then** it is processed through the same command entry path as player and CPU commands, is based only on `AIObservationSnapshot` with `decision_age_ticks >= min_ai_decision_age_ticks` using presented running ticks, and cannot use hidden state.

**Given** the training dummy is configured as `qa_fixture_dummy`, **When** the command is submitted outside a test/dev build, **Then** runtime rejects it with `rejection_reason = qa_fixture_disallowed_in_build`.

**Given** any training dummy mode attempts an auto-block or counter command using same-tick hit/collision/player-input/half-updated state, **When** the command is evaluated, **Then** the runtime rejects it and records `rejection_reason = hidden_state_source`, `future_observation`, `observation_tick_mismatch`, `decision_age_too_young`, or the more specific applicable rejection reason.

### AC-FLR-51 — MVP tuning defaults are observable and enforced

**Given** the MVP fixed-logic runtime is initialized, **When** QA inspects runtime configuration or startup trace, **Then** the default values are `combat_ticks_per_second = 60`, `catch_up_max_ticks = 2`, `catch_up_wall_clock_budget_ms = 3.0`, `trace_window_ticks = 180`, `trace_max_events_per_tick = 32`, `trace_max_payload_bytes_per_entry = 4096`, `trace_max_objects_per_tick = 64`, `trace_warning_throttle_per_second = 10`, `min_ai_decision_age_ticks = 6 presented running ticks`, `combat_critical_ack_deadline_ms = 50`, `presentation_ack_deadline_ms = 250`, `host_frame_p95_budget_ms = 16.67`, `host_frame_p99_budget_ms = 25.0`, `runtime_normal_tick_budget_ms = 1.0`, `max_single_frame_stall_ms = 50.0`, `profiling_heap_growth_fail_bytes = 4194304`, `profiling_gc_pause_fail_ms = 8.0`, `trace_total_memory_budget_bytes = 1048576`, `snapshot_summary_max_bytes = 2048`, `event_batch_max_bytes = 8192`, `ai_observation_snapshot_max_bytes = 2048`, `countdown_direction_pre_read_enabled = true`, and `focus_restore_requires_explicit_confirm = true`.

**And** for MVP builds, `combat_ticks_per_second` remains locked to `60` unless an approved test configuration explicitly overrides it.

### AC-FLR-52 — Safe tuning ranges reject invalid values

**Given** runtime tuning values are loaded, **When** a value falls outside its safe range, **Then** the runtime rejects the invalid value or falls back to an approved safe/default value, and the trace records the invalid value and the effective value.

| Tuning knob | MVP safe range | Explicit debug/stress override |
|---|---:|---:|
| `combat_ticks_per_second` | 30–120; MVP locked 60 | none without ADR |
| `catch_up_max_ticks` | 1–4 | `0` allowed only in explicit debug/stress config |
| `catch_up_wall_clock_budget_ms` | 1.0–4.0 ms | lower/higher only in profiling config |
| `trace_window_ticks` | 120–600 | larger only in temporary QA/dev config |
| `trace_max_events_per_tick` | 8–64 | larger only in temporary QA/dev config |
| `trace_max_payload_bytes_per_entry` | 1024–8192 bytes | larger only in temporary QA/dev config |
| `trace_max_objects_per_tick` | 16–128 | larger only in temporary QA/dev config |
| `trace_warning_throttle_per_second` | 1–30 warnings/sec | none |
| `min_ai_decision_age_ticks` | 6–18 presented running ticks; technical floor 6 | below 6 forbidden in exported gameplay; first-round/simple CPU default must be >=12 in downstream CPU GDD |
| `combat_critical_ack_deadline_ms` | 16.67–100 ms; MVP default 50 | no unbounded wait; changes require Presentation Delivery ADR |
| `presentation_ack_deadline_ms` | 100–500 ms; MVP default 250 | non-combat UI/recovery only; no unbounded wait; changes require Presentation Delivery ADR |
| `host_frame_p95_budget_ms` | <= 16.67 ms target | none without Web profiling sign-off |
| `host_frame_p99_budget_ms` | 16.67–33.33 ms | none without Web profiling sign-off |
| `runtime_normal_tick_budget_ms` | 0.5–2.0 ms | none without Runtime ADR |
| `max_single_frame_stall_ms` | 33.33–100.0 ms | none without Web profiling sign-off |
| `profiling_heap_growth_fail_bytes` | 1048576–8388608 bytes over 60s | none without Profiling/QA ADR |
| `profiling_gc_pause_fail_ms` | 4.0–16.0 ms when available | none without Profiling/QA ADR |
| `trace_total_memory_budget_bytes` | 262144–4194304 bytes | larger only in temporary QA/dev config |
| `snapshot_summary_max_bytes` | 1024–4096 bytes | larger only with data-contract ADR |
| `event_batch_max_bytes` | 2048–16384 bytes | larger only with data-contract ADR |
| `ai_observation_snapshot_max_bytes` | 1024–4096 bytes | larger only with data-contract ADR |

### AC-FLR-53 — Deterministic QA re-simulation of the same test inputs produces the same observable results

**Given** the same initial round state, same tuning values, same player commands, same dummy commands, same CPU observation snapshots, same CPU script/config hash, same deterministic RNG seed/call counter if randomness is used, and same runtime requests, **When** the fixed-logic runtime is executed twice in a deterministic QA test harness, **Then** both runs produce the same committed tick IDs, snapshots, runtime state transitions, command acceptance/rejection results, and ordered event sequence keys, and no presentation frame rate, HUD, VFX, audio, camera, or Godot physics timing difference changes the authoritative combat result.

This criterion defines a QA/test re-simulation expectation only; it does not create a formal player replay, rollback, spectator, networking, or cross-version replay requirement.

### AC-FLR-54 — UI runtime state payload supports pause and focus recovery

**Given** the runtime enters `paused`, `focus_suspended`, `countdown`, or `round_ended`, **When** UI/menu/HUD/debug consumers read the runtime state payload, **Then** the payload exposes `runtime_state`, `state_reason`, `pause_reason` when applicable, `focus_loss_reason` when applicable, `previous_runtime_state` or `resume_target_state`, `runtime_state_sequence_key`, `requires_player_confirm`, `combat_input_policy`, `held_input_cleanup_required`, `held_input_cleanup_state`, `countdown_phase`, `player_facing_recovery_label`, `safe_resume_countdown_id`, `resume_confirm_consumed_by_ui`, `backlog_classification`, `catch_up_stop_reason` when applicable, `presentation_ack_required`, `presentation_ack_id`, `display_watermark_target`, `round_instance_id`, and `round_instance_sequence`.

**Given** the runtime enters recovery due to unsafe backlog, wall-clock guard, or fairness guard, **When** UI reads the payload, **Then** `runtime_state = paused`, `state_reason = recovery_pause`, `backlog_classification` remains observable, and `runtime_state` is never the value `recovery_pause`.

### AC-FLR-55 — Trace and instrumentation are bounded for Web

**Given** trace, rejected-command warnings, snapshot summaries, AI observation snapshots, or event batches are emitted, **When** emitted data exceeds `trace_window_ticks`, `trace_max_events_per_tick`, `trace_max_payload_bytes_per_entry`, `trace_max_objects_per_tick`, `trace_warning_throttle_per_second`, `trace_total_memory_budget_bytes`, `snapshot_summary_max_bytes`, `event_batch_max_bytes`, or `ai_observation_snapshot_max_bytes`, **Then** old bounded debug trace data may be evicted and warnings may be throttled, but authoritative snapshot fields, authoritative event facts, event ordering keys, command acceptance/rejection facts, and AI visibility facts are not truncated or dropped.

**Given** a non-authoritative diagnostic payload is evicted, summarized, or throttled, **When** QA inspects bound diagnostics, **Then** the trace records `bound_exceeded_type`, `configured_limit`, `observed_value`, `entries_evicted_or_throttled`, and `authoritative_fact_loss = false`.

**Given** a current authoritative snapshot summary, event batch, or AI observation snapshot exceeds its configured data-contract budget, **When** QA validates the packet, **Then** the result is a data-contract failure requiring schema/content reduction or ADR-approved budget change; the runtime must not silently truncate authoritative fields to pass the byte cap.

### AC-FLR-56 — Burst and energy commands use the runtime command and event path

**Given** a burst, energy, or projectile command is submitted by player, dummy, or CPU, **When** the command is accepted or rejected, **Then** the decision occurs through the same command entry path and state-machine legality checks as other combat commands, and committed results appear only through snapshots and ordered events such as `energy_meter_changed`, `burst_ready`, `burst_started`, or rejected-command trace warnings.

### AC-FLR-57 — Critical readability phase events are observable

**Given** a QA fixture move defines startup, active, recovery, and punishable windows, **When** the move transitions into or out of each phase, **Then** ordered event batches or snapshot summaries expose `actor_id`, `move_id`, `action_tick_index`, `phase_name`, `phase_started_or_ended`, `window_id` when applicable, `committed_tick_index`, and `runtime_event_sequence_key`.

**Given** the fixture reaches each required phase, **When** QA inspects events, **Then** the required observable events are `startup_started`, `active_frame_started`, `recovery_started`, `recovery_ended`, `punish_window_opened`, `punish_window_closed`, and `control_resumed`. Only facts delivered through catch-up/recovery/resume/stale-delivery risk and classified as critical hidden causality are `presentation_must_show`; normal realtime phase transitions remain observable without forcing presentation ack waits.

### AC-FLR-58 — Hitstop input buffering preserves first legal running-tick evaluation

**Given** input is captured during hitstop, **When** hitstop ends and the next legal running tick begins, **Then** the runtime delivers the captured input record to the input system with committed runtime tick context, running/action-time context, capture state, release/reconfirm state, and first-legal-running-tick marker. The runtime does not execute that input by itself; it only proves the input was preserved and handed off deterministically for the input-buffering system to accept or reject.

### AC-FLR-59 — Recovery pause uses safe resume countdown

**Given** the runtime enters `runtime_state = paused` with `state_reason = recovery_pause`, **When** the player resumes, **Then** UI shows the recovery reason, held combat inputs are cleaned or reconfirmed, the confirm input is consumed by UI/menu context, `countdown_phase = resume_ready` remains active until at least 0.5 real seconds have elapsed and a `countdown_visual_ack_id` records `phase = resume_ready`, `host_frame_index`, `host_time_ms`, and `minimum_visual_cue_met = true`, `countdown_phase = resume_go` remains active until at least 0.2 real seconds have elapsed and a second countdown visual ack records `phase = resume_go`, and no movement, guard, attack, projectile, burst, dummy command, or CPU command executes from the confirm input.

### AC-FLR-60 — Internal menu focus does not become unsafe focus suspension

**Given** the game is intentionally in `paused` and UI focus moves between pause-menu controls, **When** focus metadata is reported to the runtime with `focus_loss_reason = internal_menu_focus` or `hover_change`, **Then** ordinary menu focus movement does not trigger `focus_suspended`.

**Given** focus metadata is reported with `focus_loss_reason` from `{page_hidden, browser_window_blur, canvas_blur, keyboard_focus_lost, fullscreen_gate}`, **When** the runtime is in `running`, `hitstop`, `paused`, or `countdown`, **Then** the runtime enters `focus_suspended` before any further combat tick advancement.

**Given** audio unlock, muted tab, browser autoplay policy, or audio device failure occurs without canvas/keyboard focus loss, **When** the runtime is in `running`, `hitstop`, `paused`, or `countdown`, **Then** the runtime records `audio_degradation_reason`, emits audio degradation telemetry, does not enter `focus_suspended` for audio alone, does not block combat tick advancement, and does not count audio-only delivery as satisfying display watermark.

### AC-FLR-61 — Duplicate presentation delivery does not replay one-shot feedback

**Given** HUD, VFX, audio, camera, or debug consumers receive the same `RuntimePresentationPacket`, event, event batch, or runtime state transition more than once, **When** the consumer processes the duplicate delivery, **Then** the idempotency scope `(presentation_generation_id, round_instance_id, runtime_event_sequence_key or runtime_state_sequence_key, event_type, source_actor_id, target_actor_id, move_id_or_cause)` does not replay one-shot hit sounds, block sounds, whiff sounds, burst sounds, countdown sounds, KO sounds, HUD pulses, VFX bursts, or debug one-shots more than once.

### AC-FLR-62 — Event batch delivery contexts support catch-up compression

**Given** multiple committed event batches are delivered in one host/render frame due to catch-up, **When** HUD/VFX/audio/debug consumers receive them, **Then** `RuntimePresentationPacket.event_batch_delivery_contexts` contains one context per ordered batch with `delivery_mode = catch_up`, `delivery_tick_start`, `delivery_tick_end`, `batch_index_in_delivery`, `batch_count_in_delivery`, `final_snapshot_committed_tick_index`, `presentation_generation_id`, `round_instance_id`, `contains_must_show`, `presentation_ack_required`, and `presentation_ack_id`, allowing non-authoritative presentation compression without dropping or reordering authoritative facts.

### AC-FLR-63 — Snapshot, event, and trace payloads are immutable consumer data

**Given** a presentation, UI, audio, debug, or QA consumer receives snapshot, event, UI state, or trace data, **When** the consumer mutates its local copy or reference, **Then** authoritative combat state is unchanged, and the payload does not expose live Godot `Node`, mutable authority `Resource`, or shared mutable `Array` / `Dictionary` that can change gameplay state.

### AC-FLR-64 — Runtime state transitions are ordered and idempotent

**Given** pause, focus suspension, recovery pause, resume, restart, countdown, or round-end UI state transitions occur outside committed combat tick advancement, **When** presentation consumers inspect them, **Then** each transition exposes `runtime_state_sequence_key`, previous state, new state, state reason, resume target when applicable, confirmation requirement, and `idempotency_key`, and the transition is not represented as a committed combat event.

### AC-FLR-65 — AI-integrated deterministic QA produces the same CPU and dummy commands

**Given** the same initial state, same committed `AIObservationSnapshot` stream, same `observation_snapshot_hash` values, same `visibility_ack_watermark` values, same CPU/dummy `script_or_config_id`, same `script_or_config_hash`, and same deterministic RNG seed / stream id / call counter if randomness is used, **When** the CPU or training-dummy decision generator runs twice, **Then** both runs produce the same command entries, same `based_on_committed_tick_index`, same `target_committed_tick_index`, same `decision_age_ticks`, same RNG call counter after decision, same command acceptance/rejection results, and therefore the same committed snapshots/events after runtime execution.

### AC-FLR-66 — Catch-up event tiers produce the required fairness behavior

**Given** catch-up preflight detects a `fairness_stop_required` fact for delayed tick `T`, **When** the runtime handles the backlog, **Then** tick `T` is not committed during silent catch-up, the runtime remains at the previous committed tick, enters `runtime_state = paused` with `state_reason = recovery_pause`, and records `catch_up_stop_reason = fairness_guard` with `blocked_tick_index = T`.

**Given** catch-up commits a tick containing a `presentation_must_show` fact without a fairness stop, **When** the delivery packet is built, **Then** the delivery context marks `presentation_ack_required = true`, provides `presentation_ack_id` and `display_watermark_target`, and blocks later AI/dummy decisions, automatic resume, and additional silent catch-up ticks that depend on that fact until display watermark or explicit visual degradation is recorded.

**Given** catch-up emits only `compressible_phase` facts, **When** presentation consumers receive the delivery packet, **Then** authoritative facts remain complete and ordered, while non-authoritative pulse/VFX/audio display may be compressed without changing perceived cause/result order.

### AC-FLR-67 — Catch-up stop reason precedence is deterministic

**Given** multiple catch-up stop conditions become true during the same catch-up attempt, **When** the runtime records `catch_up_stop_reason`, **Then** the selected reason follows this precedence: `focus_suspended`, `invalid_backlog`, `none` when `catch_up_attempted = false`, `fairness_guard`, `blocking_state`, `presentation_ack_guard`, `completed_backlog`, then `wall_clock_guard` only when backlog remains.

**Given** `catch_up_attempted = false`, **When** catch-up classification is `none`, **Then** trace records `catch_up_stop_reason = none`.

**Given** `catch_up_attempted = true`, `remaining_backlog_ticks = 0`, and no focus, invalid backlog, fairness preflight, blocking state, presentation ack, or wall-clock guard applies, **When** catch-up ends, **Then** trace records `catch_up_stop_reason = completed_backlog`.

### AC-FLR-68 — AI observation snapshots expose only fair visible facts

**Given** an `AIObservationSnapshot` is created for simple CPU or `fair_reactive_dummy` decision making, **When** QA validates the snapshot, **Then** it is built by runtime visible-observation builder from committed snapshot plus acknowledged/degraded presentation watermark, and includes `round_instance_id`, `round_instance_sequence`, `committed_tick_index`, `presented_running_tick_index`, `visibility_ack_generation_id`, `visibility_ack_watermark`, `tick_execution_state`, `post_commit_runtime_state`, `runtime_state`, each actor's stable id / position band / facing / visible action state / visible action phase bucket / coarse phase age bucket, HUD-visible health / energy / timer, coarse projectile facts with stable projectile id and visible threat state, visible round state, `observation_snapshot_id`, and `observation_snapshot_hash`.

**Given** QA validates `AIObservationSnapshot` bucket values, **When** it inspects position/action/projectile fields, **Then** `position_band`, `visible_action_state`, `visible_action_phase_bucket`, `coarse_phase_age_bucket`, and `visible_threat_state` use only the MVP enums defined in Detailed Rules, and do not expose exact pixel position, exact action tick range, exact phase tick count, raw input, input buffer, hidden collision state, or unacked must-show facts.

**Given** hidden collision internals, same-tick player input, live Godot nodes, debug-only trace internals, unacked must-show facts, or uncommitted state are available elsewhere in the program, **When** the AI observation snapshot is built, **Then** those hidden facts are absent from the snapshot and cannot affect CPU/dummy command generation.

**Given** the AI decision function is invoked, **When** QA inspects the call boundary, **Then** it receives only value-copy `AIObservationSnapshot`, value-copy `AIConfig`, and deterministic `AIRngStream`, and it cannot access live runtime nodes, mutable Resources, event bus state, input buffers, or debug trace internals.

### AC-FLR-69 — CPU and dummy decisions freeze behind unpresented critical catch-up facts

**Given** catch-up preflight stops before a `fairness_stop_required` tick or a committed packet contains `presentation_ack_required = true`, **When** the target tick's `catch_up_preflight_descriptor`, display-watermark queue, or command validation context references facts at or after the blocked tick, **Then** runtime does not invoke the CPU/dummy decision generator for that target tick, does not enqueue a placeholder command, and records `decision_generation_blocked_by = fairness_guard` or `presentation_ack_guard` plus current `interruption_epoch` until the player-facing `RuntimePresentationPacket` has display watermark, explicit visual degradation, or terminal failure policy recorded.

**Given** a CPU or dummy command was generated before the critical fact was acknowledged to the player, **When** runtime command entry validates it, **Then** the command is rejected with `rejection_reason = stale_due_to_runtime_interruption`, `decision_age_too_young`, `future_observation`, `observation_tick_mismatch`, or `hidden_state_source` as applicable.

### AC-FLR-70 — Rejected command trace uses a complete reason enum

**Given** a command is rejected by runtime command entry, input policy, observation validation, ordering validation, build-policy validation, or state-machine legality, **When** QA inspects the rejected-command record, **Then** it contains one reason from: `stale_round`, `duplicate_command_id`, `target_tick_already_processing`, `invalid_presented_tick_mapping`, `invalid_source_actor`, `invalid_runtime_state`, `missing_observation_snapshot`, `observation_tick_mismatch`, `observation_hash_mismatch`, `duplicate_observation_snapshot_id`, `future_observation`, `ack_watermark_missing`, `ack_watermark_stale`, `ack_watermark_hash_mismatch`, `decision_age_too_young`, `stale_due_to_runtime_interruption`, `hidden_state_source`, `conflicting_same_actor_command`, `invalid_enum`, `disallowed_by_input_policy`, `qa_fixture_disallowed_in_build`, or `state_machine_rejected`.

**Given** two invalid conditions apply to the same command, **When** the command is rejected, **Then** the runtime uses a deterministic validation order so the same input produces the same `rejection_reason` across repeated QA runs.

### AC-FLR-71 — Runtime presentation delivery is atomic

**Given** runtime presentation data is delivered after one or more committed ticks or state transitions, **When** HUD, VFX, audio, camera, debug, or UI consumers receive it, **Then** the data arrives as one atomic `RuntimePresentationPacket` or equivalent read-only bundle with `presentation_generation_id`, `packet_sequence_key`, `round_instance_id`, `round_instance_sequence`, `final_snapshot_committed_tick_index`, `delivery_tick_start`, `delivery_tick_end`, `snapshot_summary`, `ordered_event_batches`, `event_batch_delivery_contexts`, `ui_runtime_state_payload`, `runtime_state_transition_records`, `presentation_ack_requirements`, `packet_size_bytes`, and `stale_policy_result`.

**Given** `ordered_event_batches` contains more than one batch, **When** QA validates the packet, **Then** `event_batch_delivery_contexts` contains exactly one context per batch with matching `delivery_context_id`, `batch_index_in_delivery`, and `batch_count_in_delivery`.

**Given** the packet contains `presentation_ack_requirements`, **When** QA validates the packet, **Then** each requirement includes `presentation_ack_id`, `presentation_generation_id`, `ack_owner`, `required_visual_consumer_set`, `required_visual_consumer_policy`, `optional_audio_consumer_set`, `must_show_fact_ids`, `must_show_fact_ids_hash`, `ack_requirement_items`, `display_watermark_target`, `ack_deadline_policy`, `ack_deadline_ms`, `ack_state`, `ack_result`, `ack_degraded_reason`, `ack_wait_started_at_host_ms`, `ack_wait_duration_ms`, and `ack_response_record_ids`, and each `ack_requirement_item` includes `minimum_visual_cue_type`, `minimum_visual_cue_requirement`, `allowed_consumer_types`, `hud_only_allowed`, `spatial_combat_fact`, `non_color_backup_required`, and `minimum_visible_duration_ms`.

**Given** runtime enters `runtime_state = paused` with `state_reason = presentation_ack_wait`, **When** QA inspects UI runtime state and ack diagnostics, **Then** UI/HUD shows `player_facing_recovery_label = showing_key_moment`, `combat_input_policy = blocked_all` or `resume_confirm_only` as specified by the transition, `presentation_ack_id`, `display_watermark_target`, `ack_deadline_ms`, and `ack_wait_duration_ms` are visible to diagnostics, and the wait resolves only through a valid ack response, explicit visual degradation, terminal failure policy, stale discard, or quick restart invalidation.

**Given** an event batch from tick `T` and a snapshot from tick `T + 2` exist in the same host frame, **When** presentation consumers update, **Then** they cannot pair tick `T` one-shot feedback with tick `T + 2` current values unless the packet explicitly marks that pairing as a catch-up compression result.

### AC-FLR-72 — Quick restart invalidates stale presentation delivery

**Given** quick restart creates a new round instance, **When** the first presentation packet for the new round is delivered, **Then** it uses a new `round_instance_id`, incremented `round_instance_sequence`, incremented `presentation_generation_id`, and `committed_tick_index = 0` initial snapshot.

**Given** a stale command, event batch, snapshot, UI payload, audio/VFX request, debug record, ack, or presentation packet from the old round arrives after restart, **When** consumers validate `round_instance_id`, `round_instance_sequence`, or `presentation_generation_id`, **Then** the stale delivery returns `stale_policy_result = discarded_stale_generation` or `discarded_stale_round`, is not processed as current, and cannot affect current gameplay or one-shot presentation.

### AC-FLR-73 — Web performance budgets are measurable

**Given** the exported Web MVP runs the real fixed-runtime steady-state profiling scenario, **When** QA collects frame and runtime metrics, **Then** the report measures actual browser/runtime values rather than only checking configured constants, and each run passes only if: host frame p95 `<= host_frame_p95_budget_ms = 16.67ms`, host frame p99 `<= host_frame_p99_budget_ms = 25.0ms`, normal runtime tick work p95 `<= runtime_normal_tick_budget_ms = 1.0ms`, catch-up runtime work per host update including descriptor/preflight p95 `<= catch_up_wall_clock_budget_ms = 3.0ms` unless it enters recovery pause, no single measured visible stall exceeds `max_single_frame_stall_ms = 50.0ms`, `heap_growth_bytes_over_60s <= profiling_heap_growth_fail_bytes = 4194304`, no measured GC pause exceeds `profiling_gc_pause_fail_ms = 8.0ms` when that metric is available, and no run reports `authoritative_fact_loss = true`, `compromised_exchange_count_per_round > 0` in the first round, or combat-critical ack failure.

**Given** QA runs the real browser profiling scenario, **When** the scenario is configured, **Then** it uses exported Web release or release-debug builds on the MVP blocking browser matrix: desktop Chrome stable and desktop Safari stable on the declared MVP device class, 3 runs per browser, 10-second warm-up excluded from metrics, then 60 seconds measured per run, one player actor, one training dummy or simple CPU actor, HUD enabled, minimal actor/projectile/combat VFX and audio consumers enabled, debug trace in MVP minimal mode, at least one scripted hit/block/hitstop sequence, at least one projectile-threat sequence, one naturally injected or harness-injected 2-tick preflight-safe catch-up, one fairness preflight stop, one combat-critical presentation ack/degrade/fail check, one non-combat UI/recovery ack check, and one quick restart stale-delivery check. A browser passes only if all 3 runs pass; the MVP matrix passes only if both blocking browsers pass. Firefox/Edge profiling is advisory until a Profiling/QA Instrumentation ADR promotes it to blocking.

**Given** QA validates cold first-round readiness, **When** the game loads from a fresh page/session into the first playable round, **Then** countdown cannot reach `go` until pre-round readiness reports HUD, actor sprite, projectile/combat VFX, audio unlock/degradation state, and debug-minimal trace as ready or explicit degraded-ready; the first hit/block, first projectile, first burst, first HUD pulse, first audio degradation, and first quick restart must not exceed `max_single_frame_stall_ms`, must not trigger a live-exchange recovery pause, and must not produce `authoritative_fact_loss = true`.

**Given** QA validates backlog conformance rather than real browser frame performance, **When** the harness forces a 5-tick backlog, **Then** the fixture is marked `synthetic_backlog_conformance = true`, is excluded from host-frame p95/p99 and single-stall pass/fail calculations, and must prove that the 5-tick backlog enters `runtime_state = paused` with `state_reason = recovery_pause` before silent catch-up commits any unsafe delayed tick.

**Given** QA calculates p95 and p99 frame metrics, **When** the report is produced, **Then** the sample window, warm-up exclusion, quantile method, browser name/version, device class, OS, build type, run count, pass/fail per run, aggregate pass/fail per browser, GC/allocation metrics or explicit `gc_metrics_unavailable = true`, allocation proxy source when GC metrics are unavailable, host-frame source, runtime-work timer source, heap-growth source, synthetic fixture exclusion list, and discarded/throttled diagnostic counts are included so a later team can reproduce the pass/fail result. If GC metrics are unavailable, the report cannot claim GC safety; it must pass heap/allocation proxy thresholds and mark GC pause status as inconclusive advisory.

### AC-FLR-74 — Recovery, focus, pause, hitstop, and restart transition matrix is testable

**Given** QA drives each transition row defined in the runtime transition matrix, including running hit/block into hitstop, running KO/timeout into round end, focus loss before the next tick, pause before combat action phase, pause/focus/restart during hitstop, recovery-pause resume, focus restore, quick restart, and round-ended restart, **When** the runtime processes each scenario, **Then** the observed `runtime_state`, `state_reason`, `resume_target_state`, `runtime_state_sequence_key`, `committed_tick_index`, and combat advancement/freeze result match the specified row.

**Given** `recovery_pause_count_per_round` exceeds 2 or 4 in one round, **When** QA inspects runtime diagnostics, **Then** values above 2 produce a warning and values above 4 mark the Web experience test as failed unless an approved Runtime ADR or performance tuning change overrides the threshold.

**Given** `presentation_ack_wait_count_per_round` is greater than 0, **When** QA inspects runtime diagnostics, **Then** each wait has a matching `presentation_ack_id`, `display_watermark_target`, ack or degradation result, wait duration, and reason so the team can distinguish expected fairness pauses from broken presentation delivery.

### AC-FLR-75 — Presentation ack responses are validated before advancing dependent systems

**Given** a must-show packet generates `presentation_ack_id = A`, **When** a presentation consumer returns an ack response, **Then** runtime accepts it only if `presentation_generation_id`, `round_instance_id`, `round_instance_sequence`, `presentation_ack_id`, `must_show_fact_ids_hash`, and `acknowledged_presented_running_tick_index` match the current ack requirement.

**Given** an ack response is stale, superseded, from an old round, uses the wrong display watermark target, fails per-fact cue policy, reports HUD-only delivery for a normal-combat spatial fact, or reports audio-only delivery, **When** runtime validates it, **Then** the response is discarded with `stale_policy_result` or ack diagnostics, does not advance `presented_running_tick_index`, does not unlock AI/dummy decisions, and does not allow additional silent catch-up.

**Given** `ack_deadline_ms` expires for a must-show fact, **When** visual ack is still missing, **Then** the responsible visual consumer or UI/HUD fallback must display the defined degraded visual cue from that fact's `ack_requirement_item`, record `ack_result = visually_degraded`, and only then may dependent automatic resume, AI/dummy decisions, or silent catch-up proceed. If the degraded cue also fails, runtime records `ack_result = failed_terminal`, increments `combat_critical_ack_failure_count_per_round` when applicable, and blocks dependent automatic resume, AI/dummy decisions, or silent catch-up until stale discard, restart, or explicit recovery policy resolves the failure.

### AC-FLR-76 — Presented running tick index advances only on valid policy/result pairs

**Given** a running tick has no must-show facts and `running_presentation_advance_policy = auto_advance`, **When** its packet reaches the primary snapshot consumer and the host-frame boundary is recorded, **Then** `running_presentation_advance_result = auto_advanced` advances `presented_running_tick_index` by exactly 1 if it is the next sequential running tick.

**Given** a running tick has must-show facts and `running_presentation_advance_policy = display_watermark_required`, **When** the valid ack response records `acknowledged` or `visually_degraded`, **Then** `presented_running_tick_index` advances by exactly 1 if it is the next sequential running tick.

**Given** policy/result are mismatched, the packet is stale, or a later tick acks before an earlier tick, **When** runtime evaluates the presented-running watermark, **Then** the index does not advance and diagnostics record the hold reason.

### AC-FLR-77 — Audio degradation is telemetry, not combat focus suspension

**Given** Web audio unlock, muted tab, browser autoplay policy, or audio device failure occurs without unsafe canvas/keyboard focus loss, **When** audio consumers process runtime presentation, **Then** audio degradation telemetry records `audio_degradation_reason`, affected `presentation_generation_id`, optional audio consumer id, and failed audio request id, but runtime does not enter `focus_suspended`, does not block combat tick advancement, and does not count audio-only delivery as satisfying display watermark.

**Given** quick restart or stale generation invalidates pending audio requests, **When** audio consumers receive old one-shots or loop requests, **Then** they discard or stop them using `presentation_generation_id` plus audio request/loop id and do not replay old round sounds.

### AC-FLR-78 — First-round runtime recovery does not masquerade as fair combat

**Given** QA runs the cold first-round scenario on the MVP blocking browser matrix, **When** any live combat exchange enters `recovery_pause`, `presentation_ack_wait` terminal failure, or `exchange_integrity = compromised_by_runtime_recovery`, **Then** the first-round experience test fails, diagnostics record the compromised exchange reason, and normal match resume returns through safe neutral/reacquisition rather than granting a tactical read advantage inside the interrupted exchange.

**Given** the same recovery happens in training mode, **When** the player resumes, **Then** the UI may explicitly replay or extend the telegraph as a training aid, but the trace must mark the exchange as training-assisted and not equivalent to uninterrupted normal combat.

### AC-FLR-79 — Player-facing causality remains understandable after runtime interruptions

**Given** a QA/playtest build exposes post-round diagnostic prompts, **When** a round includes hit/block/whiff, punish, projectile threat, burst, recovery pause, visual degradation, or CPU/dummy reaction, **Then** QA can map each player-facing outcome to ordered runtime facts and ask the player to identify the cause category; a first-round readiness pass requires no report that browser pause, HUD warning, stale audio, or CPU input-reading appeared to decide the exchange.

### Instrumentation Note

The following criteria require explicit QA-facing instrumentation if it does not already exist: ordered event sequence keys, committed snapshot summaries, runtime state transition trace, backlog classification trace, rejected-command warnings, trace window bounds, command-source metadata for player/dummy/CPU command entry, UI runtime state payload, catch-up preflight descriptors and hashes, catch-up wall-clock/fairness guard decisions, catch-up stop reason precedence, committed-to-presented-running tick mapping, AI observation snapshots and hashes, interruption epochs, immutable payload diagnostics, atomic runtime presentation packets, per-fact ack requirement items, event delivery context, stale presentation diagnostics, runtime state sequence keys, Web performance budget metrics, cold first-round readiness metrics, compromised exchange counters, combat-critical ack failure counters, recovery/focus interruption counters, and bounded trace payload diagnostics.

## Visual/Audio Requirements

本系统不直接拥有视觉资产、音效资产、动画资产或相机表现；它只定义这些表现系统可以信任和消费的 runtime facts。

- Runtime 必须用原子 `RuntimePresentationPacket` 或等价只读包为 HUD、VFX、音效、相机和调试显示提供 committed snapshot summaries、ordered event batches、runtime state payload、runtime state transition records、逐批 `event_batch_delivery_contexts`、presentation ack requirements、tick id、round instance、`presentation_generation_id`、position context、result type 和 idempotency key。
- `startup_started`、`active_frame_started`、`recovery_started`、`recovery_ended`、`punish_window_opened`、`punish_window_closed`、`control_resumed`、`hit_landed`、`blocked`、`whiffed`、`hitstop_started`、`energy_meter_changed`、`burst_started`、`round_ended` 等事件必须足够明确，让表现层不需要自行推断 combat result。
- VFX、音效、动画、相机震动和 HUD 闪烁可以延迟、压缩或去重表现，但不得改变、补写、删除或重排 authoritative combat facts；对于 `presentation_must_show`，required visual consumer 必须按 per-fact `ack_requirement_item` 达到 display watermark、明确视觉降级或 terminal failure policy。normal-combat spatial facts 必须由 actor sprite、projectile VFX 或 combat VFX 满足；HUD/UI 只能辅助，除非模式明确是 training/debug/accessibility assist。audio-only ack 不足以恢复依赖该事实的 AI/补跑。
- hitstop 中允许表现层继续播放已触发的 VFX/音效和 HUD 动画，但不得推进 combat action frames、判定、硬直、timer 或 projectile combat advancement。
- catch-up 期间 runtime 仍输出完整事件顺序；每个事件批次必须带有 `delivery_mode`、`delivery_tick_start`、`delivery_tick_end`、`final_snapshot_committed_tick_index`、`presentation_generation_id`、`contains_must_show`、`presentation_ack_required`、`presentation_ack_id`、`display_watermark_target` 和 per-fact `ack_requirement_items` when applicable。表现层只能压缩不会改变玩家因果理解的非权威展示；hit/block/whiff/result、projectile threat、punish window、control return 和 round end 不得被压缩成错误顺序或 HUD-only 替代。
- 表现层如果重复收到同一 packet、state transition record 或 event batch，必须用 `presentation_generation_id`、event sequence key 和 idempotency key 去重；不得重复造成权威 gameplay 变化或重复播放 HUD/VFX/audio/UI/debug one-shot。audio one-shot 必须带 `audio_request_id` 和 idempotency key；audio loop 必须带 `audio_loop_id`、`presentation_generation_id` 和 stop policy，quick restart 不得留下旧 round loop。Web audio unlock、muted tab、browser autoplay policy 或 audio device failure 只能产生 audio degradation telemetry；audio 不能单独满足 display watermark，也不能阻塞 combat tick。
- 本系统不触发 asset-spec 生产；后续 `combat-hud-feedback`、`readable-combat-vfx`、`combat-audio-feedback` 和 `sprite-animation-presentation` GDD 会定义具体资产和表现规格。

## UI Requirements

本系统不拥有正式玩家 UI 屏幕，但必须向下游 UI、菜单和调试系统暴露稳定状态。

- HUD 必须能读取当前 `round_instance_id`、`round_instance_sequence`、`committed_tick_index`、runtime state、round timer remaining、ordered event batches、`presentation_generation_id` 和必要 snapshot summary。
- HUD 当前数值必须以 committed snapshot summary 为权威；event batch 只触发一次性反馈、pulse、音效/VFX 请求或 debug log。若二者冲突，snapshot wins。
- 暂停菜单和 Web focus recovery UI 必须通过 runtime request 进入/离开 `paused` 或 `focus_suspended`，不得直接修改 combat state。
- UI runtime state payload 必须表达 `runtime_state`、`state_reason`、适用时的 `pause_reason`、适用时的 `focus_loss_reason`、适用时的 `audio_degradation_reason`、`previous_runtime_state` 或 `resume_target_state`、`runtime_state_sequence_key`、`requires_player_confirm`、`combat_input_policy`、`held_input_cleanup_required`、`held_input_cleanup_state`、`countdown_phase`、`countdown_visual_ack_id` when applicable、`player_facing_recovery_label`、`safe_resume_countdown_id`、`resume_confirm_consumed_by_ui`、`exchange_integrity` when applicable、`backlog_classification`、适用时的 `catch_up_stop_reason`、`presentation_ack_required`、`presentation_ack_id`、`display_watermark_target`、适用时的 `ack_deadline_ms`、适用时的 `ack_wait_duration_ms`、`round_instance_id` 和 `round_instance_sequence`。
- 当 `focus_restore_requires_explicit_confirm = true` 时，UI 必须能表现“恢复焦点后需要确认继续”的状态；确认输入由 UI/menu context 消费，不得作为 combat command 进入移动、防御、攻击、气弹或爆气流程。
- 调试显示必须能读取 tick id、runtime state、backlog classification、trace window bounds、最近 event sequence keys、rejected request warnings、catch-up guard decisions、`catch_up_stop_reason`、presentation stale diagnostics 和 bounded trace diagnostics。
- UI/HUD/Debug 只读 combat facts；除明确 runtime request（pause、resume、restart、exit）外，不得写入血量、气槽、硬直、判定、timer、combo 或胜负状态。

## Open Questions

| Question | Owner | Target Resolution | Notes |
|---|---|---|---|
| Godot 4.6.2 中 fixed tick scheduling、accumulator、pause/focus handling 的具体实现方式是什么？ | Runtime ADR / technical-director / godot-specialist | Before runtime implementation starts | GDD 只定义行为；实现必须查 `docs/engine-reference/godot/`，不能靠记忆写 Godot API。 |
| Runtime architecture、fixed tick scheduling、input snapshot、event batch/signal delivery、snapshot immutability、trace/instrumentation、Godot 2D physics boundary、numeric determinism、presentation interpolation、Web audio unlock、testing strategy 是否都已有 ADR？ | Runtime ADR set / technical-director / godot-specialist | Before runtime implementation starts | design-review 指出这些是 implementation gate；GDD 冻结最小合同，ADR 决定 Godot 实现方式。 |
| `catch_up_max_ticks = 2` 和 `catch_up_wall_clock_budget_ms = 3.0` 是否在 Web 导出实测中合适？ | Runtime ADR / performance-analyst | During first browser prototype week | GDD 给出小步追赶默认值和安全范围；最终值需要 Web 性能验证，但不得在未复审情况下回到大步 silent catch-up。 |
| `trace_window_ticks = 180` 是否足够 QA 复现输入、hitstop、focus 和 event ordering 问题？ | QA Lead / Runtime ADR | Before QA instrumentation task | 如 QA 发现 3 秒窗口不足，可在安全范围内提高。 |
| Event batch 的最终字段名、序列化格式和存储位置是什么？ | Runtime ADR / lead-programmer | Before event bus implementation | GDD 只规定必须包含的信息类别和排序 key。 |
| 输入缓冲在 hitstop、pause、countdown、focus restore 中如何过期或清理？ | 输入映射与输入缓冲 GDD | During input-buffering GDD | 本系统只规定 runtime state 和“不得自动触发 held input”。 |
| HUD/VFX/音效如何表现 catch-up 后连续多个 event batches？ | HUD/VFX/Audio GDDs | During presentation GDDs | Runtime 保证顺序和完整性；表现层决定是否压缩展示。 |

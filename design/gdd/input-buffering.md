# 输入映射与输入缓冲

> **Status**: Revised — latest fresh re-review MAJOR REVISION blockers addressed; Pending Fresh Re-review
> **Author**: SteveZhang + Claude Code Game Studios
> **Last Updated**: 2026-05-13
> **Implements Pillar**: 读招定胜负；短连招，高回合；第一回合必须好玩
> **System ID**: input-buffering
> **Priority**: MVP
> **Layer**: Foundation
> **Depends On**: design/gdd/fixed-logic-runtime.md
> **Design Review**: MAJOR REVISION NEEDED 2026-05-13 on first full review; revised same day. Fresh re-review on 2026-05-13 again found MAJOR REVISION NEEDED because input authority, one-shot lifecycle, held guard/direction semantics, UI routing, accessibility, performance measurement, schemas, and QA ownership were still not implementation-ready. Latest fresh re-review on 2026-05-13 again found MAJOR REVISION NEEDED because same-frame high-value command conflict, guard+attack option-select risk, UI context routing, Accessibility profile details, toggle guard lifecycle, player-visible feedback, remap scope, ghosting fallback, and player-experience ACs were not implementation-ready. This revision hardens those contracts and remains pending fresh re-review.

## Overview

输入映射与输入缓冲系统负责把玩家的物理输入（MVP 以 Web 键盘为阻塞标准）转换成固定逻辑运行时可以验证、排序、追踪和消费的权威输入记录、sealed snapshot、战斗候选命令和 runtime/UI request。它定义唯一输入权威管线、Godot/Web 事件采集边界、`pressed` / `held` / `released` / axis 语义、固定 tick 与非推进 context 的 seal 规则、one-shot buffer 生命周期、direction/guard continuous intent、pause/focus/recovery/presentation wait 的安全恢复、UI context 路由、无障碍输入配置、Web 性能预算和 QA 证据要求。该系统不决定命中、防御成功、取消、气槽消耗、硬直或胜负；它只保证玩家意图被公平采集、按确定性规则保留或清理，并在正确的 runtime 边界提交给 fixed runtime 与后续战斗状态机。

## Player Fantasy

玩家应该感觉自己的按键是可靠的：该防住时能防住，该出手时能出手，合理提前按下的攻击、气弹或爆气不会被浏览器帧率、渲染延迟、hitstop 或 UI 焦点变化吞掉。输入缓冲的幻想不是“系统替我自动赢”，而是“我提前读到了，所以我的操作被公平接住”；玩家在短连招、确反、防守切换和起身后重新抢节奏时，应该相信失败来自自己慢了、贪了、资源不够或读错了，而不是因为隐藏输入规则漂移。

第一回合中，输入系统必须降低新玩家的挫败感：方向和防御可以按直觉提前准备，攻击和爆气不会在倒计时或恢复后因为旧按住状态自动偷跑，hitstop 中的合理反应不会被静默清掉，暂停、失焦恢复和菜单确认也不会泄漏成战斗动作。被系统拒绝或清理的输入必须能被 QA 精确追踪；在训练、倒计时、焦点恢复、暂停恢复和爆气不可用这类玩家容易误解的场景中，必须有简短、可读、非颜色唯一、可本地化的提示，让玩家知道“系统收到了，但现在不能执行”。

## Detailed Rules

1. **MVP 输入范围以 Web 键盘为阻塞标准，但键盘不等于固定布局**
   MVP 必须完整支持键盘游玩、键盘 UI 导航、键盘 smoke test 和键盘无障碍配置校验。手柄、触控和 adaptive controller 可以保留抽象接口字段，但不得阻塞 MVP 验收，也不得让键盘规则依赖非键盘特性。非键盘设备若被检测到，只能产生 bounded diagnostics 或显示“当前 MVP 需要键盘”的提示，不能伪装成 `keyboard_primary`。

2. **输入配置必须有 Standard、Alternate 和 Accessibility 三套可玩 profile**
   MVP 至少提供三套 round-start 前可选择、可校验、可 hash、可 QA 复现的键盘 profile：

   | Profile | Purpose | Combat layout | UI layout | Assist behavior | Hash-visible fields |
   |---|---|---|---|---|---|
   | `standard_keyboard_profile` | 默认手感与平衡测试布局。 | Arrow keys = direction；Z = held guard；X = light；C = heavy；V = projectile；B = burst；P = pause。 | Enter = confirm；N = cancel；Arrow keys = UI navigation when UI owns context。 | 无 combat assist。 | key bindings、profile id、same-frame conflict policy、buffer class values。 |
   | `alternate_keyboard_profile` | 避开常见浏览器快捷键、键盘 ghosting 和低端键盘 rollover 风险。 | WASD = direction；J = held guard；K = light；L = heavy；I = projectile；O = burst；P = pause。 | Enter = confirm；N = cancel；WASD 或 Arrow keys 可用于 UI navigation only when UI owns context。 | 无 combat assist；不得扩大 combat buffer。 | key bindings、profile id、ghosting combo set id、same-frame conflict policy、buffer class values。 |
   | `accessibility_keyboard_profile` | 降低同时按键压力和恢复流程认知负担。 | Arrow keys = direction；Space = toggle guard；J = light；K = heavy；L = projectile；U = burst；P = pause。 | Enter = confirm；N = cancel；Arrow keys = UI navigation when UI owns context。 | Toggle guard、较长 UI/prompt 显示时间、明确状态提示；不扩大 combat buffer window。 | key bindings、profile id、toggle guard enabled、toggle state seed/reset rules、accessibility prompt timing knobs。 |

   三套 profile 都必须通过 required combo smoke：direction + guard、direction + light、direction + heavy、direction + projectile、direction + burst、guard + light、guard + burst、pause/menu navigation。若 Standard profile 在目标浏览器/键盘组合上失败，MVP 必须提供实际可玩的 Alternate 或 Accessibility profile 通过同一 combo set；仅显示 warning 不算通过。

   MVP 不包含自由 remapping capture modal。后续 remapping UI 属于下游系统；本 GDD 只要求 profile selection、冲突检测、hash 生成和 deterministic validation。任何 profile 变更都必须发生在 round start 前，increment `input_generation_id`，invalidate stale buffer/request，并禁止在 round 中静默改变。

3. **默认键盘布局必须覆盖战斗与菜单语义**

   | Context | Action | Default key | Notes |
   |---|---|---|---|
   | Combat | `left` | ArrowLeft | Gameplay canvas active 时 suppress browser scroll。 |
   | Combat | `right` | ArrowRight | 同上。 |
   | Combat | `up` | ArrowUp | 用于跳跃/小跳或后续移动系统解释。 |
   | Combat | `down` | ArrowDown | 用于蹲伏/下方向或后续移动系统解释。 |
   | Combat | `guard` | Z | Standard profile 为 held guard；assist profile 可配置 toggle guard。 |
   | Combat | `light_attack` | X | one-shot pressed command。 |
   | Combat | `heavy_attack` | C | one-shot pressed command。 |
   | Combat | `projectile` | V | one-shot pressed command。 |
   | Combat | `burst` | B | one-shot emergency command。 |
   | Runtime/UI | `pause` | P | runtime/UI request。 |
   | UI | `confirm` | Enter | UI/menu context request only。 |
   | UI | `cancel` | N | UI/menu context request only。 |
   | UI | `ui_left` | ArrowLeft | `menu_only` / overlay context 中复用方向键，不产生 combat axis。 |
   | UI | `ui_right` | ArrowRight | 同上。 |
   | UI | `ui_up` | ArrowUp | 同上。 |
   | UI | `ui_down` | ArrowDown | 同上。 |

   默认键位不得依赖 browser/system modifier chord（如 Ctrl/Meta/Alt 组合）。同一 active context 内，一个物理键不得同时绑定多个动作；跨 context 复用必须由 router 隔离。配置加载必须拒绝 duplicate combat mapping、missing required action、reserved-key conflict、priority tie、out-of-range tuning、invalid assist profile 或 context reuse 未声明的配置。

4. **central input router 是唯一输入权威入口**
   所有物理 keydown/keyup、focus/page/fullscreen/canvas focus signal、profile selection event 和 UI navigation event 必须先进入 central input router 并生成或更新权威记录，然后才允许 combat、runtime 或 UI 层消费语义动作。Godot `Control`、pause menu、overlay、debug UI、animation callback、visual frame、`_unhandled_input` handler 或 gameplay node 不得直接根据物理键生成 combat command、runtime request 或 UI selection result。

   行为合同固定如下；具体 Godot API 由后续 ADR 选择：

   - Router 必须在项目可控制的最早输入阶段记录 raw event。
   - Godot `Input.is_action_pressed` / polling 不得用于反向补造过去 tick 的 `pressed`。
   - Polling 只允许在 focus/fullscreen/page restore 后做 `key_state_reconciliation`，且只能清理或阻塞 held state，不能制造新的 `pressed`。
   - `Control` 只能消费 router 发出的 semantic UI action / request，不得消费未记录的 physical key。
   - Text entry、profile selection、browser/system-reserved shortcut 是 MVP 例外路径，但必须进入专门 `text_entry_or_profile_selection` context，并记录 request/decision。
   - Future remapping capture 若后续加入，必须先写入独立 GDD/ADR 或本 GDD 修订；MVP 不允许用未定义 remap modal 作为阻塞验收入口。
   - SceneTree pause 或 Godot process mode 不得停止 router、focus recovery overlay、release cleanup 或 request logging。

5. **Web default suppression 必须按 context 决定**

   | Situation | Browser default policy |
   |---|---|
   | Gameplay canvas active, combat context | Suppress defaults for mapped combat keys, pause key, and arrows. |
   | Pause menu / result menu / overlay focused | Suppress defaults for mapped UI navigation, confirm, cancel, pause, and arrows. |
   | Profile selection menu | Suppress mapped UI navigation, confirm, cancel, pause, and arrows; profile changes are allowed only before round start. |
   | Text-entry field focused | Do not suppress normal text editing keys unless they are explicitly captured by the field adapter. |
   | Browser/system shortcut chord | Do not capture as gameplay; record `reserved_shortcut_blocked` if observed. |
   | Focus/page hidden/fullscreen transition | Do not accept gameplay input; enter focus recovery rules. |

   QA 必须能验证 Arrow keys、Enter、P、N、Z/X/C/V/B、W/A/S/D、J/K/L/I/O、Space 在 gameplay、menu、overlay、profile selection 和 text-entry context 下不会滚动页面、双重提交、触发浏览器 UI 或泄漏成 combat command。Future remap context 不属于 MVP 阻塞验收。

6. **InputSample 是 compact raw edge 的权威记录，不是无限诊断日志**
   每个 gameplay-relevant physical edge 必须生成 compact `InputSample`。`InputSample` 至少包含：`input_schema_version`、`physical_event_sequence_index`、`captured_host_frame_index`、`event_order_in_host_frame`、`captured_monotonic_ms_quantized`、`device_source`、`player_slot_id`、`actor_slot_id_if_bound`、`physical_key_code`、`logical_key_label`、`mapped_action_if_any`、`active_input_context_id`、`event_edge = key_down/key_up/focus_signal/reconcile_signal`、`is_browser_repeat`、`modifier_ctrl`、`modifier_alt`、`modifier_shift`、`modifier_meta`、`is_reserved_shortcut_chord`、`browser_default_policy_decision`、`is_focus_safe_at_capture`、`input_config_hash`、`ui_context_stack_top_id_if_any`、`ui_focus_owner_stable_id_if_any`、`ui_focus_owner_generation_if_any`、`raw_event_diagnostic_flags`。

   Action state 必须由 `InputSample` 和 router-owned physical-key state 推导，不能从 Godot 当前 action polling 反向补造过去 tick 的输入。额外浏览器 diagnostics、repeat bursts 和 high-volume events 只能进入 bounded diagnostic summary，不得扩大权威 payload 到无上限。

7. **canonical serialization / hashing 必须可重复**
   所有权威输入记录的 canonical serialization 必须遵守：

   - 字段顺序使用本 GDD schema 顺序，不使用 Godot `Dictionary` 迭代顺序。
   - Enum 使用 registry-stable string 或 integer code；同一 build 内不得混用。
   - Array 使用明确排序：action 使用 canonical action order，records 使用 sequence id 升序。
   - Optional field 缺失编码为 explicit `null` / `none`，不得省略导致 hash 差异。
   - 不编码 live Node reference、Godot object id、scene tree order 或非稳定 NodePath 作为权威字段。
   - 时间使用 fixture-provided integer tick/frame/quantized ms；不得使用浮点 wall-clock 参与 deterministic hash。
   - Unknown schema version、unknown enum 或字段缺失必须 fail closed。

   正式实现前可由 ADR 选择 binary、JSON 或 Resource 序列化格式，但不得改变以上 canonical 行为。

8. **每个 target 使用统一 seal boundary；request 与 combat 不分裂封口**
   Runtime 在消费任何 combat tick 或非推进 UI/runtime context 前，必须调用一次 input seal。Seal 开始时记录 `seal_cutoff_physical_event_sequence_index = latest_sample_sequence_index_arrived_before_seal_start`；本次 snapshot 只包含 sequence `<= cutoff` 且尚未被更早 seal 消费的 samples。Seal 开始后到达的 samples 必须进入后续 target，不能补写当前 snapshot。

   同一个 sealed snapshot 同时产生 request branch 和 combat branch。消费顺序固定为：

   1. Build `InputSnapshot`。
   2. Route runtime/UI requests。
   3. 若 request 被接受并改变 runtime/UI context，按规则清理或阻塞同 snapshot combat input。
   4. 若 request rejected/no-op 且 combat policy 允许，再评估 combat branch。

   这保证 pause+attack、confirm+held guard、overlay+menu 等情况只有一个权威排序。

9. **catch-up 禁止 retroactive input read**
   catch-up 时，runtime 不得把当前 host frame 新收到的输入 retroactively 套用到历史 delayed tick。若 fixed runtime 正在同一 host frame 内处理多个 delayed ticks：

   - 已存在的 sealed snapshot 按自己的 target tick 使用。
   - 缺失历史 snapshot 的 delayed tick 生成 `no_new_physical_input` snapshot。
   - `no_new_physical_input` 只能继承 safe continuous state：same round、same `input_generation_id`、same `interruption_epoch`、same config hash、not pending-release、focus-safe、combat policy 允许 continuous、且来自最近一个已 sealed authoritative snapshot 的 resolved direction/guard held。
   - `no_new_physical_input` 不得产生 `pressed`、`released`、one-shot buffer entry、runtime request 或 UI navigation。
   - 任一继承条件不满足时，direction 输出 neutral，guard 输出 false，并记录 `catch_up_continuous_state_dropped`。
   - 当前 host frame 新到达的 physical events 只能封存到 catch-up batch 之后的第一个 future realtime target。

10. **非推进状态使用 context snapshot，不伪造 committed tick**
    countdown、player pause、focus_suspended、recovery_pause、automatic `presentation_ack_wait`、player recovery prompt、resume countdown 和 round_ended 中，如果 fixed runtime 没有正在提交 combat tick，snapshot 必须使用 `target_committed_tick_index = null`，并记录 `target_context_sequence_id`、`runtime_state_at_capture`、`tick_execution_state_at_capture = none`、`post_commit_runtime_state_at_capture = runtime_state_at_capture`、`state_reason` / `pause_reason`、`combat_input_policy_at_capture`。不得用 `current_committed_tick_index + 1` 伪造目标 tick。

11. **InputSnapshot schema 必须包含 actor、context 和 trace ownership**
    每个 `InputSnapshot` 至少包含：`input_schema_version`、`input_snapshot_id`、`round_instance_id`、`round_instance_sequence`、`player_slot_id`、`actor_runtime_id_if_bound`、`input_generation_id`、`interruption_epoch`、`input_config_hash`、`device_source`、`target_committed_tick_index`、`target_running_tick_index_if_applicable`、`target_context_sequence_id_if_applicable`、`captured_host_frame_index`、`input_seal_sequence_index`、`seal_cutoff_physical_event_sequence_index`、`runtime_state_at_capture`、`tick_execution_state_at_capture`、`post_commit_runtime_state_at_capture`、`state_reason`、`pause_reason`、`combat_input_policy_at_capture`、`top_ui_context_id_if_any`、`ui_context_generation_if_any`、canonical action order、每个 action 的 resolved `pressed` / `held` / `released` / `axis`、raw held bitset、combat-visible held bitset、pending-release bitset、generated `InputBufferEntry` ids、generated `InputDecisionRecord` ids、generated `InputRequestRecord` ids。

12. **UI context stack 必须使用稳定数据，不使用 live Control 当权威**
    每个 UI context 至少有：`ui_context_id`、`ui_context_type`、`ui_context_generation`、`priority_layer`、`modal_flag`、`parent_context_id_if_any`、`focus_owner_stable_id_if_any`、`focus_owner_generation_if_any`、`opened_at_context_sequence_id`、`closed_at_context_sequence_id_if_any`、`accepts_actions`、`pass_through_actions`、`captures_actions`、`combat_input_policy_override`。Top context 由 `(priority_layer desc, modal_flag desc, opened_at_context_sequence_id desc, ui_context_id asc)` 确定。

    MVP routing 默认 **不 fallthrough**：topmost modal context 阻止所有 lower context；topmost non-modal context 只有在 action 明确列入 `pass_through_actions` 时才允许 lower context 处理。HUD/toast/prompt 若不可交互，不得注册为 capturing top context。Parent context 关闭时，其 child contexts 必须同一 `context_sequence_id` 内关闭，并记录 close reason；同帧 physical event 与 context open/close 的排序使用 `context_sequence_id` 升序和 `physical_event_sequence_index` 升序，不依赖 scene tree order。

    Stable focus owner 必须来自 UI control registry，而不是 live Node reference。每个 registered control 至少有：`control_stable_id`、`control_role`、`control_generation`、`owning_ui_context_id`、`visible`、`enabled`、`focusable`、`accepts_actions`、`semantic_command_if_confirmed`。Control 被销毁、隐藏、禁用、generation mismatch 或离开 owning context 时，请求必须在 request dispatch 前 fail closed 为 `stale_ui_context_or_focus_owner`，不得落到下层 menu 或 combat。

    UI context/control record stream 必须可重放。每个 `UIContextRecord` 至少包含：`ui_context_record_id`、`event_type = opened/closed/focus_changed/control_registered/control_unregistered/context_updated`、`ui_context_id`、`ui_context_generation`、`context_sequence_id`、`priority_layer`、`modal_flag`、`parent_context_id_if_any`、`focus_owner_stable_id_if_any`、`focus_owner_generation_if_any`、`control_stable_id_if_any`、`control_generation_if_any`、`reason`、`resulting_top_context_id_if_any`。这些 records 参与 canonical hash，并且必须先于引用它们的 `InputRequestRecord` 出现在同一 trace stream 中。

13. **InputRequestRecord schema 必须可重放和可验收**
    每个 runtime/UI request 至少包含：`input_request_record_id`、`request_type = pause/confirm/cancel/ui_navigation/quick_restart/exit_to_menu/profile_select/no_op`、`source_action`、`source_snapshot_id`、`physical_event_sequence_index`、`request_boundary_sequence_id`、`player_slot_id`、`top_ui_context_id_if_any`、`top_ui_context_type_if_any`、`ui_context_generation_if_any`、`focus_owner_stable_id_if_any`、`focus_owner_generation_if_any`、`input_generation_id`、`interruption_epoch`、`idempotency_key`、`acceptance_result = accepted/rejected/no_op/consumed`、`consumed_by_context_id_if_any`、`consumed_by_control_stable_id_if_any`、`semantic_command_if_any`、`reject_or_noop_reason_if_any`、`clears_combat_input = true/false`。

    Request result 语义固定如下：`accepted` 表示 runtime state 或 persistent UI state 被该 request 改变；`consumed` 表示 focused UI control 使用该 action 但不一定改变 runtime state；`no_op` 表示 action 在当前 state 合法但没有效果，例如 already paused 的 pause；`rejected` 表示 action 不被当前 context 接受。Quick restart / exit 使用二阶段记录：confirm 先被 focused control `consumed`，随后同一 boundary 生成 `quick_restart` 或 `exit_to_menu` semantic request 并 `accepted`。危险 semantic request 的 `idempotency_key` 必须包含 focused control id、control generation、source snapshot id 和 physical event sequence；browser repeat 或 held confirm 不得重复触发。

14. **InputBufferEntry schema 必须区分 press attempt 与 command intent**
    每个 one-shot combat press 生成一个 `buffer_attempt_id`。每个 pending entry 至少包含：`input_buffer_entry_id`、`buffer_attempt_id`、`command_intent_id`、`player_slot_id`、`actor_runtime_id`、`action`、`exclusive_combat_command_group`、`created_from_snapshot_id`、`created_from_physical_event_sequence_index`、`created_at_committed_tick_index_if_any`、`created_at_running_tick_index`、`created_at_context_sequence_id_if_any`、`created_at_monotonic_ms_quantized`、`first_legal_running_tick_index_if_deferred`、`input_generation_id`、`interruption_epoch`、`input_config_hash`、`priority_value`、`buffer_lifecycle_state`、`last_legality_query_result_if_any`、`last_evaluated_at_running_tick_index_if_any`。

    `buffer_attempt_id` 永远不复用；duplicate press 不是刷新旧 attempt，而是创建新 attempt，并按 lifecycle 规则 supersede 旧 attempt。

15. **InputDecisionRecord 必须覆盖 buffer 和 no-buffer 结果**
    每个 consumed、rejected、expired、superseded、invalidated、capacity-evicted 的 buffer entry 必须记录 lifecycle。每个没有创建 buffer entry 的关键输入也必须记录 `InputDecisionRecord`：key repeat ignored/merged、countdown action blocked、opposite-axis neutralized、runtime request rejected、pending-release blocked、same-snapshot request clearing combat input、stale snapshot rejected、focus unsafe blocked、round ended blocked、transient press/release folded、future_remap_capture_blocked、ghosting fallback required、trace overflow summary。

16. **pressed / held / released 语义必须稳定且按动作类型解释**
    `pressed` 只在从 physical up 到 down 的第一个 sealed snapshot 为 true；持续按住只产生 `held`；松开产生 `released`。浏览器 repeat keydown 不得制造多个 `pressed`。攻击、气弹和爆气只由 `pressed` 创建 one-shot buffer entry；持续 `held` 不会自动创建重复攻击、重复气弹或重复爆气。

    Direction 和 guard 是 continuous intent：每个允许的 running tick 重新解析当前 combat-visible held state。Guard 在 Standard / Alternate profile 中是 held intent；在 Accessibility profile 中是 deterministic toggle guard，toggle state 必须进入 snapshot/hash，并只能在 round start 前选择。Toggle guard 初始状态每个 new round 强制 `off`；round restart、quick restart、exit_to_menu、profile change、focus_suspended unsafe recovery begin、player recovery prompt begin 会清空为 `off` 并记录 `toggle_guard_cleared_by_interruption`。Browser repeat 不得重复 toggle；UI/menu/profile context 中 guard key 不改变 combat toggle state。

17. **方向、防御和 one-shot 动作分层处理**
    同一 snapshot 内先解析 continuous state，再解析 one-shot command：

    - Direction layer：`left/right/up/down` 解析为 resolved axis；左右或上下同时 held 时该轴 neutral。
    - Guard layer：`guard` 解析为 continuous guard intent；是否成为实际 guard state 由 combat state machine 在合法 running tick 判断。
    - One-shot command layer：`burst`、`light_attack`、`heavy_attack`、`projectile` 只在 `pressed` 时创建 candidate buffer entry。
    - Runtime/UI request layer：`pause`、`confirm`、`cancel`、`ui_*` 使用 request/context routing，不参与 combat priority。

    方向 held 不得因为同 tick one-shot action 而消失；one-shot action 也不得被 movement held 自动压掉。Guard held 与 one-shot attack 同 tick 时不得形成 option select：若 `light_attack`、`heavy_attack` 或 `projectile` candidate 被选中，输入系统必须在 candidate command 上标记 `guard_suppressed_by_attack_attempt = true`，并向下游提交 `ordinary_guard_intent_suppressed` decision。下游 combat state machine / guard GDD 必须保证同一 vulnerability window 内玩家不能同时获得普通防御和攻击尝试的最优结果；除非未来系统显式设计 guard-cancel、reversal 或 burst 例外，否则攻击尝试视为放弃普通 held guard。

18. **same-snapshot 高价值命令严格；跨 tick 使用最新意图优先**
    只有 one-shot combat command 进入同一 `exclusive_combat_command_group`。MVP 同 snapshot 规则分两层：

    - `burst` 是 high-value emergency command，必须是该 actor、该 snapshot、该 group 内唯一 one-shot 才能创建 valid pending entry。
    - 若 `burst` 与 `light_attack`、`heavy_attack` 或 `projectile` 在同一 snapshot 同组出现，全部同组 one-shot entries 立即标记 `ambiguous_high_value_conflict`，不提交、不 fallback、不消耗气槽；玩家必须 fresh press 表达明确意图。
    - 若同 snapshot 只有 non-burst one-shot 冲突，MVP 使用固定优先级 `light_attack > heavy_attack > projectile`；较低优先级 entry 标记 `superseded_by_higher_priority_same_snapshot`。

    跨 snapshot / 跨 running tick 评估 pending entries 时，选择键为：

    `(created_at_running_tick_index_desc, created_from_snapshot_id_desc, priority_value_desc, input_buffer_entry_id_desc)`

    也就是较新的有效 press attempt 优先于较旧 attempt；priority 只在同一时间层级内打平。若较新的 selected attempt terminal reject，不自动 fallback 到旧 attempt；同 group 旧 pending entries 标记 `superseded_by_newer_attempt` 并清理，玩家需要 fresh press 表达新意图。continuous direction 不因 one-shot reject 被删除；guard 另受 attack attempt suppression 规则约束。

19. **输入系统只拥有意图生命周期，不拥有完整战斗合法性**
    输入系统可以做 schema、context、generation、buffer age、capacity、priority、pending-release、trace cap 和 obvious-policy 检查；攻击是否可取消、是否有气槽、是否 projectile lockout、是否处于受击/硬直、是否命中或防住，由 fixed runtime / combat state machine 负责。若输入系统需要判断 entry 是否 `retain_until_legal`，必须调用 combat state machine 暴露的 deterministic read-only legality query。

    Query contract 至少返回：`legal_now`、`retain_until_legal`、`terminal_reject`、`unknown_blocked`，并附带 reason enum。`unknown_blocked` 在 MVP 中 fail closed 为 `retain_query_unavailable`，不得由输入系统猜测状态机规则。

20. **one-shot lifecycle 是显式状态机**

    | Lifecycle state/result | Meaning | Buffer behavior |
    |---|---|---|
    | `created` | press attempt 被采集并通过 context/generation/schema 检查 | 进入 pending。 |
    | `retained_until_legal` | 当前还没到可行动窗口，但 age 仍有效 | 保留到下一 running tick 再评估。 |
    | `selected_for_submission` | 在本 tick arbitration 中被选中 | 提交 candidate command。 |
    | `terminal_reject` | 资源不足、状态禁止、round ended、projectile lockout、burst unavailable 等不会在该 entry 生命周期内自动变合法 | 立即消费并记录，不 retry。 |
    | `superseded_by_higher_priority_same_snapshot` | 同 snapshot 被更高优先级 one-shot 覆盖 | 清理，不提交 runtime。 |
    | `superseded_by_newer_attempt` | 跨 tick 被较新的 valid attempt 覆盖 | 清理，不 fallback。 |
    | `consume_without_action` | pause accepted、UI consumed、policy clearing 等 | 清理并记录。 |
    | `expired_by_running_age` | running tick age 超过窗口 | 清理并记录。 |
    | `expired_by_hitstop_realtime_cap` | hitstop 中保留过久，超过实时上限 | 清理并记录。 |
    | `invalidated_by_interruption` | pause/focus/recovery/ack wait/quick restart 等 unsafe boundary 使旧 entry 不安全 | 清理并记录。 |
    | `capacity_evicted` | 超过 per-actor entry cap | 按 deterministic ordering 清理并记录。 |

21. **缓冲时间按 running ticks 计算，但 hitstop 还必须有 real-time cap**
    one-shot buffer entry 使用 running tick age。running tick age 只在 `tick_execution_state = running` 的 committed tick 后推进；hitstop tick、pause、focus_suspended、countdown、recovery_pause、automatic `presentation_ack_wait`、player recovery prompt 和 round_ended 不推进 running age。

    hitstop 中新创建的 one-shot buffer entry 使用 `created_at_running_tick_index = first_legal_running_tick_index_after_hitstop`，因此第一枚恢复 running tick 上 `buffer_age_running_ticks = 0`。同时必须检查 `hitstop_entry_realtime_age_ms = first_legal_running_tick_monotonic_ms_quantized - created_at_monotonic_ms_quantized`；超过 `hitstop_buffer_realtime_cap_ms` 的 entry 过期为 `expired_by_hitstop_realtime_cap`，避免超长 hitstop 把很久以前的输入变成“刚按”。

22. **hitstop 保留战斗意图，但不执行战斗命令**
    Hitstop tick 可以采集 direction、guard 和 one-shot input，并把合法 one-shot buffer entry 排到第一个后续 running tick 检查。Hitstop 中不得执行 movement、attack、projectile 或 burst command。Hitstop 中完整 press→release 的 one-shot tap 仍可创建 buffer entry，并在 first legal running tick 评估；release 不取消该 pressed entry，除非后续系统明确设计 charge / hold-cancel 机制。

23. **unsafe interruption 同时使用 generation 与 epoch，禁止 OR 语义**
    `input_generation_id` 用于 stale rejection；任何会改变输入解释的事件都必须 increment：unsafe interruption、round restart、quick restart、config/profile change、future remap change、focus recovery begin、UI context hard reset。

    `interruption_epoch` 用于 combat-visible held recovery；任何会让旧 held combat input 不安全的事件都必须 increment：player pause enter、focus_suspended enter、recovery_pause enter、player recovery prompt enter、visible automatic `presentation_ack_wait` safe-neutral recovery、quick restart、round restart。

    进入上述 unsafe boundary 时，输入系统必须：

    - increment `input_generation_id`；
    - increment `interruption_epoch` if combat-visible held state must be release-gated；
    - 将所有当前 combat-held actions 标记为 `pending_release`；
    - invalidate 所有尚未提交的 one-shot combat buffer entries；
    - 清空 pending runtime/UI requests，除非 request 被当前 top UI context 明确消费；
    - 记录 lifecycle 与 decision records。

24. **toggle guard 使用独立状态机，不改变 Standard combat timing**
    Accessibility profile 的 toggle guard state 至少为：`off`、`on`、`pending_clear_by_interruption`、`disabled_by_context`。Toggle key 在 `running` 和允许 combat continuous input 的 `hitstop` 中可切换 `off/on`；在 round-start countdown 中可预读为 GO 后的 initial guard intent，但不产生 one-shot；在 resume countdown 中只有当 pending-release 已清理且 context 明确为 combat pre-read 时才允许恢复为 `on`。Pause menu、result menu、profile selection、focus recovery overlay、player recovery prompt、automatic presentation ack wait、round_ended 和 future remap context 中，toggle key 不改变 combat toggle state。Toggle state 变化必须产生 `InputDecisionRecord` 和 player-visible state feedback；它不得扩大 `normal_attack_buffer_ticks`、`projectile_buffer_ticks` 或 `burst_buffer_ticks`。

25. **pending_release 使用物理键状态机，不能让玩家永久卡死**
    每个 combat action 的 physical key state 至少为：`trusted_up`、`trusted_down`、`pending_release_unknown`、`pending_release_observed_down`、`reconciled_up_blocked_press`。恢复后：

    - 旧 held combat action 在 `pending_release_*` 状态下对 combat 输出 blocked。
    - UI `confirm` 不会把旧 combat held 重新确认为 pressed。
    - 若 post-restore 收到可信 keyup，则清除 pending release，状态回到 `trusted_up`。
    - 若 focus restore reconciliation 能证明该 physical key 当前为 up，则清除 pending release，记录 `pending_release_cleared_by_reconciliation`；该清理不得产生 `released` 或 `pressed` combat edge。
    - 若平台无法证明 key up，且玩家下一次 keydown 到达同一 pending key，则该 keydown 只清理 pending release 为 `reconciled_up_blocked_press`，不创建 combat `pressed`；需要后续 keyup→fresh keydown 才能产生 combat input。
    - UI 必须显示可读提示，例如 localization key `input.release_keys_to_resume`，并且提示不能只靠颜色表达。

26. **round-start countdown 允许方向和防御预读，不允许 one-shot 预存**
    `countdown_direction_pre_read_enabled = true` 时，倒计时期间允许 `left/right/up/down` 记录为 `countdown_direction_pre_read`，用于 `go` 后第一枚 running tick 的朝向或移动准备。Guard 在 round-start countdown 中允许作为 continuous guard pre-read：若玩家在 `go` 时仍物理 held 且不处于 pending-release，则第一枚 running tick 可输出 guard held intent；它不产生 `pressed` edge，不创建 one-shot，不触发攻击，不消耗资源。

    `light_attack`、`heavy_attack`、`projectile`、`burst` 不得在 countdown 中预存成开局攻击或开局爆气；若玩家一直 held 到 `go` 后，仍不产生 one-shot `pressed`，必须 release 后重新 press。玩家可见提示应说明“移动/防御可提前按；攻击/爆气从 GO 后重新按”。

27. **resume countdown 比 round-start countdown 更保守**
    从 focus/pause/recovery/presentation wait 恢复时，如果 fixed runtime 进入 resume countdown，则 direction 只有在来源为 combat context、不是 menu/overlay navigation、且 pending-release 已清理时才可 pre-read；由 pause menu、result menu、profile selection、focus overlay 或 recovery prompt 中 held Arrow/WASD 产生的 UI navigation 不得泄漏成恢复第一帧 movement。Guard 只有在 pending-release 已被可信清理且当前 physical state 明确为 down 时才可作为 continuous guard intent；Accessibility toggle guard 只有在 context 明确允许 combat pre-read 时才可恢复。旧 held guard 不得因为 resume confirm 自动恢复为 guard。One-shot actions 仍必须 release+fresh press 后才可创建 `pressed`。

28. **runtime/UI request 使用 topmost context 单次消费**

    | Runtime / UI context | `pause` | `confirm` | `cancel` | `ui_*` | Combat input policy |
    |---|---|---|---|---|---|
    | `running`, no modal UI | accepted as player pause request at request boundary | rejected unless prompt owns it | rejected unless prompt owns it | rejected unless prompt owns it | `combat_execute_allowed` |
    | `hitstop` | accepted after current hitstop-safe boundary | rejected unless prompt owns it | rejected unless prompt owns it | rejected unless prompt owns it | `capture_only` |
    | round-start countdown | accepted as player pause request | rejected unless prompt owns it | rejected unless prompt owns it | rejected unless prompt owns it | `direction_guard_pre_read_only` |
    | player pause menu focused | no-op if already paused | consumed by focused menu control | consumed by focused menu/back control | consumed by menu navigation | `menu_only` |
    | focus recovery overlay focused | rejected/no-op unless overlay owns pause | consumed by overlay to begin resume flow | consumed only if UX allows staying paused/menu | consumed by overlay if it has selectable controls | `resume_confirm_only` |
    | `recovery_pause` player prompt | rejected unless UI allows menu open | consumed by prompt / resume flow | consumed only if UX allows abort/menu | consumed by prompt if selectable | `resume_confirm_only` |
    | automatic `presentation_ack_wait` without player prompt | rejected/no-op | rejected; confirm never satisfies display watermark | rejected/no-op | rejected/no-op | `blocked_all` or runtime-defined capture block |
    | `focus_suspended` unsafe | no gameplay request accepted | no gameplay request accepted | no gameplay request accepted | no gameplay request accepted | `blocked_all` |
    | `round_ended` result/menu | routed to result/menu UI only | consumed by result/menu UI | consumed by result/menu UI | consumed by result/menu navigation | `menu_only` |
    | profile selection menu | no-op unless menu owns it | consumed by focused profile/control | consumed by profile/menu back control | consumed by profile/menu navigation | `menu_only` |

    Quick restart and exit are not default global hotkeys in MVP. They are semantic requests produced by a focused menu/result/debug control after `confirm`, with `request_type = quick_restart` or `exit_to_menu`. They must still generate `InputRequestRecord` and unsafe-boundary cleanup. Dangerous controls must be protected from accidental held/repeat confirm: newly opened result/pause menus must ignore the first confirm if it originates from a held key or browser repeat, dangerous controls cannot be the default focus unless the menu is a dedicated result screen, and repeated Enter within `dangerous_request_repeat_lockout_ms` must be rejected as duplicate idempotency.

29. **automatic presentation ack wait 与 player recovery prompt 必须分开**
    Automatic `presentation_ack_wait` 是 runtime/presentation watermark gate；玩家按 `confirm` 不能满足显示水位、不能推进 runtime ack、不能把等待变成 resume。只有显式 player recovery prompt 或 focus recovery overlay 可以拥有 `confirm`，并且它们消费 confirm 后只进入 resume flow，不产生 combat input。

30. **stale snapshot / stale generation 必须 fail closed**
    如果 input snapshot、buffer entry、request record、decision record 或 command 的 `round_instance_id`、`round_instance_sequence`、`player_slot_id`、`actor_runtime_id`、`input_generation_id`、`interruption_epoch`、`input_config_hash`、UI context generation 或 target tick/context sequence 与当前 runtime 不匹配，必须拒绝为 `stale_*` 具体 reason。旧回合、旧焦点世代、旧暂停恢复前、旧 quick restart 前、旧配置下的输入不得污染当前回合。

31. **trace 必须 bounded，且权威记录与诊断记录分离**
    MVP 默认 `input_trace_mode = minimal_export`。权威 compact records 用于 gameplay correctness；diagnostic records 用于 QA/debug。QA / dev 可以启用 `qa_bounded` 或 `dev_verbose`，但 per snapshot、per tick、per trace window 和 total bytes 都必须有上限。超限时不得丢失权威 gameplay input；必须写入 bounded summary record，包含 configured limit、observed count、summarized count、dropped_diagnostic_count、`authoritative_input_loss = false`。输入 trace 是 QA/debug 工具，不是 MVP formal replay 系统。

32. **debug overlay 是 UI context，不是后门输入通道**
    Debug/tuning overlay 若在 dev build 中存在，必须注册为 UI context，使用 request routing，不能直接读 physical keys 改 combat state。Exported MVP build 默认不显示 dev overlay。Debug hotkey 若存在，必须在 config 中声明、写入 hash、记录 request，并且不能绕过 pause/focus/pending-release cleanup。

33. **玩家可见反馈必须把内部 reason 转成可理解结果**
    输入系统必须提供 player-visible feedback matrix。每个内部 reason 至少映射到：`silent`、`training_history_only`、`light_hud_hint`、`blocking_overlay_prompt`、`menu_message` 之一，并记录 localization key、触发条件、最小显示时间、重复冷却、优先级、是否可被覆盖、是否允许在 combat 中显示。MVP 最小映射：`ambiguous_high_value_conflict` → light HUD / training history；`countdown_action_blocked` → light HUD；`pending_release` → blocking overlay；`reconciled_up_blocked_press` → blocking overlay with “再按一次”说明；`burst_unavailable` → light HUD / training history；`superseded_by_newer_attempt` → training history；`stale_ui_context_or_focus_owner` → silent + diagnostics；`menu_owns_input` → first-time menu hint only。玩家-facing 文案不得显示内部 enum。

34. **Prompt 和视觉无障碍必须有客观验收标准**
    所有 recovery、countdown、profile、toggle guard、ghosting fallback 和 dangerous request 提示必须可本地化、非颜色唯一、支持 200% UI scale，在 1280×720 Web canvas 内不被 HUD 裁切。文本对比目标至少 4.5:1，非文本 focus indicator / icon 对比至少 3:1；MVP 字号不得低于 18px 等效高度；图标必须配文字；状态不能只靠闪烁或颜色表达。Accessibility profile 可使用 `accessibility_prompt_min_visible_ms` 和 `accessibility_prompt_repeat_cooldown_ms` 增强可读性，但不得扩大 combat buffer window。

35. **Keyboard-only Web flow 是 MVP 阻塞体验**
    从页面加载开始，玩家必须能不使用鼠标完成：获得 game/canvas keyboard focus、选择 Standard/Alternate/Accessibility profile、开始回合、暂停、恢复、查看 release-keys 提示、完成 quick restart 或 exit menu。若浏览器要求 click/tap 才能 audio unlock 或 fullscreen，UI 必须提供 keyboard-reachable fallback state 和明确提示；不得把“需要鼠标点击画面”作为键盘 MVP 的隐藏前提。

## Formulas

1. **输入缓冲年龄**

   `buffer_age_running_ticks = evaluated_at_running_tick_index - created_at_running_tick_index`

   | Variable | Definition | Valid Range |
   |---|---|---|
   | `created_at_running_tick_index` | buffer entry 开始按 running tick 计龄的 tick。正常 running 中为创建 entry 的 running tick；hitstop 中创建的 entry 使用 `first_legal_running_tick_index_after_hitstop`。 | integer `>= 0` |
   | `evaluated_at_running_tick_index` | 本次尝试消费、保留或过期 entry 的 running tick。 | integer `>= 0` |
   | `buffer_age_running_ticks` | entry 已经过的 running tick 数。 | integer；必须 `>= 0` 才可进入有效性判断 |

   若 `buffer_age_running_ticks < 0`，entry 必须被拒绝为 `future_or_corrupt_buffer_age`，不得因为 `age <= buffer_ticks` 被判为有效。

   Example：`created_at_running_tick_index = 120`，`evaluated_at_running_tick_index = 124`，则 `buffer_age_running_ticks = 4`。

2. **动作缓冲有效性按动作类型分层**

   `is_action_buffer_valid = 0 <= buffer_age_running_ticks <= max_buffer_ticks_for_action_class(action)`

   | Variable | Definition | Default | Safe Range |
   |---|---|---:|---:|
   | `normal_attack_buffer_ticks` | `light_attack`、`heavy_attack` 的 one-shot pressed buffer window。 | 6 | 3-8 |
   | `projectile_buffer_ticks` | `projectile` 的 one-shot pressed buffer window；比普通攻击更短，避免远程牵制被过度自动化。 | 4 | 2-6 |
   | `burst_buffer_ticks` | `burst` 的 one-shot pressed buffer window；高价值反杀命令必须更严格。 | 3 | 2-4 |

   在 60Hz running tick 下：

   - `normal_attack_buffer_ticks = 6` 约等于 `100ms`。
   - `projectile_buffer_ticks = 4` 约等于 `66.7ms`。
   - `burst_buffer_ticks = 3` 约等于 `50ms`。

   Examples：

   - light attack age `0`、`6`：valid；age `7`：expired。
   - projectile age `4`：valid；age `5`：expired。
   - burst age `3`：valid；age `4`：expired。
   - age `-1`：reject `future_or_corrupt_buffer_age`。

3. **方向/防御 transition grace 有效性**

   `is_direction_guard_transition_grace_valid = 0 <= buffer_age_running_ticks <= direction_guard_buffer_ticks`

   | Variable | Definition | Default | Safe Range |
   |---|---|---:|---:|
   | `direction_guard_buffer_ticks` | direction/guard pressed-edge transition grace；continuous held direction/guard 不靠该值长期保留。 | 3 | 1-4 |

   在 60Hz running tick 下，3 tick 约等于 50ms：

   `3 / 60 = 0.05 seconds = 50ms`

   Examples：

   - guard pressed 2 running ticks before guard becomes legal：transition grace valid。
   - guard held 30 running ticks before incoming attack：不是靠 3-tick buffer 保存，而是 continuous held intent 由 combat state machine 判断。
   - age `4`：transition grace expired，但当前 held guard intent 仍可在合法 running tick 被状态机读取。

4. **hitstop realtime cap**

   `hitstop_entry_realtime_age_ms = first_legal_running_tick_monotonic_ms_quantized - created_at_monotonic_ms_quantized`

   `is_hitstop_entry_realtime_valid = 0 <= hitstop_entry_realtime_age_ms <= hitstop_buffer_realtime_cap_ms`

   | Variable | Definition | Default | Safe Range |
   |---|---|---:|---:|
   | `hitstop_buffer_realtime_cap_ms` | one-shot entry 在 hitstop 中允许保留的最大真实时间。 | 150ms | 66-200ms |
   | `created_at_monotonic_ms_quantized` | InputSample 创建时的整数毫秒时间，由 fixture 或 runtime 提供。 | required | integer `>= 0` |
   | `first_legal_running_tick_monotonic_ms_quantized` | 第一个可评估 running tick 对应的整数毫秒。 | required | integer `>= created` |

   Examples：

   - `first_legal = 1050ms`、`created = 960ms`、cap `150ms`：age `90ms`，valid。
   - `first_legal = 1200ms`、`created = 960ms`、cap `150ms`：age `240ms`，`expired_by_hitstop_realtime_cap`。
   - `first_legal < created`：reject `future_or_corrupt_hitstop_time`。

5. **runtime request 不进入 combat buffer**

   `runtime_request_buffer_ticks = 0`

   `pause`、`confirm`、`cancel`、`ui_*`、menu-produced quick restart / exit 不创建 combat `InputBufferEntry`。若当前 context 不接受 request，结果立即为 rejected/no-op；不得延迟到后续 combat tick 自动执行。

6. **同 snapshot one-shot conflict value**

   `has_ambiguous_high_value_conflict = burst_pressed && non_burst_one_shot_count > 0`

   `selected_same_snapshot_non_burst_action = max_by(input_priority_value(action), action_ordinal_tiebreaker)` only when `has_ambiguous_high_value_conflict = false` and no burst is present.

   | One-shot Action | Priority | Ordinal tiebreaker | Same-snapshot rule |
   |---|---:|---:|---|
   | `burst` | 400 | 10 | Valid only if it is the only one-shot in the exclusive group. |
   | `light_attack` | 300 | 20 | Can beat heavy/projectile only when burst is absent. |
   | `heavy_attack` | 200 | 30 | Can beat projectile only when burst is absent. |
   | `projectile` | 100 | 40 | Lowest non-burst priority. |

   Equal priority values are invalid configuration and must fail before combat starts. Direction and guard are not in this one-shot priority table. If `has_ambiguous_high_value_conflict = true`, every same-group one-shot entry is rejected as `ambiguous_high_value_conflict`; no priority selection or fallback occurs.

7. **跨 tick one-shot arbitration**

   `selected_pending_entry = max_by(created_at_running_tick_index, created_from_snapshot_id, priority_value, input_buffer_entry_id)`

   | Variable | Definition | Valid Range |
   |---|---|---|
   | `created_at_running_tick_index` | entry 计龄起点；较大表示更新 intent。 | integer `>= 0` |
   | `created_from_snapshot_id` | 产生 entry 的 snapshot sequence；较大表示更新 seal。 | integer `>= 0` |
   | `priority_value` | 同时间层级内 one-shot priority。 | 100/200/300/400 |
   | `input_buffer_entry_id` | deterministic sequence id；较大表示同条件下更新记录。 | integer `>= 0` |

   Example：running tick 100 有 pending burst，tick 102 有 pending light attack，两者都有效；tick 102 的 light attack 是更新 intent，优先于 tick 100 的 burst。若 light terminal reject，不 fallback 到旧 burst。

8. **buffer 剩余时间**

   `remaining_buffer_ticks_raw = max_buffer_ticks_for_entry - buffer_age_running_ticks`

   `remaining_buffer_ticks_clamped = max(0, remaining_buffer_ticks_raw)`

   | Variable | Definition | Valid Range |
   |---|---|---|
   | `max_buffer_ticks_for_entry` | entry 所属类型的 max buffer window。 | normal attack: 3-8；projectile: 2-6；burst: 2-4；direction/guard transition: 1-4 |
   | `remaining_buffer_ticks_raw` | QA/debug raw value；expired 后可为负。 | integer |
   | `remaining_buffer_ticks_clamped` | UI/debug 可显示值；不为负。 | integer `>= 0` |

   Example：`max = 6`，age `7`：`raw = -1`、`clamped = 0`、result `expired_by_running_age`。

9. **capacity eviction ordering**

   当 pending entries 超过 `max_buffer_entries_per_actor` 时，先移除 expired/invalidated/superseded entries；若仍超限，按以下 deterministic key 升序 eviction：

   `(priority_value_ascending, created_at_running_tick_index_ascending, created_from_snapshot_id_ascending, input_buffer_entry_id_ascending)`

   低优先级、较旧、sequence 更小的 entry 先被清理。新 entry 只有在按同一排序排在最前时才可被清理，且必须记录 `capacity_evicted`。

10. **trace cap summary**

   `diagnostic_overflow_count = max(0, observed_diagnostic_records - max_input_diagnostics_per_tick)`

   `trace_window_overflow_bytes = max(0, projected_trace_window_bytes - max_input_trace_memory_bytes)`

   若任一 overflow `> 0`，写入 bounded summary record，而不是无限扩展 trace。

   | Variable | Definition | Default | Safe Range |
   |---|---|---:|---:|
   | `max_input_diagnostics_per_tick` | 单 tick diagnostic record 上限。 | 32 | 16-64 |
   | `max_input_trace_memory_bytes` | input trace window 总 byte 上限。 | 1,048,576 | 262,144-2,097,152 |
   | `projected_trace_window_bytes` | 本次写入后 trace window 预计 bytes。 | measured | integer `>= 0` |

11. **performance percentile calculation**

   `input_processing_elapsed_ms = seal_end_monotonic_ms_quantized - seal_start_monotonic_ms_quantized`

   `p95 = percentile(input_processing_elapsed_ms_samples, 95)`

   `p99 = percentile(input_processing_elapsed_ms_samples, 99)`

   | Variable | Definition | Default / Requirement |
   |---|---|---|
   | `input_processing_elapsed_ms_samples` | exported Web minimal trace mode 下每次 input seal 的耗时样本，包含 capture drain、snapshot build、request route、buffer processing。 | At least 3 runs × 60s each after 5s warmup |
   | `input_processing_budget_ms_p95` | p95 预算。 | `<= 0.20ms` |
   | `input_processing_budget_ms_p99` | p99 预算。 | `<= 0.50ms` |

   Performance measurement 不得在 `dev_verbose` trace mode 下进行；否则结果 invalid。

12. **input-owned heap growth**

   `input_owned_heap_growth_mb_per_60s = (input_owned_bytes_at_60s - input_owned_bytes_after_warmup) / 1,048,576`

   | Variable | Definition | Default / Requirement |
   |---|---|---|
   | `input_owned_bytes_after_warmup` | 5s warmup 后 input-owned records、buffers、trace windows 的 tracked bytes。 | measured |
   | `input_owned_bytes_at_60s` | 60s steady scenario 结束时 tracked bytes。 | measured |
   | `input_heap_growth_budget_mb_per_60s` | input-owned heap growth 预算。 | `<= 1.0MB` |

   若浏览器 JS heap API 不可用，MVP 至少必须记录 input-owned data structure tracked bytes；如果两者都不可测，该 AC fail，不能以“浏览器不支持”跳过。

13. **UI prompt timing does not change combat timing**

   `effective_prompt_min_visible_ms = max(base_prompt_min_visible_ms, profile_prompt_min_visible_ms)`

   `can_repeat_same_prompt = elapsed_since_same_prompt_ms >= prompt_repeat_cooldown_ms`

   | Variable | Definition | Default | Safe Range |
   |---|---|---:|---:|
   | `base_prompt_min_visible_ms` | Standard / Alternate profile 的关键提示最小可见时间。 | 1200ms | 800-2000ms |
   | `accessibility_prompt_min_visible_ms` | Accessibility profile 的关键提示最小可见时间。 | 2400ms | 1500-4000ms |
   | `prompt_repeat_cooldown_ms` | 同一提示重复显示冷却，避免刷屏。 | 1500ms | 500-4000ms |
   | `dangerous_request_repeat_lockout_ms` | quick restart / exit 等危险 request 的重复确认锁定时间。 | 500ms | 300-1000ms |
   | `ui_repeat_initial_delay_ms` | UI navigation held repeat 首次重复延迟。 | 350ms | 250-600ms |
   | `ui_repeat_interval_ms` | UI navigation held repeat 后续间隔。 | 100ms | 75-200ms |

   这些 UI/prompt timing knobs 只影响可读性、菜单导航和防误触；不得扩大 combat buffer、不得改变 hitstop cap、不得改变 guard/attack legality。

## Edge Cases

1. **浏览器重复 keydown**
   如果浏览器因长按产生重复 keydown，同一物理键不得重复生成 `pressed`。Repeat events 聚合为 `ignored_key_repeat` 或 `merged_key_repeat` decision record，记录 first/last host frame 与 count，而不是无限逐条写 trace。

2. **press→release 在同一 host frame 内发生**
   如果同一物理键在同一 host frame 内发生 keydown→keyup，最终 `held = false`，但 transient decision record 必须保留。若该 keydown 映射到 one-shot combat action，且 context 允许捕获，则仍创建一个 pressed buffer entry；若 context 不允许，则记录具体 reject reason。多个 press/release burst 超过 `max_transient_records_per_snapshot` 时聚合为 bounded summary。

3. **左右或上下同时按下**
   同一轴上的相反方向同时 held 时，该轴输出 neutral，不生成方向 transition entry，并记录 `opposite_axis_neutralized`。这不影响同 snapshot 的 one-shot attack 捕获。

4. **held guard 与 pressed attack 同 tick**
   held guard 属于 continuous guard intent，pressed attack 属于 one-shot command。若 `light_attack`、`heavy_attack` 或 `projectile` candidate 被选中，输入系统仍保留 raw held guard record，但 combat-visible ordinary guard intent 标记为 `ordinary_guard_intent_suppressed`，candidate command 标记 `guard_suppressed_by_attack_attempt = true`。状态机最终决定 attack 是否允许或 terminal reject，但不得在同一 vulnerability window 让玩家同时获得普通防御和攻击尝试的最优结果。

5. **burst + attack 同 snapshot**
   Burst 是 high-value emergency command，必须没有竞争 one-shot 才能进入 pending。若 `burst` 与 light/heavy/projectile 同 snapshot 同组出现，全部同组 one-shot entries 标记 `ambiguous_high_value_conflict`，不提交、不 fallback、不消耗气槽。训练/HUD 可以显示 localization key `input.ambiguous_burst_conflict`，提示玩家重新明确按下。

6. **较旧 burst 与较新 light attack 跨 tick 竞争**
   若 burst 在 running tick 100 buffered，light attack 在 running tick 102 buffered，二者在 tick 103 都有效，light attack 因较新 intent 被选中。若 light terminal reject，不 fallback 到旧 burst，旧 burst 清理为 `superseded_by_newer_attempt`。

7. **pause 与 combat input 同 snapshot**
   如果 `pause` 被当前 context 接受，该 snapshot 中未提交的 one-shot combat entries 必须 `consume_without_action`，reason `runtime_request_accepted_same_snapshot`。Continuous held combat input 标记 `pending_release`，直到 recovery rules 清理。若 `pause` 被拒绝或 no-op，combat input 按当前 `combat_input_policy_at_capture` 正常处理，并记录 pause request decision。

8. **失焦导致 keyup 丢失**
   如果浏览器窗口失焦、标签页切走、全屏状态变化、canvas 失焦、keyboard focus lost 或 page hidden，系统进入 unsafe focus recovery：increment `input_generation_id` 和 `interruption_epoch`，所有 held combat input 标记为 pending release，所有 pending one-shot combat entries invalidated。恢复时先尝试 key-state reconciliation；能证明 key up 则清理但不产生 combat edge，不能证明则显示 release-keys prompt。下一次同 key keydown 只能作为 cleanup press，不产生 combat pressed。

9. **round-start countdown 中提前按攻击、防御或方向**
   Direction 可作为 countdown pre-read。Guard 可作为 continuous guard pre-read，并在 `go` 后第一枚 running tick 成为 guard held intent，但不产生 pressed edge。One-shot attacks/projectile/burst 不创建 buffer；held 到 `go` 后也不会自动攻击，必须 release+fresh press。

10. **resume countdown 中旧 held guard**
    如果 guard 是 unsafe boundary 前的旧 held，resume countdown 中仍是 pending release，不会自动成为 guard。只有 post-restore reconciliation 或 trusted keyup 清理后，当前 physical down 才可作为 continuous guard intent。

11. **hitstop 中多次输入**
    Hitstop 中可以采集 direction、guard 和 one-shot pressed input，不执行 combat command，不推进 buffer age。Hitstop 中创建的 one-shot entry 在 first legal running tick 的 running age 为 0，但仍受 hitstop realtime cap 限制。若多个 one-shot 在同一 target running tick 竞争，先按跨 tick newest-intent，再按同 snapshot priority；continuous guard/direction 单独保留。

12. **旧 round、旧 generation、旧 interruption epoch 的输入到达**
    如果 input snapshot、request、buffer entry、decision 或 command 的 round/generation/epoch/config hash/UI context generation 与当前 runtime 不匹配，必须 fail closed，记录 stale reason。不得把旧 quick restart、旧 focus generation、旧 pause 前、旧 config 下的输入套到当前回合。

13. **round_ended 后仍收到输入**
    `round_ended` 后，combat buffer 必须清空。继续收到的攻击、移动、防御、气弹或爆气输入只能被 result/menu context 使用或记录为 no-buffer decision，不能生成 combat command。下一回合开始时必须使用新的 `round_instance_id`、`round_instance_sequence`、初始 `input_generation_id` 和新的 context sequence。

14. **buffer entry 数量超过上限**
    如果同一 actor pending entry 超过 `max_buffer_entries_per_actor`，系统按 capacity eviction ordering 清理并记录。不得无记录丢弃 latest input，也不得依赖 Dictionary/Array 非确定顺序。

15. **runtime request 在 nested UI context 中到达**
    如果 focus recovery overlay 叠在 pause menu 上，topmost overlay 先消费 confirm/cancel/ui navigation。一次 key press 只能产生一个 accepted `InputRequestRecord`；下层 menu 不得同时收到该 confirm，也不得把它转成 combat input。

16. **menu navigation 与 combat direction 复用箭头键**
    当 top context 为 `menu_only`、profile selection 或 overlay owns navigation，Arrow keys / WASD 生成 `ui_*` request，不生成 combat axis。退出 menu 后，旧 held arrow keys 必须经过 pending-release 或 context transition cleanup，不得在 resume 第一帧自动移动；resume countdown 只允许来自 combat context 的 direction pre-read，不能继承 menu/overlay navigation held。

17. **quick restart / exit through menu confirm**
    Quick restart 或 exit to menu 只能由 focused menu/result/debug control 把 confirm 转译成 semantic request。Request accepted 时进入 unsafe boundary：generation/epoch 更新、combat buffers invalidated、held combat pending release、新 round 或 menu state 使用新 context sequence。刚打开菜单后的 held/repeat confirm 不得触发危险 request；重复 confirm 在 `dangerous_request_repeat_lockout_ms` 内必须被 duplicate idempotency 拒绝。

18. **MVP 外设备输入**
    Gamepad、touch 或其他非 `keyboard_primary` 来源可以记录 bounded diagnostics，或显示“键盘为当前 MVP 阻塞输入方式”。这些输入不得影响 keyboard rules、不得进入 combat command flow，也不得成为 MVP 阻塞条件。

19. **键盘 ghosting / rollover 限制**
    如果硬件或浏览器没有报告某个实际按下的键，输入系统不能凭空推断。QA 必须记录 tested key combos；默认布局必须避免常见 browser/system shortcut 与高风险 ghosting chord。检测到冲突时，系统必须提供实际通过 required combo set 的 Alternate 或 Accessibility profile；warning-only fallback 不算通过。

20. **duplicate mapping 与 context-specific reuse**
    同一 active context 内一个 physical key 绑定多个 combat actions 时，config validation fail。菜单 context 可以复用 combat key 作为 confirm/cancel/navigation，但必须由 context router 隔离，并写入 `input_config_hash`。MVP 不验证自由 remap capture modal；profile selection 只能在 round start 前改变整套 profile。

21. **debug overlay 打开时输入不可泄漏**
    Debug overlay 若打开并 topmost，所有 confirm/cancel/ui navigation 由 overlay context 处理；combat input policy 为 `menu_only` 或 `blocked_all`。Overlay 关闭时必须记录 context generation 变化，旧 overlay request 不得在 gameplay 中重放。

## Dependencies

### Hard Dependency

| Dependency | File | Status | Why it matters |
|---|---|---|---|
| 固定逻辑步进与战斗运行时 | `design/gdd/fixed-logic-runtime.md` | Exists; revised after ninth fresh review; pending fresh re-review | 提供 committed tick、running tick、hitstop tick、`runtime_state`、`tick_execution_state`、`post_commit_runtime_state`、`combat_input_policy`、`state_reason` / `pause_reason`、`recovery_pause`、`presentation_ack_wait`、round lifecycle、focus/recovery semantics。输入系统必须按这些状态采集、冻结、清理或提交输入。 |

`input-buffering` 不得直接绕过 fixed runtime 执行动作。它只提交 candidate combat command 或 runtime/UI request；命中、防御、取消、气槽消耗、硬直、胜负和 presentation 结果都由后续 runtime / combat state systems 判定。

### Registry Dependency

| Dependency | File | Status | Required sync |
|---|---|---|---|
| Entity / formula registry | `design/registry/entities.yaml` | Exists | 当本 GDD 通过 review 后，输入 buffer windows、priority values、trace caps、schema fields、reason enums、profile constants、request types 和 profiling constants 若成为跨系统常量或 schema 字段，必须同步进 registry。 |

### Downstream Integration Points

这些不是当前 GDD 的 hard dependency，但后续系统必须消费本 GDD 的输出，不能重新定义输入语义。

| Future system | Expected relationship |
|---|---|
| Combat state machine / action legality | 消费 candidate combat command，判断当前 actor 状态是否允许该动作；提供 deterministic read-only legality query；不得重新解释 raw pressed/held/released。 |
| Movement / facing system | 消费 resolved axis 与 countdown/resume direction pre-read；必须尊重 opposite-axis neutralization 和 menu/resume cleanup。 |
| Guard / block / hit reaction | 消费 continuous guard intent；必须区分 round-start countdown guard pre-read、resume pending-release guard、toggle guard assist 和 actual guard state。 |
| Attack / projectile / burst systems | 消费 `light_attack`、`heavy_attack`、`projectile`、`burst` candidate command；负责成本、取消、命中和 terminal reject reason。 |
| UI / pause / menu flow | 消费 `pause`、`confirm`、`cancel`、`ui_*` request；提供 stable UI context stack、focus owner ids、quick restart/exit semantic request；必须防止恢复确认输入进入 combat command flow。 |
| Web platform shell | 提供 focus/page/fullscreen/canvas focus signals、prevent-default boundary、keyboard reconciliation capability、audio/focus recovery overlay。 |
| Input settings / profile selection | MVP 只提供 Standard / Alternate / Accessibility profile selection、键位配置、冲突检测、reserved-key warning、ghosting fallback、input config hash；不得允许同 active context duplicate combat mapping。自由 remapping capture 是后续系统，不是当前 MVP 阻塞依赖。 |
| QA trace / replay diagnostics | 消费 input sample、snapshot、buffer entry、decision record、request record、UI context record 和 command record，用于调查吞键、焦点恢复、过早/过晚输入和 browser event 合并问题。 |
| Debug / tuning display | 只能作为 UI context 消费 request；不得绕过 router 改 combat state 或读取 physical key。 |

### Bidirectional Notes

- `fixed-logic-runtime.md` 已声明“输入映射与输入缓冲”为 Hard dependency interaction：它需要输入系统提供 tick id、runtime state 下的采集/执行/冻结/清理规则，并禁止输入绕过 runtime 执行动作。
- 因 `fixed-logic-runtime` 仍 pending fresh re-review，本 GDD 可以把它当作当前工作依赖，但不能把未复审通过的细节当作最终实现批准。若后续 fresh re-review 改动 runtime state、ack wait、recovery pause、catch-up 或 tick terminology，本 GDD 必须同步修订。
- Input Snapshot / Serialization ADR、Web Focus/Audio Unlock Shell ADR、Godot Pause/Time-scale ADR、UI Context Stack ADR 仍是实现前技术门槛；本 GDD 固定行为合同，不指定最终 Godot API。

## Tuning Knobs

所有输入调参必须是数据驱动配置，不得硬编码在战斗逻辑里。MVP 默认 Standard Profile 对所有角色、第一回合和普通回合一致；不得按角色、敌人、隐藏状态或失败次数偷偷放宽输入窗口。Alternate / Accessibility profile 必须在 UI 明确标记、round start 前选择、写入 `input_config_hash`，并在 QA evidence 中记录；不能改变 Standard Profile 的 deterministic rules。

| Knob | Default | Safe Range | MVP Lock | Affects |
|---|---:|---:|---|---|
| `normal_attack_buffer_ticks` | 6 | 3-8 | Tunable between builds; not during round | light/heavy one-shot 攻击提前输入容忍度。 |
| `projectile_buffer_ticks` | 4 | 2-6 | Tunable between builds; not during round | projectile 提前输入容忍度；默认短于普通攻击以避免远程牵制自动化。 |
| `burst_buffer_ticks` | 3 | 2-4 | Tunable between builds; not during round | burst 提前输入容忍度；高价值反杀命令默认更严格。 |
| `direction_guard_buffer_ticks` | 3 | 1-4 | Tunable between builds; not during round | direction/guard pressed-edge transition grace；continuous held state 不靠该值长期保留。 |
| `hitstop_buffer_realtime_cap_ms` | 150 | 66-200 | Tunable between builds; not during round | hitstop 中 one-shot input 可保留的最大真实时间。 |
| `runtime_request_buffer_ticks` | 0 | 0 only | Locked for MVP | pause/confirm/cancel/ui navigation 不进入 combat buffer。 |
| `countdown_direction_pre_read_enabled` | true | true/false | Tunable between builds | countdown/resume countdown 中是否允许方向预读。 |
| `countdown_guard_pre_read_enabled` | true | true/false | Tunable between builds | round-start countdown 中是否允许 guard continuous pre-read。 |
| `countdown_action_pre_buffer_enabled` | false | false only | Locked for MVP | 是否允许 countdown 中预存攻击/气弹/爆气；MVP 必须 false。 |
| `pending_release_required_after_unsafe_interruption` | true | true only | Locked for MVP | focus/pause/recovery/visible ack wait 后是否要求 release/reconcile cleanup。 |
| `allow_polling_for_focus_reconciliation_only` | true | true only | Locked for MVP | Polling 只能用于恢复时证明 key up/down，不能制造 pressed。 |
| `opposite_axis_policy` | neutralize | neutralize only | Locked for MVP | SOCD 处理。 |
| `max_buffer_entries_per_actor` | 8 | 4-12 | Tunable between builds; not during round | 单 actor pending one-shot buffer entry 上限。 |
| `input_priority_burst` | 400 | fixed table | Locked for MVP | same-snapshot one-shot priority。 |
| `input_priority_light_attack` | 300 | fixed table | Locked for MVP | same-snapshot one-shot priority。 |
| `input_priority_heavy_attack` | 200 | fixed table | Locked for MVP | same-snapshot one-shot priority。 |
| `input_priority_projectile` | 100 | fixed table | Locked for MVP | same-snapshot one-shot priority。 |
| `max_input_diagnostics_per_tick` | 32 | 16-64 | Tunable between builds | 单 tick input diagnostics 上限。 |
| `max_transient_records_per_snapshot` | 8 | 4-16 | Tunable between builds | 同 snapshot press/release burst 记录上限。 |
| `input_trace_window_ticks` | 600 | 120-1800 | Tunable between builds | QA trace retention window；600 ticks 约 10 秒 running time。 |
| `max_input_trace_memory_bytes` | 1,048,576 | 262,144-2,097,152 | Tunable between builds | input trace window 总 byte 上限。 |
| `input_snapshot_max_bytes` | 1024 | 512-2048 | Tunable between builds | 单 snapshot serialized payload 预算。 |
| `input_decision_record_max_bytes` | 512 | 256-1024 | Tunable between builds | 单 decision/request/lifecycle record 预算。 |
| `input_processing_budget_ms_p95` | 0.20 | 0.10-0.50 | Tunable between builds | Web p95 input capture + snapshot + buffer processing budget。 |
| `input_processing_budget_ms_p99` | 0.50 | 0.25-1.00 | Tunable between builds | Web p99 input processing budget。 |
| `input_heap_growth_budget_mb_per_60s` | 1.0 | 0.5-2.0 | Tunable between builds | exported Web steady-state input-owned heap growth budget。 |
| `toggle_guard_assist_allowed` | true in accessibility profile only | true/false | Profile-fixed before round | Guard accessibility；must be hash-visible and trace-visible。 |
| `base_prompt_min_visible_ms` | 1200 | 800-2000 | Tunable between builds | Standard / Alternate profile 关键提示最小显示时间；不影响 combat timing。 |
| `accessibility_prompt_min_visible_ms` | 2400 | 1500-4000 | Tunable between builds | Accessibility profile 关键提示最小显示时间；不影响 combat timing。 |
| `prompt_repeat_cooldown_ms` | 1500 | 500-4000 | Tunable between builds | 同一提示重复显示冷却，避免刷屏。 |
| `dangerous_request_repeat_lockout_ms` | 500 | 300-1000 | Tunable between builds | quick restart / exit 等危险 request 防重复确认窗口。 |
| `ui_repeat_initial_delay_ms` | 350 | 250-600 | Tunable between builds | UI navigation held repeat 首次重复延迟。 |
| `ui_repeat_interval_ms` | 100 | 75-200 | Tunable between builds | UI navigation held repeat 后续间隔。 |

### Tuning Rules

- Any tuning change must record before/after values and a short reason: “reduce missed early light attacks”, “prevent buffered burst from feeling automatic”, “reduce focus recovery confusion”, etc.
- Input buffer tuning must be tested on exported Web/browser target, not only in editor.
- Tuning must not compensate for broken runtime timing, frame stalls, focus bugs, state-machine rejection bugs, stale generation, wrong buffer age, or lost cleanup. Fix the bug instead of widening the buffer.
- Values outside safe range require GDD revision and fresh review before implementation uses them.
- `qa_bounded` / `dev_verbose` trace may exceed minimal trace volume only within explicit caps; trace overflow must summarize, not allocate unbounded records.
- Assist/profile differences must be visible to the player and QA; Standard Profile cannot silently inherit assist behavior.
- Accessibility prompt timing and toggle guard do not change combat buffer windows in MVP. If a future assist mode widens combat timing, it requires explicit GDD revision, separate profile labeling, separate playtest data, and fresh review.

## Acceptance Criteria

### MVP Blocking — Authority, Serialization, and Schemas

- **AC-IB-01 — Canonical determinism**: Owner QA automation. Fixture `IB-AUTH-001`. Given identical input-event fixture, UI context sequence, runtime-state sequence, target tick sequence, round IDs, player/actor IDs, generation/epoch IDs, config hash, tuning config and deterministic ID seed, when the input system runs twice, then canonical serialized `InputSample`, `InputSnapshot`, `InputBufferEntry`, `InputDecisionRecord`, `InputRequestRecord`, `UIContextRecord`, and candidate command streams have identical hashes and identical ordered records.
- **AC-IB-02 — Required schema fields**: Owner QA automation. Fixture `IB-AUTH-002`. Every `InputSample`, `InputSnapshot`, `InputBufferEntry`, `InputDecisionRecord`, `InputRequestRecord`, and `UIContextRecord` includes the required fields from Detailed Rules 6, 11, 12, 13, 14, and 15; missing field, unknown enum, live Node reference, or unstable NodePath authority fails validation.
- **AC-IB-03 — Router is sole authority**: Owner integration QA. Fixture `IB-AUTH-003`. Given gameplay, pause menu, overlay, profile selection and text-entry contexts, physical keys are recorded by central router before any semantic consumption; no Godot `Control` or gameplay node can produce combat command or accepted request without matching router record.
- **AC-IB-04 — Combat/request branch separation**: Owner unit QA. Fixture `IB-AUTH-004`. `pause`, `confirm`, `cancel`, `ui_*`, quick restart and exit create `InputRequestRecord` only and never create combat `InputBufferEntry`; combat actions never directly create runtime/UI request records.
- **AC-IB-05 — Unified seal cutoff**: Owner unit QA. Fixture `IB-AUTH-005`. Given samples before, at, and after `seal_cutoff_physical_event_sequence_index`, only samples `<= cutoff` enter the current snapshot; later samples enter a future target and cannot mutate the sealed snapshot.
- **AC-IB-06 — Catch-up no retroactive input**: Owner integration QA. Fixture `IB-AUTH-006`. When runtime processes multiple delayed ticks in one host frame, physical events received during that host frame are not applied to historical delayed ticks; missing historical ticks receive `no_new_physical_input` snapshots only.
- **AC-IB-07 — Catch-up safe continuous inheritance**: Owner unit QA. Fixture `IB-AUTH-007`. `no_new_physical_input` inherits direction/guard only when same round, generation, epoch, config, focus-safe, not pending-release and policy permits continuous state; otherwise direction is neutral, guard is false, and `catch_up_continuous_state_dropped` is recorded.
- **AC-IB-08 — Non-advancing context snapshots**: Owner unit QA. Fixture `IB-AUTH-008`. During countdown, pause, focus_suspended, recovery_pause, automatic presentation ack wait, player recovery prompt, resume countdown and round_ended with no combat tick commit, snapshot has `target_committed_tick_index = null`, valid `target_context_sequence_id`, and no forged `current_committed_tick_index + 1`.
- **AC-IB-09 — Browser key repeat suppression**: Owner Web smoke QA. Fixture `IB-AUTH-009`. Given one keydown, five browser repeat keydowns, then one keyup for the same key in exported Web, exactly one snapshot has `pressed = true`; repeats create no additional pressed edges and are aggregated as repeat diagnostics.
- **AC-IB-10 — Same-host-frame tap is not swallowed**: Owner unit QA. Fixture `IB-AUTH-010`. Given keydown→keyup for `light_attack` before one snapshot is sealed, final held is false, transient press/release is traceable, and one valid one-shot buffer entry is created if context allows capture.

### MVP Blocking — Buffer Lifecycle and Combat Intent

- **AC-IB-11 — Negative buffer age fails closed**: Owner unit QA. Fixture `IB-LIFE-001`. Given `evaluated_at_running_tick_index < created_at_running_tick_index`, the entry is rejected as `future_or_corrupt_buffer_age` and never considered valid.
- **AC-IB-12 — Action-class buffer boundaries across safe range**: Owner unit QA. Fixture `IB-LIFE-002`. For `normal_attack_buffer_ticks` values 3, 6, and 8; `projectile_buffer_ticks` values 2, 4, and 6; and `burst_buffer_ticks` values 2, 3, and 4, age `0..max` is valid, age `max + 1` expires, and age `-1` rejects as corrupt/future for the matching action class.
- **AC-IB-13 — Direction/guard transition boundaries**: Owner unit QA. Fixture `IB-LIFE-003`. For `direction_guard_buffer_ticks` values 1, 3, and 4, transition age `0..max` is valid, age `max + 1` expires, and continuous held direction/guard remains represented separately from transition grace.
- **AC-IB-14 — Hitstop running-age anchor**: Owner integration QA. Fixture `IB-LIFE-004`. Given running tick `T` creates hitstop and `light_attack` is pressed during hitstop, no command executes during hitstop; the entry is evaluated no earlier than first resumed running tick; `buffer_age_running_ticks = 0` on that first running tick.
- **AC-IB-15 — Hitstop realtime cap**: Owner performance/gameplay QA. Fixture `IB-LIFE-005`. Given one hitstop press at realtime age `<= hitstop_buffer_realtime_cap_ms` and one at `cap + 1ms`, the first can be evaluated and the second expires as `expired_by_hitstop_realtime_cap` even though running age is 0.
- **AC-IB-16 — Same snapshot high-value conflict is strict**: Owner unit QA. Fixture `IB-LIFE-006`. Given `burst` plus any `light_attack`, `heavy_attack`, or `projectile` are pressed for one actor in one snapshot, all same-group one-shot entries reject as `ambiguous_high_value_conflict`, no command is submitted, no fallback occurs, and no burst resource is consumed. Given only non-burst one-shot conflicts, `light_attack > heavy_attack > projectile` priority applies.
- **AC-IB-17 — Cross-tick newest intent wins**: Owner unit QA. Fixture `IB-LIFE-007`. Given older burst and newer light attack are both valid pending entries, the newer light attack is selected; if it terminally rejects, no fallback to older burst occurs and older entries are `superseded_by_newer_attempt`.
- **AC-IB-18 — Burst unavailable does not fallback to attack**: Owner integration QA. Fixture `IB-LIFE-008`. Given an unambiguous selected burst terminally rejects for unavailable resource, burst is consumed as `terminal_reject = burst_unavailable`, no one-shot fallback executes, continuous direction remains available, and guard follows attack-suppression rules only if a separate valid attack attempt exists.
- **AC-IB-19 — Retain-until-legal is bounded and query-owned**: Owner integration QA. Fixture `IB-LIFE-009`. Given combat legality query returns `retain_until_legal`, the entry remains only while `is_action_buffer_valid` is true; missing query returns `retain_query_unavailable` and fails closed according to policy rather than guessing combat state.
- **AC-IB-20 — Capacity eviction is deterministic**: Owner unit QA. Fixture `IB-LIFE-010`. When adding a pending entry would exceed `max_buffer_entries_per_actor`, expired/invalidated/superseded entries are removed first; remaining eviction uses the defined ordering and records `capacity_evicted`.
- **AC-IB-21 — Attack attempt suppresses ordinary guard option-select**: Owner integration QA. Fixture `IB-LIFE-011`. Given `guard` held and `light_attack` pressed in the same valid running snapshot, raw guard held remains traceable, the attack candidate is represented, `guard_suppressed_by_attack_attempt = true` is attached, `ordinary_guard_intent_suppressed` is recorded, and downstream validation cannot grant both ordinary guard protection and attack attempt payoff in the same vulnerability window.

### MVP Blocking — Interruption, Countdown, Focus, and Recovery

- **AC-IB-22 — Unsafe interruption invalidates one-shot buffers**: Owner integration QA. Fixture `IB-REC-001`. Given a pending `light_attack` before player pause, focus loss, recovery pause, player recovery prompt, visible presentation safe-neutral recovery, quick restart or round restart, the entry is invalidated as `invalidated_by_interruption` and cannot fire after resume.
- **AC-IB-23 — Generation and epoch update are not OR**: Owner unit QA. Fixture `IB-REC-002`. Unsafe interruption increments `input_generation_id`; boundaries that make held combat unsafe also increment `interruption_epoch`; stale records with either old value fail closed.
- **AC-IB-24 — Pending release trusted keyup cleanup**: Owner Web smoke QA. Fixture `IB-REC-003`. After unsafe interruption, pre-existing held combat keys are blocked as pending release; UI confirm does not clear them; trusted post-restore keyup clears pending release without producing combat pressed.
- **AC-IB-25 — Pending release reconciliation cleanup**: Owner Web smoke QA. Fixture `IB-REC-004`. If focus restore reconciliation proves a pending key is currently up, pending release clears as `pending_release_cleared_by_reconciliation`; no combat `released` or `pressed` edge is generated.
- **AC-IB-26 — Pending release no-deadlock fallback**: Owner Web smoke QA. Fixture `IB-REC-005`. If no keyup arrives and reconciliation cannot prove key up, the next keydown for the pending key is consumed only as cleanup, records `reconciled_up_blocked_press`, and requires keyup→fresh keydown before combat-visible pressed.
- **AC-IB-27 — Round-start countdown direction and guard pre-read**: Owner gameplay QA. Fixture `IB-REC-006`. During round-start countdown, direction can affect first running tick preparation and guard held can become continuous guard intent at GO; neither creates pressed edges or one-shot entries.
- **AC-IB-28 — Countdown one-shot blocked**: Owner gameplay QA. Fixture `IB-REC-007`. During round-start countdown, `light_attack`, `heavy_attack`, `projectile`, and `burst` create no combat buffer entry; held-through-GO does not attack until release+fresh press.
- **AC-IB-29 — Resume countdown old guard remains blocked**: Owner integration QA. Fixture `IB-REC-008`. After pause/focus/recovery, old held guard remains pending release during resume countdown until trusted cleanup; confirm does not restore guard.
- **AC-IB-30 — SOCD neutralization**: Owner unit QA. Fixture `IB-REC-009`. Simultaneous left+right or up+down produces neutral axis, creates no direction transition entry for that axis, and records `opposite_axis_neutralized` without deleting unrelated one-shot inputs.

### MVP Blocking — UI, Request Routing, and Web Shell

- **AC-IB-31 — Request context routing is one-shot**: Owner UI integration QA. Fixture `IB-UI-001`. Given focus recovery overlay above pause menu, one confirm press is consumed only by the overlay and cannot also activate pause menu or produce combat input.
- **AC-IB-32 — Pause plus combat same snapshot**: Owner integration QA. Fixture `IB-UI-002`. If pause and `light_attack` occur in the same snapshot and pause is accepted, no combat command from that snapshot executes or survives resume; the combat input records `runtime_request_accepted_same_snapshot`.
- **AC-IB-33 — Menu navigation does not move combat actor**: Owner UI integration QA. Fixture `IB-UI-003`. In pause menu, result menu, recovery overlay and profile selection, Arrow keys / WASD produce `ui_*` request according to context and produce no combat axis; held navigation keys cannot leak into resume countdown direction pre-read without release+fresh combat-context press.
- **AC-IB-34 — Quick restart and exit are menu-owned semantic requests with anti-misclick protection**: Owner UI integration QA. Fixture `IB-UI-004`. Confirm on focused quick restart/exit control first creates a consumed confirm record, then generates `request_type = quick_restart` or `exit_to_menu`, records consumed context/control ids and idempotency key, triggers unsafe cleanup, and never creates combat input. Held/repeat confirm on newly opened menus and duplicate confirm within `dangerous_request_repeat_lockout_ms` cannot trigger the dangerous request.
- **AC-IB-35 — Automatic presentation ack wait ignores confirm**: Owner integration QA. Fixture `IB-UI-005`. During automatic `presentation_ack_wait` without player prompt, confirm does not satisfy presentation watermark, does not resume runtime, and does not create combat input; a player recovery prompt can consume confirm only if it owns top context.
- **AC-IB-36 — Stale UI focus fails closed**: Owner UI unit QA. Fixture `IB-UI-006`. If a focused Control is destroyed, hidden, or has stale generation, request rejects as `stale_ui_context_or_focus_owner` and does not fall through to lower menu or combat.
- **AC-IB-37 — Web prevent-default and modifier matrix**: Owner Web smoke QA. Fixture `IB-UI-007`. Exported Web build verifies arrows, Enter, P, N, Z/X/C/V/B, W/A/S/D, J/K/L/I/O, Space and Ctrl/Alt/Shift/Meta chord cases in gameplay, menu, overlay, profile selection and text-entry contexts do not scroll page, double-submit, trigger browser UI, or leak across contexts except where explicitly allowed; `InputSample` records modifier and reserved shortcut decisions.

### MVP Blocking — Accessibility and Input Configuration

- **AC-IB-38 — Config validation rejects unsafe mappings**: Owner unit QA. Fixture `IB-ACC-001`. Duplicate combat binding in the same active context, missing required action, duplicate one-shot priority, reserved-key conflict, out-of-range tuning, nonzero `runtime_request_buffer_ticks`, invalid profile, invalid toggle guard state, invalid prompt timing, or `countdown_action_pre_buffer_enabled = true` fails validation before combat starts.
- **AC-IB-39 — Standard, Alternate, and Accessibility profiles are available and hash-visible**: Owner UX/accessibility QA. Fixture `IB-ACC-002`. MVP exposes Standard, Alternate, and Accessibility keyboard profiles before round start; each profile has concrete key bindings, required combo evidence, profile id, hash-visible settings, and selected profile appears in `input_config_hash` and QA evidence.
- **AC-IB-40 — Toggle guard assist lifecycle is deterministic**: Owner accessibility QA. Fixture `IB-ACC-003`. In Accessibility profile, toggle guard state is serialized in snapshot/hash and produces identical streams across repeated fixture runs covering round start, countdown, running, hitstop, pause, focus loss, recovery pause, resume countdown, profile selection, round end, quick restart, and browser repeat; UI/menu contexts cannot silently change combat toggle state.
- **AC-IB-41 — Ghosting fallback is pass/fail, not only logged**: Owner Web smoke QA. Fixture `IB-ACC-004`. QA tests required movement+guard+action combos for Standard, Alternate, and Accessibility profiles; if Standard fails a required combo on target browser/hardware, Alternate or Accessibility must actually pass the combo set before MVP acceptance. Warning-only fallback fails this AC.
- **AC-IB-42 — Recovery prompts meet measurable accessibility standards**: Owner UX/accessibility QA. Fixture `IB-ACC-005`. Focus/pause/recovery resume flow displays localization-keyed, 200%-scale-safe, non-color-only prompt text with minimum 18px equivalent text, 4.5:1 text contrast, 3:1 focus/icon contrast, visible focus ownership, minimum display time from prompt timing knobs, and no internal enum strings; confirm/cancel cannot leak into combat input.
- **AC-IB-43 — Countdown prompt matches actual rules**: Owner UX QA. Fixture `IB-ACC-006`. First-round/training prompt states movement/guard can be held before GO and attack/burst must be pressed after GO; resume countdown prompt separately states old held keys must be released/fresh-pressed according to recovery rules; both prompts are non-color-only, measurable under AC-IB-42, and do not use internal enum strings.

### MVP Blocking — Performance and Trace Bounds

- **AC-IB-44 — Input processing budget**: Owner performance QA. Fixture `IB-PERF-001`. In exported Web minimal trace mode, after 5s warmup and at least 3 × 60s runs per browser smoke scenario, input capture drain + snapshot build + request route + buffer processing meets `input_processing_budget_ms_p95 <= 0.20` and `input_processing_budget_ms_p99 <= 0.50`.
- **AC-IB-45 — Catch-up processing budget included**: Owner performance QA. Fixture `IB-PERF-002`. Performance scenarios include a catch-up batch of at least 4 delayed ticks in one host frame; no retroactive input reads occur and per-seal processing remains within p99 budget or records a blocking performance failure.
- **AC-IB-46 — Input-owned heap growth budget**: Owner performance QA. Fixture `IB-PERF-003`. In exported Web minimal trace mode, 60 seconds of steady keyboard play stays within `input_heap_growth_budget_mb_per_60s <= 1.0` for input-owned tracked bytes; inability to measure tracked bytes fails the AC.
- **AC-IB-47 — Trace overflow is bounded by count and bytes**: Owner unit/performance QA. Fixture `IB-PERF-004`. When diagnostic records exceed per-tick, per-snapshot or total trace byte caps, the system writes bounded summary records with `authoritative_input_loss = false` and does not exceed `max_input_trace_memory_bytes`.
- **AC-IB-48 — Byte budget check is canonical**: Owner unit QA. Fixture `IB-PERF-005`. Canonical serialized `InputSnapshot`, `InputDecisionRecord`, and `InputRequestRecord` samples in QA fixtures stay within configured byte budgets or fail with bounded diagnostic reason; measurement uses canonical serialization, not editor object size.
- **AC-IB-49 — Functional trace and performance profiling are separated**: Owner performance QA. Fixture `IB-PERF-006`. Performance ACs run only in `minimal_export` trace mode; `qa_bounded` and `dev_verbose` results are marked invalid for p95/p99 approval.

### MVP Blocking — Player Trust and UX Evidence

- **AC-IB-50 — Player-visible feedback matrix is complete**: Owner UX QA. Fixture `IB-UX-001`. Every input decision reason used by MVP maps to a feedback category, localization key or explicit silent rationale, minimum display time, repeat cooldown, priority, and combat/menu visibility rule; no player-facing prompt displays internal enum strings.
- **AC-IB-51 — Keyboard-only Web flow is playable**: Owner UX/accessibility QA. Fixture `IB-UX-002`. Starting from page load in exported Web, a tester using only keyboard can focus the game, choose each profile, start a round, pause, recover from focus loss, resolve pending release, quick restart, and exit to menu without mouse input or combat leakage.
- **AC-IB-52 — Training input history explains blocked input**: Owner gameplay/UX QA. Fixture `IB-UX-003`. In training/debug evidence mode, buffered, rejected, superseded, ambiguous, pending-release, countdown-blocked, burst-unavailable, and stale UI input decisions appear with player-readable explanations tied to their source snapshot/request/entry ids.
- **AC-IB-53 — Standard profile player trust smoke**: Owner design QA. Fixture `IB-UX-004`. In a 5-minute keyboard smoke walkthrough, tester can demonstrate light attack, heavy attack, projectile, guard, burst, pause, resume, countdown blocked attack, and pending-release recovery; observed failures must be attributable to documented rules, not unexplained dropped input.
- **AC-IB-54 — Assist data is separated from Standard balance evidence**: Owner production/QA. Fixture `IB-UX-005`. Standard, Alternate, and Accessibility profile smoke/playtest results are labeled separately; Accessibility toggle guard or prompt timing evidence cannot be used as Standard profile combat balance proof.

### Review / Integration Gates

- **AC-IB-55 — Fresh review before implementation authority**: Owner production. This revised GDD is not implementation authority until `/design-review design/gdd/input-buffering.md` resolves blocking issues.
- **AC-IB-56 — Registry sync after approval**: Owner systems/design. After approval, exported constants, schema fields, reason enums, request types, profile constants and profiling knobs from this GDD must be synced to `design/registry/entities.yaml` before implementation tasks consume them.
- **AC-IB-57 — ADR gate awareness**: Owner technical direction. Implementation must not start until input serialization/snapshot, Web focus/audio shell, Godot pause/time-scale and UI context stack ADR gates are resolved or explicitly scoped as prototype-only exceptions.
- **AC-IB-58 — Fixed-runtime dependency remains provisional**: Owner design review. If fresh re-review of `fixed-logic-runtime.md` changes runtime state names, ack wait, recovery pause, catch-up or tick terminology, this GDD must be updated before approval.

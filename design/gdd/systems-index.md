# Systems Index: 星核斗魂

> **Status**: Reviewed — CD-SYSTEMS concerns accepted; GDD authoring in progress
> **Created**: 2026-05-10
> **Last Updated**: 2026-05-11
> **Source Concept**: design/gdd/game-concept.md
> **Art Bible**: design/art/art-bible.md

> **TD-SYSTEM-BOUNDARY**: CONCERNS accepted — keep 30 systems, but explicitly include combat event contract, combat data loading/validation, and MVP scope limits.
> **PR-SCOPE**: CONCERNS accepted — 4-week solo MVP is only realistic if gas projectile, CPU, HUD, VFX, audio, debug, menu, asset pipeline, and accessibility are implemented as minimal playable rules.
> **CD-SYSTEMS**: CONCERNS accepted — system set is creatively valid, with mandatory scope boundaries on movement, defense, projectile, and combo systems.

---

## Creative Director Notes

- **移动与距离控制 / MVP**：小跳和跳跃只服务距离、读招和节奏变化；禁止空中冲刺链、浮空追击、空中长连和复杂空战。
- **防御、格挡与受击反馈 / MVP**：只做基础防御、格挡硬直、受击反馈；禁止上下段、完美格挡、投技拆解、破防槽和多层防反猜拳。
- **气弹 / 能量攻击 / MVP**：只作为可读牵制与节奏变化；禁止对波、蓄力炮、反弹、多弹种或主导胜负的远程系统。
- **短连招与取消规则 / MVP**：必须锁定 2–4 段上限和快速回合重置原则，禁止演变成长连段、自由取消或浮空追击系统。

---

## Overview

《星核斗魂》需要一组小而严密的 2.5D Web 格斗系统，用来验证“读招、防御、抓硬直、短连、气槽爆气反杀”是否好玩。系统设计必须优先保护 60fps Web 运行、键盘输入可信、固定逻辑时序、数据驱动 hitbox/hurtbox、清楚的战斗状态、短连限制、气槽不接管胜负，以及符合 Art Bible 的星核拳馆可读表现。MVP 只做 1 个均衡武斗家、1 个训练木桩、1 个极简脚本 CPU、1 张星核拳馆场景、基础 HUD、最小 VFX/音效和 Web 键盘单人闭环。

---

## MVP Scope Lock

MVP 只验证一个核心假设：玩家能否通过读招、防御、抓硬直、短连和爆气反杀，获得“我读懂了，所以我赢了”的爽感。

### MVP 必须保持最小规则

| 系统 | MVP 范围限制 |
|---|---|
| 气弹 / 能量攻击 | 只做 1 种直线小气弹；不做蓄力、对波、反弹、多弹种。 |
| 简单脚本 CPU | 只做靠近、防御、轻攻击、偶尔气弹；不做行为树、学习型 AI 或复杂难度。 |
| 星核拳馆场景与战斗相机 | 只做 1 张静态场景 + 简单跟随 / 限边相机。 |
| HUD 与战斗信息反馈 | 只显示血量、气槽、计时、胜负、基础命中/防御/满气提示。 |
| VFX 可读性系统 | 只做命中闪光、防御火花、爆气特效、气弹特效。 |
| 动画播放与 sprite 表现 | 只覆盖站立、移动、攻击、防御、受击、胜负、爆气、气弹。 |
| 音效反馈 | 只接入占位事件音效：攻击、防御、命中、受击、爆气、气弹、胜负。 |
| Web 平台壳与浏览器焦点 | 只验证浏览器运行、键盘输入、焦点丢失暂停/提示、音频解锁。 |
| 暂停与基础菜单 | 只做暂停、继续、重开、返回标题；不做设置菜单。 |
| 调参与调试显示 | 只做 hitbox/hurtbox、状态、帧窗口、气槽事件的最小显示。 |
| 美术资产生产管线 | 只定最小命名、尺寸、导入和 AI 生成记录规则。 |
| 基础可访问性与按键提示 | 只做按键提示、可读字体、高对比 HUD；不做完整键位重绑。 |

### Explicitly Not MVP

- 本地双人完整支持。
- 第二个完整角色。
- 能量波对轰完整系统。
- 复杂指令搓招。
- 长连段、自由取消、浮空追击。
- 复杂防御猜拳。
- 剧情模式、关卡推进、角色成长。
- 联网、匹配、排行榜、回放、反作弊。
- 移动端适配。
- 商业级美术音效。

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | 固定逻辑步进与战斗运行时 | Foundation | MVP | Revised after fifth full review — Pending Fresh Re-review | design/gdd/fixed-logic-runtime.md | — |
| 2 | 输入映射与输入缓冲 | Foundation | MVP | Not Started | design/gdd/input-buffering.md | 1 |
| 3 | 角色数据与招式数据 | Foundation | MVP | Not Started | design/gdd/character-move-data.md | — |
| 4 | 动画帧数据与资产元数据 | Foundation | MVP | Not Started | design/gdd/animation-frame-metadata.md | 3, 25 |
| 5 | 移动与距离控制 | Core | MVP | Not Started | design/gdd/movement-spacing.md | 1, 2, 3 |
| 6 | 战斗状态机 | Core | MVP | Not Started | design/gdd/combat-state-machine.md | 1, 2, 3, 4, 5 |
| 7 | Hitbox / Hurtbox 判定 | Core | MVP | Not Started | design/gdd/hitbox-hurtbox.md | 1, 3, 4, 6 |
| 8 | 血量、伤害、计时与胜负 | Core | MVP | Not Started | design/gdd/health-damage-round-rules.md | 1, 3, 6, 7, 10 |
| 9 | 防御、格挡与受击反馈 | Core | MVP | Not Started | design/gdd/guard-block-hit-reaction.md | 3, 6, 7, 8, 10 |
| 10 | 命中停顿与硬直窗口 | Core | MVP | Not Started | design/gdd/hitstop-stun-windows.md | 1, 3, 6, 7 |
| 11 | 短连招与取消规则 | Core | MVP | Not Started | design/gdd/short-combo-cancel-rules.md | 3, 6, 7, 8, 9, 10 |
| 12 | 气槽与爆气反杀 | Core | MVP | Not Started | design/gdd/energy-meter-burst-reversal.md | 3, 6, 7, 8, 9, 10, 11 |
| 13 | 气弹 / 能量攻击 | Feature | MVP | Not Started | design/gdd/energy-projectile.md | 3, 4, 6, 7, 8, 9, 10, 12 |
| 14 | 对局流程与快速重开 | Feature | MVP | Not Started | design/gdd/round-flow-restart.md | 1, 2, 8, 12, 22 |
| 15 | 训练木桩 | Feature | MVP | Not Started | design/gdd/training-dummy.md | 1, 6, 7, 8, 9, 10, 18 |
| 16 | 简单脚本 CPU | Feature | MVP | Not Started | design/gdd/simple-scripted-cpu.md | 1, 5, 6, 7, 8, 9, 10, 11, 12, 14 |
| 17 | 星核拳馆场景与战斗相机 | Presentation | MVP | Not Started | design/gdd/starcore-gym-camera.md | 5, 14, 25 |
| 18 | HUD 与战斗信息反馈 | Presentation | MVP | Not Started | design/gdd/combat-hud-feedback.md | 8, 12, 14, 22 |
| 19 | VFX 可读性系统 | Presentation | MVP | Not Started | design/gdd/readable-combat-vfx.md | 4, 7, 8, 9, 10, 12, 13, 25 |
| 20 | 动画播放与 sprite 表现 | Presentation | MVP | Not Started | design/gdd/sprite-animation-presentation.md | 4, 5, 6, 7, 25 |
| 21 | 音效反馈 | Presentation | MVP | Not Started | design/gdd/combat-audio-feedback.md | 6, 7, 8, 9, 10, 12, 13, 14 |
| 22 | Web 平台壳与浏览器焦点 | Platform | MVP | Not Started | design/gdd/web-platform-shell.md | 1, 2 |
| 23 | 暂停与基础菜单 | Platform | MVP | Not Started | design/gdd/pause-basic-menu.md | 1, 2, 14, 22 |
| 24 | 调参与调试显示 | Tooling | MVP | Not Started | design/gdd/debug-tuning-display.md | 1, 3, 4, 5, 6, 7, 8, 9, 10, 12, 16 |
| 25 | 美术资产生产管线 | Production | MVP | Not Started | design/gdd/art-asset-pipeline.md | design/art/art-bible.md |
| 26 | 基础可访问性与按键提示 | Polish | MVP | Not Started | design/gdd/accessibility-key-prompts.md | 2, 18, 22, 23 |
| 27 | 训练课题 / 挑战 | Polish | Vertical Slice | Not Started | design/gdd/training-challenges.md | 14, 15, 16, 18, 24 |
| 28 | 本地双人对战 | Feature | Alpha | Not Started | design/gdd/local-versus.md | 1, 2, 5, 6, 7, 8, 9, 10, 11, 12, 14, 18, 22 |
| 29 | 多角色 / 角色差异 | Feature | Alpha | Not Started | design/gdd/multiple-characters.md | 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 20, 25 |
| 30 | 能量波对轰 | Feature | Full Vision | Not Started | design/gdd/beam-clash.md | 3, 6, 7, 8, 9, 10, 12, 13, 18, 19, 21 |

---

## Categories

| Category | Description | Typical Systems |
|----------|-------------|-----------------|
| **Foundation** | 其他系统依赖的底层合同 | 固定逻辑、输入、数据、动画元数据、平台壳、资产管线 |
| **Core** | 决定格斗是否成立的核心规则 | 移动、状态机、判定、硬直、伤害、防御、短连、气槽 |
| **Feature** | 基于核心战斗扩展出的可玩功能 | 气弹、对局流程、训练木桩、脚本 CPU、本地双人、多角色 |
| **Presentation** | 将规则结果转译为玩家能看懂/听懂的反馈 | 场景、相机、HUD、VFX、动画、音效 |
| **Platform** | Web 与菜单壳层 | 浏览器焦点、音频解锁、暂停、重开 |
| **Tooling** | 调试和调参支持 | hitbox 显示、状态显示、数值热调 |
| **Production** | 资产生产约束 | 地图、sprite、VFX、UI 的生成与导入规则 |
| **Polish** | 可读性、训练、可访问性和后续体验增强 | 按键提示、训练挑战、辅助提示 |

---

## Priority Tiers

| Tier | Definition | Target Milestone | Design Urgency |
|------|------------|------------------|----------------|
| **MVP** | 没有这些就无法验证“读招 + 短连 + 气槽爆气”是否好玩 | 4 周 Web 可玩原型 | Design FIRST |
| **Vertical Slice** | 让一个完整训练/对战体验更像正式 demo | Vertical slice / demo | Design SECOND |
| **Alpha** | 完整机械范围的粗糙版本 | Alpha milestone | Design THIRD |
| **Full Vision** | 可选热血机制、后期内容和高成本扩展 | Beta / Release | Design only if core is validated |

---

## Dependency Map

### Foundation Layer

1. **固定逻辑步进与战斗运行时** — 战斗时序、tick、更新顺序、战斗事件合同的基础。
2. **输入映射与输入缓冲** — 依赖固定逻辑步进；Web 键盘输入必须先稳定。
3. **角色数据与招式数据** — 招式帧、伤害、硬直、取消、气槽收益的数据源。
4. **动画帧数据与资产元数据** — 依赖角色/招式数据和资产管线；连接 sprite 帧、pivot、VFX 点、判定时间点。
5. **Web 平台壳与浏览器焦点** — 依赖固定逻辑和输入；保证浏览器运行、焦点和音频解锁不破坏战斗。
6. **美术资产生产管线** — 依赖 Art Bible；约束 generate2dmap、generate2dsprite、codex-gateway-imagegen 的资产产出。

### Core Layer

1. **移动与距离控制** — 依赖固定逻辑、输入、角色数据。
2. **战斗状态机** — 依赖固定逻辑、输入、角色数据、动画元数据、移动。
3. **Hitbox / Hurtbox 判定** — 依赖固定逻辑、角色数据、动画元数据、状态机。
4. **命中停顿与硬直窗口** — 依赖固定逻辑、角色数据、状态机、判定。
5. **血量、伤害、计时与胜负** — 依赖状态机、判定、硬直窗口。
6. **防御、格挡与受击反馈** — 依赖状态机、判定、伤害、硬直窗口。
7. **短连招与取消规则** — 依赖状态机、判定、伤害、防御、硬直窗口。
8. **气槽与爆气反杀** — 依赖状态机、判定、伤害、防御、硬直、短连规则。

### Feature Layer

1. **气弹 / 能量攻击** — 依赖状态机、判定、硬直、伤害、防御、气槽；MVP 只做 1 种直线小气弹。
2. **对局流程与快速重开** — 依赖固定逻辑、输入、血量胜负、气槽、Web 平台壳。
3. **训练木桩** — 依赖状态机、判定、伤害、防御、硬直、HUD。
4. **简单脚本 CPU** — 依赖移动、状态机、判定、防御、短连、气槽、对局流程；MVP 只做极简行为表。

### Presentation Layer

1. **星核拳馆场景与战斗相机** — 依赖移动、对局流程、资产管线。
2. **HUD 与战斗信息反馈** — 依赖血量胜负、气槽、对局流程、Web 平台壳。
3. **VFX 可读性系统** — 依赖动画元数据、判定、硬直、防御、气槽、气弹、资产管线。
4. **动画播放与 sprite 表现** — 依赖动画元数据、移动、状态机、判定、资产管线。
5. **音效反馈** — 依赖状态机、判定、伤害、防御、硬直、气槽、气弹、对局流程。
6. **暂停与基础菜单** — 依赖输入、Web 平台壳、对局流程。
7. **调参与调试显示** — 依赖核心战斗系统；MVP 只做关键调试显示。
8. **基础可访问性与按键提示** — 依赖输入、HUD、Web 平台壳、菜单。

### Polish / Later Layer

1. **训练课题 / 挑战** — 依赖对局流程、木桩、CPU、HUD、调试显示。
2. **本地双人对战** — 依赖完整核心战斗、HUD、Web 平台壳。
3. **多角色 / 角色差异** — 依赖角色数据、动画元数据、完整核心战斗、资产管线。
4. **能量波对轰** — 依赖气槽、气弹、判定、VFX、HUD、音效；Full Vision 可选。

---

## Technical Boundary Contracts

### Combat Event Contract

战斗运行时必须发布事实事件，Presentation 系统只能订阅，不得反向驱动战斗逻辑。

示例事件：

- `attack_started`
- `active_frame_started`
- `hit_landed`
- `blocked`
- `whiffed`
- `hitstun_started`
- `blockstun_started`
- `combo_counter_changed`
- `energy_meter_changed`
- `burst_ready`
- `burst_started`
- `round_ended`

HUD、VFX、音效、相机、调试显示只能消费这些事件，不能决定命中、防御、硬直、取消或爆气结果。

### Combat Data Loading and Validation

角色数据、招式数据、动画帧数据、hitbox/hurtbox 必须在加载或调试阶段校验：

- 招式引用的动画是否存在。
- 前摇、活跃、收招帧是否连续且合法。
- hitbox/hurtbox 是否只在合法帧出现。
- 取消窗口是否落在允许阶段内。
- VFX spawn point 是否存在。
- pivot 是否稳定。
- 气槽收益、伤害、硬直是否在安全范围内。

---

## Recommended Design Order

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | 固定逻辑步进与战斗运行时 | MVP | Foundation | systems-designer, technical-director | M |
| 2 | 输入映射与输入缓冲 | MVP | Foundation | systems-designer, ux-designer | M |
| 3 | 角色数据与招式数据 | MVP | Foundation | systems-designer, gameplay-programmer | M |
| 4 | 动画帧数据与资产元数据 | MVP | Foundation | systems-designer, technical-artist | M |
| 5 | 移动与距离控制 | MVP | Core | systems-designer, gameplay-programmer | M |
| 6 | 战斗状态机 | MVP | Core | systems-designer, gameplay-programmer | L |
| 7 | Hitbox / Hurtbox 判定 | MVP | Core | systems-designer, gameplay-programmer | L |
| 8 | 命中停顿与硬直窗口 | MVP | Core | systems-designer, gameplay-programmer | M |
| 9 | 血量、伤害、计时与胜负 | MVP | Core | systems-designer, game-designer | M |
| 10 | 防御、格挡与受击反馈 | MVP | Core | systems-designer, game-designer | M |
| 11 | 短连招与取消规则 | MVP | Core | systems-designer, game-designer | M |
| 12 | 气槽与爆气反杀 | MVP | Core | systems-designer, game-designer | L |
| 13 | 对局流程与快速重开 | MVP | Feature | game-designer, ux-designer | S |
| 14 | 训练木桩 | MVP | Feature | systems-designer, qa-tester | S |
| 15 | 气弹 / 能量攻击 | MVP | Feature | systems-designer, technical-artist | S |
| 16 | 简单脚本 CPU | MVP | Feature | ai-programmer, systems-designer | M |
| 17 | HUD 与战斗信息反馈 | MVP | Presentation | ux-designer, ui-programmer | M |
| 18 | 动画播放与 sprite 表现 | MVP | Presentation | technical-artist, gameplay-programmer | M |
| 19 | VFX 可读性系统 | MVP | Presentation | technical-artist, art-director | M |
| 20 | 星核拳馆场景与战斗相机 | MVP | Presentation | level-designer, technical-artist | M |
| 21 | 音效反馈 | MVP | Presentation | sound-designer, audio-director | S |
| 22 | Web 平台壳与浏览器焦点 | MVP | Platform | ux-designer, godot-specialist | S |
| 23 | 暂停与基础菜单 | MVP | Platform | ux-designer, ui-programmer | S |
| 24 | 调参与调试显示 | MVP | Tooling | tools-programmer, gameplay-programmer | S |
| 25 | 美术资产生产管线 | MVP | Production | art-director, technical-artist | M |
| 26 | 基础可访问性与按键提示 | MVP | Polish | accessibility-specialist, ux-designer | S |
| 27 | 训练课题 / 挑战 | Vertical Slice | Polish | systems-designer, ux-designer | M |
| 28 | 本地双人对战 | Alpha | Feature | network-programmer, gameplay-programmer | L |
| 29 | 多角色 / 角色差异 | Alpha | Feature | systems-designer, art-director | L |
| 30 | 能量波对轰 | Full Vision | Feature | systems-designer, technical-artist | L |

---

## Circular Dependencies

No blocking circular dependencies found.

### Pseudo-cycle: Animation Frame Metadata ↔ Hitbox / Hurtbox

Animation and hitbox data both need to know startup, active, recovery, VFX points, and contact timing.

**Resolution**: Define frame phases in `角色数据与招式数据`; animation metadata and hitbox/hurtbox both reference that contract.

### Pseudo-cycle: VFX ↔ Combat Feedback

VFX needs combat events to show hits, blocks, bursts, and whiffs; combat feedback needs VFX to be readable.

**Resolution**: Core combat publishes event facts; VFX subscribes and displays them, but never decides combat results.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|------------------|------------|
| 固定逻辑步进与战斗运行时 | Technical | Web 下帧率波动会破坏硬直、输入缓冲、连招可信度。 | 早期做浏览器导出验证；固定 tick；事件日志可检查。 |
| 输入映射与输入缓冲 | Technical / Feel | 浏览器键盘冲突、焦点丢失、延迟会让格斗手感失真。 | 优先测试真实浏览器；明确失焦暂停和重新聚焦提示。 |
| 战斗状态机 | Technical / Design | 状态优先级混乱会导致取消、受击、爆气、格挡互相冲突。 | 先写明确状态优先级矩阵和非法转移规则。 |
| Hitbox / Hurtbox 判定 | Technical / Feel | 视觉与判定不一致会破坏“读招定胜负”。 | 数据驱动判定；调试显示；命中点和动画帧校验。 |
| 命中停顿与硬直窗口 | Feel | 命中停顿太短没爽感，太长破坏节奏。 | 用公式定义范围；第一版调小表驱动，逐步调参。 |
| 气槽与爆气反杀 | Design / Balance | 气槽过强会变成“按爆气就赢”。 | 限制持续时间、收益和触发条件；无气也能公平获胜。 |
| 简单脚本 CPU | Scope / Feel | CPU 太蠢无法测试，太复杂会吞掉时间。 | 极简行为表；只验证靠近、攻击、防御、偶尔气弹。 |
| VFX 可读性系统 | Visual / Performance | 能量特效遮挡拳脚或 Web overdraw 过高。 | 遵守 Art Bible；短时局部 VFX；限制粒子和透明层。 |
| 美术资产生产管线 | Production | AI 生成 sprite 可能帧间不一致或有 IP 风险。 | 使用 Art Bible、generate2dmap/generate2dsprite、codex-gateway-imagegen 记录和验收 checklist。 |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 30 |
| Design docs started | 1 |
| Design docs reviewed | 1 |
| Design docs approved | 0 |
| MVP systems designed | 1/26 |
| Vertical Slice systems designed | 0/1 |
| Alpha systems designed | 0/2 |
| Full Vision systems designed | 0/1 |

---

## Next Steps

- [x] Run CD-SYSTEMS review in Full mode.
- [x] Start first MVP GDD: `/design-system fixed-logic-runtime`.
- [ ] Re-run `/design-review design/gdd/fixed-logic-runtime.md` in a fresh session after the NEEDS REVISION fixes.
- [ ] After re-review approval, run `/map-systems next` to pick the highest-priority undesigned system (`input-buffering`).
- [ ] Run `/gate-check pre-production` when MVP systems are designed and reviewed.

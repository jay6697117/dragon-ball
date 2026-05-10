# Game Concept: 星核斗魂

*Created: 2026-05-10*
*Status: Draft*

---

## Elevator Pitch

> 《星核斗魂》是一款原创 Web 2.5D 热血能量格斗原型。玩家操控一名“星核气”均衡武斗家，在星核拳馆中通过移动、试探、防御、抓硬直、短连招和爆气反杀，验证“读招定胜负”的硬派格斗核心是否好玩。

---

## Core Identity

| Aspect | Detail |
| ---- | ---- |
| **Genre** | 2.5D 横向格斗 / 热血能量格斗 / Web 原型 |
| **Platform** | Web 首发原型 |
| **Target Audience** | 喜欢《拳皇》《街霸》式读招、反击、短连练习和本地格斗手感的中核玩家 |
| **Player Count** | MVP：单人键盘，对训练木桩 / 简单脚本 CPU；本地双人延后 |
| **Session Length** | MVP 单局 30-60 秒；单次测试 5-15 分钟 |
| **Monetization** | 暂无；原型阶段不考虑商业化 |
| **Estimated Scope** | Medium（6-9 个月，solo，全愿景）；MVP 为 4 周 Web 原型 |
| **Comparable Titles** | 《拳皇》《街霸》《龙珠斗士Z》（仅参考高速能量武斗观感，不复刻 IP、角色、招式或视觉） |

---

## Core Fantasy

玩家幻想是：成为一名原创世界里的超级武斗家，不靠数值碾压或无脑大招，而是通过观察距离、识别起手、挡住攻击、抓住硬直，在绝境中用爆气完成反杀。

这款游戏承诺的不是“我按了大招所以赢”，而是“我读懂了对手，所以我赢”。热血能量、气弹、爆气和必杀都服务于这个判断过程，让玩家在技术成长中获得强烈的战斗爽感。

---

## Unique Hook

《星核斗魂》像《拳皇》《街霸》一样强调距离、起手、硬直和确反，**并且**把原创热血能量表现压缩成可读、可控、辅助性的战斗资源：气槽不接管胜负，只在关键时刻制造爆气反杀的情绪峰值。

核心差异点：它不是完整商业格斗游戏，也不是动漫大招轰炸游戏，而是一个小范围、强手感、低内容量的 Web 格斗原型，专门验证“读招 + 短连 + 辅助气槽 + 爆气反杀”的闭环。

---

## Player Experience Analysis (MDA Framework)

### Target Aesthetics (What the player FEELS)

| Aesthetic | Priority | How We Deliver It |
| ---- | ---- | ---- |
| **Challenge**（挑战 / 精通） | 1 | 通过可读起手、硬直、格挡、反击、短连和气槽判断，让玩家靠技术成长获胜 |
| **Fantasy**（力量幻想） | 2 | 通过星核气、爆气反杀、气弹、能量高光，让玩家感觉自己是原创热血武斗家 |
| **Sensation**（感官刺激） | 3 | 通过命中停顿、打击音效、可读能量特效、爆气危险色制造冲击感 |
| **Expression**（表达） | 4 | 通过连招选择、气槽使用时机、进攻/防守风格和后续配色表达玩家风格 |
| **Fellowship**（社交连接） | Supporting | 后续可通过本地双人对战支持朋友间切磋；MVP 暂缓 |
| **Discovery**（发现） | Minimal | 主要发现来自理解招式窗口和反击机会，不做探索内容 |
| **Narrative**（叙事） | Minimal | MVP 不做剧情；世界观只作为视觉和角色设定背景 |
| **Submission**（放松） | N/A | 本项目不是放松型游戏，核心是挑战和学习 |

### Key Dynamics (Emergent player behaviors)

- 玩家会先观察对手起手，而不是无脑按攻击。
- 玩家会用轻击试探、用重击惩罚、用气弹牵制。
- 玩家会在失败后思考：“我刚才是没挡住、出招太慢，还是被对方抓了硬直？”
- 玩家会练习稳定 2-4 段短连，而不是背长连段表演。
- 玩家会把气槽留给关键时刻：爆气反杀、脱困、或扩大一次已经读对的优势。

### Core Mechanics (Systems we build)

1. **硬派读招攻防**：可见前摇、明确攻击范围、格挡、闪避/走位、收招硬直和反击窗口。
2. **短连招系统**：固定 2-4 段短连，例如轻 → 轻 → 重、轻 → 重 → 爆气反击。
3. **辅助气槽系统**：通过命中、受击、格挡少量获得，用于爆气反杀和后续必杀，不主导胜负。
4. **Web 键盘单人原型**：方向键 + 轻击 / 重击 / 气弹 / 防御 / 爆气，避免复杂搓招。
5. **训练木桩与简单脚本 CPU**：先验证手感、反馈、硬直和基础攻防，不做完整 AI。

---

## Player Motivation Profile

### Primary Psychological Needs Served

| Need | How This Game Satisfies It | Strength |
| ---- | ---- | ---- |
| **Autonomy**（自主） | 玩家选择进攻、后撤、防御、气弹牵制、爆气反杀的时机 | Supporting |
| **Competence**（能力成长） | 玩家通过读招、抓硬直、短连稳定性和气槽判断感到自己变强 | Core |
| **Relatedness**（连接） | MVP 主要是单人；后续本地双人可增强朋友间切磋感 | Minimal now / Supporting later |

### Player Type Appeal (Bartle Taxonomy)

- [x] **Achievers**（目标完成 / 技术成长）— 通过练会短连、看懂前摇、完成反击和爆气反杀获得成就感。
- [ ] **Explorers**（探索系统）— 不是主要目标，但玩家会探索攻击距离、硬直窗口和气槽时机。
- [ ] **Socializers**（社交）— MVP 暂不主打；本地双人加入后会增强朋友对战。
- [x] **Killers/Competitors**（竞争者）— 后续本地双人会服务竞争和对抗；MVP 先用 CPU/木桩验证基础。

### Flow State Design

- **Onboarding curve**：先在训练木桩中学习移动、轻击、重击、防御、气弹、爆气；再进入简单脚本 CPU 对战。
- **Difficulty scaling**：MVP 不做多难度；通过 CPU 出招频率、防御概率和反击概率做最小调参。
- **Feedback clarity**：命中、格挡、打空、受击硬直、爆气状态必须有清楚动画、音效、特效和 HUD 反馈。
- **Recovery from failure**：单局 30-60 秒，失败后快速重开，让玩家马上尝试新的防守和反击判断。

---

## Core Loop

### Moment-to-Moment (30 seconds)

玩家观察距离和对手起手动作，用移动、跳跃/小跳、短冲刺、轻击、防御和气弹试探。发现对手破绽后，玩家抓住硬直打出 2-4 段短连；如果气槽足够，则选择爆气反杀或强化一次反击。核心体验是：看懂、挡住、反击、重置，再进入下一轮判断。

### Short-Term (5-15 minutes)

玩家在训练木桩或简单 CPU 中连续进行多场短对局。每一局围绕一个小学习点：距离控制、轻击试探、重击惩罚、防御反击、气弹牵制、爆气反杀。玩家失败后应能看懂原因，并产生“下一局我能打得更好”的想法。

### Session-Level (30-120 minutes)

MVP 的真实测试会更短，通常是 5-15 分钟；完整会话可以是 30 分钟左右：

1. 进入训练木桩熟悉按键和反馈。
2. 对简单脚本 CPU 打数场 30-60 秒短局。
3. 练会一套短连或成功完成一次爆气反杀。
4. 发现某个动作太强、太弱或不清楚，记录调参反馈。

自然停止点是打完数场短对局；回归理由是“我想再练一次反击和爆气时机”。

### Long-Term Progression

MVP 不做攻击力、防御力、装备、等级和数值养成。长期成长来自玩家技术：更懂距离、更会防御、更会抓硬直、更能稳定短连、更能判断何时爆气。

后续可以解锁训练课题、招式挑战、角色配色、录像标记，但不做破坏公平性的数值成长。

### Retention Hooks

- **Curiosity**：玩家想知道自己能否更稳定地挡住某个起手、反击某个破绽。
- **Investment**：玩家投入在短连练习、反击时机和爆气判断上。
- **Social**：后续本地双人加入后，朋友切磋会成为回归动力。
- **Mastery**：最大回归动力是“再打一局我能读得更准”。

---

## Game Pillars

### Pillar 1: 读招定胜负

胜负主要来自观察距离、识别起手、抓硬直惩罚，而不是靠数值、复杂连段或大招压制。

*Design test*: 如果在“招式更华丽”和“动作更清楚”之间争论，选择动作更清楚；强招必须有可见起手、风险或反制窗口。

### Pillar 2: 短连招，高回合

连招要有打击爽感，但必须短，让玩家频繁回到移动、防守、试探、惩罚的核心决策。

*Design test*: 如果在“连段更长”和“玩家更快恢复控制”之间争论，选择更快恢复控制；基础连段以 2-4 段为宜。

### Pillar 3: 气槽辅助，不接管战斗

气槽用于强化、脱困、小爆发和必杀，但不能替代立回和读招。

*Design test*: 如果在“气槽大招更强”和“无气也能公平获胜”之间争论，选择无气也能公平获胜；耗气效果不能按了就赢。

### Pillar 4: 第一回合必须好玩

第一版只验证移动、试探、防守、反击、短连和气槽爆发这一条核心循环是否好玩。

*Design test*: 如果在“加角色/剧情/联网/多场景”和“打磨 1 个完整角色 + 镜像/木桩/脚本 CPU、1 个场景、基础攻防”之间争论，选择后者。

### Pillar 5: 热血能量必须可读

原创能量武斗感负责情绪、辨识度和幻想，但特效不能遮挡判断信息。

*Design test*: 如果在“特效更炸裂”和“命中、防御、打空、硬直更易读”之间争论，选择更易读；气弹、爆气、能量波的起手、轨迹、命中、被防、打空结果必须在 0.5 秒内可辨认。

### Anti-Pillars (What This Game Is NOT)

- **NOT 现成动漫/IP 复刻**：不复刻现成动漫/IP 的角色、招式、剧情、名称或视觉，因为它会破坏原创安全可发布目标。
- **NOT 复杂空战**：不做飞行、二段跳、空中冲刺链、空中长追击，因为它会破坏硬派读招优先和 Weeks 原型范围。
- **NOT 长连段表演游戏**：不做长连段、自由取消、浮空无限追击、墙角压制表演，因为它会把胜负从读招变成背连招。
- **NOT 气槽主导胜负**：不做多层能量、攒满秒杀、觉醒套觉醒，因为气槽只能辅助战术判断。
- **NOT 大规模多角色格斗**：MVP 不做多角色和每人一套专属复杂机制，因为动画、判定和平衡成本会倍增。
- **NOT 线上竞技产品**：不做联网、匹配、排行榜、回放、反作弊，因为手感没验证前不投入网络系统。
- **NOT 复杂防御猜拳**：不做上下段、完美格挡、防反、投技拆解、破防槽，因为第一版必须让失败原因容易理解。

---

## Visual Identity Anchor

### Direction Name

**星核拳馆**

### One-Line Visual Rule

所有视觉都像“街机拳馆中的恒星能量训练者”，先拳脚、后能量。

### Supporting Visual Principles

1. **角色剪影必须原创**
   - 以拳击、散打、护具、绑带、短外套为主要语言。
   - *Design test*: 遮掉颜色后，也不能像任何现成动漫角色；禁止尖刺爆发发型、橙蓝道服配色、现成角色轮廓。

2. **能量只强化动作信息**
   - 星核气集中在拳脚末端、气弹轨迹、防御反馈、爆气状态，不覆盖身体和判定。
   - *Design test*: 能量特效不能遮住起手、攻击范围、受击硬直和格挡状态。

3. **特效必须帮助读招**
   - 玩家必须能快速区分轻击、重击、气弹、防御、爆气和必杀。
   - *Design test*: 每个关键交互必须在 0.5 秒内能看懂命中、被挡、打空或被惩罚。

### Color Philosophy Summary

- 主背景：深靛、炭黑，保持拳馆和 Web 原型的清晰对比。
- 能量高光：星青、热白，代表星核气的稳定输出。
- 危险/爆气状态：少量赤橙，只在爆气反杀和高风险状态出现。
- 禁止方向：七龙珠式橙蓝道服配色、龟派波式经典构图、现成角色发型和标志性招式视觉。

### Asset Pipeline Implication

- 地图资产优先使用 `.claude/skills/generate2dmap` 生成 1 张“星核拳馆”Web 原型场景。
- 角色、美术资产和动画相关素材优先使用 `.claude/skills/generate2dsprite` 生成：1 个均衡武斗家、镜像版、训练木桩、基础攻击帧、爆气反杀特效帧。
- 第一版必须少动画、强轮廓、固定色板；判定盒以设计数据为准，不依赖画面边缘。

---

## Inspiration and References

| Reference | What We Take From It | What We Do Differently | Why It Matters |
| ---- | ---- | ---- | ---- |
| 《拳皇》 | 本地格斗、短局节奏、角色对抗、读招与惩罚 | MVP 不做三人队伍和完整角色池，只做 1 角色 + 镜像/木桩/脚本 CPU | 验证硬派格斗的“再来一局”心理 |
| 《街霸》 | 距离控制、起手判断、格挡、确反、训练价值 | 降低复杂指令，Web 首版采用直接按键 | 保护读招和能力成长感 |
| 《龙珠斗士Z》 | 高速能量武斗的观感和情绪峰值 | 不复刻 IP、角色形象、招式、发型、服装、配色或剧情；只吸收抽象能量战斗幻想 | 给热血能量表现提供方向，同时保持原创安全 |

**Non-game inspirations**：街机拳馆、地下训练赛、拳击护具、散打绑带、恒星核心、星云高光、夜间霓虹训练馆。

---

## Target Player Profile

| Attribute | Detail |
| ---- | ---- |
| **Age range** | 16-35 |
| **Gaming experience** | 中核到硬核；愿意练习基础格斗技巧 |
| **Time availability** | 5-15 分钟快速试玩；后续可进行 30 分钟练习或切磋 |
| **Platform preference** | Web Demo；键盘单人首测 |
| **Current games they play** | 《拳皇》《街霸》；可能也熟悉动漫格斗游戏 |
| **What they're looking for** | 小而清楚的硬派格斗手感，带一点原创热血能量幻想 |
| **What would turn them away** | 输入延迟、反馈不清、特效遮挡、长连段失控、气槽无脑碾压、IP 山寨感 |

---

## Technical Considerations

| Consideration | Assessment |
| ---- | ---- |
| **Recommended Engine** | 待 `/setup-engine` 决策；重点比较 Web 导出成熟度、2D/2.5D 动画管线、固定逻辑步进、输入缓冲、判定盒编辑、资源体积和调试工具 |
| **Key Technical Challenges** | 浏览器 60fps、输入延迟、固定逻辑步进、硬直/命停一致性、键盘按键冲突、可读特效、脚本 CPU 稳定性 |
| **Art Style** | 2D sprite + 2.5D 场景表现；星核拳馆视觉锚点 |
| **Art Pipeline Complexity** | Medium：地图用 generate2dmap；角色/精灵/动画资产用 generate2dsprite；需要人工筛选一致性和可读性 |
| **Audio Needs** | Moderate：轻击、重击、格挡、受击、气弹、爆气、胜负反馈都需要清楚音效 |
| **Networking** | None for MVP；联网明确延后 |
| **Content Volume** | MVP：1 个角色、镜像/木桩/脚本 CPU、1 张场景、1 套基础 HUD、少量基础音效/特效 |
| **Procedural Systems** | None |

---

## Risks and Open Questions

### Design Risks

- **读招不够清楚**：如果玩家看不懂起手、硬直和反击窗口，核心乐趣失败。
- **气槽过强**：如果爆气或必杀按了就赢，会破坏硬派读招。
- **短连太长或太短**：太长会让新手失控，太短会缺少爽感。
- **CPU 行为误导手感**：如果 CPU 太蠢或像作弊读输入，会破坏学习体验。

### Technical Risks

- **Web 输入/渲染延迟**：浏览器延迟和帧率波动会直接破坏格斗判断。
- **固定逻辑步进和 60fps**：必须稳定，否则硬直、连招和防御时机会不可信。
- **生成资产一致性**：generate2dsprite 可能出现角色帧间体积、朝向、轮廓不稳定，需要控制色板和帧数。
- **特效可读性**：能量特效如果遮挡起手、判定或硬直，会破坏支柱。

### Market Risks

- **格斗游戏门槛高**：目标玩家较明确，但新手可能因为挑战而流失。
- **同类大作强势**：完整商业格斗游戏已有成熟作品，因此本项目必须先作为小而清楚的原创原型成立。
- **IP 误解风险**：如果视觉太像现成动漫角色，会被认为是山寨或侵权。

### Scope Risks

- **想要三种模式同时完整**：训练木桩、本地双人、CPU 都完整做会超出 Weeks 范围；MVP 已降级为键盘单人 + 木桩 + 简单脚本 CPU。
- **热血爆发表现膨胀**：爆气、变身、大招、能量波对轰很容易拖进特效黑洞。
- **多角色冲动**：第二个完整角色会让动画、判定、平衡和测试成本翻倍。

### Open Questions

- **哪个引擎最适合 Web 2.5D 格斗原型？** 通过 `/setup-engine` 决策。
- **爆气反杀的具体规则是什么？** 在战斗系统 GDD 中定义：触发条件、持续时间、是否无敌、是否打断受击、伤害倍率。
- **脚本 CPU 的最小行为集是什么？** 在 AI/战斗原型中验证：靠近、攻击、防御、低频反击即可。
- **生成 sprite 的动画帧数和判定盒标准是什么？** 在 art bible 和战斗系统 GDD 中锁定。

---

## MVP Definition

**Core hypothesis**：玩家能否在 Web 键盘单人原型中，通过观察敌人起手、防御/闪避、抓硬直反击、打出短连，并用一次爆气反杀获得“我读懂了，所以我赢了”的爽感？

**Required for MVP**:
1. Web 可运行，浏览器稳定 60fps，固定逻辑步进。
2. 键盘单人输入：移动、跳跃/小跳、短冲刺、轻击、重击、气弹、防御、爆气。
3. 1 个完整均衡武斗家，镜像/木桩/脚本 CPU 复用同一基础规则。
4. 1 张“星核拳馆”场景。
5. 基础战斗规则：血量、计时、胜负、前摇、命中停顿、受击硬直、格挡硬直、收招硬直。
6. 2-4 段短连，优先 2 段稳定短连。
7. 气槽与爆气反杀：满气后一次低复杂度、强可读的短时强化/反击机制。
8. 训练木桩能显示命中、格挡、硬直状态。
9. 简单脚本 CPU：靠近、攻击、偶尔防御/反击，不做完整 AI。

**Explicitly NOT in MVP**:
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

### Scope Tiers (if budget/time shrinks)

| Tier | Content | Features | Timeline |
| ---- | ---- | ---- | ---- |
| **MVP** | 1 角色、镜像/木桩/脚本 CPU、1 张星核拳馆场景 | Web 键盘单人、基础移动、轻/重/气弹、防御、硬直、2 段短连、气槽、爆气反杀、基础胜负 | 4 周 |
| **Stabilized Prototype** | MVP 内容加更多反馈和调参 | 3-4 段短连、更多 CPU 行为、爆气表现增强、更多音效/打击反馈、基础教程提示 | 6 周 |
| **Vertical Slice** | 1 个完整角色 + 更完整训练/CPU体验 + 1-2 个场景变体 | 更稳的 HUD、训练课题、简单菜单、更多动画 polish、本地双人可作为候选但需核心稳定后再加入 | 2-3 个月 |
| **Alpha** | 2-3 个原创角色、更多场景、基础菜单和测试流程 | 本地双人、角色差异初版、训练挑战、基础调参工具、更多音效和视觉反馈 | 3-6 个月 |
| **Full Vision** | 4-6 个原创角色、多个拳馆/赛事场景、完整 Web Demo 体验 | 本地双人、脚本 CPU 多行为、训练挑战、可选能量波对轰、配色解锁、完整 UI/音效/视觉 polish；不承诺联网 | 6-9 个月，solo |

---

## Next Steps

1. Run `/setup-engine` to configure the engine and populate version-aware reference docs.
2. Run `/art-bible` to create the visual identity specification — do this BEFORE writing GDDs. The art bible gates asset production and shapes technical architecture decisions (rendering, VFX, UI systems).
3. Use `/design-review design/gdd/game-concept.md` to validate concept completeness before going downstream.
4. Discuss vision with the `creative-director` agent for pillar refinement.
5. Decompose the concept into individual systems with `/map-systems` — maps dependencies, assigns priorities, and creates the systems index.
6. Author per-system GDDs with `/design-system` — guided, section-by-section GDD writing for each system identified in step 4.
7. Plan the technical architecture with `/create-architecture` — produces the master architecture blueprint and Required ADR list.
8. Record key architectural decisions with `/architecture-decision (×N)` — write one ADR per decision in the Required ADR list from `/create-architecture`.
9. Validate readiness to advance with `/gate-check` — phase gate before committing to production.
10. Prototype the riskiest system with `/prototype [core-mechanic]` — validate the core loop before full implementation.
11. Run `/playtest-report` after the prototype to validate the core hypothesis.
12. If validated, plan the first sprint with `/sprint-plan new`.

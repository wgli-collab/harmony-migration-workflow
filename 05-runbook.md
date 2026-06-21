# 05 — Runbook: Group B Execution Procedure

## Architecture: Three-Loop Dynamic Workflow

> **07 Support Agent** (side-channel, consulted throughout): All non-code questions — DevEco ops, environment errors, API lookups, function explanations, config debugging. Code tasks → 01-06 only.

```
Phase 1: PLAN
    │
    ▼
┌─ Loop 1: Per-File Micro-Loop (Phase 2 BUILD) ────────────────┐
│                                                               │
│  for each file in dependency_order:                           │
│    Write .ets ──► Auditor(R1-R29+G1-G11) ──► Build(hvigorw)   │
│        ▲              │                 │                     │
│        │         violations?          errors?                 │
│        │          yes → fix          yes → fix                │
│        │              │                 │                     │
│        └──────────────┴─────────────────┘                     │
│                   retry (max 3x)                              │
│                                                               │
│  收敛: Auditor PASS + Build 0 error                            │
│  动态触发: 同类违规≥2文件 / 单文件≥2轮 / 依赖错误              │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
Phase 3: VERIFY (full build)
    │
    ▼
┌─ Loop 2: Audit-Fix-Reaudit (Phase 4 PROJECT AUDIT) ──────────┐
│                                                               │
│  // 外循环（max 5 轮）                                        │
│  1. 阶段一：L1 (源库对齐) + L2 (独立对抗) — 全量扫描          │
│  2. 阶段二：P1 (语义) + P2 (结构) + P3 (补漏) — 窄聚焦深度扫描 │
│  3. 分层建议 (P0/P1/P2/P3)                                    │
│  4. 人工决策 (✅/❌/⏭️/📌)                                     │
│                                                               │
│  // 内循环（修复 ⇄ 增量重审，收敛驱动）                        │
│  while (有 ✅ 采纳项):                                        │
│    5. ✅ 项 → Loop 1 (逐文件修复)                              │
│    6. 对修改文件增量重审 L1+L2 + P2 结构                       │
│       → 有新 Critical/High → 追加 → 人工审核 → 继续内循环     │
│       → 无新 Critical/High → 内循环干净 → 退出                │
│                                                               │
│  内循环干净 + 无新 Critical/High → CP4 → 退出                 │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌─ Loop 3: Global Test Loop (Phase 5 TEST) ────────────────────┐
│                                                               │
│  Write tests ──► Auditor ──► Build entry ──► Runtime          │
│      ▲              │            │              │             │
│      │         violations?    errors?       failures?         │
│      │          yes → fix    yes → fix     yes → analyze      │
│      │              │            │              │             │
│      └──────────────┴────────────┴──────────────┘             │
│                    retry (no hard cap)                        │
│                                                               │
│  收敛: error/failure 单调递减 + 全部 PASS                      │
│  动态触发: 根因分类(library vs test) / 2轮不降则暂停           │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
Phase 6: COLLECT
```

| 维度 | Loop 1 (Micro) | Loop 2 (Audit) | Loop 3 (Macro) |
|------|:---:|:---:|:---:|
| 粒度 | 单文件 | 项目级（双层嵌套：外 5 轮 + 内收敛） | 整体 entry 模块 |
| 反馈源 | Auditor + Compiler | Auditor 两阶段(L1+L2+P1+P2+P3) + Compiler | Auditor + Compiler + Runtime |
| 失败回退范围 | 只改当前文件 | 修复 → 增量重审 → 再修复 → 再重审（内循环至干净） | 可能涉及 library + test |
| 硬上限 | 3 轮 → 未通过则转入 Phase 4 审计（不静默失败） | 外循环 5 轮；内循环收敛驱动无硬上限 | 不限（收敛性判断） |
| 动态干预重点 | Executor prompt 补丁 | 人工审核决策（内循环每轮新增项也需审核） | 根因分类（test vs library） |

---

## CP Checkpoints 速查

| CP | Phase 边界 | 检查项 | 跳过后果 |
|----|-----------|--------|---------|
| CP0 | Session 启动 → Phase 1 | 文档加载：6 文档（按 Session 变体）全部 Read + 📂 LOADED 确认行 | 关键属性错误 → Phase 2-5 行为异常 |
| CP1 | 1→2 | Plan 完整性：9 自动 + 1 人工 | Plan 有误 → Phase 2 批量翻车 |
| CP2 | 2→3 | Build 完成：文件数 / 指标记录 / G2 触发 | 缺指标 → 无法对照 |
| CP3 | 3→4 | 编译验证：build/HAR/依赖/扫描/导出/风险 (6 项) | 库不稳定 → 审计白费 |
| CP4 | 4→5 | 审计闭环：两阶段审计(L1+L2+P1+P2+P3)完成 + 修复队列清空 + 增量重审无新 Critical/High | 遗留安全问题 → 测试白测 |
| CP5 | 5→6 | 运行时验证 + Library 变更审计 | 运行未验证 → 报告缺关键数据 |

---

## 07 Trigger Map — AI Hint Points

During Loop execution, when the AI encounters specific error patterns, it will output a directed hint. The operator decides whether to pause and look up `07-environment.md`.

| # | Error Pattern / Scenario | AI Hint | → 07 § |
|---|--------------------------|---------|--------|
| E1 | `hvigorw: command not found` / `DEVECO_SDK_HOME` not set | `💡 07 §1 — 环境变量问题，建议 @07-environment.md §1.1 后继续` | §1.1 |
| E2 | Error code `11211120` (resource pack error) | `💡 07 §6 — 资源文件错误，建议 @07-environment.md §6 后继续` | §6 |
| E3 | Error code `00401021` / `00500001` (module config) | `💡 07 §5 — module.json5 配置错误，建议 @07-environment.md §5 后继续` | §5 |
| E4 | Build error mentioning `signingConfig` / `compileSdkVersion` / `deviceTypes` / `runtimeOS` | `💡 07 §4 — 工程配置魔法字符串，建议 @07-environment.md §4 后继续` | §4 |
| E5 | DevEco Studio won't run / emulator not responding / app doesn't launch | `💡 07 §2 — DevEco/模拟器问题，建议 @07-environment.md §2 后继续` | §2 |
| E6 | Unsure if an ArkTS API exists / function explanation needed | `💡 07 §3 — API 兼容性查询，建议 @07-environment.md §3` | §3 |
| E7 | `hdc` not found / device not detected | `💡 07 §1.2 — hdc 工具链问题，建议 @07-environment.md §1.2 后继续` | §1.2 |
| E8 | npm registry / dependency download fails | `💡 07 §7 — 依赖/registry 问题，建议 @07-environment.md §7 后继续` | §7 |
| E9 | Build error persists ≥2 rounds with same message（非代码违规——配置/环境/签名类错误，文件未被修改） | `💡 本轮 error 连续出现，非代码违规，建议 @07-environment.md 按错误码索引排查` | §8 |
| E10 | `arkts-no-any-unknown` / `ESObject type is restricted` / `catch (error: Error)` | `💡 07 §9.1 — 类型系统错误，建议 @07-environment.md §9.1` | §9.1 |
| E11 | `Object literal must correspond to...` / object spread type error | `💡 07 §9.2 — 对象字面量/接口错误，建议 @07-environment.md §9.2` | §9.2 |
| E12 | `Function return type inference is limited` / `Function.bind` / `Cannot find name 'this'` | `💡 07 §9.3 — 函数/this 错误，建议 @07-environment.md §9.3` | §9.3 |
| E13 | `Property 'X' does not exist on type 'typeof window'` / API type mismatch | `💡 07 §9.5 — API 兼容性错误，建议 @07-environment.md §9.5` | §9.5 |
| E14 | `Color.X does not exist` / `fontColor` on container / `@Entry` duplicate / `IDataSource` required | `💡 07 §9.6 — UI 组件错误，建议 @07-environment.md §9.6` | §9.6 |
| E15 | Build 错误连续 ≥2 轮完全相同（**确认文件已修改但错误不变**——代码级错误，疑似 daemon 缓存） | `💡 07 §1.4 — hvigor daemon 缓存，建议杀 daemon + 清理 build 缓存后重试` | §1.4 |
| E16 | `arkts-no-obj-literals-as-types` / `arkts-no-implicit-return` / `@State` on non-struct / `@StorageLink` missing default | `💡 07 §9.4 — 类型系统限制，建议 @07-environment.md §9.4` | §9.4 |

> **AI does NOT load 07 automatically.** AI 仅输出 💡 提示。操作者回复 `fix` 时，AI 加载 07-environment.md 对应节次并执行修复。若 07 未覆盖该问题（对应节次无匹配方案），AI 不可仅回复"07 未覆盖"就放弃——必须自行搜索/查证找到解决方案，修复后将该问题追加到 07 Appendix。操作者回复 `recheck` 时，AI 重跑当前 CP（不加载 07，操作者已手动修复）。操作者回复 `Next` 时，忽略提示继续。

---

## Session Setup

1. Open the **Group B Claude Code session** in VS Code
2. Target library repo is at `{{WORK_DIR}}/{{LIBRARY_NAME}}/` (baseline branch), e.g. `~/harmony-migration/axios/`
3. Working directory: create new directory `{{ARKTS_PROJECT}}/`, e.g. `~/output/axios-arkts/`
4. Load the prompt files in order as each phase begins
5. **Before any build command**: confirm `DEVECO_SDK_HOME` is set and `hvigorw` path is correct (see `07-environment.md` §1.1)
6. **Support Agent `07-environment.md`**: ALL non-code questions go here — DevEco how-to, API queries, function explanations, env errors, config debugging. Do NOT pollute agent prompts 01-06 with these.

---

## 全局交互规则

**语言。** 所有面向人工的输出（G1 门控行、心跳、等待提示、报告、询问、Phase 过渡、CP 结果、Token 检查点）一律使用中文。代码注释、变量名、bash 命令除外。此规则不依赖上下文信号，全程生效。

**Compaction 恢复。** ⚠️ 此协议优先级高于 CP0。每次上下文压缩（compaction）发生后，AI 必须立即优先执行以下恢复步骤——不可凭记忆继续，不可先跑 CP0 完整加载（除非步骤 3 磁盘降级也无法定位时）：

1. **扫描对话（或 compaction 摘要）中最后一条 G1 输出行** — G1 格式天然包含恢复所需的全部状态：当前文件（filename）、进度（X/Y）、质量（streak）、Token（used/budget）。Compaction 摘要通常会保留 `╔═╗` 视觉块内容——先扫摘要，再扫对话。此即完整状态快照。
2. **若 G1 行不可见**（Phase 1 中途压缩，尚无 G1 输出） — 定位最后一条 CP 输出行（CP1 PASS 等，同样扫描对话或摘要）确定当前 Phase。此时磁盘上尚未完成任何文件，从该 Phase 起点恢复是合理的。⛔ **不可仅凭 HAR 文件存在推断 Phase 3 已完成**——HAR 可能来自 CP1 脚手架编译产物、前次 session 残留、或仅构建了部分模块。必须以对话/摘要中实际的 `✅ CP3 PASS` 输出行为准。无 CP3 输出行则视为 Phase 3 未执行，从 Phase 3 恢复。
3. **若 G1 和 CP 行均不可见**（极端情况，摘要也未保留任何断点证据） — 降级到磁盘恢复：(a) Read `06-metrics.md` 获取最后记录的 CP 状态和文件进度表；(b) `ls` 输出目录统计 .ets 文件数和 HAR 存在性；(c) 检查 git tag。若 06-metrics.md 也尚未创建（CP1 之前压缩）→ 从 CP0 完整加载开始。
4. **重新 Read `05-runbook.md`** — compaction 可能已移除关键规则（CP 检查项、G2 触发条件、输出格式）。此步骤不可跳过。若当前 Phase ≥ 3，额外精读即将进入的 Phase 的 CP 门控段——Phase 3 恢复 → §3.2（CP3 自检）；Phase 4 恢复 → §4.6（CP4 自检）；Phase 5 恢复 → §5.4 Step A-D 完整协议（含子步骤 C.1-C.3）（含 `╔═ HARD GATE ═╗` 视觉块和 STOP 规则）；Phase 6 恢复 → §6.1（Final Report 格式）。不可仅凭记忆重构门控细节。
5. MUST output exactly：`🔄 Compaction 恢复 — Phase {N}，{X}/{Y} 文件完成，最后 G1: {filename} (streak={streak}, Token={used}K/{budget}K)。请确认后说 Next。`
   - 若通过磁盘降级（步骤 3）恢复：`🔄 Compaction 恢复 — Phase {N}（磁盘降级），{X}/{Y} 文件完成，来源: 06-metrics.md + ls（{ets_count} .ets, HAR={有/无}）。请确认后说 Next。`
   - 若 06-metrics.md 也尚未创建（CP1 之前压缩，目录为空）：`🔄 Compaction 恢复 — Phase 0（无 CP 无产出），目录为空。将从 CP0 完整加载开始。请确认后说 Next。`

> ⛔ **以下是唯一合法格式。任何修改（包括改为表格、改为"Session 恢复诊断"、改为"断点推断"、改为"将从 Phase X 恢复"）均视为恢复协议未执行。** 格式不匹配 → 下一步操作（CP0 还是继续）的判断依据丢失 → 必须补输出正确格式行后才能继续。

此恢复协议不可跳过。不可假设"我知道跑到哪了"而不执行恢复步骤。

> ⛔ **历史教训**：semver session `7d1de1b4` 中，AI 在 compaction 后未执行恢复协议——没有扫描 G1/CP 行、没有重读 05-runbook、没有输出 🔄 行。AI 凭（错误的）记忆将 Phase 3（全量编译）误解为 Loop 2（深度审计），跳过了 Session Bridge，直接在同一 Session 内进入 Phase 4。根因：协议存在但 AI 没有触发执行——上下文压缩时协议本身被移除了。

#### 三层恢复机制（Compaction Recovery Escape Ladder）

```
compaction 发生
    │
    ├── 第 1 层：AI 自动执行（⚠️ 实际成功率极低，不可依赖）
    │    原理：AI 扫描摘要中的 G1/CP 行 → Read 05-runbook → 输出 🔄 行
    │    现实：compaction 通常将协议文本一并移除，AI 看不到协议→不会执行
    │    ⚠️ 操作员切勿等待 AI 自动输出 🔄 行——等待 = 浪费时间。
    │       若 AI 未在 compaction 后的第一条回复中输出 🔄 行 → 立即进入第 2 层。
    │
    ├── 第 2 层：人工强制触发（← 这是实际操作中的标准路径）
    │    触发：操作员看到 compaction 后，直接发送：
    │         "Read 05-runbook.md §全局交互规则，执行 Compaction 恢复协议"
    │    效果：AI 被迫加载协议原文 → 执行 5 步 → 输出 🔄 行 → 定位恢复
    │    成功 → 确认 🔄 行断点正确 → "Next" → 继续
    │    ⚠️ 若 🔄 行显示 streak≥5：AI 必须在恢复后立即重提 Template C（批量生成提议），
    │       不可直接进入逐文件 G1 模式。操作员也可主动说 "批量生成" 跳过 Template C 直接进入批量模式。
    │    失败 → 🔄 行断点与操作员所知不符 → 进入第 3 层
    │
    └── 第 3 层：人工定位（最可靠兜底）
         触发：第 2 层定位不准，或操作员确切知道断点位置
         操作员直接告诉 AI 当前位置，例如：
              "我们在 Phase 2，文件 8/15，刚才完成了 lrucache.ets"
              "我们在 Phase 2，streak=5，批量生成剩余文件"
              "我们在 Phase 3，刚 CP3 PASS，需要输出 Session Bridge"
              "我们在 Phase 5 Step A，等待手动测试结果"
         效果：AI Read 05-runbook → 找到对应 Phase → 输出确认行 → 从指定断点继续
         格式：
              ```
              📍 人工定位恢复 — Phase {N}，{文件/步骤}。已加载 05-runbook.md，从指定断点继续。
              ```

⛔ **第 2 层强制规则**：AI 若在 compaction 后收到包含 "Read 05-runbook.md" 和 "恢复" 字样的人工指令，必须立即执行恢复协议 5 步，不得先做任何迁移操作（写文件、编译、审计）。此规则无例外。

⛔ **第 3 层强制规则**：AI 若收到包含 "Phase" + 文件位置信息的人工定位指令，必须 (a) 先 Read 05-runbook.md 找到对应 Phase，(b) 输出 📍 人工定位恢复确认行，(c) 从指定断点按 runbook 流程继续。不可质疑人工定位的准确性——人工知道自己的项目状态。

> **关于 G1/CP 行 vs 06-metrics.md**：G1/CP 行（存在于对话或 compaction 摘要中）仍是恢复的首选数据源——06-metrics.md 可能在 CP1 之前尚未创建。06-metrics.md 仅在 G1 和 CP 行均不可见的极端情况下作为降级兜底，提供磁盘上最后记录的状态。

**文档加载确认。** 加载 01-08 中任一阶段文档（01/02/03/04/05/06/08）后，AI 必须输出加载确认行：

```
📂 LOADED: {文件名} ({关键属性})
```

示例：
- `📂 LOADED: 01-planner.md (20 sources S1-S20, 🔴🟠🟡 risk categories, JSON plan schema)`
- `📂 LOADED: 02-executor.md (OUTPUT_RULES OR1-OR4, HAR SDK target, task spec only — no ArkTS rules)`
- `📂 LOADED: 03-auditor.md (29 rules R1-R29, anti-pattern catalog AP1-AP9, STATUS: PASS format)`
- `📂 LOADED: 04-test-generator.md (Module Config magic values, L1-L5 coverage, TA1-TA15 anti-patterns)`
- `📂 LOADED: 05-runbook.md (CP0-CP5, E1-E16 Trigger Map, G1 format, Compaction recovery protocol)`
- `📂 LOADED: 06-metrics.md (per-file tracking table, Violations rule slug format)`
- `📂 LOADED: 08-project-audit.md (18 rules R1-R18, B1-B10 blind-spot checklist, 2-stage audit)`

07-environment.md 不在此列（按需加载，操作者回复 `fix` 时加载对应节次并修复；07 未覆盖则 AI 自行搜索解决后追加到 07 Appendix）。

> **Phase 5 入口额外要求**：进入 Phase 5 前，除标准 `📂 LOADED: 05-runbook.md (...)` 外，还需额外输出 `📂 LOADED: 05-runbook.md §5.4` 确认行（Step A-D 完整协议（含子步骤 C.1-C.3） + HARD GATE 视觉块 + STOP 规则），证明 §5.4 细节已被加载（非仅靠 compaction 前的上下文记忆）。此确认行是 CP5 前置条件 #3 的追溯依据。

确认行是 CP 检查追溯依据：若某 Phase 要求加载的文档无确认行 → 对应 CP 的文档加载检查项 FAIL。

**Token Checkpoint 输出格式。** 每个 CP 边界，AI 必须在每次回复末尾输出 token 消耗汇报（不可省略、不可自由发挥）：

```
📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N | ≥80%⚠️ / ≥95%🛑
```

累计值计算方式（优先级递减）：(1) 若 Claude Code 环境支持 `/cost` 命令或 transcript 文件访问 → 使用精确值；(2) 否则基于本轮输入输出长度估算累计。估算偏差 ±15% 可接受——token checkpoint 的目的是提醒预算边界，非精确审计。

此格式行是 CP 自检追溯依据：若 CP 输出区无 `📊 TOKEN:` 行 → 对应 CP 自检的 Token Checkpoint 项 FAIL。AI 不可仅凭 G1 逐文件行的 Token 字段推算——CP 边界 checkpoint 必须显式输出此格式行。

---

## CP 门控自检 — 通用规则

每个 CP 输出 `✅ CP{N} PASS` 之前，AI 必须执行自检，逐项确认该 CP 的前置门控均已执行。此自检是 CP PASS 输出的**硬前置条件**——不自检就不能说 PASS。

自检格式：

```
🔍 CP{N} 门控自检（输出 PASS 前逐项确认）：
- [ ] {门控项 1}
- [ ] {门控项 2}
  ...
以上任一项为"否" → 🛑 禁止输出 CP{N} PASS，先补执行缺失项
```

各 CP 的自检项定义在对应 Phase 的检查点章节中。自检输出后，紧跟 CP PASS 行。

> **关于 05-runbook 加载**：CP1 自检包含 `📂 LOADED: 05-runbook.md` 确认项。CP2-CP5 不再逐 CP 重复此项——05-runbook 的重读由 §全局交互规则 中的 Compaction 恢复协议（步骤 4）保障：每次 compaction 后强制重读 05。若 session 未发生 compaction 且 05 已在 CP1 加载，则无需重复加载。（04-test-generator.md 和 08-project-audit.md 在对应 CP（CP4/CP5）仍需逐次验证——它们是 Phase 特定文档，非全程适用。）

---

## CP0: Document Loading Hard Gate（硬门控 — 文档加载验证）

**Trigger**: Session startup（非 compaction 恢复），BEFORE any migration work begins. This is NOT optional — CP0 is a hard gate with the same enforcement level as CP1-CP5. **Compaction 恢复时**：若 §全局交互规则 中的 Compaction 恢复协议已成功定位断点，则跳过 CP0 完整加载，直接从断点 Phase 恢复。CP0 完整加载仅适用：(a) 全新 session 启动，或 (b) 恢复协议无法定位任何 G1 行或 CP 行。

**Purpose**: CP0 replaces the old "Phase 0 document inventory table." In past sessions, the AI treated the table output as a cognitive shortcut — outputting the table satisfied the instruction without actually loading the documents. CP0 fixes this by requiring actual Read tool calls + 📂 LOADED confirmations before allowing entry to Phase 1.

**⚠️ 为什么是硬门控**：历史 session（semver c883d832 等）中，AI 输出 Phase 0 表格后即跳过 02/03/04/08 的实际加载，因为 AI 认为这些文档的领域知识可从训练数据推断。CP0 不信任 AI 的"我应该不需要读这个"判断——每个文档必须在 CP0 中被显式 Read，并输出 📂 LOADED 确认行。

### CP0 执行步骤（不可跳过任何一步）

**Step 0.1 — 实际加载每个文档（Read tool call，非仅表格输出）**：

AI 必须使用 Read 工具逐一加载以下文档（07 除外——07 是按需加载的支撑文档，不参与 CP0 强制加载；具体加载哪些文档取决于 Session——见下方两 Session 差异表）：

| # | 文档 | 必须 Read | 必须输出 📂 LOADED | 验证关键属性 |
|---|------|:---:|:---:|------|
| 1 | 01-planner.md | ✅ | ✅ | 20 个官方文档来源（S1-S20），Platform API 风险分类（🔴/🟠/🟡），Migration Plan JSON schema |
| 2 | 02-executor.md | ✅ | ✅ | OUTPUT_RULES 文件头格式、import 排序规则、禁止注释模式、HAR library SDK 6.1.1(24) 目标 |
| 3 | 03-auditor.md | ✅ | ✅ | 29 条语义规则（R1-R29）+ 11 条编译器限制（G1-G11），Rule Source Mapping 表（S1-S8 + E1-E2），输出格式 STATUS: PASS — R1-R29 全量扫描 |
| 4 | 04-test-generator.md | ✅ | ✅ | Module Config Verification Table（5 个魔法值 + 常见 AI 错误值），L1-L5 覆盖层级定义，TA1-TA15 反模式表 |
| 5 | 05-runbook.md | ✅ | ✅ | CP0-CP5 检查点，E1-E16 Trigger Map，G1 输出格式，Compaction 恢复协议 |
| 6 | 06-metrics.md | ✅ | ✅ | 逐文件记录表格结构（File/Attempts/Audit Fails/Violations/Build Fails/Status），Violations 必须带 rule slug |
| 8 | 08-project-audit.md | ✅ | ✅ | 18 条执行纪律（R1-R18），盲区清单（AI 自审计最易遗漏的 10 类问题），二阶段架构（L1+L2 → P1+P2+P3） |

> 07-environment.md 不参与 CP0——它是按需加载的支撑文档。但 AI 必须在 CP0 输出中确认 07 的触发机制已理解（operator 回复 `fix` 时加载对应节次）。

> **两 Session 架构下的 CP0 差异**：
>
> | Session | 排除文档 | 原因 | 实际加载 |
> |---------|---------|------|---------|
> | Session 1（Phase 1-3） | 08-project-audit.md | Phase 4 审计在 Session 2 由独立审计员执行，Session 1 AI 不可接触审计文档 | 6 文档（#1-6） |
> | Session 2（Phase 4-6） | 01-planner.md | migration-plan.json 已在磁盘，01 的知识（API 清单、依赖图、风险列表）已编码其中 | 6 文档（#2-6 + #8） |
>
> AI 根据当前 Session 角色自动选择对应变体。CP0 自检中，`✅ CP0 PASS` 行输出 `6/6`（非 `7/7`）。START_HERE.md 各 Prompt 中的 CP0 描述与此表一致——若 Prompt 描述与此处有出入，以此表为准。

**Step 0.2 — 输出 📂 LOADED 确认行**：

每加载一个文档后，AI 必须立即输出 📂 LOADED 确认行。格式：
```
📂 LOADED: 01-planner.md (20 sources S1-S20, 🔴🟠🟡 risk categories, JSON plan schema)
📂 LOADED: 02-executor.md (OUTPUT_RULES, HAR SDK target, task spec only — no ArkTS rules injected)
📂 LOADED: 03-auditor.md (29 rules R1-R29, rule source mapping S1-S8+E1-E2, STATUS: PASS format)
📂 LOADED: 04-test-generator.md (Module Config Verification magic values, L1-L5 coverage, anti-pattern table)
📂 LOADED: 05-runbook.md (CP0-CP5, E1-E16 Trigger Map, G1 format, Compaction recovery protocol)
📂 LOADED: 06-metrics.md (per-file tracking table, Violations rule slug format)
📂 LOADED: 08-project-audit.md (18 execution rules R1-R18, blind-spot checklist, 2-stage L1+L2→P1+P2+P3)
```

⛔ 缺任何一条确认行 → CP0 自动 FAIL。AI 不可假设"不需要读"而跳过任何文档。

### Step 0.3 — CP0 门控自检

AI 在输出所有 📂 LOADED 行后，执行自检：

```
🔍 CP0 门控自检（输出 PASS 前逐项确认）：
- [ ] 所有必加载文档（按 Session 变体：6 个）全部通过 Read 工具实际加载（非仅凭 Phase 0 表格记忆）？
- [ ] 每个文档的 📂 LOADED 确认行已输出（含关键属性，非简写）？
- [ ] 07-environment.md 触发机制已理解（fix → 加载对应节次；07 未覆盖 → AI 自行搜索后追加到 07 Appendix）？
- [ ] 关键属性与文档实际内容一致（非 Phase 0 表格中的旧属性）？
以上任一项为"否" → 🛑 禁止输出 CP0 PASS，先补执行缺失项
```

### Step 0.4 — CP0 输出

自检全部通过后：

```
✅ CP0 PASS — 6/6 文档已加载并确认（Session 变体）
   📂 LOADED 确认行: 7/7
   07-environment.md: 按需触发机制已理解

→ CP0 通过。自动进入 Phase 1。
```

⛔ CP0 未通过 → 🛑 禁止进入 Phase 1。不可凭"我觉得我知道文档内容"而跳过 CP0。

---

## Phase 1: PLAN

### Step 1.1 — Load Planner
Load `01-planner.md` as the agent's system context.

### Step 1.2 — Analyze Source
```
Read the source library at {{WORK_DIR}}/{{LIBRARY_NAME}}/
Produce a complete migration plan per the Planner format.
Output the plan as JSON.
输出文件命名为 `migration-plan.json`，写入项目根目录（与 `library/` 同级）。
```

### Step 1.3 — Verify Plan
- [ ] All public APIs identified? (cross-reference source exports)
- [ ] All function signatures ArkTS-compatible? (no any/unknown/Object)
- [ ] Dependency order correct? (leaf files first)
- [ ] Project config templates valid? (SDK 6.1.1(24), hvigor 6.24.2)
- [ ] Any API risks flagged? (Node.js APIs, browser APIs, etc.)

### Step 1.4 — Create Project Scaffold
Based on Planner's project_config, create:
```
{{ARKTS_PROJECT}}/
├── hvigor/
│   └── hvigor-config.json5
├── build-profile.json5
├── hvigorfile.ts              ← 内容模板见 configs/hvigorfile.ts（只含插件导入 + task 注册）
├── oh-package.json5
├── AppScope/
│   └── app.json5
│   └── resources/base/element/string.json
│   └── resources/base/media/app_icon.png
└── library/
    ├── build-profile.json5
    ├── hvigorfile.ts          ← 内容模板见 configs/library-hvigorfile.ts
    ├── oh-package.json5
    └── src/main/ets/
        ├── Index.ets           (barrel export, empty initially)
        └── types.ets           (shared types)
```

> ⚠️ **hvigorfile.ts 约束**：所有 `hvigorfile.ts`（根目录 / library / entry）**只含插件导入 + task 注册**，禁止硬编码绝对路径（如 `/Applications/DevEco-Studio.app/`）。DevEco Studio 路径通过环境变量 `DEVECO_SDK_HOME` 注入，不应写入源码。模板见 `configs/hvigorfile.ts`、`configs/library-hvigorfile.ts`、`configs/entry-hvigorfile.ts`。

**Step 1.4a — git 初始化（Phase 4/5 的 tag 依赖）**：

```bash
cd [project-path] && git rev-parse HEAD >/dev/null 2>&1 || (git init && git add -A && git commit -m "scaffold: initial project structure")
```

> ⚠️ 若项目已在 git 仓库中则跳过。Phase 4 CP4 需创建 `Phase4-SNAPSHOT` tag，Phase 5 C.2 需 `git diff Phase4-SNAPSHOT`——两者都依赖至少一次 commit 存在。未 init 则 tag 无法创建，C.2 签名检测无法执行。

**Metrics to record:**
- `plan_files_total`: N (number of .ets files in plan)
- `plan_apis_total`: N (total public API count)
- `plan_risks_flagged`: N

### Step 1.5 — CP1: Plan Integrity Check + Token Checkpoint

> ⚠️ **幂等性保护**：若以下条件同时满足 → 先询问人工 "检测到已有 migration-plan.json + 编译产物，是否从 Phase 2 恢复？" 而非重新执行 Phase 1：
> - `migration-plan.json` 已存在且非空
> - `library/build/default/outputs/default/*.har` 已存在
> - 对话上下文中无 `✅ CP3 PASS` 输出行（若有则可直接跳到 Phase 4）
> 此检查防止 compaction 恢复时因误判 HAR 来源而重复执行 Phase 1 全流程。

After Plan output + scaffold created, run these checks. Each item = PASS / FAIL / WARN.

| # | 检查项 | 方法 |
|---|--------|------|
| 1 | API 数量正确？ | `grep -c "export function\|export class\|export const"` 源码 → 与 `plan_apis_total` 对比。差异 > 5% → FAIL |
| 2 | 每文件导出 ≤ 15？ | 统计 Plan 中每个 file 的 exports 数量。超标 → FAIL，列出超标文件 |
| 3 | 签名合规？ | 扫描 Plan 所有 function signature，不含 `any`/`unknown`/`Object`。命中 → FAIL，列违规签名 |
| 4 | 动态键值用 Map？ | 若 Plan 描述含 "dynamic key" / "任意键" / "obj[key]" → 检查是否指定 Map 替代。未指定 → WARN |
| 5 | call-signature 接口模式检测 | Plan 签名中出现 call-signature 模式 `{ (x: X): Y }` → FAIL |
| 6 | `dependency_order` 正确？ | 拓扑排序：leaf（无本地 import）在前。顺序错 → FAIL，列出正确顺序。有循环依赖 → 检查是否已按 01-planner Output Rule #6 策略处理：已提取公共类型后重排 → PASS；已标注在 risks 中 → WARN；未处理 → FAIL |
| 7 | `project_config` 使用模板值？ | SDK = `6.1.1(24)`、hvigor = `6.24.2`、runtimeOS = `HarmonyOS`、deviceTypes = `["default"]`。不匹配 → FAIL |
| 8 | 脚手架文件已创建？ | `ls` 检查 build-profile.json5 / hvigorfile.ts / oh-package.json5 / hvigor/hvigor-config.json5 / AppScope/app.json5 / library/。缺失 → FAIL |
| 9 | 空项目编译通过？ | 执行 `hvigorw assembleHar`。非 0 → FAIL，贴错误 |

**人工判断 — 分类合理？** 请确认每个文件的 API 分组是否合理。不需修改可直接 Next，需调整请说。

**输出：**

先执行自检：
```
🔍 CP1 门控自检（输出 PASS 前逐项确认）：
- [ ] 9 项自动检查全部执行？
- [ ] 人工分类确认已收集？
- [ ] 📂 LOADED: 01-planner.md (20 sources S1-S20, risk categories, plan schema) 确认行存在？
- [ ] 📂 LOADED: 05-runbook.md (CP0-CP5, E1-E16, G1 format) 确认行存在？
- [ ] 📂 LOADED: 06-metrics.md (tracking table, rule slug format) 确认行存在？
- [ ] 若本 session 发生过 compaction，是否已输出 🔄 Compaction 恢复 行（标准格式或磁盘降级格式）？无此行 → 05-runbook §全局交互规则 恢复协议未执行
- [ ] 📊 TOKEN 行已输出（格式: 📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N）？
以上任一项为"否" → 🛑 禁止输出 CP1 PASS，先补执行缺失项
```

自检全部通过后：
```
如果全部通过：
✅ CP1 PASS — X/9 项通过（Y WARN）
→ 请填写 06-metrics.md Phase 1: plan_files_total / plan_apis_total / plan_risks_flagged
→ 填写完毕说 「Next」 进入 Phase 2 BUILD

如果有 FAIL：
❌ CP1 FAIL — [列出失败项]
→ 修复后说 「recheck」（只改 Plan/配置，不写代码）
```

**Token Checkpoint:** AI 输出强制格式行 `📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N | ≥80%⚠️ / ≥95%🛑`（累计值：优先 /cost 或 transcript 精确值，否则估算，±15% 可接受）。≥80% → warn，≥95% → stop。≥1 compaction → 视为 ≥50%，请核实。

---

## Phase 2: BUILD — Loop 1 (Per-File Micro-Loop)

### Step 2.0 — Phase 2 入口门控（⛔ 不可跳过 — 防止 G1 漏执行）

进入 Phase 2 后、Step 2.1 之前，AI 必须执行以下入口门控——不可从 CP1 PASS 直接冲进文件编写：

**A. 文档加载（2 个必加载）**：

| # | 文档 | 📂 LOADED 格式 | 原因 |
|---|------|---------------|------|
| 1 | 02-executor.md | `📂 LOADED: 02-executor.md (OR1-OR4 输出规则, HAR SDK target, task spec only — no ArkTS rules)` | Phase 2 逐文件编写必须遵守 OR1-OR4 |
| 2 | 03-auditor.md | `📂 LOADED: 03-auditor.md (29 rules R1-R29 + 11 compiler G1-G11 = 40 rules, AP1-AP9, STATUS: PASS format)` | Loop 1 每次审计依据 |

⚠️ 文档加载必须在第一个文件的 Step 2.1 之前完成——不可"边写边补"，不可"CP0 时已读过所以跳过"（新 Phase 新上下文，必须刷新）。

**B. 依赖顺序确认**：Read `migration-plan.json` → 提取 `dependency_order` → 输出：
```
📋 Phase 2 依赖顺序：N 个文件
  1. first.ets (leaf)
  2. second.ets
  ...
  N. Index.ets (barrel)
开始第 1 个文件：[first.ets]。
```

**C. G1 门控自检**：AI 必须输出以下确认行：
```
🔒 G1 门控已理解：
  - 每文件 DONE 后 → 输出 ╔═╗ G1 块 → 硬停止
  - 禁止同一回复中开始下一个文件
  - 禁止 "Now file N: xxx" 紧接 G1 后
  - 必须等待操作者 "Next" 后才能继续
```

⛔ 未完成 Step 2.0 A+B+C 三项 → 不可进入 Step 2.1。

### Loop Control (with Dynamic Triggers)

```
for each file in dependency_order:
  attempt = 0
  max_attempts = 3
  passed = false

  while attempt < max_attempts:
    attempt++
    → Step 2.1 (Execute)
    → Step 2.2 (Audit)
    → if FAIL and attempt < max_attempts: continue
    → Step 2.3 (Build)
    → if FAIL and attempt < max_attempts: 
        → 若此为同一文件同一错误的第 ≥2 次出现 → check 07 Trigger Map (E1-E16)。首次出现 → 仅记录错误到 metrics，不触发 07 检查
        → continue
    → if PASS: passed = true; break

  After each file DONE → MUST output exactly:

  ```
  ╔══════════════════════════════════════════════════════════╗
  ║  ⛔  G1 文件门控 — 硬停止（与 Phase 4 HARD GATE 同级）     ║
  ║                                                          ║
  ║  ✅ [filename] DONE — 等待 Next | {X}/{Y} | 连续1轮通过: {streak}/5 | Token: {used}K/{budget}K
  ║                                                          ║
  ║  输出此行后，你必须立即结束当前回复。                       ║
  ║  ⛔ 禁止在同一个回复中开始下一个文件的编写。                 ║
  ║  ⛔ 禁止 "outputting G1 gates for files 1-2 and proceeding"  ║
  ║  ⛔ 禁止 "Now file N: xxx" 紧接在 G1 行之后。               ║
  ║                                                          ║
  ║  必须等待用户输入 "Next" 后才能继续下一个文件。             ║
  ╚══════════════════════════════════════════════════════════╝
  ```

  > **G1 文件门控说明**：硬停止，人不发 Next 就停。G1 输出行不可省略、不可缩写 — 进度/streak/Token 由此强制输出。T3 靠 streak 触发。Compaction 后此行即是完整状态快照，无需依赖上下文记忆。
  >
  > **streak 重置规则**：
  > - streak 计数「首次即通过（1 轮 PASS）」的连续文件数
  > - 非 1 轮通过（≥2 轮、3 轮耗尽即 exhausted）→ streak 归零
  > - T3 触发且批量生成获批 → streak 归零（批量跳过了逐文件 G1 门控，不应累加）
  > - T5 紧急熔断后 → streak 归零，从下一个文件重新计数

> ⛔ **禁止的简化格式（反面示例，不可使用）**：
> ```
> ✅ **file.ets DONE — 等待 Next** | **X/Y** | **连续1轮通过: streak/5** | **Token: ~30K/800K**
> ```
> 此类纯文本行缺少 `╔═╗` 视觉块外壳，在 compaction 压缩时更容易被丢弃。只有 `╔═╗` 包围的完整 G1 块才算有效输出。

  ⛔ Turn-boundary 规则：G1 行必须是一个回复的最后一条内容。禁止在 G1 行之后紧跟 "Now file N: xxx" / "接下来写文件 N" / "proceeding" 等任何继续信号。AI 在看到用户 "Next" 之前不可在新回复中主动启动下一个文件的 Step 1。

> ⚠️ **决策时机**：3 轮耗尽时不在 Phase 2 阻塞。AI 记录文件到 exhausted 列表，输出模板 B 分析后由人工说 "Next" 继续下一个文件。遗留问题在 Phase 4 Step 4.3 统一决策，不在 Phase 2 当场要求人工选择 skip/split/skip→known。

  **if not passed（3 轮耗尽仍未通过）**：
    不静默失败。输出 "⛔ [filename] 3 轮未通过 — 转入 Phase 4 审计"
    → 递增 `consecutive_3x_exhausted` 计数器。若任一文件通过（1 轮或 ≥2 轮）→ 计数器归零。
    → 若计数器 = 3 → T5 紧急熔断（详见 G2 触发器），先处理系统性异常再继续。
    → AI 自动输出模板 B API fit 分析，记录文件到 exhausted 列表。
    → 人工说 "Next" 继续下一个文件（不在 Phase 2 当场决策 skip/split/skip→known）。
    遗留问题在 Phase 4 Step 4.3 统一决策。

  **Support file discovery（Phase 2 中段发现隐式依赖）**：
    当前文件依赖一个不在 migration plan 中的文件时：
    → 暂停当前文件，先创建 support file，走标准 Loop 1（写→审→编→G1）
    → Support file 通过后，加入文件队列末尾。输出 "📎 [support_file] 新增（plan 外）— 文件进度: {done}/{plan + new}"
    → 回到原文件继续
    → 新增文件数 `new` 由每次 support file discovery 累加，Heartbeat 同步引用动态分母
    → CP2 #5 对此类文件 WARN 是预期行为，Phase 6 对照 plan vs actual 时解释 delta

Dynamic Triggers — G2 模式检测 (check after EACH file completes):

  Heartbeat (fires automatically every N files, does NOT pause):
    N = max(5, ceil(total_files/10))
    → total_files 为动态值：plan_files_total（Phase 1 计划数）+ 已创建的 support file 数
    → Output progress summary:
      "📊 进度: {done}/{plan + new} 文件完成 ({pct}%)
         首轮通过率: {first_round_pass}/{done} ({pct}%)
         当前 Token: {used}K / {budget}K ({pct}%)"
    → Execution continues WITHOUT waiting for human input.
  ┌──────────────────────────────────────────────────────────┐
  │ T1 (→ Template A): Same violation ID in ≥2 consecutive  │
  │   files. Report: "⚠️ [RX] 连续出现在 [FileA] 和 [FileB]" │
  │   → STOP. Human may apply Executor prompt patch.         │
  │                                                          │
  │ T2: File took ≥2 attempts (passed)                      │
  │   → 文件通过（经 N 轮挣扎后 PASS）：                       │
  │     "⚠️ [FileX] 经 {N} 轮通过（已记录到 metrics）"       │
  │     → 不暂停。原因复盘合并到 CP2（Step 2.5）批量回顾。     │
  │     → 3 轮耗尽的情况由上方独立分支处理，T2 不重复触发       │
  │                                                          │
  │ T2b | 单文件 3 轮耗尽未通过 | 模板 B：AI 输出 API fit 分析 → 记录到 exhausted → 人工说 Next 继续 → Phase 4 统一决策（不在 Phase 2 阻塞） │
  │                                                          │
  │ T3 (→ Template C): G1 streak 计数器达到 5             │
  │   → G1 输出行已显示 "连续1轮通过: 5/5"                    │
  │   → STOP. Human may choose batch generation for          │
  │     remaining files of same category.                    │
  │   → 操作员回复 "批量生成" / "Next" 进入批量模式              │
  │   → 操作员回复 "继续逐文件" 拒绝批量，维持 G1 节奏           │
  │                                                          │
  │ 💡 Phase 2 通用批量触发（不限于 T3）：操作员随时可说          │
  │    "批量生成" 或 "批量生成剩余 N 个文件" → AI 跳过逐文件      │
  │    G1，一次性生成所有剩余文件 → 逐个 Audit → 逐个 Build →     │
  │    最后输出结果汇总（通过/失败/耗尽），无逐文件 G1 门控。       │
  │    ⚠️ Compaction 恢复后尤其有用——streak 记忆丢失可直接批量。  │
  │ T4 (→ Template D): Build errors point to same dependency│
  │   → Report: "⚠️ Build 错误指向 [dep.ets]"                │
  │   → STOP. Fix dependency first before continuing.        │
  │                                                          │
  │ T5 (→ Emergency Stop): 连续 3 个文件各耗尽 3 轮仍未通过  │
  │   → Report: "🛑 连续 3 文件各 3 轮耗尽 — 疑似系统性异常" │
  │   → STOP. Human decides:                                 │
  │     a) 检查/补丁 Executor prompt                         │
  │     b) 调整策略（标记不兼容 API 为 skipped）              │
  │     c) 继续（确认为巧合，重置计数器）                      │
  └──────────────────────────────────────────────────────────┘
```

### Step 2.1 — Execute
Load `02-executor.md` as context.
Provide: single file task from Planner (path, signature, description).
Output: .ets file content.

### Step 2.2 — Audit
Load `03-auditor.md` as context.
Provide: generated .ets file.
Auditor checks against 29 条语义规则（R1-R29）+ 11 条编译器限制（G1-G11，由 hvigorw 编译阶段自动拦截，Auditor 可选预检）。

**Rules of Engagement:**
- **Attempt 1**: Auditor lists violations ONLY (no fix suggestions)
- **Attempt 2+**: Auditor lists violations WITH fix suggestions
- **Output format (强制，非建议)**: 
  ```
  STATUS: PASS — R1-R29 全量扫描，无违规
  ```
  或
  ```
  STATUS: FAIL
  VIOLATIONS: R1(arkts-no-any-unknown)×2, R5(arkts-no-props-by-index)×1
  ```
  ⛔ PASS 必须显式声明 "R1-R29 全量扫描"。仅写 "STATUS: PASS"（无 R1-R29 声明）视为不完整 → 视为 Auditor 未执行。
  ⛔ 禁止使用自创的简化表格（如"\| 检查项 \| 结果 \|"）替代此格式。违规输出视为 Auditor 未执行。

### Step 2.3 — Build
Run: `cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHar`
- If 0 errors → FILE DONE ✅
- If errors → extract error messages, feed back to Step 2.1 as additional context

### Step 2.4 — Track Metrics Per File（每完成一个立即记）

每个文件 DONE 后，**立即**在 06-metrics.md Phase 2 表格中记一行，不要攒到 Phase 2 结束再补：

| File | Attempts | Audit Fails | Violations | Build Fails | Status |
|------|:--------:|:-----------:|------------|:-----------:|:------:|
| types.ets | 1 | 0 | — | 0 | DONE |

**Violations 列必须记录规则 ID + 子类型，不可仅记数量。** 格式：`R5(arkts-no-props-by-index)×3, R1(arkts-no-any-unknown)×1`。仅记 `R5×3` 是不够的 — T1 跨 compaction 检测需要子类型来判断"同一违规"是否在连续文件中重复出现（T1 只看违规 ID，但审计追溯需要子类型）。子类型缺失会被 CP2 #3 标记为数据不完整。

> 逐行记录是 CP2 检查的前提。缺行会被 CP2 拦住。

### Step 2.5 — CP2: Build Completion Check + Token Checkpoint

Trigger: Phase 2 Loop 1 全部文件 DONE（含 escalated 已标记）。

| # | 检查项 | 方法 |
|---|--------|------|
| 1 | 所有文件已生成？ | Plan 文件列表 vs `library/src/main/ets/*.ets` 实际文件。缺失 → FAIL |
| 2 | 逐文件指标已记录？ | 06-metrics Phase 2 表格中，Plan 每个文件都有一行（File / Attempts / Audit Fails / Violations / Build Fails / Status）。缺行 → FAIL |
| 3 | 记录数据完整？ | 每行 Attempts ≥ 1、Status 已填。缺失 → FAIL |
| 4 | G2 触发执行？ | Template A/C/D（T1/T3/T4 触发）已执行？如触发 3轮耗尽→模板 B 已执行？未执行 → WARN |
| 5 | 多余文件？ | `library/` 中有 Plan 未列的 .ets → WARN |
| 6 | 03-auditor 已逐文件加载并执行？ | (a) 必须有 `📂 LOADED: 03-auditor.md (29 rules, R1-R29)` 确认行。(b) 每个文件的 Auditor 输出必须为 `STATUS: PASS — R1-R29 全量扫描，无违规` 或 `STATUS: FAIL\nVIOLATIONS: R1(rule)×N` 格式。PASS 未声明 "R1-R29 全量扫描" → FAIL。自创简化表格 → FAIL。(c) 06-metrics 每行的 Violations 列与 Auditor 输出一致。不一致 → FAIL |

> ⚠️ **R5 误标提醒**：06-metrics.md 顶部的勘误（L3）记录了历史 L5-L7 数据的系统性 R5 误标（大量违规归为 R5 但实为 R1/R2/R3/R4/R11/R16/R27 等）。本迁移的 Violations 记录必须使用 `R#(subtype)×N` 精确格式，规则 ID 严格对应 03-auditor.md 的 R1-R29 编号——不可将所有 ArkTS strict-mode 违规笼统归为 R5。

**输出：**

先执行自检：
```
🔍 CP2 门控自检（输出 PASS 前逐项确认）：
- [ ] 6 项检查全部执行？
- [ ] 逐文件指标已记录（每文件一行）？
- [ ] Violations 列使用完整格式 R5(subtype)×N？
- [ ] 📂 LOADED: 02-executor.md (OUTPUT_RULES, HAR SDK target, task spec only) 确认行存在？
- [ ] 📂 LOADED: 03-auditor.md (29 rules, R1-R29) 确认行存在？
- [ ] 每个文件的 Auditor 输出为 STATUS: PASS — R1-R29 全量扫描 / STATUS: FAIL + VIOLATIONS 格式（非自创简化表格）？
- [ ] G2 触发的 Template 已执行（如有）？
- [ ] 若本 session 发生过 compaction，是否已输出 🔄 Compaction 恢复 行（EXACT 格式: 🔄 Compaction 恢复 — Phase {N}，{X}/{Y} 文件完成，最后 G1: {filename} (streak={streak}, Token={used}K/{budget}K)。请确认后说 Next。无此行 → 恢复协议未执行）？
- [ ] 📊 TOKEN 行已输出（格式: 📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N）？
以上任一项为"否" → 🛑 禁止输出 CP2 PASS，先补执行缺失项
```

自检全部通过后：
```
✅ CP2 PASS — X/Y 文件通过（Z escalated）
→ 请更新 06-metrics.md Phase 2 汇总: 总文件数/通过数/escalated / Audit/Build 首轮通过率 / 按规则 ID 违规统计
→ 填写完毕说 「Next」 进入 Phase 3 VERIFY
```

**Token Checkpoint:** AI 输出强制格式行 `📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N | ≥80%⚠️ / ≥95%🛑`（累计值：优先 /cost 或 transcript 精确值，否则估算，±15% 可接受）。≥80% → warn，≥95% → stop。≥1 compaction → 视为 ≥50%，请核实。

---

## Phase 3: VERIFY (Library Compilation)

### Step 3.1 — Full Library Build
Run: `cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHar`
- [ ] All files compile together with 0 errors
- [ ] HAR output exists at `library/build/default/outputs/default/*.har`

### Step 3.2 — CP3: Full Verification + Token Checkpoint

After `hvigorw assembleHar` runs, execute these checks:

| # | 检查项 | 方法 |
|---|--------|------|
| 1 | 0 编译错误？ | 解析 Build 输出。非 0 → FAIL，贴完整错误。**若是 env/config 类（路径/hvigor/SDK/签名）→ 💡 查 07-environment.md §1/§4/§5**；**若是 library 代码错误（类型不匹配/重复导出/import 路径错误）→ 列出错误文件，回到 Phase 2 逐文件修复，修完重新跑 Phase 3** |
| 2 | HAR 文件生成？ | `ls -lh library/build/default/outputs/default/*.har`。不存在或 0 字节 → FAIL |
| 3 | 无循环依赖？ | 解析所有 `from './X'` import → 拓扑检测。有环 → 检查 Phase 1 risks 是否已标注：已标注且已提取公共类型处理 → PASS；已标注但无公共类型可提取且人工接受 → PASS（标注原因）；已标注但未处理 → FAIL（Plan 承诺了但没做）；未标注 + 新发现 → FAIL，列出环路，需在 Phase 4 审计中处理 |
| 4 | Anti-Pattern 扫描 | grep 全量 .ets：`any`/`unknown`/`Symbol(`/`delete `（排除 Map.delete）/`for...in`/对象 spread（排除 rest）/`.bind(`/`.call(`/`.apply(`/`WeakMap`/`WeakSet`/`Object.create(null`/`__proto__`。区分真违规 vs 注释/字符串 |
| 5 | Export 审计 | 每文件 export ≤ 15。Index.ets barrel 覆盖所有 public export。缺失 → FAIL |
| 6 | 风险复核 | 逐条对照 Phase 1 `plan_risks_flagged`：已解决 / 已缓解 / 仍存留 |

**输出：**

先执行自检：
```
🔍 CP3 门控自检（输出 PASS 前逐项确认）：
- [ ] 6 项检查全部执行？
- [ ] hvigorw assembleHar 实际执行（非缓存/非记忆）？
- [ ] HAR 文件实际存在且非空？
- [ ] Anti-pattern scan 实际执行（非跳过）？
- [ ] 📂 LOADED: 03-auditor.md (29 rules, R1-R29) 确认行存在（Anti-pattern scan 参考规则依据）？
- [ ] 若本 session 发生过 compaction，是否已输出 🔄 Compaction 恢复 行（EXACT 格式: 🔄 Compaction 恢复 — Phase {N}，{X}/{Y} 文件完成，最后 G1: {filename} (streak={streak}, Token={used}K/{budget}K)。请确认后说 Next。无此行 → 恢复协议未执行）？
- [ ] 📊 TOKEN 行已输出（格式: 📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N）？
以上任一项为"否" → 🛑 禁止输出 CP3 PASS，先补执行缺失项
```

自检全部通过后：
```
如果全部通过：
✅ CP3 PASS — 6/6 项通过
   Build: SUCCESSFUL, 0 errors · HAR: library.har (XX KB)
   Deps: N modules, 0 cycles · Scan: N hits, 0 violations
   Export: ≤15/file, barrel OK · Risk: 已解决 X, 已缓解 Y, 存留 Z

→ 请更新 06-metrics.md:
	  - Phase 2 Summary: 总文件数/通过数/escalated / Audit/Build 首轮通过率 / 平均轮次 / 违规按规则 ID 统计
	  - Phase 3: Full build / HAR output / Anti-pattern scan / Circular deps / Export audit / Risk recheck

→ 填写完毕说 「Next」 → ⛔ 进入 Step 3.3（Session Bridge）。不可跳过——Phase 3 未完成。
```

**Token Checkpoint:** AI 输出强制格式行 `📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N | ≥80%⚠️ / ≥95%🛑`（累计值：优先 /cost 或 transcript 精确值，否则估算，±15% 可接受）。≥80% → warn，≥95% → stop。≥1 compaction → 视为 ≥50%，请核实。

---

### Step 3.3 — CP3 Session Bridge: 输出 Session 2 启动提示词（⛔ 强制，Phase 3 最后一步）

⛔ **AI 不可在 CP3 PASS 输出后停止**。Step 3.3 是 Phase 3 的最后一步——未执行 Step 3.3 = Phase 3 未完成。

CP3 PASS + Token Checkpoint + 人工 "Next" 后，AI 必须立即执行 3.3-A 和 3.3-B，输出完毕后**结束当前回复**。AI **不进入 Phase 4**。

---

**Step 3.3-A — 加载 Prompt Audit 原文（⛔ 不可凭记忆跳过，不可自己编写替代内容）**：

```
╔══════════════════════════════════════════════════════════╗
║  ⛔  CP3 HARD GATE — Step 3.3-A 强制加载                  ║
║                                                          ║
║  ⛔ 你现在必须使用 Read 工具加载 START_HERE.md。           ║
║                                                          ║
║  ⛔ 禁止以下行为：                                        ║
║     ❌ "我知道 Prompt Audit 内容，不需要重新读"           ║
║     ❌ "我根据记忆总结一下就行了"                         ║
║     ❌ "Session 2 需要的是摘要，我自己写"                 ║
║     ❌ 输出任何非 Prompt Audit 原文的内容替代              ║
║                                                          ║
║  ✅ 正确做法：                                            ║
║     1. 立即执行 Read START_HERE.md                       ║
║     2. 定位 "### Prompt Audit — Phase 4-6 ..." 标题      ║
║     3. 将该标题下的完整内容逐行加载到上下文               ║
║     4. 进入 Step 3.3-B，将原文内联输出                   ║
║                                                          ║
║  ⛔ 不执行此 Read = Session Bridge 无效                   ║
║  ⛔ 不执行此 Read = Session 2 收到残缺入口                ║
║  ⛔ 不执行此 Read = 不可进入 Step 3.3-B                   ║
╚══════════════════════════════════════════════════════════╝
```

> ⛔ AI 不可凭"之前 session 读过 START_HERE.md"的记忆跳过此 Read。Session 1 结束时的上下文可能已被 compaction 裁剪——START_HERE.md 的内容不可假设仍存在。必须重新 Read 获得 Prompt Audit 的当前版本原文，确保 Session 2 收到的是最新版。
>
> ⛔ 历史上 AI 多次跳过此步骤，自创简化版"Session 2 启动提示"替代——结果 Session 2 缺少 Step 0（磁盘验证）、CP0 文档加载表、收敛条件、执行纪律等关键内容。**Prompt Audit 原文 ~115 行，AI 自创版往往只有 ~20 行**——差距就是 Session 2 执行质量差距。不可重犯此错误。

**Step 3.3-B — 输出统一的 🔗 Phase 4-6 启动提示词**：

```
╔══════════════════════════════════════════════════════════╗
║  ⛔  CP3 HARD GATE — Step 3.3-B 强制输出                  ║
║                                                          ║
║  将 Step 3.3-A 加载的 Prompt Audit 原文完整内联输出。      ║
║                                                          ║
║  ⛔ 禁止以下行为：                                        ║
║     ❌ 用占位符 "[在此粘贴...]" 替代原文                  ║
║     ❌ 将原文缩写、总结、改写                             ║
║     ❌ "详细流程见 START_HERE.md Prompt Audit"            ║
║     ❌ 分成两个代码块输出                                 ║
║     ❌ 输出 "Session 2 Launch Prompt" 自创标题           ║
║     ❌ 自创 Phase 4/5/6 简化清单替代 Prompt Audit 原文    ║
║                                                          ║
║  ✅ 正确输出 = Session 1 参数块 + Prompt Audit 完整原文    ║
║     （合并为一个代码块，标题: 🔗 Phase 4-6 启动提示词）     ║
╚══════════════════════════════════════════════════════════╝
```

AI 将以下两部分**合并输出为一个完整的代码块**（不可分成两块，不可用占位符替代）：

1. **Session 1 动态参数**（填入实际值）：
   - 项目路径、源码路径、迁移计划路径、指标文件路径
   - CP3 摘要：.ets 文件数 / HAR 大小 / Build 结果 / exhausted 列表 / 风险状态

2. **Prompt Audit 完整正文**（从 Step 3.3-A 加载的原文，不省略、不缩写）：
   - ⚠️ Prompt Audit 中的 `[Token预算]`、`[输出项目路径]`、`[级别]`、`[库名]` 等占位符由 AI 根据 Session 1 的实际值填入
   - ⚠️ Prompt Audit 中的 `[填入与 Session 1 相同的 Token 预算]` 替换为 Session 1 的实际 Token 预算数值

输出格式（唯一合法格式，不可偏离）：

```
🔗 Phase 4-6 启动提示词 — 复制到新 Session
============================================
⚠️ 本提示词由 Session 1 AI 在 CP3 PASS 时自动生成。
   参数已根据实际执行结果填入，Prompt Audit 原文已内联。
   整体复制此代码块到新 Session 粘贴即可，无需手动拼装。

项目路径：{{ARKTS_PROJECT}}（如 ~/output/semver-arkts/）
源码路径：{{WORK_DIR}}/{{LIBRARY_NAME}}（如 ~/harmony-migration/semver/）
迁移计划：{{ARKTS_PROJECT}}/migration-plan.json
指标文件：{{ARKTS_PROJECT}}/06-metrics.md

CP3 摘要：
  .ets 文件数: N / HAR: library.har (XX KB) / Build: SUCCESSFUL, 0 errors
  exhausted 列表（Phase 2 3轮耗尽未通过）: [文件列表 或 无]
  风险状态: 已解决 X, 已缓解 Y, 存留 Z

============================================
[此处内联 Step 3.3-A 加载的 Prompt Audit 完整原文]
============================================
```

**🔍 Step 3.3 门控自检（输出前逐项确认，缺一项 = 🛑 禁止输出）**：
```
🔍 Step 3.3 门控自检：
- [ ] Step 3.3-A 实际执行了吗？Read START_HERE.md 了吗？（非"我记得"）
- [ ] Prompt Audit 原文是否完整加载（~115 行，非 ~20 行自创摘要）？
- [ ] 动态参数全部填入实际值（项目路径/源码路径/计划路径/指标路径）？
- [ ] CP3 摘要已填入实际值（文件数/HAR大小/Build结果/exhausted列表/风险状态）？
- [ ] [Token预算] 占位符已替换为 Session 1 的实际 Token 预算数值？
- [ ] 代码块中内联的是 Prompt Audit 完整原文吗？（非占位符，非自创简化版）
以上任一项为"否" → 🛑 禁止输出，先补执行缺失项
```

⛔ **硬门控规则**：
- AI 输出此块后必须结束回复，不进入 Phase 4
- `[此处内联...]` 处必须输出 Prompt Audit 的**完整原文**（从 Step 3.3-A 加载），不可用占位符替代——占位符依赖人工事后填充，是 Session 2 入口残缺的根因。**AI 自创简化版（如"Phase 4/5/6 各6项"清单）同样禁止**——必须是 START_HERE.md Prompt Audit 原文
- 若 exhausted 列表非空 → 参数块必须包含完整的 exhausted 文件列表（Session 2 Step 4.3 需呈现）
- 用户整体复制此代码块到新 Session 粘贴即可启动 Phase 4-6，无需打开 START_HERE.md 手动拼装

→ **Step 3.3 完成后，AI 结束回复。Phase 3 完成。Session 1 终止。**

---

## Phase 4: PROJECT AUDIT — Loop 2 (Audit-Fix-Reaudit)

> ⚠️ **Phase 4 在 Session 2 执行**。Session 1 CP3 PASS 后终止，用户在新 Session 粘贴 START_HERE.md Prompt Audit 启动 Phase 4-6。Phase 4 流程（Step 4.1-4.6）在新 Session 中完整执行，AI 作为独立审计员（未写过代码）。

Trigger: Phase 3 CP3 PASS，全量编译通过，库代码稳定。

### Loop 2 Control

Loop 2 是**嵌套循环**结构：

- **外循环**（max 5 轮）：每轮一次审计（首轮全量 → 后续仅增量）+ 人工决策
- **内循环**（收敛驱动，无硬上限）：修复 → 增量重审 → 再有新发现 → 再修复 → 再增量重审 → 直到干净

```
round = 0
max_rounds = 5

while (round < max_rounds):
  round++

  // === 外循环：审计 + 人工决策 ===
  if round == 1:
    → Step 4.1 全量审计（阶段一 L1+L2 全量扫描 → 阶段二 P1+P2+P3 窄聚焦；覆盖所有源文件）
    → Step 4.2 (输出分层建议)
  else:
    → Step 4.1 增量审计（L1+L2+P2 结构；仅覆盖上次修复涉及的文件）
    → Step 4.2 (输出分层建议)

  → Step 4.3 (人工审核) — HARD STOP，等待人工回复，不可替人工预判
  人工回复后：
  if 无 ✅ 采纳项:
    break  // 无需修复 → 外循环退出 → CP4

  let inner = 0  // 内循环计数器（每次进入 Step 4.4 的修复→再审计循环递增）
  // === 内循环：修复 ⇄ 增量重审，直到干净（软上限 10 轮）===
  while true:
    inner++  // 递增内循环计数器

    if inner >= 10:
      → STOP. Report: "⚠️ 内循环已达上限 10 轮，暂停"
      → Human decides: 继续（重置计数器）/ 标记残余为已知限制 / 放弃修复

    → Step 4.4 (Loop 1 逐文件修复本期 ✅ 项)
    → Step 4.5 (对本次修改的文件增量重审 L1 + L2)

    if 有新 Critical/High:
      → 追加到问题清单
      → 回到 Step 4.3 (人工审核新发现)
      → if 人工有 ✅ 采纳 → continue  // 继续内循环（再修复 → 再重审）
      → else → break                  // 人工全部驳回 → 退出内循环

    else if 有新 Medium/Low:
      → 输出新增 Medium/Low 清单（不阻塞 CP4）
      → 人工决定: 改 / 不改
      → 改：选取 ✅ 项 → 回到 Step 4.4（进入内循环修复）；未选 → 记残余
      → 不改 → 全部记残余 → break      // 有残余 → 回到外循环顶部（非干净退出）

    else:
      内循环干净退出 = true  // ← 唯一设置点：无任何新发现
      → break

  // 内循环以"无新 Critical/High 且人工定夺完毕"退出 → CP4
  if 内循环干净退出:
    break

  // 📊 心跳（每次修复-重审周期后自动输出，不暂停）：
  // 在 Step 4.5 完成后 → 输出摘要：
  //   "📊 Loop 2 Round {round}/{max_rounds} 内循环 #{inner}: 本轮修了 {fixed}, 剩余 {remaining}, 新发现 {new_found}"
  // → 执行继续，不等待人工输入。

if round >= max_rounds:
  → STOP. Report: "⚠️ Loop 2 已达上限 {max_rounds} 轮，仍有 N 个残余"
  → Human decides: 强制退出 / 提高上限继续 / 标记残余为已知限制
```

### Step 4.1 — 执行两阶段审计

AI 加载 `08-project-audit.md`，对完整编译产物执行两阶段审计（详见 08 §八）：

**阶段一：全量扫描（广度覆盖）**

1. **Layer 1 — 源库对齐审计**：移植版 vs 原库，逐 API 对比行为偏差
2. **Layer 2 — 独立对抗审计**：安全 + 质量 + 性能，完全抛开原库
→ 产出 L1 清单 + L2 清单

**阶段二：窄聚焦深度扫描（深度覆盖）**

3. **Pass 1 语义审计**（08 §8.3）：只看行为对不对，不管代码脏不脏
4. **Pass 2 结构审计**（08 §8.3）：只看代码干不干净，不管行为对不对
   ⚠️ 含全文件打勾表（08 §8.4），全部打勾后才输出 P2 完成
5. **Pass 3 补漏审计**（08 §8.3）：P1+P2 漏了什么？
→ 产出 P1 清单 + P2 清单 + P3 清单

**合并** L1+L2+P1+P2+P3 → 进入 Step 4.2 输出统一问题清单

### Step 4.2 — 输出分层建议

AI 完成 Step 4.1 内部审计后，按格式输出问题清单，每个问题标注来源（`[L1]`/`[L2]`/`[L1-P1]`/`[L2-P1]`/`[L1-P2]`/`[L2-P2]`/`[L1-P3]`/`[L2-P3]`）和严重程度：

> 两 Session 架构下，Session 2 AI 审计即独立审计——不再区分"内/外部"，无需合并步骤。

```
=== Phase 4 审计完成：{{LIBRARY_NAME}} ===
发现 N 个问题
  阶段一（L1+L2）: N 项    阶段二（P1+P2+P3）: N 项

🔴 P0 — 立即修复
  #1  [L2-P1]  [R4]  SSRF 无内网 IP 过滤                harmonyHttpAdapter.ets:15
  #2  [L1]     [R1]  combineURLs 正则 bug               combineURLs.ets:8

🟠 P1 — 尽快修复
  #7  [L1]     [R7]  flatMapDepth 忽略 depth 参数        collection_group.ets:131

🟡 P2 — 常规修复
  #14 [L1-P2]  [R14] flipInX/flipInY 完全相同            flippers.ets:66

🔵 P3 — 可延后
  #19 [L2]     [B5]  类型导出不完整                       Index.ets:17

---
以上分层建议仅供人工决策参考，不可在人工回复前自行修复任何条目。
回复格式: #1 ✅ | #2 ❌ 原因 | #3 ⏭️ 已知限制 | #4 📌 后续提醒
         或: P0 ✅ | P1 #8 #11 ✅ | P2 P3 📌 全部
```

### Step 4.3 — 人工审核

AI 输出 Step 4.2 分层建议后，**必须 STOP**（此行为硬门控，不可跳过）。

> ⚠️ **Phase 2 耗尽项**：若 Phase 2 Loop 1 有 3 轮耗尽的文件（exhausted 列表非空），必须与审计发现一同在此呈现。格式：`🛑 Phase 2 耗尽项（延迟至此决策）:` + 每个耗尽文件的模板 B API fit 分析摘要。操作者在此对耗尽项做出决定（✅ 接受 → 进入修复队列 / ⏭️ 已知限制 / 📌 后续提醒），与审计发现统一处理。

MUST output exactly:

```
╔══════════════════════════════════════════════════════════╗
║  ⏸️  HARD GATE — Phase 4 人工决策门控                     ║
║                                                          ║
║  请逐条决策（回复格式）：                                   ║
║     #1 ✅ | #2 ❌ [原因] | #3 ⏭️ 已知限制 | #4 📌 后续提醒  ║
║     或: P0 ✅ | P1 #8 #11 ✅ | P2 P3 📌 全部               ║
║                                                          ║
║  ⛔ 在收到你的决策之前，我不会进入 Step 4.4 修复。           ║
║  ⛔ 不会输出 CP4 PASS。                                    ║
╚══════════════════════════════════════════════════════════╝
```

不可替人工预判。不可在人工回复前进入 Step 4.4。
不可假设"P0 显然严重，人工肯定会采纳"而直接修复，跳过询问。
不可假设"不严重，人工应该不会采纳"而跳过询问。
"立即修复""尽快修复"是给人工的优先级标签，不是 AI 绕过门控的许可。任何严重级别都必须经人工决策后才能动手修。

用户回复后，AI 解析规则见 `08-project-audit.md §5.3`。

### Step 4.4 — Loop 1 逐文件修复

✅ 采纳项 → Loop 1（逐文件修复）。流程与 Phase 2 Loop 1 相同：修复 → 构建。

> ⚠️ **修复完成后不得直接进入 CP4**。必须执行 Step 4.5 增量重审，产出重审结果后才能进入 CP4。

### Step 4.5 — 增量重审 + 循环判断（强制，不可跳过）

**所有修复完成后，必须对修改过的文件重跑 L1 + L2 + P2 结构。此步骤不可跳过。**

增量重审具体范围：
- **L1（源库对齐）**：修改行是否仍与原库行为一致？修复是否引入了行为偏差？
- **L2（安全/质量）**：修改引入了新风险吗？（覆盖全文件，不仅限于变更行）
- **P2（结构）**：以下三项必须全部执行，不可省略任一项：
  - R13 注释信任验证：`KNOWN-LIMITATION` / `INTENTIONAL` 等注释是否仍然准确？
  - R15 并排对比：修改文件与修复前的版本逐行对比，确认无关变更未混入
  - 全文件打勾表（仅修改文件）：对照 08 §8.4 格式，全部打勾后才算 P2 完成

（P1/P3 在增量重审阶段可跳过——仅重审结构和安全，无需重跑完整语义对比。）

输出格式：

```
🔍 Step 4.5 增量重审 — Round {round}
   重审文件: [修改过的文件列表]
   重审结果:
   - 新 Critical: [N 项 / 无]
   - 新 High:     [N 项 / 无]
   - 新 Medium:   [N 项]
   - 新 Low:      [N 项]
```

判定：
- **有新 Critical/High** → 追加到问题清单 → 回到 Step 4.3（人工审核新发现；新增项先按 Step 4.2 格式输出分层建议）
- **无 Critical/High 但有新 Medium/Low** → 输出新增 Medium/Low 清单（不阻塞 CP4）→ 人工决定：
  → **改**：选取 ✅ 项 → 回到 Step 4.4（进入内循环修复）；未选取的记入残余清单
  → **不改** → 全部记入残余清单 → 退出循环 → 进入 Step 4.6 CP4
- **无任何新发现** → 退出循环 → 进入 Step 4.6 CP4

### Step 4.6 — CP4: Audit Completion + Token Checkpoint

**前置条件**：Step 4.5 增量重审已执行且输出可查。未执行 Step 4.5 → CP4 自动 FAIL。

**🔖 CP4 快照（必须执行，不可跳过）：**

CP4 PASS 后立即打 git tag，作为 Phase 5 C2.1 签名变更检测的基准：

```bash
cd {{ARKTS_PROJECT}}
git add library/src/main/ets/
git commit -m "Phase 4 CP4 PASS — 审计闭环，冻结 library 基线"
git tag Phase4-SNAPSHOT
```

> ⚠️ **非 git 项目回退**：若 `git rev-parse --git-dir` 失败（项目不在 git 仓库中）：
> - 跳过 git tag 创建
> - 改为输出 `📌 CP4-SNAPSHOT: 非 git 项目，以 HAR md5 代替：{md5}`
> - 计算并记录 `md5 library/build/default/outputs/default/library.har`
> - Phase 5 C.2 改为对比 HAR md5（`md5 library.har` 当前值 vs CP4 记录值），而非 `git diff Phase4-SNAPSHOT`

> 没有此 tag → Phase 5 Step C2.1 无法 diff → 🛑 停止，先补 tag。

**输出：**

先执行自检：
```
🔍 CP4 门控自检（输出 PASS 前逐项确认）：
- [ ] 阶段一 L1+L2 全量扫描完成？
- [ ] 阶段二 P1 语义 + P2 结构 + P3 补漏全部完成（P2 全文件打勾表全部 ✓）？
- [ ] 📂 LOADED: 08-project-audit.md (18 rules R1-R18, blind-spot checklist, 2-stage L1+L2→P1+P2+P3) 确认行存在？
- [ ] Step 4.3 人工决策已收集（非替人工预判、非跳过）？
- [ ] Step 4.5 增量重审（L1+L2+P2 结构）已执行？
- [ ] 无新 Critical/High（或已回到 Step 4.3 决策并处理完毕）？
- [ ] git tag Phase4-SNAPSHOT 已创建？（非 git 项目 → HAR md5 已记录？）
- [ ] 若本 session 发生过 compaction，是否已输出 🔄 Compaction 恢复 行（EXACT 格式: 🔄 Compaction 恢复 — Phase {N}，{X}/{Y} 文件完成，最后 G1: {filename} (streak={streak}, Token={used}K/{budget}K)。请确认后说 Next。无此行 → 恢复协议未执行）？
- [ ] 📊 TOKEN 行已输出（格式: 📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N）？
以上任一项为"否" → 🛑 禁止输出 CP4 PASS，先补执行缺失项
```

自检全部通过后：
```
✅ CP4 PASS — 审计闭环完成
   Step 4.5 增量重审: 已执行, Round {N}/{max_rounds}, 新 Critical: 0, 新 High: 0
   🔖 Phase4-SNAPSHOT tag: 已创建 (commit: {hash})
   P0: X 发现 → Y ✅ / Z ❌ / W ⏭️ / V 📌
   P1: ...
   P2: ...
   P3: ...
   修复后残余: ...（如有）

→ 请更新 06-metrics.md「Phase 4 Project Audit」段
→ 填写完毕说 「Next」 进入 Phase 5 TEST。
```

**Token Checkpoint:** AI 输出强制格式行 `📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N | ≥80%⚠️ / ≥95%🛑`（累计值：优先 /cost 或 transcript 精确值，否则估算，±15% 可接受）。≥80% → warn，≥95% → stop。≥1 compaction → 视为 ≥50%，请核实。

---

## Phase 5: TEST — Loop 3 (Global Test Loop)

> ⚠️ **Phase 5 在 Session 2 执行**。Session 1 CP3 PASS 后终止，用户在新 Session 粘贴 START_HERE.md Prompt Audit 启动 Phase 4-6。

> ⚠️ **Phase 5 入口强制加载**：进入 Phase 5 前（CP4 PASS → Next 后），必须先 Read 05-runbook.md §5.4（Step A-D 完整协议（含子步骤 C.1-C.3）），尤其确认 Step A 的 HARD GATE 视觉块格式（`╔═ HARD GATE ═╗`）和 STOP 规则。不可仅凭 CP4→CP5 过渡时的上下文记忆执行 Phase 5——compaction 可能已移除 §5.4 细节。此加载是 CP5 前置条件 #3（📱 手动运行提示已输出）的前提：若 AI 上下文中没有 §5.4 的完整视觉块格式，Step A 的 HARD GATE 将不会被正确输出。
>
> **Phase 5 硬前置条件（缺一不可进入 Step 5.1）**：
> - [ ] `📂 LOADED: 05-runbook.md §5.4` 确认行已输出（Step A-D 完整协议（含子步骤 C.1-C.3） + HARD GATE 视觉块 + STOP 规则）
> - [ ] `📂 LOADED: 04-test-generator.md` 确认行已输出（Module Config Verification 魔法值 + L1-L5 覆盖层级 + TA1-TA15 反模式表）
> - [ ] `📂 LOADED: 03-auditor.md` 确认行已输出（29 rules R1-R29——测试代码也需审计，不仅限 library 代码）
> - [ ] `entry/` 目录存在且结构完整（build-profile.json5 + hvigorfile.ts + oh-package.json5 + src/main/module.json5 + src/main/ets/pages/Index.ets + src/main/ets/entryability/EntryAbility.ets + 资源文件）。若目录不存在或结构不完整 → 🛑 停止，先按 `04-test-generator.md` TASK 1 + 3 + 4 创建脚手架（目录结构 + module.json5 + 资源文件 + 注册 entry 到 root build-profile.json5；**不含 TASK 2**——测试代码在 Step 5.1 生成）。不可跳过 TASK 3 资源文件（缺少 string.json / main_pages.json 会导致运行时白屏或 11211120 错误）
> 以上四条全部满足 → 方可进入 Step 5.1。任一条缺失 → 🛑 停止，先补加载或补创建。

### Loop Control (Convergence-Based, No Hard Cap)

```
attempt = 0
prev_errors = Infinity  // must monotonically decrease

while true:
  → Step 5.1 (Generate/Repair Tests)
  → Step 5.2 (Audit Test Code)
  → Step 5.3 (Build entry: hvigorw assembleHap)

  current_errors = audit_violations + build_errors

  if current_errors == 0:
    → Step 5.4 (Runtime Verification)
    break

  if current_errors > 0:
    → check 07 Trigger Map (E1-E16). If error pattern matches → output "💡 07 §X" hint

  if current_errors >= prev_errors:
    → STOP. Report: "⚠️ Loop 3 不收敛：本轮 error={current_errors}，上轮={prev_errors}"
    → Human intervenes.

  prev_errors = current_errors
  attempt++

  if attempt % 3 == 0:
    → STOP. Report: "Loop 3 已进行 {attempt} 轮，当前 error={current_errors}"
    → Human decides: continue or intervene.
```

### Loop 3 Gatekeepers (Convergence + Root Cause + Forced Checkpoint)

```
  ┌──────────────────────────────────────────────────────────┐
  │ G-CONV (收敛检查): Current errors < previous errors      │
  │   Flat or rising → STOP, human analyzes root cause.      │
  │                                                          │
  │ G-ROOT (根因分类):                                       │
  │   T1: Build errors point to library files                │
  │     → Report: "Build 错误指向 library/{x}.ets"           │
  │     → STOP. Go back to Phase 2 to fix library file.      │
  │   T2: Runtime failures (not assertion bugs)              │
  │     → Report: "运行时失败 {N} 个，疑似 library 代码 bug" │
  │     → STOP. Analyze root cause.                          │
  │                                                          │
  │ G-CHECK (强制检查点): Every 3 rounds → pause, report     │
  │     status, human decides continue or intervene.         │
  └──────────────────────────────────────────────────────────┘

  Heartbeat (non-checkpoint rounds only, does NOT pause):
    After each round where attempt % 3 != 0 AND convergence is OK:
    → Output round summary:
      "📊 Loop 3 Round {attempt}: 当前 error={current_errors} (上轮={prev_errors}), 累计修复 {total_fixed}"
    → Execution continues WITHOUT waiting for human input.
    → On checkpoint rounds (attempt % 3 == 0) and convergence-stop rounds, the existing
      pause mechanisms take precedence — no duplicate heartbeat needed.
```

### Step 5.1 — Generate Tests
Load `04-test-generator.md` as context.
Provide: Planner's API surface + compiled library code.
Output: complete entry/ module with test code.

### Step 5.2 — Audit Test Code
Load `03-auditor.md` as context.
Provide: generated test code (entry/src/main/ets/pages/Index.ets).
Auditor checks test code against 29 rules (R1-R29).
Output format 与 Step 2.2 相同：`STATUS: PASS — R1-R29 全量扫描，无违规` 或 `STATUS: FAIL\nVIOLATIONS: R1(rule)×N`。
⛔ PASS 必须显式声明 "R1-R29 全量扫描"，仅写 "STATUS: PASS" 视为不完整。
⛔ 禁止自创简化表格替代。格式不符 → 视为未审计。

### Step 5.3 — Build Test Code

**Pre-build check** — 在 `assembleHap` 之前，执行以下配置验证（按顺序，任一步 FAIL → 🛑 停止构建）：

```
🔍 Pre-build: 配置验证
1. module.json5 关键字段：
   grep "ohos.want.action.home" entry/src/main/module.json5 → 必须非空（TA1 最高频幻觉）
   grep "entity.system.home" entry/src/main/module.json5     → 必须非空
   grep "EntryAbility" entry/src/main/module.json5           → 必须在 mainElement 和 abilities[].name 各出现一次
   grep '"pages"' entry/src/main/module.json5                → 必须非空（TA8 — 缺此字段 → 运行时白屏，hvigorw 不检查）
   grep '"phone"' entry/src/main/module.json5                → 必须非空（TA15 — deviceTypes 必须为 ["phone"]；["default"] 在部分模拟器版本上导致启动失败）
2. 资源文件（缺任一个 → 11211120 错误）：
   cat entry/src/main/resources/base/profile/main_pages.json → 必须存在且含 "pages/Index"（路由表文件，缺 → 运行时白屏）
   cat entry/src/main/resources/base/element/string.json     → 必须存在且含 module_desc / EntryAbility_desc / EntryAbility_label
   cat entry/src/main/resources/base/element/color.json      → 必须存在且含 start_window_background
3. 工程注册（漏注册 → hvigorw 不编译 entry，无 HAP 产出）：
   grep '"entry"' build-profile.json5                        → 根 build-profile.json5 modules 数组必须包含 entry（04 TASK 4）
   grep '"icon"' AppScope/app.json5                          → 必须存在（缺 → build 失败）
   grep '"label"' AppScope/app.json5                         → 必须存在（缺 → build 失败）
```

验证 FAIL → 🛑 停止构建，修正后重试。验证 PASS → 继续。

Run: `cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHap`

### Step 5.4 — CP5: Manual Runtime Loop + Token Checkpoint

Trigger: Phase 5 `hvigorw assembleHap` 0 error 通过。

**CP5 前置条件表（缺一不可）：**

| # | 前置条件 | 说明 | 不满足 → |
|---|---------|------|:--|
| 1 | `hvigorw assembleHap` 0 error | 编译通过是必要但不充分条件 | 🛑 禁止 CP5 PASS |
| 2 | 测试代码已通过 03-auditor 审计 | (a) 必须有 `📂 LOADED: 03-auditor.md (29 rules, R1-R29)` 确认行。(b) 输出 `STATUS: PASS — R1-R29 全量扫描，无违规`（格式同 Step 2.2，禁止自创简化表格，PASS 必须声明 R1-R29 全量扫描） | 🛑 禁止 CP5 PASS |
| 3 | 已输出 📱 手动运行提示 | 不可假设"编译通过 = 运行通过"而跳过。**不可用 hdc / aa test / hdc install 等命令行替代手动运行。** | 🛑 禁止 CP5 PASS |
| 4 | 已收到人工测试结果 | 必须人工贴回，不可替人工生成/推测。不可假设"编译通过 = 运行通过"而跳过。**不可用 hdc / aa test / hdc install 等命令行替代手动运行。** 无设备/模拟器时，可用 Node.js oracle 对比验证替代（需显式声明"无设备运行时验证，以 oracle 对比代替"）。 | 🛑 禁止 CP5 PASS |
| 5 | 测试结果 0 失败 | 有失败 → 回到 Step B 修复循环 | 🛑 禁止 CP5 PASS |
| 6 | C.1/C.2/C.3 已完成 | Library 变更检查 + 签名检测 + 安全复核 | 🛑 禁止 CP5 PASS |

> 以上 6 条全部满足 → 方可进入 Step D 输出 CP5 PASS。任一条未满足 → 🛑 禁止输出 CP5 PASS。

**Step 0 — 模拟器前置验证（Step A 的前置步骤）**：

在执行 Step A 手动运行提示之前，AI 必须完成以下验证。任一条 FAIL → 🛑 不输出 Step A HARD GATE，先修复。

| # | 检查项 | 方法 | 不满足 → |
|---|--------|------|:--|
| 1 | entry/ 结构完整 | `ls entry/src/main/ets/entryability/EntryAbility.ets entry/src/main/ets/pages/Index.ets entry/src/main/module.json5 entry/src/main/resources/base/element/string.json entry/src/main/resources/base/profile/main_pages.json` — 全部存在 | 按 04-test-generator.md TASK 1 + TASK 3 补创建 |
| 2 | AppScope 存在且含 bundleName | `grep bundleName AppScope/app.json5` 返回非空 | 按 04-test-generator.md TASK 3 创建 AppScope/app.json5 |
| 3 | root build-profile.json5 含 entry 模块 | `grep '"name": "entry"' build-profile.json5` 返回非空 | 按 04-test-generator.md TASK 4 注册 entry |
| 4 | module.json5 srcEntry 指向文件存在 | `ls entry/src/main/ets/entryability/EntryAbility.ets` 返回文件路径 | 按 04-test-generator.md TASK 3 创建 EntryAbility.ets |
| 5 | module.json5 关键魔法值全部正确 | `grep "ohos.want.action.home" entry/src/main/module.json5` 返回非空 **且** `grep '"phone"' entry/src/main/module.json5` 返回非空（deviceTypes 必须为 `["phone"]`；`["default"]` 在部分模拟器版本上导致启动失败） | 修正 actions 为 `"ohos.want.action.home"`（非 `action.system.home`）；修正 deviceTypes 为 `["phone"]` |
| 6 | HAP 编译通过 | `hvigorw assembleHap 2>&1 \| grep "BUILD SUCCESSFUL"` 返回非空 | 回到 Step 5.3 修复编译错误 |

> ⚠️ 以上 6 项全部 PASS 后，方可输出 Step A HARD GATE。此验证是 CP5 前置条件 #3（📱 手动运行提示）正确执行的前提——若 entry 结构不完整、AppScope 缺 bundlename、或模拟器配置值错误，人工打开 DevEco Studio 后会发现项目无法加载或白屏，浪费人工时间。

**Step A — 提示手动运行（HARD STOP，不可跳过）：**

AI MUST output exactly:

```
╔══════════════════════════════════════════════════════════╗
║  ⏸️  HARD GATE — Step 5.4 手动运行门控                    ║
║                                                          ║
║  📱 请在 DevEco Studio 模拟器上手动运行应用：               ║
║     1. DevEco Studio 打开项目目录                          ║
║     2. 选择模拟器 → Run ▶️                                 ║
║     3. 观察测试页面 ✅/❌/⬜ 数量                            ║
║     4. 将结果贴回来：通过: N  失败: N  跳过: N              ║
║                                                          ║
║  ⛔ 在收到你的测试结果之前，我不会继续。                     ║
║  ⛔ 不会输出 Step B/C/D。不会输出 CP5 PASS。               ║
║  ⛔ 不可假设"编译通过就是运行通过"而跳过手动运行。          ║
║  ⛔ 禁止在此阶段使用 hdc / aa test / hdc install 等         ║
║     命令行替代手动运行——门控的目的是让人亲眼看到            ║
║     模拟器屏幕上的测试结果。                                ║
║  ⛔ 模拟器启动和操作由人工在 DevEco Studio GUI 中完成。      ║
╚══════════════════════════════════════════════════════════╝
```

> ⚠️ **自动化例外**：仅当人工**明确**说"自动运行"或"用 hdc 跑"时，才允许使用 `hdc` 命令行替代手动运行。此时仍需输出 📱 提示（改为"将自动部署到模拟器"），但可绕过 DevEco Studio GUI。人工未明确授权 → 禁止自动化。

**Step B — 解析结果：**

人工贴回结果后，AI 解析并执行：

```
若全部通过（0 失败）→ 进入 Step C

若有失败：
  → 列出失败项 → 分析根因（test bug / library bug）
  → 提示修复 → 修复完毕 → 回到 Step A（重新提示手动运行，再次等待结果）
  → 循环直到 0 失败
```

**Step C — Library 变更检查 & 增量审计：**

**C.1 — 变更文件清单 + 03-auditor：**

检查 Phase 5 期间是否修改了 `library/src/main/ets/*.ets`：
- 无变更 → 跳过 C.2 / C.3，直接进入 Step D
- 有变更 → 列出变更文件 → 回跑 03-auditor (R1-R29) → 有新违规 → 修复 → 回跑 `hvigorw assembleHar` → 进入 C.2

**C.2 — 签名变更检测（硬 gate，不靠 AI 判断）：**

对每个变更文件执行 diff，检查是否涉及函数签名或公共 API 契约变更：

```bash
# 检测 export 函数签名变更（新增/删除/参数变化）
git diff Phase4-SNAPSHOT -- library/src/main/ets/ -- '*.ets' \
  | grep -E '^[+-].*export (function|class|const|interface|type)'

# 检测公共 API 契约变更（Index.ets barrel export 增删）
git diff Phase4-SNAPSHOT -- library/src/main/ets/Index.ets
```

> ⚠️ **Phase4-SNAPSHOT** 是 Phase 4 CP4 PASS 时创建的 git tag（05 Step 4.6）。tag 缺失 → 🛑 停止，先补 tag。

**判定：**
- grep 无输出 → 签名/API 契约未变 → C.3 走路径 A（L2 only）
- grep 有输出 → 签名/API 契约变更 → C.3 走路径 B（L1 + L2）

**C.3 — 增量安全复核：**

**路径 A — 纯 L2（签名未变）：**

Phase 5 改动为内部实现细节（修 bug、调逻辑），不涉及对外契约：

- 输入：变更文件的**完整源码**，标注哪些行是本次变更的
- 对变更文件跑 08 Layer 2（安全检查，覆盖全文件，不仅限于变更行）
- 无新 Critical/High → 过，进入 Step D
- 有新 Critical/High → 输出新增项清单 → 人工决策（✅/❌/⏭️/📌）
  - ✅ 项 → Loop 1 修复 → 增量重审（L2 only）→ 循环至干净
  - 其余标记按 08 流程处理

**路径 B — L1 + L2（签名变更）：**

Phase 5 改动涉及导出的函数签名、类型定义或 barrel export：

- 输入：变更文件的**完整源码**，标注哪些行是本次变更的、哪些函数签名变更了
- 对变更文件跑 08 Layer 1 + Layer 2（覆盖全文件，签名变更部分深审，其余部分快速安全检查）
- L1 检查项：签名参数数量/类型/可选性/返回值是否与原版一致、默认值是否对齐
- 合并 L1 + L2 发现 → 按严重度输出 → 人工决策
- ✅ 项 → Loop 1 修复 → 增量重审（L1 + L2）→ 循环至干净

**输出格式：**
```
🔍 C.2 签名变更检测: [无变更 → 路径A / 有变更 → 路径B，涉及 N 个函数]
🔍 C.3 审计结果: L2 [PASS / N 项]（路径 B 追加 L1 [PASS / M 项]）
```

**Step D — 输出：**

先执行自检（对照 CP5 前置条件表逐项确认）：
```
🔍 CP5 门控自检（输出 PASS 前逐项确认）：
- [ ] 前置条件 #1: assembleHap 0 error？
- [ ] 前置条件 #2: 测试代码已通过 03-auditor 审计（📂 LOADED: 03-auditor.md 确认行存在，输出 STATUS: PASS — R1-R29 全量扫描 / STATUS: FAIL + VIOLATIONS 格式，非自创简化表格）？
- [ ] 前置条件 #3: 📱 手动运行提示已输出（非跳过、非遗忘）？
- [ ] 前置条件 #4: 人工测试结果已收到（非假设、非推测、非"编译通过所以应该没问题"）？
- [ ] 前置条件 #5: 测试结果 0 失败？
- [ ] 前置条件 #6: C.1 变更检查 + C.2 签名检测 + C.3 安全复核 全部完成？
- [ ] 📂 LOADED: 04-test-generator.md (Module Config Verification magic values, L1-L5 coverage, anti-pattern table) 确认行存在？
- [ ] 若本 session 发生过 compaction，是否已输出 🔄 Compaction 恢复 行（EXACT 格式: 🔄 Compaction 恢复 — Phase {N}，{X}/{Y} 文件完成，最后 G1: {filename} (streak={streak}, Token={used}K/{budget}K)。请确认后说 Next。无此行 → 恢复协议未执行）？
- [ ] 📊 TOKEN 行已输出（格式: 📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N）？
以上任一项为"否" → 🛑 禁止输出 CP5 PASS，先补执行缺失项
```

自检全部通过后：
```
✅ CP5 PASS — 运行时验证通过
   通过: N  失败: 0  跳过: M
   Library 变更: [无 / 已回跑 03-Auditor (C.1) + 08 安全复核 (C.3 路径A: L2 / 路径B: L1+L2), PASS]

→ 请更新 06-metrics.md Phase 5 + Phase 6:
  - 测试函数数 / 轮次
  - 运行时通过/失败/跳过
  - 测试覆盖对齐：测试函数数 ≥（Plan API 数 − SKIP API 数）× 2？不满足时标注原因（不阻塞 CP5 PASS）
  - Overall: 反馈轮次 / API 幻觉率 / Token 消耗 / 07 咨询次数

→ 填写完毕说 「Next」 进入 Phase 6 COLLECT。
```

**Token Checkpoint:** AI 输出强制格式行 `📊 TOKEN: 累计=XXXK / 预算=YYYK (ZZ%) | compaction=N | ≥70%⚠️ / ≥90%🛑`（累计值：优先 /cost 或 transcript 精确值，否则估算，±15% 可接受）。≥70% → warn（提醒为 Phase 6 预留空间），≥90% → stop。≥1 compaction → 视为 ≥50%，请核实。

> ⚠️ CP5→CP6 阈值比其它 CP 更严格（70%/90% vs 80%/95%），因为 Phase 6（Final Report + 三方对照 + 06-metrics 补全）是迁移交付的底线产物，不可因 Token 耗尽而跳过。

---

## Phase 6: COLLECT

> ⚠️ **Phase 6 在 Session 2 执行**。Session 1 CP3 PASS 后终止，用户在新 Session 粘贴 START_HERE.md Prompt Audit 启动 Phase 4-6。

> ⚠️ **Token 预算保护**：若当前 Token ≥85%，精简执行：Step 6.0 简化为 5 分钟内 API 数量快速对照（Plan API 数 vs HAR export 数 vs 测试覆盖 API 数），不执行完整 triple alignment 逐项审查。Step 6.1 完整执行。Step 6.2 中不属于 6.0/6.1 的指标标注"因 Token 不足跳过收集"。若 Token ≥95%，输出精简版 Final Report（仅含文件统计 + 测试结果 + 三方对照 + 已知限制，跳过整体评分）。

### Step 6.0 — 📌 后续提醒复核 + 三方对照

**📌 后续提醒复核**：Phase 4 审计中标记 📌 的问题，逐一复查——是否需要在此时修复、或继续延后、或标记为已知限制。

**三方对照**：AI computes alignment across phases:

| Checkpoint | Source | Value |
|-----------|--------|-------|
| Plan API 数 | Phase 1 `plan_apis_total` | N1 |
| HAR 实际导出数 | Phase 3 CP3 #5 (Export 审计) | N2 |
| 测试覆盖数 | Phase 5 测试函数 written | N3 |

Report: `三方对照 — Plan: N1 API → HAR 导出: N2 → 测试覆盖: N3`
- N2 < N1 by >10% → 列出缺失导出
- N3 < N1 by >10% → 列出未测试 API（或 SKIP 原因）

### Step 6.1 — Final Report（HARD OUTPUT，不可跳过）

AI 汇总全流程产出，**MUST output exactly** 以下格式的最终报告。此输出为硬门控，不可省略、不可缩写。

```
============================================
{{LEVEL}} {{LIBRARY_NAME}} — 迁移最终报告
============================================

📊 文件统计
  计划文件: N   通过: N   失败(escalated): N
  首轮通过率: Audit X% / Build X%
  平均轮次: X.X 轮/文件

🔍 Phase 4 项目审计
  发现总数: N
    阶段一 L1 源库对齐: N, L2 独立对抗: N
    阶段二 P1 语义: N, P2 结构: N, P3 补漏: N
  多源同时发现: N（同一问题被 ≥2 个来源独立报告）
  
  严重程度分布:
    🔴 Critical: N → ✅修了 N, ❌驳回 N, ⏭️已知限制 N
    🟠 High:     N → ✅修了 N, ❌驳回 N, ⏭️已知限制 N
    🟡 Medium:   N → ✅修了 N, 📌后续提醒 N
    🔵 Low:      N → ✅修了 N, 📌后续提醒 N
  
  修复后残余: N 项 (列表)

⏭️ 已知限制
  - [ID] 描述 — 原因（平台限制 / ArkTS 不支持）
  ...

📌 后续提醒（Phase 6 复核结论）
  - [ID] 描述 — 决策（仍延后 / 转为已知限制 / 现在修）
  ...

🧪 测试结果
  测试函数: N   通过: N   失败: N   跳过: N
  通过率: X%
  失败根因: ...

📋 三方对照
  Plan API: N → HAR 导出: N → 测试覆盖: N
  差异: (如有)

⭐ 整体评分
  安全性:     X/10
  功能正确性: X/10
  代码质量:   X/10
  API 兼容性: X/10

💡 后续建议
  - ...
============================================
```

### Step 6.2 — Fill 06-metrics + Done

AI 将 Final Report 中的数字填入 06-metrics.md。

全部填写完毕后，**MUST output exactly**：

```
✅ 全部完成。请确认最终报告无误后回复 「Done」
```

此行为硬门控。不可在人工回复 Done 前自行结束会话或认为任务完成。


---

## Parameter Configuration

Before each run, replace these placeholders with actual values（`{{FILE_COUNT}}` / `{{API_COUNT}}` 在 Phase 1 Plan 产出后回填）：

| Variable | Description | Example |
|----------|-------------|---------|
| `{{LEVEL}}` | Level label | L3 |
| `{{LIBRARY_NAME}}` | Target library name | axios |
| `{{SOURCE_INSTRUCTION}}` | Source instruction (pick template from START_HERE.md) | 见 START_HERE.md |
| `{{ARKTS_PROJECT}}` | ArkTS project output dir | ~/output/axios-arkts/ |
| `{{WORK_DIR}}` | Working directory for cloned source repos | ~/harmony-migration/ |
| `{{MATERIAL_DIR}}` | Group B material directory (01-08.md + configs/) | ~/harmony-migration/group-b/ |
| `{{TOKEN_BUDGET}}` | Token budget | 500K |
| `{{FILE_COUNT}}` | Estimated file count (fill exact value after Phase 1) | ~28 |
| `{{API_COUNT}}` | Estimated API count (fill exact value after Phase 1) | TBD |

Use the **same material files** (01-08 + configs/) for any target library.

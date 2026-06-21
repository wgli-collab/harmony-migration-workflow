# Group B — Dynamic Workflow for HarmonyOS Library Migration

## Overview

This directory contains the complete **Dynamic Workflow** setup for migrating open-source TypeScript/JavaScript libraries to HarmonyOS ArkTS. The system combines AI agents with human-in-the-loop gatekeepers in a 6-phase, 3-loop architecture.

---

## Full Architecture

```
                        ┌──────────────────┐
                        │   07 Support Agent │ ← Any non-code question:
                        │   env/IDE/API/FAQ  │   DevEco ops, API lookup,
                        └──────┬─────────────┘   function explanation,
                               │                 config debugging
                               │ (consulted throughout)
                               ▼
╔══════════════════════════════════════════════════════════════════╗
║              🟢  SESSION 1 — Phase 1-3（PLAN → BUILD → VERIFY）  ║
║              AI = orchestrator，同一 session 写全部代码            ║
╚══════════════════════════════════════════════════════════════════╝
┌──────────────────────────────────────────────────────────────────┐
│                        PHASE 1: PLAN                              │
│  01-planner.md → analyze source → output migration plan JSON     │
│  Steps: 1.1 scaffold → 1.2 plan → 1.3 review → 1.4 create        │
│  Step 1.4a: git init（Phase 4/5 tag 依赖）                        │
│  Gate: human approves plan        CP1 自检 → ✅ CP1 PASS          │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                   LOOP 1: Per-File Micro-Loop                     │
│                   (Phase 2: BUILD, dependency order)              │
│                                                                    │
│   for each file:                                                   │
│     ┌──────────┐    ┌───────────────┐    ┌──────────────┐        │
│     │ 02 Executor│ → │  03 Auditor   │ → │ hvigorw Build│        │
│     │  write .ets│    │ R1-R29+G1-G11 │    │  assembleHar │        │
│     └─────┬─────┘    └──────┬────────┘    └──────┬───────┘        │
│           │          violations?         errors?                   │
│           │           yes → fix          yes → fix                 │
│           └──────────────┴──────────────────┘                      │
│                       retry (max 3 rounds)                         │
│                                                                    │
│   ← Auditor PASS + Build 0 error = FILE DONE                      │
│                                                                    │
│   ██ GATEKEEPERS ██                                                │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ G1 文件门控 │ ╔═╗ visual block per file（硬停止）         │   │
│   │             │ "✅ [file] DONE — 等待 Next | {X}/{Y} |     │   │
│   │             │  连续1轮通过: {streak}/5 | Token: ~XXK/YYK" │   │
│   │             │ STOP until human says "Next"                 │   │
│   ├────────────┼──────────────────────────────────────────────┤   │
│   │ G2 模式检测 │ T1: Same violation ≥2 consecutive files     │   │
│   │  (5+1项)   │ T2: Single file ≥2 rounds (logged, no pause)│   │
│   │             │ T2b: 3轮耗尽 → API fit 分析 → exhausted     │   │
│   │             │ T3: streak ≥5 → 批量生成提议(C)             │   │
│   │             │ T4: Build errors → same dependency(D)       │   │
│   │             │ T5: 3 files × 3 rounds → EMERGENCY STOP     │   │
│   └────────────┴──────────────────────────────────────────────┘   │
│   Dynamic Intervention（G2 触发器，详见 05-runbook §G2）:            │
│     T1→A: 违规模式注入 Executor prompt                              │
│     T3→C: 批量生成提议（streak≥5 → 人工决定）                       │
│     T4→D: 依赖优先修复（Build 错误指向同一依赖文件）                 │
│     T5: 紧急诊断（连续 3 文件各 3 轮耗尽）                          │
│   streak 重置: 非1轮通过 / T3批量 / T5熔断 → streak=0              │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      PHASE 3: VERIFY                              │
│  Full library build: hvigorw assembleHar → HAR artifact          │
│  CP3 门控自检（6 项: 编译/文件/指标/CP2/风险/Git） → ✅ CP3 PASS  │
│  🔗 输出 Phase 4-6 启动提示词（含项目路径/CP3摘要/exhausted列表） │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ╔════════╧════════╗
                    ║  ⏸️ SESSION 1  ║ ← 终止，不进入 Phase 4
                    ║     终    止    ║
                    ╠════════════════╣
                    ║  🔗 新 SESSION  ║ ← 复制 🔗 Phase 4-6 提示词
                    ║     启    动    ║    粘贴到新 Session 继续
                    ╚════════╤════════╝
                             │
╔══════════════════════════════════════════════════════════════════╗
║  🔵  SESSION 2 — Phase 4-6（PROJECT AUDIT → TEST → COLLECT）    ║
║  AI = 独立审计员，未写过代码，以对抗视角审查                       ║
║  加载 6 文档（02/03/04/05/06/08——不含 01，migration-plan 在磁盘） ║
╚══════════════════════════════════════════════════════════════════╝
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                  LOOP 2: Audit-Fix-Reaudit (Nested Loop)          │
│                  (Phase 4: PROJECT AUDIT, outer max 5 rounds)     │
│                                                                    │
│   ┌──────────────────────┐  ┌───────────────┐   │
│   │ Step 4.1-4.2         │→ │ Step 4.3      │   │
│   │ 08 Two-Stage Audit   │  │ Human Decision │   │
│   │ Stage1: L1+L2 (scan) │  │ ✅/❌/⏭️/📌   │   │
│   │ Stage2: P1+P2+P3     │  └───────┬───────┘   │
│   │ (narrow-focus deep)  │           │           │
│                        no ✅? → exit → CP4                          │
│                        has ✅ → Inner Loop (convergence, max 10):   │
│                          Step 4.4: Loop 1 fix → Step 4.5: reaudit   │
│                          new Critical/High? → back to Human (4.3)   │
│                          new Medium/Low? → list → human decides     │
│                          no new? → 内循环干净 → exit outer           │
│                                                                    │
│   ██ GATEKEEPERS ██                                                │
│   ┌──────────────────────────────────────────────────────────┐    │
│   │ Human decision gate │ Each finding: ✅ fix / ❌ reject    │    │
│   │                      │ / ⏭️ known limit / 📌 defer→Phase6 │    │
│   ├──────────────────────┼────────────────────────────────────┤    │
│   │ Incremental reaudit  │ After fixes: L1+L2+P2 on modified   │    │
│   │                      │ → Critical/High → force fix         │    │
│   │                      │ → Medium/Low → human decides        │    │
│   └──────────────────────┴────────────────────────────────────┘    │
│   Heartbeat: each fix-reaudit cycle (auto, no pause)              │
│   CP4: git tag Phase4-SNAPSHOT → ✅ CP4 PASS                      │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                  LOOP 3: Global Test Loop                         │
│                  (Phase 5: TEST, convergence-based)               │
│                                                                    │
│   Step 0 — 模拟器前置验证（🆕 Phase 5 入口前强制）：               │
│     entry/ 结构完整 + AppScope/bundleName + entry 模块注册        │
│     + EntryAbility.ets 存在 + 魔法值正确 + HAP 编译通过            │
│                                                                    │
│   ┌──────────────┐  ┌──────────┐  ┌──────────────┐  ┌─────────┐ │
│   │04 Test Gen   │→ │03 Auditor│→ │hvigorw Build │→ │Runtime  │ │
│   │write entry + │  │R1-R29+   │  │ assembleHap  │  │emulator │ │
│   │test pages    │  │  G1-G11  │  └──────┬───────┘  └────┬────┘ │
│   └──────┬───────┘  └────┬─────┘         │               │       │
│          │         violations?      errors?         failures?    │
│          │          yes → fix       yes → fix      yes → analyze │
│          └──────────────┴──────────────┴───────────────┘         │
│                        retry (no hard cap, must converge)          │
│                                                                    │
│   ← error/failure monotonically decreasing + ALL PASS             │
│                                                                    │
│   ██ GATEKEEPERS ██                                                │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ G-CONV │ Current errors < previous (flat/rise → STOP)     │   │
│   │ G-ROOT │ T1: Build errors → library → Phase 2 re-entry   │   │
│   │        │ T2: Runtime failures → analyze root cause       │   │
│   │ G-CHECK│ Every 3 rounds → pause, report status           │   │
│   └────────┴─────────────────────────────────────────────────┘   │
│                                                                    │
│   Step 5.4 — CP5 手动运行门控（6 前置条件）：                       │
│     1. HAP 编译 0 error   2. 测试代码审计 PASS                     │
│     3. 📱 HARD GATE 手动运行提示已输出                              │
│     4. 人工测试结果已贴回  5. 测试 0 失败                           │
│     6. C.1/C.2/C.3 完成（library 变更+签名+安全复核）              │
│   → ✅ CP5 PASS                                                   │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      PHASE 6: COLLECT                             │
│  Triple alignment（Plan API ≡ HAR exports ≡ test coverage）       │
│  📌 后续提醒复核 + 06-metrics.md 汇总填完                         │
│  Final Report → ✅ 迁移完成                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Agent Roles

| Agent | File | Responsibility | Input | Output |
|-------|------|---------------|-------|--------|
| **Planner** | `01-planner.md` | Analyze source, output migration plan | Source repo | JSON plan + project scaffold |
| **Executor** | `02-executor.md` | Write ArkTS code | Task spec from Planner | `.ets` file |
| **Auditor** | `03-auditor.md` | Check against R1-R29 + G1-G11 (40 rules total) | `.ets` file | STATUS: PASS — R1-R29 全量扫描，无违规 / STATUS: FAIL + VIOLATIONS |
| **Test Generator** | `04-test-generator.md` | Create entry module + tests | Library API surface | Full `entry/` module |
| **Orchestrator** | `05-runbook.md` | Execute all 6 phases | All agent prompts | Completed migration |
| **Metrics** | `06-metrics.md` | Record all execution data | Phase outputs | Filled metrics table |
| **Support Agent** | `07-environment.md` | Answer non-code questions | Human queries | Env fix / API answer / IDE guide |
| **Project Auditor** | `08-project-audit.md` | Two-stage audit (Stage1: L1+L2 full scan → Stage2: P1 semantic+P2 structural+P3 gap-fill deep scan) | Full compiled library | Issue list → fix queue |

### 07 Support Agent — Boundary

```
         NON-CODE QUESTIONS               │           CODE TASKS
                                           │
  • DevEco how-to (open, run, screenshot)  │  01 Planner    → plan
  • Environment errors (hvigorw, hdc)      │  02 Executor   → write .ets
  • ArkTS API queries (does X exist?)      │  03 Auditor    → check rules
  • Config debugging (magic strings)       │  04 Test Gen   → write tests
  • Function explanations                  │  05 Runbook    → orchestrate
  • Build/runtime error lookup             │
                                           │
  ──→ ALL go to 07 ──────────────────────  │  ──→ 01-06 stay clean
```

### 07 Trigger Mechanism

During Loop execution, AI performs **deterministic pattern matching** against Build/Runtime errors. When an error matches one of 16 known patterns (E1-E16), AI outputs a directed hint (e.g. `💡 07 §6 — 资源文件错误`). The **operator** decides whether to look up `07-environment.md`. AI does NOT auto-load 07.

| Pattern | Error Signature | AI Hint |
|---------|----------------|---------|
| E1 | `hvigorw` not found / SDK path | `💡 07 §1` |
| E2 | `11211120` resource pack error | `💡 07 §6` |
| E3 | `00401021` module config error | `💡 07 §5` |
| E4 | `signingConfig` / `deviceTypes` / etc. | `💡 07 §4` |
| E5 | DevEco/emulator not responding | `💡 07 §2` |
| E6 | API existence/func explanation needed | `💡 07 §3` |
| E7 | `hdc` not found | `💡 07 §1.2` |
| E8 | npm registry / dependency fails | `💡 07 §7` |
| E9 | Same error ≥2 rounds (non-code) | `💡 07 §8` |
| E10 | `arkts-no-any-unknown` / type system | `💡 07 §9.1` |
| E11 | Object literal / interface mismatch | `💡 07 §9.2` |
| E12 | Function return type / `this` / null | `💡 07 §9.3` |
| E13 | API type mismatch (window, notification...) | `💡 07 §9.5` |
| E14 | UI component / decorator errors | `💡 07 §9.6` |
| E15 | Build error persists ≥2 rounds identical after fixing code | `💡 07 §1.4` |
| E16 | `arkts-no-obj-literals-as-types` / `@State` on non-struct / `@StorageLink` | `💡 07 §9.4` |

Full trigger map and integration: `05-runbook.md` §07 Trigger Map (E1-E16) and `07-environment.md` §Trigger Map.

---

### Loop 1 Gatekeepers (Phase 2: Per-File)

| Gate | Mechanism | Trigger | Action |
|------|-----------|---------|--------|
| **G1 File Gate** | Mandatory stop | After EACH file loop completes | AI MUST output ╔═╗ visual block `✅ [file] DONE — 等待 Next | {X}/{Y} | 连续1轮通过: {streak}/5 | Token: {used}K/{budget}K`（硬停止，与 Phase 4 HARD GATE 同级，详见 05-runbook §Phase 2）; waits for human "Next" |
| **G2 Pattern Detection** | 5 automated triggers (4 Templates + 1 Emergency Stop) | See below | AI pauses, reports trigger, human applies intervention template |
| **Heartbeat** | Auto, no pause | Every N files (N=max(5, ceil(total/10))) | AI outputs `📊 进度: X/Y (Z%) 首轮通过率: … Token: …`; continues without waiting |

**5 Triggers (4 Templates + 1 Emergency Stop):**

| # | Trigger | Meaning | Template |
|---|---------|---------|----------|
| T1 | Same violation ID in ≥2 consecutive files | Executor keeps making same mistake | **A**: Inject rule patch into Executor prompt |
| T2 | Single file passed but took ≥2 rounds | Executor struggled — possible systemic issue | Soft notification — logged to metrics, batch-reviewed at CP2（无模板，不暂停） |
| — | Single file exhausts 3 rounds (not passed) | Executor cannot produce compliant code for this file | **Stop**: AI outputs Template B analysis, records to exhausted list, human says "Next"（决策延迟到 Phase 4 Step 4.3 统一处理） |
| T3 | G1 streak counter reaches 5 | Executor is fluent, can accelerate | **C**: Offer batch generation for same-category files |
| T4 | Build errors from multiple files → same dependency | Dependency file may be buggy | **D**: Pause current, fix dependency first |
| T5 | 3 consecutive files each exhaust 3 rounds | Possible systemic Executor failure | **Emergency Stop**: Human diagnoses prompt/strategy |

### Loop 3 Gatekeepers (Phase 5: Test)

| Gate | Mechanism | Trigger | Action |
|------|-----------|---------|--------|
| **Convergence** | monotonic decrease | errors stop decreasing | STOP, human analyzes root cause |
| **Root cause classify** | error source detection | Build errors → library / Runtime failures → library bug | May re-enter Phase 2 |
| **Forced checkpoint** | round counter | Every 3 rounds | Pause, report status, human decides |
| **Heartbeat** | Auto, no pause | Non-checkpoint rounds (attempt % 3 ≠ 0) | AI outputs `📊 Loop 3 Round N: 当前 error=…`; continues without waiting |

### CP1-CP5 Checkpoints (Every Phase Transition)

Each Phase boundary includes an automated checkpoint. AI runs the checklist, reports PASS/FAIL, prompts metrics recording, then reports token consumption.

| CP | Transition | Scope |
|----|-----------|-------|
| CP1 | Phase 1→2 | Plan integrity (9 automated + 1 human judgment: API count / ≤15 exports / signatures / Map / arrow fn / dep order / config / scaffold / build + classification) |
| CP2 | Phase 2→3 | Build completion (files generated, per-file metrics, G2 triggers) |
| CP3 | Phase 3→4 | Full verification (6 items: build, HAR, deps, anti-pattern, export, risk) |
| CP4 | Phase 4→5 | Audit closure（两阶段内部审计 L1+L2+P1+P2+P3 完成 + 修复队列清空 + 增量重审无新 Critical/High + 人工决策门控通过 → git tag Phase4-SNAPSHOT）
| CP5 | Phase 5→6 | Runtime manual run + library change audit + 03-auditor + 08 security recheck. **6 preconditions** (compile→audit→prompt→result→0-fail→C.1/C.2/C.3) — all must pass before CP5 PASS |

All CP logic embedded in `05-runbook.md`. Token budget: ≥80% warn, ≥95% stop（CP5→CP6 例外：≥70% warn / ≥90% stop——Phase 6 Final Report 不可跳过）. Compaction treated as ≥50% proxy.

---

## Material Inventory

### Core Agent Prompts (P0-P1)

| # | File | Lines | Type |
|---|------|:-----:|------|
| 01 | `01-planner.md` | ~250 | P0 — Ground Truth |
| 02 | `02-executor.md` | ~134 | P1 — Task spec only |
| 03 | `03-auditor.md` | ~250 | P0 — 29 semantic rules (R1-R29) + 11 compiler restrictions (G1-G11) = 40 total |
| 04 | `04-test-generator.md` | ~570 | P1 — Entry module + test generation |
| 08 | `08-project-audit.md` | ~797 | P1 — Two-stage project audit (L1+L2 full scan + P1/P2/P3 narrow-focus deep scan) |

### Process & Data (P0-P2)

| # | File | Lines | Type |
|---|------|:-----:|------|
| 05 | `05-runbook.md` | ~1280 | P2 — Three-loop procedure |
| 06 | `06-metrics.md` | ~1029 | P0 — Execution metrics |
| 07 | `07-environment.md` | ~517 | P0 — Support Agent |

### Operator-Facing (P3)

| File | Purpose |
|------|---------|
| `START_HERE.md` | Session startup + first message template |
| `OPERATION_GUIDE.md` | Full operator manual (templates, checklists, dynamic intervention) |
| `QUICK_REFERENCE.md` | Interactive quick-reference card (AI hints & operator commands) |
| `configs/` | 13 validated project config templates |

### Document Hierarchy

```
P0 (Ground Truth)         P1 (Reference)            P2 (Orchestrate)     P3 (Operator)
01-planner.md ──────────→ 04-test-generator ──────→ 05-runbook ────────→ START_HERE
03-auditor.md ───────────→ 02-executor                    │               OPERATION_GUIDE
06-metrics.md ───────────→ 08-project-audit               │               QUICK_REFERENCE
07-environment.md (P0 side-channel — loaded on-demand      │                    │
  by operator, not auto-loaded by AI; consulted            │                    │
  throughout all phases via 💡 hint triggers)              │                    │
                                                           │                    │
                          Sync direction: always P0 → P1 → P2 → P3
                          Never reverse. 06-metrics always wins data conflicts.
```

---

## Execution Flow (Operator View)

```
START: Open VS Code → New Claude Code Session → Paste first message from START_HERE.md
  │
  ├─ Phase 1: AI analyzes source → outputs plan
  │     → CP1: auto-check 9 items + human confirms classification → record metrics → Next
  │
  ├─ Phase 2: AI loops per-file with G1 (file gate) + G2 (pattern detection)
  │     Human: says "Next" per file, applies Template A/B/C/D if triggered
  │     → CP2: verify all files done + per-file metrics recorded → record summary → Next
  │
  ├─ Phase 3: AI runs full library build
  │     → CP3: auto-check 6 items (build/HAR/deps/anti-pattern/export/risk) → record metrics → Next
  │
  │  🛑 Session 1 终止 — AI 自动读取 START_HERE.md Prompt Audit，填好参数，输出完整 🔗 启动提示词
  │     → 用户复制到新 Session（无需手动拼装 Prompt Audit）
  │
  ├─ Phase 4 (08 Audit): 🔵 Session 2 — Two-stage internal audit — Stage 1: L1+L2 full scan → Stage 2: P1(semantic)+P2(structural)+P3(gap-fill) deep scan
  │     → AI outputs issue list with P0/P1/P2/P3 severity → human reviews → Loop 1 fixes
  │     📊 Heartbeat: each repair round → "Loop 2 Round N/M: 本轮修了 X, 剩余 Y" (auto, no pause)
  │     → CP4: verify audit closure (fix queue clear, reaudit clean) → create git tag Phase4-SNAPSHOT → record metrics → Next
  │
  ├─ Phase 5: AI generates tests → convergence loop
  │     → CP5: prompt manual run → repair loop until 0 failures → library change audit (03 + 08 C.2/C.3) → record metrics
  │
  └─ Phase 6: AI does 📌 follow-up + three-way cross-reference (Plan/HAR/Test) → fills 06-metrics → Done
  
Throughout: any non-code question → consult 07-environment.md
```

---

## Design Principles

1. **Executor prompt**: task spec only. No ArkTS rules injected — compliance enforced downstream by Auditor.
2. **Auditor rules**: from official HarmonyOS public documentation (S1-S8 in 03-auditor.md) + community-verified sources (E1-E2). All rules must be cited by ID (R1-R29).
3. **Progressive Auditor**: 1st audit = violations only. 2nd+ audit = violations + fix suggestions.
4. **File gating**: One file at a time, human-controlled pacing. No batch generation unless T3 (Template C, G1 streak=5) is triggered and human explicitly approves.
5. **Dynamic intervention**: pattern detection triggers human-in-the-loop decisions (Templates A/B/C/D).
6. **Convergence-based exit**: Loop 3 exits on monotonic error decrease + all-pass, not a fixed round count.
7. **Token budget**: hard constraints with human checkpoints at every phase boundary.
8. **Support isolation**: 07 handles all non-code questions; 01-06 stay unpolluted.
9. **Two-stage audit**: 08 runs after Phase 3 — Stage 1: L1+L2 full scan (breadth coverage); Stage 2: P1 semantic + P2 structural + P3 gap-fill narrow-focus deep scan (depth coverage). Independent from 03's per-file compliance checks. Two stages are complementary — Stage 1 ensures nothing is missed by category, Stage 2 ensures nothing is skipped by attention failure.

## 参数化

> **占位符约定**：`{{变量名}}` = AI 模板中替换的运行时参数。`【变量名】` = 操作员填入的实际值。两种语法等价，选择其一即可。

每个库迁移前，替换以下占位符：

| 变量 | 说明 | 示例 | 填写时机 |
|------|------|------|---------|
| `{{LEVEL}}` | 级别标签 | L3 | 迁移前 |
| `{{LIBRARY_NAME}}` | 目标库名称 | axios | 迁移前 |
| `{{SOURCE_INSTRUCTION}}` | 源码获取指令（按来源选择模板，见 START_HERE.md） | 见 START_HERE.md | 迁移前 |
| `{{ARKTS_PROJECT}}` | ArkTS 项目输出目录 | `~/output/axios-arkts/` | 迁移前 |
| `{{WORK_DIR}}` | 工作目录，存放克隆的源码 | `~/harmony-migration/` | 迁移前 |
| `{{MATERIAL_DIR}}` | Group B 物料目录，01-08.md + configs/ | `~/harmony-migration/group-b/` | 迁移前 |
| `{{TOKEN_BUDGET}}` | Token 预算 | 500K | 迁移前 |
| `{{FILE_COUNT}}` | 预估文件数（Plan 产出后回填精确值） | ~28 | Phase 1 后 |
| `{{API_COUNT}}` | 预估 API 数（Plan 产出后回填精确值） | TBD | Phase 1 后 |

Same Group B materials (01-08 + configs) for any target library.

# 06 — Metrics Tracking

> ⚠️ **历史教训（保留供参考）**：早期迁移（2026-06-17 之前）的 Violations 列存在系统性 R5 误标——大量违规被标注为 `R5(arkts-no-*)` 但实际分属 R1（any/unknown）、R2（无类型字面量）、R3（call/construct signatures）、R4（structural typing）、R11（spread）、R16（for...in）、R27（obj-literals-as-types）等。根因：早期 session 中 Auditor 将所有 ArkTS strict-mode 违规统一归类为 R5，未按实际规则 ID 拆分。
>
> **当前规范**：Violations 列必须使用 `R#(subtype)×N` 精确格式（如 `R1(arkts-no-any-unknown)×3`），规则 ID 严格对应 03-auditor.md 的 R1-R29 编号。不可将所有 ArkTS strict-mode 违规笼统归为 R5。此规范从 2026-06-17 起生效。

---

## 使用说明

每个库迁移时，复制下方空白模板，按 Phase 逐段填写。每个 CP 边界（CP1-CP5）填写对应 Phase 的指标。Phase 6 补完汇总，不从零开始写。

### 空白模板

```markdown
## {{LEVEL}}: {{LIBRARY_NAME}} → {{ARKTS_PROJECT}}

### Phase 1: Plan

| Metric | Value |
|--------|-------|
| `plan_files_total` | ___ |
| `plan_apis_total` | ___ |
| `plan_risks_flagged` | ___ |
| Plan approval needed? | ___ (CP1 PASS — _/9) |

### Phase 2: Library Build (per file)

| File | Attempts | Audit Fails | Violations | Build Fails | Status |
|------|:--------:|:-----------:|------------|:-----------:|:------:|
| ... | | | | | |

**TOTAL** | ... | ... | ... | ... | ... |

Summary:

| Metric | Value |
|--------|-------|
| Files total | ___ |
| Files passed | ___ |
| Files escalated (3轮耗尽) | ___ |
| Build 1st-round pass rate | ___ |
| … | |

### Phase 3: Library Compilation

| Metric | Value |
|--------|-------|
| Full build (hvigorw assembleHar) | ___ |
| HAR output | ___ |
| Anti-pattern scan | ___ |
| Export audit | ___ |
| Risk recheck | ___ |
| **CP3** | ___ |

### Phase 4: Project Audit

| Metric | Value |
|--------|-------|
| `audit_findings_total` | ___ |
| `audit_P0_critical` | ___ |
| `audit_P1_high` | ___ |
| `audit_P2_medium` | ___ |
| `audit_P3_low` | ___ |
| Human adoption rate | ___ |
| Re-audit result | ___ |
| **CP4** | ___ |

（若执行了独立审计 Session 2，追加 Re-Audit 子表及 Findings detail 表）

### Phase 5: Test Integration

| Metric | Value |
|--------|-------|
| Test functions written | ___ |
| HAP build result | ___ |
| Runtime rounds | ___ |
| Runtime pass rate | ___ |
| Test bugs found | ___ |
| Library bugs found | ___ |
| **CP5** | ___ |

（追加 Test coverage by API group 表）

### Phase 6: COLLECT

| Metric | Value |
|--------|-------|
| Total sessions | ___ |
| Total token consumption | ___ |
| API hallucination rate | ___ |
| Triple alignment | ___ |
| CP checkpoints | ___ |
| Known gaps | ___ |
```

---

## 填写示例：semver v7.8.4 → semver-arkts

> 以下是 semver 库的完整迁移记录，作为各 Phase 填写格式参考。你的迁移数据会与此不同。

### Phase 1: Plan

| Metric | Value |
|--------|-------|
| `plan_files_total` | 15 |
| `plan_apis_total` | 46 (core) + 6 (ArkTS additions: Options, ReleaseType, VersionString, RangeString, AnySentinel, ANY_SENTINEL) |
| `plan_risks_flagged` | 12 (🔴5: Symbol()→AnySentinel, circular dep Comparator↔Range, RegExp.replace callbacks, for...in→indexed for, obj[key] dynamic access · 🟠3: any/unknown elimination, do...while→while(true), catch(e) throw e · 🟡4: debug no-op, typeof type narrowing, RegExp.exec structural typing, parseInt behavior) |
| Plan approval needed? | Yes ✅ (CP1 PASS — 9/9) |

### Phase 2: Library Build (per file)

| File | Attempts | Audit Fails | Violations | Build Fails | Status |
|------|:--------:|:-----------:|------------|:-----------:|:------:|
| types.ets | 1 | 0 | — | 0 | ✅ DONE |
| constants.ets | 1 | 0 | — | 0 | ✅ DONE |
| debug.ets | 1 | 0 | — | 0 | ✅ DONE |
| parse-options.ets | 1 | 0 | — | 0 | ✅ DONE |
| lrucache.ets | 1 | 0 | — | 0 | ✅ DONE |
| identifiers.ets | 1 | 0 | — | 0 | ✅ DONE |
| regex.ets | 1 | 0 | — | 0 | ✅ DONE |
| semver-class.ets | 1 | 0 | — | 0 | ✅ DONE |
| compare-functions.ets | 1 | 0 | — | 0 | ✅ DONE |
| comparator-range.ets | 2 | 0 | R1(arkts-no-any-unknown)×1, R2(type-mismatch:boolean↔Options)×5, R2(type-mismatch:SemVer-null-guard)×3 | 1 | ✅ DONE |
| parse-functions.ets | 2 | 0 | G11(arkts-limited-throw)×1, arkts-no-types-in-catch×1 | 1 | ✅ DONE |
| version-parts.ets | 1 | 0 | — | 0 | ✅ DONE |
| version-ops.ets | 3 | 0 | R2(type-mismatch:RegExpExecArray→RegExpMatchArray)×1, R2(type-mismatch:Options→boolean)×?, arkts-no-structural-typing×1, arkts-limited-throw×? | 2 | ✅ DONE |
| range-functions.ets | 2 | 0 | R2(type-mismatch:Options→boolean)×4, R1(arkts-no-any-unknown:null→SemVer)×1, R2(type-mismatch:null→Comparator)×1 | 1 | ✅ DONE |
| Index.ets | 1 | 0 | — | 0 | ✅ DONE |
| **TOTAL** | **20** | **0** | **R1(arkts-no-any-unknown)×2, R2(type-mismatch)×14, G11(arkts-limited-throw)×1, arkts-no-types-in-catch×1, arkts-no-structural-typing×1** | **5 build fails** | **15/15** ✅ |

Summary:

| Metric | Value |
|--------|-------|
| Files total | 15 |
| Files passed | 15 |
| Files escalated (3轮耗尽) | 0 |
| Build 1st-round pass rate | 11/15 (73%) |
| Audit pass rate (inline) | 15/15 (100%) |
| Total build rounds | 20 across 15 files |
| Build failures fixed | 5 |
| G1 gate streak (1-round files) | max 3 |
| Heaviest files | comparator-range.ets (~500 lines), range-functions.ets (~390 lines), semver-class.ets (~290 lines) |

### Phase 3: Library Compilation

| Metric | Value |
|--------|-------|
| Full build (hvigorw assembleHar) | ✅ BUILD SUCCESSFUL |
| HAR output | library.har (16,825 bytes) |
| Build time | 595 ms |
| Circular dependency check | 1 resolved (Comparator↔Range → combined single file comparator-range.ets) |
| Anti-pattern scan | 0 violations |
| Export audit | 46/46 APIs + 6 ArkTS additions → Index.ets barrel OK |
| Risk recheck | 12/12 addressed |
| **CP3** | ✅ PASS — 10/10 |

### Phase 4: Project Audit (Session 1 — Inline)

| Metric | Value |
|--------|-------|
| `audit_findings_total` | 5 |
| `audit_CRITICAL` | 0 |
| `audit_HIGH` | 0 |
| `audit_MEDIUM` | 0 |
| `audit_LOW` | 2 |
| `audit_INFO` | 3 |
| **CP4** | ✅ PASS — 10/10 |

### Phase 4 Re-Audit (Session 2 — Independent Auditor)

独立审计员（非原 Session 1 写代码的 AI）执行 08-project-audit.md 两阶段审计。

| Metric | Value |
|--------|-------|
| `audit_findings_total` | 9 |
| `audit_P0_critical` | 0 |
| `audit_P1_high` | 1 |
| `audit_P2_medium` | 4 |
| `audit_P3_low` | 4 |
| Human adoption rate | 9/9 (100%) |
| Files modified in fix | 9 |
| Re-audit result | 0 new Critical/High/Medium/Low |
| 修复后残余 | 🔴0 · 🟠0 · 🟡0 · 🔵0 |

**与原始审计对比**：独立审计员（无创作记忆）发现 9 项 vs 原始审计仅 5 项（且 5 项均为 LOW/INFO 级别）。独立审计的对抗视角成功识别了原始审计忽略的 1 个 P1（代码重复）和 4 个 P2（逻辑 bug + 结构问题）。——这验证了 Two-Session 架构的价值。

### Phase 5: Test Integration

| Metric | Value |
|--------|-------|
| Oracle test pass rate | 60/60 (100%) |
| HAP build result | ✅ BUILD SUCCESSFUL |
| Runtime rounds | 3 (R1: 62✅/4❌; R2: 65✅/1❌; R3: 66✅/0❌) |
| Test bugs found | 5 (all in Index.ets test code) |
| Library bugs found | 0 |
| **CP5** | ✅ PASS — 10/10 |

### Phase 6: COLLECT

| Metric | Value |
|--------|-------|
| Total sessions | 2 (Session 1: Phase 1-3; Session 2: Phase 4-6) |
| Token consumption Session 1 | ~400K / 800K (50%) |
| Token consumption Session 2 | ~230K / 1500K (15.3%) |
| Token consumption total | ~630K |
| API hallucination rate | 0% |
| **Triple alignment** | ✅ Plan 52 API ≡ HAR 52 exports ≡ Test 52 covered (100%) |
| CP checkpoints | CP1 ✅ · CP2 ✅ · CP3 ✅ · CP4 ✅ · CP5 ✅ — ALL 5 PASS |

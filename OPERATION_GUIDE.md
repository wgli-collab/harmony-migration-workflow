# Group B 操作指导

> **占位符说明**：`{{WORK_DIR}}` = 工作目录，存放克隆的源码（如 `~/harmony-migration/`）；`{{MATERIAL_DIR}}` = 物料目录，01-08.md + configs/ 所在（如 `~/harmony-migration/group-b/`）；`{{ARKTS_PROJECT}}` = ArkTS 项目输出目录（如 `~/output/axios-arkts/`）。启动前按实际路径替换。

---

## 目录

1. [环境确认](#1-环境确认)
2. [Session 启动](#2-session-启动)
3. [Phase 1：PLAN（迁移规划）](#3-phase-1plan)
4. [Phase 2：BUILD（库代码生成+审计+编译循环）](#4-phase-2build)
5. [Phase 3：VERIFY（库全量编译）](#5-phase-3verify)
6. [Phase 4：PROJECT AUDIT](#6-phase-4project-audit)
7. [Phase 5：TEST（测试代码+运行时验证）](#7-phase-5test)
8. [Phase 6：COLLECT（指标汇总）](#8-phase-6collect)
9. [参数配置](#9-参数配置)
10. [常见问题](#10-常见问题)

---

## 1. 环境确认

> 💡 **环境问题先查 07**：终端/IDE/配置类问题优先查阅 `07-environment.md`（Support Agent），覆盖错误码索引、DevEco 操作、API 查询等。本节是快速确认清单。

### 1.1 确认终端环境

打开终端，验证以下命令都能正常执行：

```bash
# 1. hvigorw 可用
which hvigorw
# 预期: /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw

# 2. DEVECO_SDK_HOME 已设置
echo $DEVECO_SDK_HOME
# 预期: /Applications/DevEco-Studio.app/Contents

# 3. npmrc 配置了鸿蒙 registry
cat ~/.npmrc
# 预期: @ohos:registry=https://repo.harmonyos.com/npm/

# 4. 工作目录结构正确
ls {{WORK_DIR}}/
# 预期: 物料目录（01-08.md + configs/）及目标库源码目录

# 5. 工作目录干净（~/ 根目录无残留产物）
ls {{ARKTS_PROJECT}} 2>/dev/null
# 预期: No such file or directory
```

如果某项不通过，**先修复再继续**。

### 1.2 确认 IDE 环境

- DevEco Studio 已安装并可以打开
- 模拟器可用（用于 Phase 5 运行时验证）
- VS Code Claude Code 插件已安装

---

## 2. Session 启动

### 2.0 新建 Session

在 VS Code 中：

1. 打开命令面板：`Cmd + Shift + P`
2. 输入 `Claude Code: New Session`
3. 回车，一个新的 Claude Code 对话窗口打开

> **为什么用新 Session？** 确保从干净的上下文开始，避免之前会话的残留信息干扰。

### 2.1 Token 预算

本次迁移的 Token 预算：`{{TOKEN_BUDGET}}`。超出时 AI 会提示，避免无限消耗：

> 每次迁移开**独立新 Session**，预算不跨库累积。预估文件数 `{{FILE_COUNT}}` 可作为预算参考。

### 2.2 发送第一条消息

**打开 `START_HERE.md`，选择对应来源的完整 Prompt（A/B/C/D），把 `[方括号]` 填好，直接粘贴到对话框。**

四种来源：Prompt A（本地已有）/ Prompt B（GitHub）/ Prompt C（Gitee）/ Prompt D（其他 Git URL）。

不需要来这里拼装参数。

### 2.3 预期行为

发送后，AI 会：
0. **CP0**：输出文档清单 & 协议确认表（6 个必加载文档，按 Session 变体：Session 1 加载 01-06 / Session 2 加载 02-06+08；07 按需加载 + 关键约束）→ 关键属性错误会停止
1. 读取 `01-planner.md` 和 `05-runbook.md`
2. 分析目标库源码
3. 输出一份结构化的迁移计划
4. 创建项目脚手架

**你在这一步需要做什么**：
- **CP0 快速扫一眼**：确认 03-auditor.md 写的是 "R1-R29 + G1-G11 (40 rules total)"（不是其他数字）、07 写的是 "按需加载"（不是 Phase 1 加载）
- 确认 AI 理解了 3 Loop 结构（Loop 1: Per-File 逐文件 | Loop 2: Audit-Fix-Reaudit 审计修复 | Loop 3: Global Test 全局测试）
- 确认 AI 读取了正确数量的源文件
- 检查迁移计划是否遗漏了重要的公开 API
- 确认项目脚手架创建正确

---

## 3. Phase 1：PLAN

### 3.1 AI 输出内容

Planner 会输出：
```json
{
  "library_name": "{{LIBRARY_NAME}}",
  "total_functions": {{API_COUNT}},
  "total_files": {{FILE_COUNT}},
  "files": [
    {
      "path": "library/src/main/ets/array/chunk.ets",
      "category": "Array",
      "exports": [
        {
          "name": "chunk",
          "kind": "function",
          "signature": "<T>(array: T[], size?: number) => T[][]",
          "description": "将数组拆分为多个 size 长度的区块"
        }
      ]
    },
    // ... 更多文件
  ],
  "dependency_order": ["types.ts", "internal/...", "array/...", "collection/...", "..." ],
  "project_config": { ... }
}
```

### 3.2 你的检查清单

确认以下各项，不要跳过：

- [ ] API 数量正确？（对照源码导出）
- [ ] 分类合理？
- [ ] 每个文件的函数/导出 ≤ 15 个？
- [ ] 所有签名不含 `any`/`unknown`/`Object` 类型？
- [ ] 动态键值对使用了 `Map<K,V>` 替代？
- [ ] 回调模式使用了箭头函数而非 call-signature interface？
- [ ] `dependency_order` 正确？（类型文件和工具函数先于具体实现）
- [ ] `project_config` 使用了模板中的值？（SDK 6.1.1(24), hvigor 6.24.2）
> 以上为撰写时的最新版本。若 DevEco Studio 已更新，以实际安装版本为准（查看方式：DevEco Studio → Help → About → HarmonyOS SDK 版本）。
- [ ] 项目脚手架文件已创建在目标目录？
- [ ] `hvigorw assembleHar` 在空项目上能成功？（验证脚手架正确）

### 3.3 记录指标

打开 `{{MATERIAL_DIR}}/06-metrics.md`，在对应级别的 Phase 1 部分填写：

```
plan_files_total: ___     （计划生成的文件数）
plan_apis_total: ___      （计划覆盖的 API 总数）
plan_risks_flagged: ___   （标记的 API 风险数）
```

### 3.4 CP1：Phase 1 → 2

AI 必须执行 05-runbook Step 1.5 CP1（§3.2 的 9 项自动 + 1 项人工判断）→ PASS 后提示录指标 → Token Checkpoint（自动计算累计消耗并汇报）。

- 累计 ≥ 预算的 80% → AI 警告并暂停，等你决定
- 累计 ≥ 预算的 95% → 停止新工作，只收尾当前 Phase
- 发生过上下文压缩（compaction）→ 视为已消耗 ≥ 50%，主动检查

---

## 4. Phase 2：BUILD

这是核心阶段。对 Planner 输出的**每个文件**，按照 `dependency_order` 的顺序，逐文件执行循环。

**双门控机制**：每个文件经过 **G1 文件门控**（完成后必须等待 "Next"）和 **G2 模式检测**（5 个触发器 — 4 模板 + 1 紧急熔断，详见 §4.5）。

> 📊 **心跳（自动，不暂停）**：Phase 2 每 N 个文件（N=max(5, ⌈总数/10⌉)）、Phase 4 每轮修复后、Phase 5 非检查点轮次，AI 会自动输出进度摘要。不需要你做任何操作，只管看着。详见 `05-runbook.md`。

### 4.1 单文件循环（G1 文件门控）

对于每个文件，AI（你作为 orchestrator）需要执行以下步骤：

```
文件: library/src/main/ets/array/chunk.ets
尝试次数: 1/3
──────────────────────────────────────

Step A — EXECUTE:
  加载 02-executor.md 角色
  提供 Planner 中该文件的任务描述:
    "路径: library/src/main/ets/array/chunk.ets
     导出: chunk<T>(array: T[], size?: number) => T[][]
     描述: 将数组拆分为多个 size 长度的区块
     依赖: 无（leaf 文件）"
  Executor 生成 .ets 代码 → 保存到文件

Step B — AUDIT:
  加载 03-auditor.md 角色 → 输出 📂 LOADED: 03-auditor.md (29 rules, R1-R29) 确认行
  提供 Executor 生成的 .ets 文件
  Auditor 逐行检查 29 条规则（R1-R29）
  
  第 1 次审计（首次打回）: 只输出违规名单，不给修复建议
  输出: STATUS: PASS — R1-R29 全量扫描，无违规
        或
        STATUS: FAIL
        VIOLATIONS: R1(arkts-no-any-unknown)×2, R5(arkts-no-props-by-index)×1

Step C — 决策:
  IF Auditor PASS → 进入 Step D
  IF Auditor FAIL AND attempt < 3 → 将违规名单回传给 Executor，attempt++，回到 Step A
  IF Auditor FAIL AND attempt = 3 → 触发 "3 轮耗尽" 流程 → AI 输出模板 B 分析，记录文件到 exhausted 列表，人工说 Next 继续（决策延迟到 Phase 4 Step 4.3 统一处理）

Step D — BUILD:
  执行: cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHar
  
  IF 0 errors → 文件 DONE ✅
  IF errors AND attempt < 3 → 将编译错误回传给 Executor，attempt++，回到 Step A  
  IF errors AND attempt = 3 → 触发 "3 轮耗尽" 流程 → AI 输出模板 B 分析，记录文件到 exhausted 列表，人工说 Next 继续（不在 Phase 2 当场决策）
```

> ⚠️ 3 轮耗尽时不在 Phase 2 当场决策。AI 记录文件并输出模板 B 分析，人工说 "Next" 继续下一个文件。遗留问题统一在 Phase 4 Step 4.3 决策。

### 4.2 实际操作方式

> ⚠️ **注意**：以下为角色分工概念示意（展示 Executor/Auditor 的职责边界），非实际对话模板。实际执行中 AI 自动推进文件，仅在 G1 门控处等待 "Next"，不需要人工说 "现在处理下一个文件"。

> **07 提示机制**：Build 失败时，如果 AI 检测到错误匹配已知的环境/配置模式（如 `11211120`、`hvigorw: command not found`、`signingConfig` 等），会输出 `💡 07 §X — …建议 @07-environment.md §X` 的提示。你可以选择：(a) 去查 07 对应节，修复后说 "Next" 继续，或 (b) 忽略提示继续。详细触发映射见 `05-runbook.md` §07 Trigger Map。

在会话中，对每个文件，你可以这样交互：

```
现在处理下一个文件：library/src/main/ets/array/chunk.ets

=== EXECUTOR ===
请以 Executor 角色（加载 02-executor.md）编写此文件。

任务规格：
- 路径: library/src/main/ets/array/chunk.ets
- 导出: chunk<T>(array: T[], size?: number) => T[][]
- 描述: 将数组拆分为多个 size 长度的区块，不足的放在最后一个区块
- 依赖: 无

只输出代码，不要加解释。
```

AI 输出代码后，保存到对应文件，然后：

```
=== AUDITOR ===
请以 Auditor 角色（加载 03-auditor.md）检查以下文件。

[粘贴文件内容]

这是第 1 次审计，只输出违规编号和行号，不给修复建议。
```

如果 Auditor PASS：

```
Auditor 通过。执行编译验证：
cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHar
```

如果编译通过，记录指标，继续下一个文件。

如果 Auditor FAIL 或编译 FAIL：

```
Auditor/编译 未通过。违规/错误如下：
[粘贴违规或错误]

这是第 2 次尝试。请以 Executor 角色重新生成此文件，修复上述问题。
```

### 4.2.1 依赖文件发现（Support File）

当 Executor 发现当前文件依赖了迁移计划之外的模块/函数（如 `src/helpers/uuid.ets` 不在 plan 中），AI 会暂停当前文件并输出 `📎 [support_file] 新增` 提示。此时 AI 会：

1. 为发现的依赖文件创建任务规格
2. 对该依赖文件执行标准 Loop 1（Executor → Auditor → Build）
3. 依赖文件完成后，回到原文件继续

你只需正常回复 `Next` 即可。发现 support file 的情况会被记录到 06-metrics。

### 4.3 记录指标

为**每个文件**在 `06-metrics.md` 的 Phase 2 表格中记录：

| File | Attempts | Audit Fails | Violations | Build Fails | Status |
|------|:--------:|:-----------:|------------|:-----------:|:------:|
| array/chunk.ets | 1 | 0 | — | 0 | DONE |
| collection/groupBy.ets | 3 | 2 | R1,R2,R5 | 1 | DONE |
| ... | | | | | |

### 4.4 预期典型流程

一个典型的文件可能经历：

```
文件 A (简单，无违规):
  Executor → Auditor PASS → Build PASS → DONE ✅  (1 轮)

文件 B (有 any 类型 + 动态索引):
  Executor → Auditor FAIL (R1, R5) → 
  Executor (修复) → Auditor PASS → Build PASS → DONE ✅  (2 轮)

文件 C (有 call-signature + untyped literal):
  Executor → Auditor FAIL (R2, R3) →
  Executor (修复) → Auditor FAIL (R2 未修好) →
  Executor (修复) → Auditor PASS → Build PASS → DONE ✅  (3 轮)

文件 D (类型问题 Auditor 抓不出，Build 才报错):
  Executor → Auditor PASS → Build FAIL (类型不匹配) →
  Executor (修复) → Auditor PASS → Build PASS → DONE ✅  (2 轮)
```

### 4.5 动态干预（Human-in-the-loop）

AI 执行 Phase 2 时，会按照"执行纪律"在以下情况暂停。当 AI 报告触发条件时，从下方复制对应模板发送。

| # | 触发条件 | 你的动作 | 模板 |
|---|---------|---------|------|
| 1 | 同一违规 ID 在 ≥2 个连续文件中出现 | 给 Executor 追加针对该规则的补丁指令 | 模板 A |
| 2 | 单文件 ≥2 轮后通过 | （不暂停，指标已记录）CP2 批量回顾 | — |
| 2b | 单文件 3 轮耗尽仍未通过 | AI 输出模板 B 分析 → 记录到 exhausted 列表 → Next 继续（Phase 4 统一决策） | 模板 B（API fit 分析）|
| 3 | G1 streak 计数器达 5 | 暂停，询问是否对同类文件批量生成 | 模板 C |
| 4 | Build 错误指向同一依赖文件 | 暂停，优先修依赖文件 | 模板 D |
| 5 | 连续 3 个文件各耗尽 3 轮仍未通过 | 🛑 紧急暂停。检查 Executor prompt 是否被污染、策略是否需要调整、或这些 API 根本上不兼容 | 人工诊断 |

---

**模板 A — 同一规则连续出现（以 R1 为例）**

```
检测到 R1 (any/unknown) 在 [文件1] 和 [文件2] 连续出现，共 N 处违规。

从下一个文件起，对 Executor 追加以下硬约束：

"执行规则补丁 — 从现在开始，你生成的每个 .ets 文件必须满足：
1. 禁止使用 any 关键字
2. 禁止使用 unknown 关键字
3. 所有函数参数必须有显式类型（string、number、boolean、T 泛型、联合类型）
4. 所有函数返回值必须有显式类型
5. 回调参数类型必须明确标注（如 (item: T, index: number)）

这是硬约束，不是建议。违反任何一条都会被打回。"

确认收到后我发 Next。
```

如果连续出现的是 R5（动态索引 `obj[key]`），把上面换成：
```
"执行规则补丁：
1. 禁止 obj[key] 或 obj["prop"] 写法
2. 所有属性访问用 obj.prop 点号
3. 需要动态 key 时用 Map<K, V>.get(key)"
```

如果连续出现的是 R2（无类型字面量 `{}`），换成：
```
"执行规则补丁：
1. 禁止未声明类型的对象字面量 { key: val }
2. 必须先用 interface 或 class 声明类型，再使用
3. 简单键值对用 Record<K, V>"
```

---

**模板 B — 单文件卡住（AI 自动输出分析，人工说 Next 继续。决策延迟到 Phase 4）**

```
[文件名] 已 3 轮未收敛。

当前违规/错误： [从 AI 报告中粘贴]

API Fit 分析（AI 自动输出，不在此处阻塞等待决策）：
1. 这些违规是否因为这个 API 本身不适合 ArkTS？
   - 如果是 → 在 migration plan 中标记该 API 为 skipped，注明原因
   - 如果是（例如依赖 Node.js / 浏览器 API）→ 写一个 stub，标注 // NOT_SUPPORTED
2. 如果只是写法问题 → AI 提出正确写法建议。

→ 文件已记录到 exhausted 列表。人工说 "Next" 继续下一个文件。
→ 遗留问题在 Phase 4 Step 4.3 统一决策。
```

---

**模板 C — 可加速**

```
最近 5 个文件都 1 轮通过。当前文件类型：[如 string_*.ets]。

是否可以对剩余同类型文件（[列出文件]）批量生成？规则：
- 一次性生成所有列出文件的 .ets 代码
- 我会逐个文件执行 Auditor 审计
- Build 仍然逐个进行
确认后我发 Next。
```

---

**模板 D — 依赖优先**

```
多个文件的 Build 错误都指向 [依赖文件名]。这说明依赖文件本身可能有问题。

暂停当前文件。回到 [依赖文件名]，重新执行 Executor→Auditor→Build。
修复完成后再回到当前文件。
```

---

### 4.6 Token Checkpoint：Phase 2 → 3

AI 必须执行 05-runbook Step 2.5 CP2 → PASS 后提示录指标 → Token Checkpoint（方法同 §3.4）。

---

## 5. Phase 3：VERIFY

所有文件生成完毕后：

### 5.1 全量编译

```bash
# 注意：如果 hvigorw 不在 PATH 中，使用绝对路径并设置 DEVECO_SDK_HOME
export DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents"
cd {{ARKTS_PROJECT}} && /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHar
```

> ⚠️ **已知问题**：`hvigorw` 可能不在 `PATH` 中（Claude Code 沙箱或终端环境未继承）。使用上面的绝对路径 + DEVECO_SDK_HOME 可解决。如果仍然找不到，在终端手动执行并把结果贴回 AI。

### 5.2 检查

- [ ] 0 编译错误？
- [ ] HAR 文件生成在 `library/build/default/outputs/default/*.har`？
- [ ] 所有文件间 import 正确（没有循环依赖）？
- [ ] Anti-Pattern 全量扫描：`any`/`unknown`/`Symbol`/`delete`/`...spread`/`for...in`/`.bind()`/`.call()`/`.apply()`/`WeakMap` → 0 violations？
- [ ] Export 审计：每个文件 ≤ 15 导出，Index.ets barrel 是否覆盖所有 public API？
- [ ] 风险复核：Plan Phase 1 中标记的 N 个风险 → 逐一确认已解决/缓解/仍存留

### 5.3 CP3 → Session 1 终止（两 Session 架构）

AI 必须执行 05-runbook Step 3.2 CP3 → PASS 后，**Session 1 终止**。

> ⚠️ **两 Session 架构**：CP3 PASS 后 AI 输出「🔗 Phase 4-6 启动提示词」代码块即停止当前 Session。用户将此提示词复制到**新 Session** 粘贴，以独立审计员身份执行 Phase 4-6。跨 Session 数据通过磁盘文件传递（migration-plan.json / 06-metrics.md / HAR / .ets 源码 / git tag）。

Phase 4-6 的完整操作指南见下方 **[§6 Phase 4](#6-phase-4project-audit)**、**[§7 Phase 5](#7-phase-5test)**、**[§8 Phase 6](#8-phase-6collect)**。审计流程细节见 `08-project-audit.md` 及 `05-runbook.md` §Phase 4 Steps 4.1–4.6。

---

## 6. Phase 4：PROJECT AUDIT

### 触发时机

Session 1 CP3 PASS 后，用户将 AI 输出的「🔗 Phase 4-6 启动提示词」粘贴到**新 Session**。AI 以独立审计员身份（未写过代码）执行 Phase 4。库代码已全量编译通过，此时审计最准确。

### 审计输出示例

```
=== 08 审计完成：axios ===
发现 21 个问题
  阶段一（L1+L2）: 12 项    阶段二（P1+P2+P3）: 9 项

🔴 P0 — 立即修复（6 项）
  #1  [L2-P1] SSRF 无内网 IP 过滤              harmonyHttpAdapter.ets:15
  #2  [L1]    combineURLs 正则 bug             combineURLs.ets:8
  ...

🟠 P1 — 尽快修复（7 项）
  ...

🟡 P2 — 常规修复（5 项）
  ...

🔵 P3 — 可延后（3 项）
  ...

---
回复格式: #1 ✅ | #2 ❌ 原因 | #3 ⏭️ 已知限制 | #4 📌 后续提醒
         或: P0 ✅ | P1 #8 #11 ✅ | P2 P3 📌 全部
```

### 你的决策

```
P0 ✅  P1 #8 #10 ✅  #7 #9 ❌ 不重要  P2 P3 📌 全部
```

### 修复执行

AI 将 ✅ 项排入 Loop 1（逐文件修复，与 Phase 2 流程相同）。修复完毕 → 增量重审（L1+L2+P2结构）→ 无新 Critical/High → AI 必须执行 05-runbook Step 4.6 CP4（含两阶段审计确认 + 外部审计确认 + git tag Phase4-SNAPSHOT，CP5 C.2 签名检测依赖此 tag）→ PASS 后提示录指标 → Token Checkpoint（方法同 §3.4）→ Next 进入 Phase 5 TEST.

---

## 7. Phase 5：TEST

### 7.1 AI 生成测试代码

在会话中：

```
=== Phase 5: TEST GENERATOR ===

请以 Test Generator 角色（加载 04-test-generator.md）生成测试模块。

输入：
1. Planner 的 API 清单（Phase 1 的输出）
2. 编译完成的库代码（{{ARKTS_PROJECT}}/library/src/main/ets/）

你的任务：
1. 创建 entry/ 模块，使用 configs/ 中的模板
2. 为每个公开 API 编写测试函数
3. 配置 module.json5（如需要 INTERNET 权限）
4. 确保 AppScope/app.json5 有 icon 和 label
5. 确保测试代码本身通过 Auditor 的 29 条规则

输出完整的 entry/ 目录结构和所有文件。
```

### 7.2 测试代码审计+编译循环

> **07 提示机制**：与 Phase 2 相同，Build/Runtime 失败时如果 AI 检测到错误匹配 E1-E16 模式，会输出 `💡 07 §X — …建议 @07-environment.md §X` 提示。操作方式同 §4.2。

与 Phase 2 相同的循环逻辑，但对象是测试代码：

```
Test Generator → Auditor → Build (DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHap)
                     ↑ FAIL         │
                     │       FAIL   │
                     └──────────────┘
                    (收敛性退出，不限轮数)
```

### 7.3 手动运行时验证

> **CP5 前置条件（6 条）**：`hvigorw assembleHap` 0 error → 测试代码已通过 03-auditor 审计 → 已输出 📱 手动运行提示 → 已收到你的测试结果 → 0 失败 → C.1/C.2/C.3 完成。6 条全部满足，AI 才会输出 CP5 PASS。输出 PASS 前还会执行自检逐项打勾。

1. 用 DevEco Studio 打开 `{{ARKTS_PROJECT}}`
2. 选择模拟器
3. 点 Run ▶️
4. 观察测试页面上的 ✅/❌ 数量
5. 将结果贴回 AI：`通过: N 失败: N 跳过: N`

AI 会以视觉块（`╔═ HARD GATE ═╗`）提示手动运行，不会在收到你的结果前继续。

**运行时通过后，AI 自动执行**（详见 05-runbook Step C.1–C.3）：
- **C.1 变更检查 + 03-auditor**：Phase 5 期间改了哪些 library 文件 → 回跑 03-auditor (R1-R29)
- **C.2 签名变更检测**：`git diff Phase4-SNAPSHOT` 检测 export 签名是否变更（硬 gate）
- **C.3 增量安全复核**：路径 A（签名未变 → 08 Layer 2 only）/ 路径 B（签名变更 → 08 Layer 1 + Layer 2）→ 新 Critical/High → 人工决策 → Loop 1 修复

### 7.4 记录指标

在 `06-metrics.md` Phase 5 记录：

```
test_functions_written: ___    （测试覆盖的 API 数）
test_attempts_to_pass: ___     （测试代码循环轮次）
runtime_tests_total: ___
runtime_tests_passed: ___
runtime_tests_failed: ___
runtime_tests_skipped: ___
```

### 7.5 Phase 5 动态干预

Phase 5 的 Loop 3 与 Phase 2 不同——失败可能来自 library 代码而非测试代码。干预策略也有区别：

| # | 触发条件 | 你的动作 |
|---|---------|---------|
| 1 | 测试 Build 失败指向 library 文件 | 暂停，回到 Phase 2 修复 library 文件，再继续测试循环 |
| 2 | 运行时测试失败（非断言 bug） | 暂停，确认是 library bug 还是测试 bug。library bug 需回到 Phase 2 |
| 3 | 测试代码自身 Audit 违规 | 与 Phase 2 相同流程：给 Test Generator 回传违规 |

**核心区别**：
- Phase 2 的动态干预重点在 **Executor prompt 补丁**（防止同类违规重复出现）
- Phase 5 的动态干预重点在 **根因分类**（测试代码问题 vs library 代码问题），因为修错方向会导致无限循环

### 7.6 Token Checkpoint：Phase 5 → 6

AI 必须执行 05-runbook Step 5.4 CP5（含 CP5 门控自检，逐项确认 6 条前置条件 + Token Checkpoint）→ PASS 后提示录指标。

> ⚠️ **CP5→CP6 Token 阈值比其它 CP 更严格**：≥70% warn / ≥90% stop（通用阈值 80%/95%）。因为 Phase 6（Final Report + 三方对照 + 06-metrics 补全）是迁移交付的底线产物，不可因 Token 耗尽而跳过。

---

## 8. Phase 6：COLLECT

### 8.1 AI 输出 Final Report

测试完成后，AI 会汇总全流程产出，输出如下完整报告：

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

你确认报告内容无误后，AI 填入 06-metrics。

### 8.2 汇总当前级别全部指标

填写 `06-metrics.md` 中当前级别的 Overall 部分。

### 8.3 关键数据

| 维度 | 值 |
|------|:--:|
| 编译错误数 | ___ |
| 反馈轮次 | ___ |
| 编译结果 | ___ |
| 运行时通过率 | ___ |
| API 幻觉率 | ___ |
| Token 消耗 | ___ |

---

## 9. 参数配置

**启动前不需要手动替换变量。** 直接打开 `START_HERE.md`，选择对应来源的完整 Prompt（A/B/C/D），把所有 `[方括号]` 填成你的实际值，粘贴到新 Session。四个来源：

| 来源 | Prompt | 需要填的值 |
|------|--------|----------|
| 本地已有 | A | `[路径]` `[输出目录]` `[级别]` `[库名]` `[Token预算]` |
| GitHub | B | `[GitHub URL]` `[库名]` `[输出目录]` `[级别]` `[Token预算]` |
| Gitee | C | `[Gitee URL]` `[库名]` `[输出目录]` `[级别]` `[Token预算]` |
| Git URL | D | `[Git地址]` `[库名]` `[输出目录]` `[级别]` `[Token预算]` |

所有级别使用**完全相同的 Group B 物料文件**（01-08.md + configs/），只是 Prompt 参数不同。

---

## 10. 常见问题

> **07 Support Agent 隔离边界**：
> - **所有非代码问题 → 07**：环境报错（按错误码索引）、DevEco 操作指南、API 可行性查询、函数解释、配置调试
> - **所有代码任务 → 01-06**：规划、执行、审计、测试生成、流程编排、指标记录
> - 这是硬边界，防止 agent prompt 污染。以下 Q&A 是流程/方法论层面的问题，07 不覆盖。

### Q1: AI 生成的文件不符合模板怎么办？
→ 把 configs/ 中的模板内容直接粘贴给 AI，说"请严格使用这个模板"。

### Q2: Auditor 漏报违规怎么办？
→ Auditor 是 AI agent，可能漏报。编译阶段（hvigorw）是最终的 ground truth。如果编译报错但 Auditor 通过了，记录在案，作为 Auditor 的 false negative 率。

### Q3: Executor 3 轮都没通过怎么办？
→ 不静默失败。AI 会输出 `⛔ 3 轮未通过 — 转入 Phase 4 审计` + 模板 B API fit 分析，文件记录到 exhausted 列表。人工说 "Next" 继续下一个文件。
**决策不在 Phase 2 当场做**——遗留问题在 Phase 4 Step 4.3 统一决策（skip→audit / split / skip→known）。

### Q4: 测试代码需要 INTERNET 权限但 planner 没标注怎么办？
→ 在 Phase 5 的 module.json5 中手动添加：
```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET", "reason": "$string:internet_reason" }
]
```

### Q5: 某次编译报错后，AI 修了但引入了新错误怎么办？
→ 正常现象。只要在 3 轮内收敛即可。超过 3 轮标记 FAILED。

### Q6: 整个流程预计需要多久？
→ 取决于库的复杂度。粗略估计：
- 小库（<15 文件）：0.5-1 小时
- 中等库（15-50 文件）：1-3 小时
- 大库（>50 文件）：2-4 小时

### Q7: 中途 Session 断了怎么办？
→ 重新打开 session 后，从 START_HERE.md 复制 Prompt R，粘贴到对话中。AI 将自动执行完整的恢复协议（compaction 恢复行 → 磁盘扫描 → 断点推断 → 文档重载 → 人工确认）。不可使用简化指令直接指定 Phase——那样会跳过验证步骤。

---

## 附录：Group B 完整文件结构

```
{{MATERIAL_DIR}}/
├── START_HERE.md              ← 启动指南（4 个完整 Prompt）
├── QUICK_REFERENCE.md         ← 交互速查卡（AI 怎么提示，你怎么回）
├── README.md                  ← 技术总览
├── 01-planner.md              ← Planner prompt（含 20 个官方来源）
├── 02-executor.md             ← Executor prompt（只含任务规格，不含 ArkTS 规则）
├── 03-auditor.md              ← Auditor prompt（29 条规则）
├── 04-test-generator.md       ← Test Generator prompt
├── 05-runbook.md              ← 执行流程
├── 06-metrics.md              ← 指标记录表
├── 07-environment.md          ← Support Agent（环境/IDE/API/函数问询）
├── 08-project-audit.md        ← 项目审计（两阶段审计 + 修复闭环）
└── configs/                   ← 验证过的工程模板（16 个文件）
    ├── app.json5
    ├── appscope-string.json
    ├── build-profile.json5
    ├── build-profile-with-entry.json5
    ├── entry-hvigorfile.ts
    ├── entry-oh-package.json5
    ├── hvigor-config.json5
    ├── hvigorfile.ts
    ├── library-build-profile.json5
    ├── library-hvigorfile.ts
    ├── library-oh-package.json5
    ├── main_pages.json
    ├── module.json5
    ├── oh-package.json5
    ├── string.json
    └── VERSION-NOTES.md
```

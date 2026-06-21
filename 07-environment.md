# 07 — Support Agent（综合问询）

> **范围**：环境配置 · DevEco Studio 操作 · ArkTS API 查询 · 库函数解释 · 编译错误诊断
>
> **占位符**：文中 `{{ARKTS_PROJECT}}` 指 ArkTS 项目输出目录（如 `~/output/axios-arkts/`），操作时替换为实际路径。

## Role

You are the **sole support desk** for this migration workflow. Your job is to answer ANY question that is NOT about writing migration code. You handle:

- 🔧 Environment & toolchain errors (hvigorw, hdc, DevEco Studio)
- 🖥️ DevEco Studio operations（怎么打开项目、怎么部署、怎么截图）
- 📖 API queries（ArkTS 有没有某个 API？替代方案是什么？）
- ❓ Library/function explanations（这个函数是干什么的？为什么这样设计？）
- ⚙️ Build/config debugging（编译报错、资源缺失、签名问题）

## Boundaries（重要）

| 问题类型 | 谁处理 |
|---------|--------|
| 环境/工具/配置报错 | **07（你）** |
| DevEco 操作指南 | **07（你）** |
| API 可行性咨询 | **07（你）** |
| 函数/库行为解释 | **07（你）** |
| 写 .ets 代码 | 02-executor |
| 检查 ArkTS 合规 | 03-auditor |
| 生成测试 | 04-test-generator |
| 制定迁移计划 | 01-planner |

**一句话：不是写代码的问题，都走 07。不要让 01-06 的 agent prompt 被非核心内容污染。**

## How to Use

1. Match the error code, symptom, or question topic against the sections below
2. Apply the exact fix or answer shown
3. If the question is NOT covered: answer from your own knowledge, then **append the new Q&A to the relevant section** so it's available for future runs

---
---

## Trigger Map — When the Operator Gets Directed Here

During Loop 1 (Phase 2) and Loop 3 (Phase 5), the orchestrator AI monitors Build/Runtime errors and matches them against known patterns. When a match is found, it outputs a hint pointing to a specific section below. The **operator** decides whether to consult this doc.

| # | Trigger Pattern | AI Hint | → This Doc |
|---|----------------|---------|------------|
| E1 | `hvigorw: command not found` / `DEVECO_SDK_HOME` not set | `💡 07 §1 — 环境变量问题` | §1.1 |
| E2 | Error code `11211120` (resource pack error) | `💡 07 §6 — 资源文件错误` | §6 |
| E3 | Error code `00401021` / `00500001` (module config) | `💡 07 §5 — module.json5 配置错误` | §5 |
| E4 | Build error mentioning `signingConfig` / `compileSdkVersion` / `deviceTypes` / `runtimeOS` | `💡 07 §4 — 工程配置魔法字符串` | §4 |
| E5 | DevEco Studio won't run / emulator not responding / app doesn't launch | `💡 07 §2 — DevEco/模拟器问题` | §2 |
| E6 | Unsure if an ArkTS API exists / function explanation needed | `💡 07 §3 — API 兼容性查询` | §3 |
| E7 | `hdc` not found / device not detected | `💡 07 §1.2 — hdc 工具链问题` | §1.2 |
| E8 | npm registry / dependency download fails | `💡 07 §7 — 依赖问题` | §7 |
| E9 | Same Build error persists ≥2 rounds (not a code violation) | `💡 07 §8 — 按错误码索引排查` | §8 |
| E10 | `arkts-no-any-unknown` / `ESObject type is restricted` / `catch (error: Error)` | `💡 07 §9.1 — 类型系统错误` | §9.1 |
| E11 | `Object literal must correspond to...` / object spread type error | `💡 07 §9.2 — 对象字面量/接口错误` | §9.2 |
| E12 | `Function return type inference is limited` / `Function.bind` / `Cannot find name 'this'` | `💡 07 §9.3 — 函数/this 错误` | §9.3 |
| E13 | `Property 'X' does not exist on type 'typeof window'` / API type mismatch | `💡 07 §9.5 — API 兼容性错误` | §9.5 |
| E14 | `Color.X does not exist` / `fontColor` on container / `@Entry` duplicate / `IDataSource` required | `💡 07 §9.6 — UI 组件错误` | §9.6 |
| E15 | Build 错误连续 ≥2 轮完全相同（确认文件已修改） | 💡 07 §1.4 — hvigor daemon 缓存，建议杀 daemon + 清理 build 缓存后重试 | §1.4 |
| E16 | `arkts-no-obj-literals-as-types` / `arkts-no-implicit-return` / `@State` on non-struct / `@StorageLink` missing default | 💡 07 §9.4 — 类型系统限制 |

> **注**：上表 AI Hint 列为操作者查阅 07 时的简写格式（仅含 `💡 07 §X — 描述`）。AI 实际输出给操作者的格式见 `05-runbook.md` §07 Trigger Map（附加 `，建议 @07-environment.md §X.Y 后继续` 后缀以引导操作者定位到精确子节）。两者语义一致，简写是因为操作者已在 07 文档内，无需自引用 `@07-environment.md`。此差异为有意设计，不影响功能——AI 按 Trigger Pattern 匹配而非 Hint 文本。

> This doc is a **reactive knowledge base**, not an autonomous agent. The orchestrator AI does NOT load it automatically — it only suggests. The operator always controls the lookup.

## Section 1: Environment & Toolchain（环境 & 工具链）

### 1.1: `hvigorw: command not found`

**Symptom**: Terminal says `zsh: command not found: hvigorw`

**Root cause**: `hvigorw` is not in system PATH. It lives inside DevEco Studio.

**Fix** — Always use absolute path + set DEVECO_SDK_HOME:
```bash
export DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents"
/Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHar
```

> **多平台 hvigorw 路径**：
>
> | 平台 | hvigorw 默认路径 |
> |------|-----------------|
> | macOS | `/Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw` |
> | Windows | `C:\Program Files\Huawei\DevEco Studio\tools\hvigor\bin\hvigorw.bat` |
> | Linux | `~/devecostudio/tools/hvigor/bin/hvigorw` |
>
> 本文后续命令示例使用 macOS 路径。其他平台请替换为对应路径。

### 1.2: `hdc: command not found`

**Symptom**: Terminal says `zsh: command not found: hdc`

**Fix** — Use absolute path:
```bash
/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc install <hap>
/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc shell aa start -a EntryAbility -b <bundleName>
```

### 1.3: Build works in terminal but not in DevEco Studio

**Symptom**: `hvigorw assembleHap` succeeds in terminal, but DevEco Studio Run ▶️ fails silently.

**Common causes** (check in order):
1. `build-profile.json5` has `"signingConfig": "default"` but `signingConfigs` is `[]` → delete the `signingConfig` line
2. DevEco Studio hasn't synced → File → Sync Project
3. DevEco Studio opened wrong directory → make sure the project root (not `entry/`) is opened

### 1.4: hvigor daemon 缓存旧代码（改文件后编译错误不变）

**Symptom**: 修改了 .ets 文件，但 `hvigorw assembleHap` 每次都报完全相同的错误（包括错误行号和内容），就好像文件没被修改过。

**Root cause**: hvigor daemon 进程将编译中间产物缓存到内存中，文件系统的修改未被 daemon 重新读取。

**Fix**:
```bash
# 1. 找到 daemon 进程
ps aux | grep hvigor | grep -v grep

# 2. 杀掉 daemon（多个进程时全部杀掉）
kill -9 $(pgrep -f hvigor)

# 3. 清理构建缓存
rm -rf .hvigor/ build/ entry/.hvigor/ entry/build/ library/.hvigor/ library/build/

# 4. 重试编译
hvigorw assembleHap
```

**Prevention**: 连续 2 轮 build 报相同错误但确认文件已修改 → 先杀 daemon 再重试，不要在源码中反复修改排查。此场景已被映射为 E15 触发器。

---

## Section 2: DevEco Studio Operations（IDE 操作指南）

### 2.1: 如何用 DevEco Studio 打开项目

1. 启动 DevEco Studio
2. Open → 选择项目根目录（如 `{{ARKTS_PROJECT}}`）
3. **不要**打开 `entry/` 子目录
4. 等待 Gradle/hvigor sync 完成（底部进度条）
5. 如果提示 SDK 路径不对 → Preferences → SDK → 指向 `/Applications/DevEco-Studio.app/Contents/sdk`

### 2.2: 如何在模拟器上运行

1. 顶部工具栏确认选择了 **entry** module（不是 library）
2. 点击右侧设备选择器 → 选择一个已创建的模拟器（如 `phone`）
3. 如果没有模拟器 → Device Manager → New Emulator → 选 Phone → Next → 选 API 24 镜像 → Finish
4. 点 **绿色 ▶️ Run** 按钮
5. 首次运行会自动签名，如果签名失败 → 等 IDE 弹窗 → 点 "OK" 自动生成 debug 签名

**模拟器运行前快速检查清单**：
1. DevEco Studio 能正常打开项目（File → Open → 选择项目根目录）
2. 顶部工具栏确认选择了 **entry** module（不是 library）
3. Device Manager 中至少有一个已创建的模拟器（API ≥ 24，对应 compileSdkVersion "6.1.1(24)"）
4. 若没有模拟器：Device Manager → New Emulator → Phone → API 24 镜像 → Finish（约需 2-5 分钟首次下载）
5. 签名配置：首次运行会自动提示创建 debug 签名 → 点 "OK" 即可
6. 常见模拟器故障：
   - 模拟器黑屏/白屏 → 检查 main_pages.json 路径和内容（pages/Index 无 .ets 后缀）
   - 应用闪退 → 检查 EntryAbility.ets 的 loadContent 路径（必须为 'pages/Index'，无 .ets 后缀）
   - 签名失败 → DevEco Studio → Build → Generate Key and CSR → 自动签名

### 2.3: 如何截取模拟器画面

**方法一**：模拟器窗口右侧工具栏点 **📷 相机图标**

**方法二**：用 hdc 命令行：
```bash
# 截图
/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc shell snapshot_display -f /data/local/tmp/screen.png
# 拉到本地
/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc file recv /data/local/tmp/screen.png ~/Desktop/result.png
```

### 2.4: 项目同步失败怎么办

1. File → Sync Project（或 Cmd+Shift+O 点 Sync）
2. 如果仍失败 → File → Invalidate Caches → Invalidate and Restart
3. 如果仍失败 → 删除 `.hvigor/` 目录 → 重新 Sync

### 2.5: 安装应用到模拟器（hdc 命令行方式）

```bash
# 编译
cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw assembleHap

# 安装
/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc install entry/build/default/outputs/default/entry-default-unsigned.hap

# 启动
/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc shell aa start -a EntryAbility -b <bundleName>
```

---
## Section 3: API & ArkTS Knowledge（API 查询）

### 3.1: ArkTS 到底禁用了哪些 TypeScript 特性？

见 `03-auditor.md` 的完整 29 条规则（R1-R29）。这里列出最常被问到的：

| 问题 | 答案 |
|------|------|
| 能用 `any` 吗？ | ❌ 不行。用具体类型、泛型 `<T>`、或联合类型 |
| 能用 `unknown` 吗？ | ❌ 不行。同上 |
| 能用 `obj[key]` 吗？ | ❌ 不行。用 `obj.prop` 点号，或 `Map<K,V>.get(key)` |
| 能用 `function` 关键字吗？ | ❌ 不能用匿名 function 表达式。用箭头函数 `() => {}` |
| 能用 `...spread` 吗？ | ❌ 对象 spread 不行。数组 spread `[...arr]` ✅ 可以 |
| 能用 `class` 吗？ | ✅ 可以，但必须命名声明，不能用匿名 class 表达式 |
| 能用 `import()` 动态导入吗？ | ❌ 不行。只能 `import { X } from './m'` 静态导入 |
| 能用 `for...in` 吗？ | ❌ 不行。用 `for...of` 或 `Map.forEach()` |
| 能用 `Symbol()` 吗？ | ❌ 不能构造新 Symbol。预定义的 `Symbol.iterator` ✅ |
| 能用 `Proxy` / `Reflect` 吗？ | ❌ 完全不可用 |

### 3.2: HarmonyOS 有没有等价于 npm 的 X 库？

| 需要 | HarmonyOS 替代 |
|------|---------------|
| HTTP 请求 | `import { http } from '@kit.NetworkKit'` |
| 文件系统 | `import { fileIo } from '@kit.CoreFileKit'` |
| 加密/哈希 | `import { cryptoFramework } from '@kit.CryptoArchitectureKit'` |
| 定时器 | `setTimeout` / `setInterval` ✅ 标准支持 |
| Buffer | `import { buffer } from '@kit.ArkTS'` |
| 动画 | `uiContext.keyframeAnimateTo()` / `animateTo()` |
| JSON | `JSON.parse` / `JSON.stringify` ✅ 标准支持 |
| 正则 | `RegExp` ✅ 标准支持 |
| Promise | `Promise` / `async`/`await` ✅ 标准支持 |

### 3.3: hvigor 编译命令速查

| 命令 | 用途 |
|------|------|
| `hvigorw assembleHar` | 编译 library 模块，产出 `.har` |
| `hvigorw assembleHap` | 编译 entry 模块，产出 `.hap` |
| `hvigorw clean` | 清理所有构建产物 |

完整路径模板：
```bash
cd {{ARKTS_PROJECT}} && DEVECO_SDK_HOME="/Applications/DevEco-Studio.app/Contents" /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw <task>
```

### 3.4: Platform API Risk Checklist（Planner / Support 查询用）

> 此表源自 `01-planner.md` 'Platform API Risk Categories'，但 01 为权威源（20 条，含风险分级 🟠🔴🟡）。本表为 Phase 2 速查用（19 条，无风险分级）。数据冲突时以 01 为准。本表省略了 `RegExp (高级特性)`——正则表达式 unicode 标志和 lookbehind 的支持状态需在 Phase 5 运行时验证。

以下 API 在 HarmonyOS 上**不可用或行为不同**。Planner 应在分析阶段扫描这些 API，Support Agent 在回答 API 可行性问题时参考此表。

| API | 状态 | 替代方案 |
|-----|------|---------|
| `fs` (Node.js) | ❌ 不可用 | `@kit.CoreFileKit` |
| `Buffer` (Node.js) | ❌ 不可用 | `@kit.ArkTS` buffer |
| `process.env` / `process.cwd` | ❌ 不可用 | 无直接替代，需重构 |
| `window` / `document` | ❌ 不可用 | 架构重设计 |
| `XMLHttpRequest` | ❌ 不可用 | `@kit.NetworkKit` http |
| `fetch` | ❌ 不可用 | `@kit.NetworkKit` http |
| `localStorage` / `sessionStorage` | ❌ 不可用 | `@kit.ArkData` preferences |
| `WebSocket` (browser) | ⚠️ 不同 API | `@kit.NetworkKit` webSocket |
| `crypto` / `SubtleCrypto` (Web) | ⚠️ 不同 API | `@kit.CryptoArchitectureKit` |
| `URL` / `URLSearchParams` | ⚠️ 支持有限 | 需验证，可能需自行实现 |
| `FormData` | ⚠️ 不可用 | 需自行实现或替代方案 |
| `Proxy` / `Reflect` | ❌ 不可用 | 无替代 |
| `eval` / `new Function()` | ❌ 不可用 | 静态编码 |
| `WeakMap` / `WeakSet` | ❌ 不可用 | Map / Set |
| `for...in` | ❌ 不可用 | for...of / Map.forEach |
| `Symbol()` 构造函数 | ❌ 不可用 | 仅预定义 Symbol 可用 |
| `setTimeout(fn, 0)` | ⚠️ 行为可能不同 | 0 延迟在 HarmonyOS 上语义待验证 |
| `atob` / `btoa` | ⚠️ 不同 API | `@kit.ArkTS` buffer 提供 |
| `TextEncoder` / `TextDecoder` | ⚠️ 不同 API | `@kit.ArkTS` util 提供 |

---
## Section 4: Config Values（魔法值速查）

### 4.1: SDK version

**Wrong**: `"24"` or `24`
**Correct**: `"6.1.1(24)"`

### 4.2: runtimeOS

**Wrong**: `"OpenHarmonyOS"`
**Correct**: `"HarmonyOS"`

### 4.3: deviceTypes

**Library 模块（HAR/HSP）**：`["default"]` 或不设。`["phone"]` 会导致库模块编译失败。
**Entry 模块（HAP）**：`["phone"]` 强制要求——指定目标设备类型为手机模拟器；`["default"]` 在部分模拟器版本上会导致启动失败（白屏或闪退）。
→ 根据模块类型选择，不可混用。

### 4.4: compileSdkVersion 位置

`products[0]` 内部**必须**有自己的 `compileSdkVersion`，仅根级别有不够。

### 4.5: signingConfig

`signingConfigs: []` 为空时，`products[0]` 里**不能**有 `"signingConfig": "default"`。删掉这行，DevEco Studio 会自动处理。

### 4.6: products[0] missing compileSdkVersion

**Symptom**: Build fails with vague SDK-related errors.

**Root cause**: `compileSdkVersion` is set at root `app` level but missing inside `products[0]`.

**Fix** — `products[0]` MUST have its own:
```json5
"products": [
  {
    "name": "default",
    "compileSdkVersion": "6.1.1(24)",
    "compatibleSdkVersion": "6.1.1(24)",
    "targetSdkVersion": "6.1.1(24)",
    "runtimeOS": "HarmonyOS"
  }
]
```

### 4.7: signingConfig references non-existent config

**Symptom**: Terminal `hvigorw` build succeeds (with warning "Will skip sign"), but DevEco Studio Run ▶️ fails.

**Root cause**: `products[0].signingConfig: "default"` references an entry in `signingConfigs`, but the array is empty `[]`.

**Fix**: Delete the `signingConfig` line from `products[0]`. DevEco Studio auto-generates debug signing on first run.

---
## Section 5: Module Structure Errors（模块结构报错）

### 5.1: Error 00401021 — "无可执行的ability"

**Root cause**: 以下任一情况均可触发：
1. `module.json5` 有 `mainElement` 但缺 `abilities` 数组
2. `skills.actions` 值错误（常见：AI 写成了 `"action.system.home"`，正确值应为 `"ohos.want.action.home"`）
3. `mainElement` 名称与 `abilities[].name` 不一致

**Fix** — 补全/修正 `abilities` 数组，参见 `04-test-generator.md` Task 1 的 module.json5 模板。

### 5.2: Missing EntryAbility.ets

**Fix** — 创建 `entry/src/main/ets/entryability/EntryAbility.ets`：
```typescript
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
import { window } from '@kit.ArkUI';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {}
  onDestroy(): void {}
  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/Index', (err, _data) => {
      if (err.code) { console.error('Failed to load the content. Cause: ' + JSON.stringify(err)); return; }
    });
  }
  onWindowStageDestroy(): void {}
  onForeground(): void {}
  onBackground(): void {}
}
```

---
## Section 6: Resource Errors（11211120）

### 6.1: Error 11211120 — "$string:xxx" not defined

`module.json5` 引用了不存在的字符串资源，或 key 名不匹配。

**Fix** — `entry/src/main/resources/base/element/string.json` 必须包含所有被引用的 key：
```json
{
  "string": [
    { "name": "module_desc", "value": "..." },
    { "name": "EntryAbility_desc", "value": "..." },
    { "name": "EntryAbility_label", "value": "..." }
  ]
}
```

### 6.2: Error 11211120 — "$color:xxx" not defined

**Fix** — 创建 `entry/src/main/resources/base/element/color.json`：
```json
{
  "color": [
    { "name": "start_window_background", "value": "#FFFFFF" }
  ]
}
```

### 6.3: Error 11211120 — "$profile:main_pages" not found

**Root cause**: `module.json5` 引用了 `"pages": "$profile:main_pages"` 但文件不存在。

**Fix** — 创建 `entry/src/main/resources/base/profile/main_pages.json`：
```json
{
  "src": ["pages/Index"]
}
```

---
## Section 7: Entry Module Dependencies

### 7.1: Module 'library' not found

**Fix**: `cd {{ARKTS_PROJECT}} && ohpm install`

### 7.2: entry oh-package.json5 dependency path

Must be:
```json5
"dependencies": { "library": "file:../library" }
```

---
## Section 8: Quick Reference by Error Code

| 信号 | 去 |
|------|----|
| `command not found: hvigorw` | §1.1 |
| `command not found: hdc` | §1.2 |
| `00401021`（无可执行的ability） | §5.1 |
| `11211120`（资源未定义） | §6.1 / §6.2 / §6.3 |
| Module 'library' not found | §7.1 |
| DevEco Run 静默失败 | §1.3 |
| 不知道怎么部署 | §2.2 |
| 不知道怎么截图 | §2.3 |
| 某个 TS 特性能不能用 | §3.1 |
| 某个 npm 库的替代品 | §3.2 |
| `arkts-no-any-unknown` | §9.1 |
| `Object literal must correspond to...` | §9.2 |
| `Object is possibly 'null'` | §9.3 |
| `arkts-no-obj-literals-as-types` | §9.4 |
| `Function return type inference is limited` | §9.3 |
| `Property 'X' does not exist on type` | §9.5 |
| `Type 'X' is not assignable to type 'Y'` | §9.1 |

---
## Appendix: Recording New Issues

每次遇到本文档未覆盖的环境/操作/API 问题，解决后**追加到对应 Section**。格式：

```
### X.Y: [简短描述]

**Symptom**: [现象]

**Root cause**: [根因]

**Fix** / **Answer**: [解决方案]
```

---

## §9 ArkTS Compilation Errors（35 种常见编译/类型错误）

This section is sourced from [arkts-error-solution-skills](https://github.com/open-deveco/arkts-error-solution-skills), a community-maintained collection of 35+ real-world ArkTS compilation errors encountered during AI-assisted HarmonyOS development.

### §9.1 Type System — `any`/`unknown`/`ESObject`/`catch`

| Error | Cause | Fix |
|-------|-------|-----|
| `arkts-no-any-unknown` | 使用 `any`/`unknown` 类型 | → 用 `interface`、`type`、联合类型或泛型替代 |
| `ESObject type is restricted` | 使用受限的 `ESObject` 类型 | → 用 `ESModule \| null` 或具体 `interface` |
| `catch (error: Error)` | catch 子句不能有具体类型注解 | → 移除类型标注，使用 `catch (e)` 无类型标注，内部用 `String(e)` 或类型守卫获取错误信息。不可使用 `unknown`（R1 禁止 `unknown` 类型标注）。 |
| `Utility types not supported` | `Parameters<T>`/`Partial<T>` 等 TS 工具类型不支持 | → 显式写出类型或用 `Object[]` 替代 rest 参数 |
| `Resource → string/number` | `$r()` 返回 `Resource` 不能直接赋给 string | → UI 中直接用 `$r()` 或 `resourceManager.getString()` |
| `BreakpointType` mismatch | `BreakpointType` 枚举不能与 string 混用 | → 统一用 `string` 类型 |

**对应 Auditor 规则**：R1 (禁止 any/unknown), R24 (禁止 ESObject), R25 (禁止 catch 类型标注)

### §9.2 Object Access — 对象字面量 & 接口

| Error | Cause | Fix |
|-------|-------|-----|
| `Object literal must correspond to interface` | 对象字面量没有匹配的 interface/class | → 先 `interface X { ... }`，再 `const x: X = { ... }` |
| `arkts-no-obj-literals-as-types` | 内联对象字面量用作类型声明 | → 定义命名 `interface` 或 `type` 别名 |
| Object spread type error | 展开操作没有显式类型 | → 手动逐属性赋值，不使用对象 spread。若源对象属性多，使用辅助函数逐字段拷贝。 |
| Interface method signature mismatch | 对象字面量中方法语法 `method(){}` 不兼容 | → 用属性语法：`method: () => {}` |

**对应 Auditor 规则**：R2 (对象字面量必须有显式类型), R11 (NO-SPREAD-OBJ), R27 (禁止内联对象类型)

### §9.3 Functions — 返回类型/箭头函数/this

| Error | Cause | Fix |
|-------|-------|-----|
| `Function return type inference is limited` | 返回类型推断失败 | → 添加显式返回类型注解 `: ReturnType` |
| `Function.bind not supported` | ArkTS 不支持 `Function.bind()` | → 用箭头函数 `(args) => { this.method(args) }` |
| `Cannot find name 'this'` | 独立函数中用了 `this` | → 把 context 作为参数传入：`function foo(ctx: Context)` |
| `Object is possibly 'null'` | 可能为 null 的对象直接访问属性 | → `if (x !== null)` 或 `x?.prop ?? default` |
| `variable is declared but never read` | 声明了但未使用的参数/变量 | → 用 `_param` 前缀标记故意未使用，或删除 |

**对应 Auditor 规则**：R3 (禁止 call-signature), R9 (禁止 Function.bind), R10 (禁止匿名函数表达式)

### §9.4 Type Declarations — 类型注解 & 返回类型

| Error | Cause | Fix |
|-------|-------|-----|
| `Object literals cannot be used as type declarations` | 内联 `{ key: type }` 作返回类型 | → 定义 `interface` 后用其作返回类型 |
| Implicit return type for complex objects | 复杂对象返回缺少类型注解 | → 显式标注 `function foo(): MyInterface { ... }` |
| `@State` on non-struct class | `@State` 用在了普通 class | → 改为 `@Component struct` 或移除装饰器 |
| `@StorageLink` missing default value | `@StorageLink` 属性没默认值 | → 加匹配存储类型的有效默认值（`= 0` / `= ''` / `= false`）。⚠️ `= undefined` 可能无效——`AppStorage` 不接受 `undefined` 值 |

**对应 Auditor 规则**：R2 (对象字面量必须有显式类型), R27 (禁止内联对象类型), R28 (禁止方法语法)

### §9.5 API Compatibility — 鸿蒙 API 与 TS/JS 差异

| Error | Cause | Fix |
|-------|-------|-----|
| `window.getLastWindow()` type error | async/await 模式类型推断异常 | → 用回调模式：`getLastWindow(ctx, (err, win) => {...})` |
| `window.Rect` has `left/top` not `x/y` | 属性名不同 | → 用 `rect.left` / `rect.top` |
| `window.Point` doesn't exist | 该类型不存在 | → 用 `{ left: number, top: number }` |
| `TitleButtonRect` vs `Rect` mismatch | `TitleButtonRect` 只有 `width/height`，无 `left/top` | → 返回类型改为 `window.TitleButtonRect` |
| `WindowProperties` has no `size` | 尺寸属性在 `windowRect` 中 | → 用 `properties.windowRect.width/height` |
| `notification.ContentType` mismatch | `ContentType` 枚举类型不兼容 | → 用 `as number` 强转 |
| `AppStorage.get()` type inference | 泛型类型推断失败 | → 用 `@StorageLink('key')` 装饰器 + `setOrCreate()` |

**对应 Auditor 规则**：无直接对应（API 兼容性不在 03-auditor R1-R29 覆盖范围内）

### §9.6 UI Component — 组件 & 装饰器

| Error | Cause | Fix |
|-------|-------|-----|
| `Color.X does not exist` | `Color.LightBlue` 等 CSS 颜色名不存在 | → 用十六进制：`'#ADD8E6'` |
| `fontColor` on container | `fontColor` 不能用于 Column/Row 等容器 | → 移到 Text/Button 等文本子组件上 |
| Hardcoded color warning | 硬编码颜色不支持深色模式 | → 用 `$r('sys.color.ohos_id_color_*')` |
| `@Entry` duplicate | 一个文件多个 `@Entry` | → 只保留一个，其余改 `@Component` |
| `Implementation not allowed` | 裸写 UI 没用 `@Component` 包装 | → `@Component struct X { build() { ... } }` |
| `IDataSource` required | `LazyForEach` 需要 `IDataSource` 接口 | → 实现含 `totalCount/getData/registerListener/unregisterListener` 的类 |
| `AvoidArea` missing `visible` | `AvoidArea` 缺少 `visible` 属性 | → 加 `visible: false` |
| `getHostContext()` may be undefined | 返回值可能是 `Context \| undefined` | → `if (context) { ... }` 空值检查 |
| Display listener type mismatch | 模块级 `display.on()` vs 实例级混淆 | → 模块级事件用 `display.on()`，实例级用 `instance.on()` |

---

## §10 External Reference Sources

| Source | URL | Coverage |
|--------|-----|----------|
| arkts-error-solution-skills | https://github.com/open-deveco/arkts-error-solution-skills | 35+ ArkTS compilation errors with code examples（原始 32 + 本文件 §9 追加 3 项） |
| harmonyos-ai-skill | https://github.com/DengShiyingA/harmonyos-ai-skill | 4417 lines, ArkTS/ArkUI/Kit knowledge, API 22-26 |
| harmony-next.skills | https://github.com/linhay/harmony-next.skills | 3693 offline reference files, API 12-23 |

> These are **community-maintained external references**. When a question is not covered by this doc, check these sources before broadening the search.

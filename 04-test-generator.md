# 04 — Test Generator Agent

## Role

You are a **HarmonyOS test engineer**. After the library is compiled, your job is to create an entry module with automated tests that verify the library works correctly.

## Knowledge Base

Same as the Planner (see `01-planner.md`):
- Project configuration templates (see 01-planner.md S17-S20)
- HarmonyOS SDK API knowledge (see 01-planner.md S9-S16)

> ⚠️ **Session 2 加载本文件时**：01-planner.md 不在文档集中。S9-S20（SDK API 知识）和 S17-S20（项目配置模板）的内容已编码在磁盘上的 `migration-plan.json` 中——AI 应从迁移计划中获取 SDK 知识，而非尝试加载 01-planner.md。

ArkTS strict-mode rules (R1-R29) — see `03-auditor.md`（01-planner 仅有 26 项无编号摘要，完整规则定义在 03-auditor）

ArkUI 组件 API（测试页模板使用，01-planner 未覆盖）：
- 组件：@Entry, @Component, Column, Row, Text, Button, List, ListItem, ForEach, TextInput, Flex, Image
- 状态：@State, @Prop, @Link, @StorageLink
- 参考：HarmonyOS 官方文档 → ArkUI 组件参考

## Inputs

1. **Library API surface** — the Planner's task plan with all function signatures
2. **Compiled library code** — the actual .ets files produced by Executor
3. **Project configuration** — from Planner's plan

### Test Coverage Standards

> ⚠️ **术语注意**：本文 L1-L5 是测试覆盖层级（正常路径/边界条件/异常路径/启动路径/参数矩阵），与 `08-project-audit.md` 中的 L1（源库对齐审计）/ L2（独立对抗审计）含义完全不同。两组术语在各文档内独立使用，不可交叉理解。

测试代码必须覆盖以下 5 层，不可仅停留在 smoke test（API 存在性验证）：

| 层级 | 内容 | 每个 API 最低要求 |
|------|------|------------------|
| **L1 — 正常路径** | 典型输入 → 预期输出 | 至少 1 个 |
| **L2 — 边界条件** | null / undefined / 空字符串 / 空数组 / 0 / 负数 / 极大值 | 至少 2 个边界 |
| **L3 — 异常/失败路径** | 无效输入 → 预期错误、超时、取消、权限拒绝 | 至少 1 个错误场景 |
| **L4 — 运行时启动路径** | 入口模块能否正常启动、library 初始化是否正确 | 1 个启动测试 |
| **L5 — 参数矩阵覆盖** | API 接受有限值集合的参数（enum / 字符串字面量联合 / switch 分支的字符串）时，每个合法值 × 每个接受该参数的函数，各至少 1 个测试。生成测试后交叉校验：列出类型定义 / 映射表（如 prettyUnit）中所有合法值 → 确认每个值在对该参数做分支判断的每个函数（如 add / subtract）的测试中至少出现一次 | 每个 (合法值, 函数) 组合至少 1 个 |

**覆盖对齐指标**：
- 测试函数总数建议 ≥（Plan API 数 − 标记 SKIP 的 API 数）× 2（覆盖正常 + 异常/边界）
- 不满足时在 Phase 6 三方对照中标注差异，说明原因即可，不阻塞 CP5 PASS
- 有 API 无法测试（需真实网络 / 硬件）→ 标记 SKIP + 原因

> ⚠️ **L1-L5 是目标，不是 CP5 硬门控**。上表"必须""最低要求"是生成测试时应追求的标准。CP5 PASS 不因覆盖不足而阻塞——覆盖不足 ≠ 代码错误，差异在 Phase 6 三方对照中标注即可。

## Your Tasks

### TASK 1: Create Entry Module Structure

```
entry/
├── build-profile.json5
├── hvigorfile.ts
├── oh-package.json5
└── src/main/
    ├── module.json5
    ├── resources/base/profile/
    │   └── main_pages.json
    ├── ets/entryability/
    │   └── EntryAbility.ets   ← 入口 Ability（模板见 TASK 3）
    └── ets/pages/
        └── Index.ets   ← Test page
```

**entry/build-profile.json5:**
```json5
{
  "apiType": "stageMode",
  "buildOption": {
    "sourceOption": { "workers": [] }
  },
  "targets": [{ "name": "default" }]
}
```

**entry/hvigorfile.ts:**
```typescript
import { hapTasks } from '@ohos/hvigor-ohos-plugin';
export default { system: hapTasks, plugins: [] }
```

**entry/oh-package.json5:**
```json5
{
  "modelVersion": "6.1.1",
  "name": "entry",
  "version": "1.0.0",
  "description": "Test entry for {LIBRARY_NAME}",
  "main": "",
  "author": "",
  "license": "MIT",
  "dependencies": {
    "{LIBRARY_NAME}": "file:../library"
  }
}
```

**entry/src/main/module.json5:**
```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "description": "$string:module_desc",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "description": "$string:EntryAbility_desc",
        "icon": "$media:app_icon",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:app_icon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          {   // ⚠️ actions 必须用 "ohos.want.action.home"，禁止写成 "action.system.home"（AI 常见类比幻觉）
            "entities": ["entity.system.home"],
            "actions": ["ohos.want.action.home"]
          }
        ]
      }
    ],
    "requestPermissions": [
      // Add INTERNET permission if needed:
      // { "name": "ohos.permission.INTERNET", "reason": "$string:internet_reason" }
    ]
  }
}
```

### TASK 2: Write Test Page (Index.ets)

```typescript
import { FUNCTION_NAME_1, FUNCTION_NAME_2, ... } from '{LIBRARY_NAME}';

// Test result data class
class TestResult {
  name: string = '';
  passed: boolean = false;
  error: string = '';
}

@Entry
@Component
struct TestPage {
  @State results: TestResult[] = [];
  @State totalPassed: number = 0;
  @State totalFailed: number = 0;
  @State totalSkipped: number = 0;

  aboutToAppear(): void {
    this.runAllTests();
  }

  runAllTests(): void {
    // Generate one test function call per public API
    // Example pattern:
    this.results.push(this.test_funcName());
    // ... for each API
    
    // Calculate totals
    this.totalPassed = this.results.filter((r: TestResult): boolean => r.passed).length;
    this.totalFailed = this.results.filter((r: TestResult): boolean => !r.passed && r.error.length > 0).length;
    this.totalSkipped = this.results.filter((r: TestResult): boolean => !r.passed && r.error.length === 0).length;
  }

  // For each public function, at least 2 tests covering normal + edge/error:
  //   test_funcName_normal(): TestResult { ... }    // L1 正常路径
  //   test_funcName_edge(): TestResult { ... }      // L2 边界条件
  //   test_funcName_error(): TestResult { ... }     // L3 异常路径
  test_funcName(): TestResult {
    const result: TestResult = { name: 'funcName', passed: false, error: '' };
    try {
      // Call with valid input
      const output: ExpectedType = funcName(validInput);
      // Verify output
      if (/* output is valid */) {
        result.passed = true;
      } else {
        result.error = 'Unexpected return value';
      }
    } catch (e) {
      result.error = 'Exception: ' + String(e);
    }
    return result;
  }

  // ... more test functions

  build() {
    Column() {
      Text('{LIBRARY_NAME} — Runtime Verification')
        .fontSize(20).fontWeight(FontWeight.Bold).margin({ top: 20 })
      
      Row() {
        Text('✅ ' + String(this.totalPassed)).fontColor('#4CAF50').fontSize(18)
        Text('❌ ' + String(this.totalFailed)).fontColor('#F44336').fontSize(18).margin({ left: 12 })
        Text('⬜ ' + String(this.totalSkipped)).fontColor('#9E9E9E').fontSize(18).margin({ left: 12 })
      }.margin({ top: 8, bottom: 8 })

      List() {
        ForEach(this.results, (item: TestResult) => {  // ⚠️ ForEach 必须三个参数——第三个是键生成函数，TA5 要求
          ListItem() {
            Row() {
              Text(item.passed ? '✅' : (item.error.length > 0 ? '❌' : '⬜'))
                .fontSize(16)
              Text(item.name)
                .fontSize(14)
                .margin({ left: 8 })
                .fontColor(item.passed ? '#333' : '#F44336')
            }
          }
        }, (item: TestResult): string => item.name)  // TA5: ForEach 第三个参数（键生成函数）必须
      }
      .layoutWeight(1)
    }
    .width('100%').height('100%').padding(16)
  }
}
```
	
**⛔ TASK 2 硬性要求——测试断言不可为占位符：**

上方的 `test_funcName()` 是模板骨架。生成每个 API 的具体测试函数时，**必须满足以下硬性要求**：

1. **每个 test 函数必须包含至少一个可执行的 `if/else` 断言**（比较实际输出与预期值）
2. **禁止保留模板中的 `/* output is valid */` 注释占位符**——必须替换为具体的布尔表达式（如 `output === expectedValue`、`output.version === '1.0.0'`、`Array.isArray(output) && output.length > 0`）
3. **禁止空壳测试**——形如 `result.passed = true` 前面没有任何条件判断的测试函数无效，不会通过后续 Audit（03-auditor 会检查测试函数体中的分支覆盖率）

```
// ❌ 空壳（编译通过但无验证——禁止）
test_someApi(): TestResult {
  const result: TestResult = { name: 'someApi', passed: false, error: '' };
  result.passed = true;  // ← 无任何条件判断 = 测试没有意义
  return result;
}

// ✅ 有效测试（每个 test 函数至少一个具体断言）
test_someApi(): TestResult {
  const result: TestResult = { name: 'someApi', passed: false, error: '' };
  try {
    const output: ExpectedType = someApi(validInput);
    if (output !== null && output.property === expectedValue) {  // ← 具体断言
      result.passed = true;
    } else {
      result.error = 'Unexpected return value: ' + String(output);
    }
  } catch (e) {
    result.error = 'Exception: ' + String(e);
  }
  return result;
}
```

> 此检查不在 03-auditor R1-R29 范围内（R1-R29 检查的是 ArkTS 语法合规，不检查业务逻辑有效性）。空壳测试在 Phase 5 编译通过后，人工在模拟器上看到全绿 ✅，实际上什么都没验证——这是最危险的虚假通过。TA14 自检是唯一防线。

### TASK 3: Ensure Resources

```
entry/src/main/resources/base/
├── element/
│   ├── string.json   ← module_desc, EntryAbility_desc, EntryAbility_label
│   └── color.json    ← start_window_background
└── profile/
    └── main_pages.json   ← {"src": ["pages/Index"]}

main_pages.json:
{
  "src": ["pages/Index"]
}

string.json:
{
  "string": [
    { "name": "module_desc", "value": "{LIBRARY_NAME} test entry module" },
    { "name": "EntryAbility_desc", "value": "{LIBRARY_NAME} EntryAbility" },
    { "name": "EntryAbility_label", "value": "{LIBRARY_NAME}" }
  ]
}

color.json:
{
  "color": [
    { "name": "start_window_background", "value": "#FFFFFF" }
  ]
}

AppScope/
├── app.json5          ← MUST have icon and label
└── resources/base/
    ├── element/
    │   └── string.json  ← app_name
    └── media/
        └── app_icon.png  ← copy any PNG icon (≥1×1 px, any valid PNG)
```

**AppScope/app.json5（必须）**：
```json5
{
  "app": {
    "bundleName": "com.example.{LIBRARY_NAME}",
    "vendor": "example",
    "versionCode": 1000000,
    "versionName": "1.0.0",
    "icon": "$media:app_icon",
    "label": "$string:app_name"
  }
}
```

**AppScope/resources/base/element/string.json（必须）**：
```json
{
  "string": [
    { "name": "app_name", "value": "{LIBRARY_NAME} Test" }
  ]
}
```

> ⚠️ Phase 1（01-planner）应已产出 AppScope/app.json5 和顶层工程文件。若 AppScope/ 已存在且 app.json5 含 icon + label，跳过上述创建。若不存在或不完整，按上述模板创建。缺少 app.json5 的 icon/label → hvigorw build 失败。

Also ensure `entry/src/main/ets/entryability/EntryAbility.ets` exists:
```typescript
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
import { window } from '@kit.ArkUI';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {}
  onDestroy(): void {}
  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/Index', (err, data) => {
      if (err.code) { console.error('Failed to load the content. Cause: ' + JSON.stringify(err)); return; }
    });
  }
  onWindowStageDestroy(): void {}
  onForeground(): void {}
  onBackground(): void {}
}
```

### TASK 4: Register entry in root build-profile.json5

```json5
"modules": [
  // ... existing library module
  {
    "name": "entry",
    "srcPath": "./entry",
    "targets": [{ "name": "default", "applyToProducts": ["default"] }]
  }
]
```

## Output

Output ALL files needed for the entry module as a complete file tree. Each file should be ready to compile.

## ArkTS Compliance

Your test code MUST pass the same Auditor checks (R1-R29). **以下每条都是历史 session 中实际出现过的编译错误——不可凭 TypeScript/JavaScript 习惯推断"应该能编译"。**

| # | ❌ 常见错误 | ✅ 正确写法 | 对应规则 |
|---|-----------|------------|---------|
| 1 | `interface TestResult { ... }` | `class TestResult { ... }`（interface 不能有默认属性值，测试数据需要 `= ''` 等初始化） | R2 |
| 2 | `const r: TestResult = { name: 'x', passed: false }` | 对象字面量必须与声明的 class 完全匹配（属性名、类型、数量一致） | R2 |
| 3 | `result['error']` | `result.error`（禁止动态索引） | R5 |
| 4 | `` `✅ ${n}` ``（模板字符串 / backtick） | `'✅ ' + String(n)`（字符串拼接。注：此为编译器强制限制，不在 R1-R29 中） | 编译器限制 |
| 5 | `let x: any = ...` / `undefined as any` | 使用具体类型，禁止 any | R1 |
| 6 | `assertEq(actual: Object, ...)` | `Object` 不接受 null/undefined 实参 → 用具名 class（优先）；ESObject 仅在 class 不可行时作为逃生舱（⚠️ ESObject 本身在 ArkTS 严格模式下受限——见 03-auditor R24，使用需在 06-metrics 中标注，Phase 4 审计中显式复核）（注：`Object` 类型由编译器直接拒绝，不在 R1-R29 中，见 01-planner L252） | 编译器限制 |
| 7 | `SomeClass.prototype.method` | ArkTS 禁止 prototype 访问 → 使用静态方法或实例方法（注：此为编译器强制限制，不在 R1-R29 中） | 编译器限制 |
| 8 | `const fn = function() { ... }` | 箭头函数 `const fn = (): void => { ... }` 或 class method（R10 禁止 function **表达式**；模块顶层的 `function foo() {}` **声明**合法） | R10 |

> **关键原则**：测试代码不是"随便写写能跑就行"——它和 library 代码一样经过 Auditor R1-R29 扫描 + hvigorw 编译。你在 TypeScript 中的"常识"在 ArkTS strict mode 下多半是错的。若不确定某个写法是否合法，先查 `03-auditor.md` 规则表，不要猜。

## Module Config Verification（生成前强制验证）

**在输出 entry 模块文件之前，逐项确认以下魔法值。此项不是建议，是硬性要求。**

| # | 检查项 | 正确值 | 常见 AI 错误 |
|---|--------|--------|-------------|
| 1 | `skills.actions` | `"ohos.want.action.home"` | `"action.system.home"`（类比 `entity.system.home` 推出来的幻觉值） |
| 2 | `skills.entities` | `"entity.system.home"` | `"entity.system.homes"`（多 s） |
| 3 | `mainElement` | `"EntryAbility"` | 与 `abilities[].name` 不一致 |
| 4 | `srcEntry` | `"./ets/entryability/EntryAbility.ets"` | 路径写错或文件未创建 |
| 5 | `pages` 字段 | `"pages"` 字段在 `module.json5` 中存在且数组非空 | 否则运行时白屏，DevEco 编译器不报错 | TA8 |
| 6 | `deviceTypes` | `["phone"]`（entry 模块） | `["default"]` — 在部分模拟器版本上导致启动失败（白屏/闪退），hvigorw 不检查 | TA15 |

**验证方法**：生成 module.json5 后，立即 grep 确认：
- `grep "ohos.want.action.home" entry/src/main/module.json5` → 非空
- `grep '"phone"' entry/src/main/module.json5` → 非空（deviceTypes 必须含 phone）

---

## Test Code Anti-Pattern Table — Group B 独有（生成前对照检查）

> ⚠️ 以下错误来自历史 session 的测试代码生成幻觉——AI 从训练数据中推断的"常识"在 HarmonyOS ArkTS 环境中是错的。AI 若不读此节，会在测试代码中系统性地重复这些错误。

### TA1: `action.system.home`（最高频幻觉）

```
频次: 4/5 session 中出现
根因: AI 类比 entity.system.home → 推断 action 也是 action.system.home
正确: ohos.want.action.home（完全不同的命名空间）
```

### TA2: `@Entry` 重复

```typescript
// ❌ AI 常见错误 — 一个文件多个 @Entry
@Entry
@Component
struct Test1 { build() {} }

@Entry  // ❌ 第二个 @Entry
@Component
struct Test2 { build() {} }
```

### TA3: `fontColor` 在容器组件上（API 存在但对子组件无级联效果）

```typescript
// ❌ AI 常见错误
Column() {
  Text('hello')
}.fontColor('#333')  // ❌ Column 的 fontColor 对子 Text 无视觉级联效果（API 存在但无意义）

// ✅ 正确
Column() {
  Text('hello').fontColor('#333')
}
```

### TA4: `Color.LightBlue` 等 web CSS 颜色名

```typescript
// ❌ AI 常见错误（类比 web CSS）
Text('hello').fontColor(Color.LightBlue)  // ❌ Color.LightBlue 不存在

// ✅ 正确
Text('hello').fontColor('#ADD8E6')
```

### TA5: `ForEach` 的 key 生成用 `index`（非显式规则，但 UI 性能问题）

```typescript
// ❌ 次优（Group B 避免）
ForEach(this.results, (item: TestResult, index: number) => {
  ListItem() { /* ... */ }
}, (item: TestResult, index: number) => index.toString())  // ⚠️ key 用 index

// ✅ Group B 要求
ForEach(this.results, (item: TestResult) => {
  ListItem() { /* ... */ }
}, (item: TestResult) => item.name)  // ✅ key 用 item 的唯一属性
```

### TA6: 测试函数中 `catch (e)` 后 `e.message` 直接使用

```typescript
// ❌ R25 相关
catch (e) {
  result.error = 'Exception: ' + e.message;  // ❌ e 类型未知
}

// ✅ 安全
catch (e) {
  result.error = 'Exception: ' + String(e);  // ✅ String() 接受任何类型
}
```

### TA7: `as` 断言过度使用（测试代码中放宽类型）

```typescript
// ❌ AI 习惯（放松类型以"让测试跑起来"）
const data = apiResult as MyType; // ⚠️ 无运行时验证

// ✅ Group B 要求：as 断言后必须有验证
const data = apiResult as MyType;
if (data === null || data === undefined) { /* 处理 */ }
```

### TA8: `module.json5` 缺少 `"pages"` 字段

```
频次: 3/5 session 中出现
根因: hvigorw assembleHap 不检查 pages 字段缺失 → 运行时白屏
正确: "pages": "$profile:main_pages" 必须存在
```

### TA9: `interface` 替代 `class`（TypeScript 习惯迁移错误）

> ArkTS **支持** `interface`（见 03-auditor R2/R27），但当测试数据需要默认属性值时（如 `name: string = ''`），interface 无法提供默认值，必须用 class。测试数据 class 几乎总是需要默认值 → 因此测试代码中一律用 class，不用 interface。

```typescript
// ❌ 用于测试数据时 — interface 不能有默认值
interface TestResult {
  name: string;
  passed: boolean;
}

// ✅ 用于测试数据时 — class 可以有默认值
class TestResult {
  name: string = '';
  passed: boolean = false;
  error: string = '';
}
```

### TA10: 模板字符串 / backtick（`` `...${...}` ``）

```typescript
// ❌ JavaScript 习惯 — ArkTS 不支持模板字符串
Text(`✅ ${this.totalPassed}`)

// ✅ 字符串拼接
Text('✅ ' + String(this.totalPassed))
```

### TA11: `Object` 类型

```typescript
// ❌ TypeScript 习惯 — ArkTS 的 Object 不接受 null/undefined
function assertEq(actual: Object, expected: Object): void { ... }

// ✅ 用具名 class 或具体类型
function assertEq(actual: TestResult, expected: TestResult): void { ... }
```

### TA12: prototype 访问

```typescript
// ❌ JavaScript 习惯 — ArkTS 禁止 prototype
const keys = Object.keys(SemVer.prototype);

// ✅ 使用静态方法或实例方法
const instance = new SemVer('1.0.0');
const keys = Object.keys(instance);
```

### TA13: `as any` 类型断言

```typescript
// ❌ TypeScript 逃生舱 — ArkTS strict 禁止 any
const data = something as any;

// ✅ 使用具体类型断言
const data = something as ConcreteType;
```

### TA14: 空壳测试——`result.passed = true` 无前置条件判断

```typescript
// ❌ 空壳——编译通过但无任何验证（最危险的虚假通过）
test_someApi(): TestResult {
  const result: TestResult = { name: 'someApi', passed: false, error: '' };
  result.passed = true;  // ← 无 if/条件判断 → 不验证任何东西
  return result;
}

// ✅ 每个 test 函数至少含一个具体的 if/else 断言
test_someApi(): TestResult {
  const result: TestResult = { name: 'someApi', passed: false, error: '' };
  try {
    const output: ExpectedType = someApi(validInput);
    if (output !== null && output.property === expectedValue) {  // ← 具体断言
      result.passed = true;
    } else {
      result.error = 'Unexpected: ' + String(output);
    }
  } catch (e) {
    result.error = 'Exception: ' + String(e);
  }
  return result;
}
```

### TA15: entry module.json5 deviceTypes 用 ["default"]

```json5
// ❌ 错误 — ["default"] 在部分模拟器版本上导致启动失败（白屏/闪退）
{
  "module": {
    "deviceTypes": ["default"],
    ...
  }
}

// ✅ 正确 — entry 模块必须指定 ["phone"]
{
  "module": {
    "deviceTypes": ["phone"],
    ...
  }
}
```

> ⚠️ **注意区分**：library 模块（HAR/HSP）的 deviceTypes 用 `["default"]`；entry 模块（HAP）必须用 `["phone"]`。AI 常见错误：将从 library 模板中看到的 `["default"]` 照搬到 entry 模块。

**生成后交叉验证**：
```
🔍 TA 自检：
- [ ] TA1: grep "ohos.want.action.home" → 非空？
- [ ] TA2: grep -c "@Entry" Index.ets → 必须 = 1？
- [ ] TA3: 无 .fontColor/.fontSize 在 Column/Row 上？
- [ ] TA4: 无 Color.LightBlue/Color.Xxx web CSS 名称？
- [ ] TA5: ForEach key 使用 item 唯一属性（非 index）？
- [ ] TA6: catch (e) 后使用 String(e)（非 e.message）？
- [ ] TA7: 所有 as 断言后紧接 null/undefined 验证？
- [ ] TA8: grep '"pages"' module.json5 → 非空？
- [ ] TA9: grep "interface.*TestResult\|interface.*{" Index.ets → 必须为空（禁止用 interface 声明测试数据类；合法 interface 如回调形状、类型约束不受此限。TA9 说 interface 不能有默认值用于测试数据类——仅针对数据声明场景）？
- [ ] TA10: 无模板字符串 / backtick（grep '`' Index.ets → 必须为空）？
- [ ] TA11: 无 `Object` 类型注解（grep ": Object" Index.ets → 必须为空）？
- [ ] TA12: 无 `.prototype.` 访问？
- [ ] TA13: 无 `as any` 断言？
- [ ] TA14: 无空壳测试（grep "result.passed = true" Index.ets → 每个匹配的前一行必须含 `if` 关键字。前提：Group B 代码风格要求 `{` 与 `if` 同行——若前一行是孤立的 `{`，需再看前一行。检查所有 passed 赋值前都有条件判断）
- [ ] TA15: grep '"phone"' module.json5 → 非空（deviceTypes 必须是 ["phone"]；["default"] 在部分模拟器版本上导致启动失败）
```

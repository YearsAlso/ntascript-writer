---
name: ntascript-writer
description: 根据自然语言需求生成合规的 NTAScript（.nta）脚本——LightTracker 自动化实验脚本语言。用户说"帮我写个脚本""写个 .nta""我要测37度重复5次"时触发。完整覆盖语法规则、15 个命令参数、常见陷阱、完整示例、错误对照与排查指南，无需项目源码即可生成正确脚本。
allowed-tools: Read, Write, Grep, Glob
---

# NTAScript 脚本生成器

把用户的自然语言实验需求，转成能在 LightTracker 软件里直接跑的 `.nta` 脚本。

## 何时使用

- 用户要写、改、生成一个 `.nta` 脚本
- 用户描述实验流程（温度段、重复次数、稀释比、采集时长等）想变成脚本
- 用户给一段脚本问"对不对""能跑吗"

## 一、语法规则（必须严格遵守）

### 1.1 词法规则

| 规则 | 说明 |
|---|---|
| 行结构 | 每行一条命令：`命令名 参数1 参数2 ...`，参数用空格/制表符分隔 |
| 空行 | 忽略 |
| 注释 | `#` 只能出现在**行首**（允许前导空白），整行注释。**不支持行内注释** |
| 大小写 | 命令名不区分大小写，统一用**大驼峰（PascalCase）**书写，如 `Capture`、`SetTemperature` |
| 字符集 | 行内只允许字母、数字、空白、下划线、点、减号。**禁止中文、全角符号、逗号、@、{}** |
| 参数 | 每个参数是一个单词，**不能含空格** |

### 1.2 三条容易踩的写法

**不支持行内注释**：
```
Capture 30   # 采集30秒    ← 错误：参数个数变多，中文还触发非法字符
# 采集30秒                  ← 正确：注释独占一行
Capture 30
```

**参数不能含空格**：
```
SetViscosity WATER          ← 正确
SetViscosity pure water     ← 错误：被当成 2 个参数
```

**不要用中文/全角符号**：
```
DetectThreshold 6           ← 正确
DetectThreshold 6，          ← 错误：全角逗号
```

## 二、命令参考（15 个）

### 2.1 速查表

| 命令 | 参数 | 取值范围 | 副作用 | 说明 |
|---|---|---|---|---|
| `RecordDilution` | 整数 | 1-1000000 | 无 | 设稀释度并勾选稀释框 |
| `SetViscosity` | 字符串 | `WATER` 或任意词 | 无 | 等于 WATER 选 Water，**其它任何值都落 Custom** |
| `SyringeLoad` | 数字 | 0.1-200 | 注射泵 | 启动灌注，单位 µL/min |
| `SyringeStop` | 无 | — | 注射泵 | 停止灌注 |
| `Delay` | 整数秒 | 建议 0-3600 | 无 | 阻塞等待 N 秒，**必须是整数** |
| `Capture` | 可选整数秒 | 建议 3-120，省略=30 | 相机 | 发起采集（异步，立即返回） |
| `SetNumberCapture` | 整数 | 1-11 | 无 | 设自动采集次数 |
| `DetectThreshold` | 非负整数 | UI 滑块 0-10 | 无 | 设检测阈值，**越界静默截断** |
| `SetTemperature` | 摄氏温度 | 0.0-50.0，最多一位小数 | 温控 | 设温+开温控+自动纠偏极性 |
| `WaitForTemperature` | 摄氏温度 | 0.0-50.0，最多一位小数 | 无 | **等待温度到达目标**，只读不写 |
| `ProcessBasic` | 无 | — | 无 | 触发"处理选中"（不可用时静默跳过） |
| `ExportResults` | 无 | — | 存储 | 触发结果导出 |
| `CameraSettingsMsg` | 任意 | — | 无 | **占位实现，生产脚本不要用** |
| `RepeatStart` | 无 | — | 无 | 循环开始标记 |
| `Repeat` | 整数 ≥1 | — | 无 | 循环结束标记 |

### 2.2 逐条说明

#### RecordDilution `<整数>`
写稀释度控件并自动勾选稀释度复选框。必须是整数，写小数会报错。

#### SetViscosity `<字符串>`
参数转大写后等于 `WATER` 时选 Water，**其它任何值一律落 Custom**。脚本无法设置 Custom 的具体粘度数值（须在界面填写）。同时勾选粘度复选框。

#### SyringeLoad `<流速>` / SyringeStop
`SyringeLoad` 启动灌注，参数即流速（µL/min）。注射器直径、灌注体积等其余参数沿用界面设置，脚本不改变。`SyringeStop` 停止灌注。

#### Delay `<整数秒>`
阻塞式等待 N 秒后继续下一条。参数为**非负整数**（`Delay 0` 合法，`Delay 1.5` 报错）。等待期间界面保持刷新。

#### Capture `[整数秒]`
设置采集时长控件并触发采集。**省略参数时按 30 秒**。采集是异步发起的：命令立即返回，脚本不会等待采集完成，**必须自行 `Delay` 预留时间**（建议 ≥ 采集时长 + 5 秒）。

#### SetNumberCapture `<1-11>`
设置自动采集次数。硬校验 1-11：超出属于执行时错误。

#### DetectThreshold `<非负整数>`
设置检测阈值滑块。UI 范围 0-10，超出会被**静默截断**（写 50 实际变 10，且无警告）。

#### SetTemperature `<摄氏温度>`
合法格式：非负数、最多一位小数。`25`、`25.0`、`30.5` 合法；`25.55`、`-5`、`50.1` 非法。

执行时做三件事，顺序固定：
1. **纠偏 M500 极性**——按执行当时的环境温度判定：目标高于环境 0.5℃ 以上设为加热，低 0.5℃ 以上设为制冷；环境温度读数无效或温差在 0.5℃ 以内时不改动极性
2. **写目标温度**（D900）
3. **开启温控**（M10）

**该命令不等温度稳定**，设温后必须用 `Delay` 或 `WaitForTemperature` 等待。

#### WaitForTemperature `<摄氏温度>`
**不写任何硬件寄存器**，只读当前实际温度（D906）。每 2 秒读一次，进入目标 ±0.5℃ 容差范围就继续。最多等 600 秒，超时会报错并中止脚本。

用法：紧跟在 `SetTemperature` 后面，替代固定的 `Delay 300`：
```
SetTemperature 37.0
WaitForTemperature 37.0     ← 温度真到 37±0.5℃ 才继续
Capture 30
```
好处：升温快就提前走，升温慢就多等，比死等 300 秒准。

#### ProcessBasic / ExportResults
分别等价于点击界面"处理选中"与结果导出。`ProcessBasic` 在按钮不可用时**静默跳过**且不报错。

## 三、循环

只支持**单层循环**，嵌套会判语法错误。`RepeatStart` 与 `Repeat` 必须一一配对。

⚠️ **`Repeat N` 表示「额外再重复 N 次」，循环体共执行 N+1 次**：

| 想要循环体执行 | 应写 |
|---|---|
| 3 次 | `Repeat 2` |
| 5 次 | `Repeat 4` |
| 1 次 | 直接写一次，不要用循环 |

```
RepeatStart
Capture 20
Delay 25
Repeat 2
```
（上面这段：Capture 20 + Delay 25 跑 3 次）

周期更长的重复测量建议优先用 `SetNumberCapture`（自动重复采集），循环只用于整段流程的重复。

## 四、五大陷阱（最容易写错的地方）

**陷阱 1：`Repeat N` 实际跑 N+1 次**（见上）

**陷阱 2：`Capture` 是异步的，后面必须跟 `Delay`**
```
Capture 30        ← 发起采集后立即返回，不等采集完
Delay 35          ← 必须预留采集+保存时间
ProcessBasic
```

**陷阱 3：`SetTemperature` 不等温度到达**
```
SetTemperature 37.0
WaitForTemperature 37.0    ← 推荐：等到真到温
# 或 Delay 300             ← 也可以：死等 5 分钟
Capture 30
```

**陷阱 4：不支持行内注释**（见 1.2）

**陷阱 5：无变量、无 if、无表达式**
所有参数必须是**字面量数字/单词**，不能写 `$temp`、`if`、`for`、计算式。

## 五、完整示例

### 模板 A：单次测量（默认骨架）
```
RecordDilution 100
SetViscosity WATER
SyringeLoad 20
Delay 10
SyringeStop
SetNumberCapture 1
DetectThreshold 6
Capture 30
Delay 35
ProcessBasic
ExportResults
```

### 模板 B：温度分段（等到温再采）
```
# 25.0 → 30.5 → 37.0 ℃ 三段采集
SetTemperature 25.0
WaitForTemperature 25.0
DetectThreshold 4
Capture 30
Delay 35
ProcessBasic
ExportResults

SetTemperature 30.5
WaitForTemperature 30.5
DetectThreshold 6
Capture 30
Delay 35
ProcessBasic
ExportResults

SetTemperature 37.0
WaitForTemperature 37.0
DetectThreshold 6
Capture 30
Delay 35
ProcessBasic
ExportResults
```

### 模板 C：温度分段（固定延时版）
如果不想用 WaitForTemperature，可以用 Delay：
```
SetTemperature 25.0
Delay 300
DetectThreshold 4
Capture 30
Delay 35
ProcessBasic
ExportResults
```

### 模板 D：循环重复采集
```
RepeatStart
Capture 20
Delay 25
Repeat 2
```
（`Repeat 2` = 跑 3 次。想跑 5 次写 `Repeat 4`）

### 模板 E：整段流程循环
```
RepeatStart
RecordDilution 100
SetViscosity WATER
SyringeLoad 20
Delay 10
SyringeStop
Capture 30
Delay 35
Repeat 2
ProcessBasic
ExportResults
```
（ProcessBasic 和 ExportResults 放循环外，跑完一次性处理导出）

### 模板 F：稀释度对比
```
SetViscosity WATER
DetectThreshold 6

RecordDilution 10
SyringeLoad 5.0
Delay 10
Capture 30
Delay 35
SyringeStop
ProcessBasic
ExportResults

Delay 30

RecordDilution 100
SyringeLoad 5.0
Delay 10
Capture 30
Delay 35
SyringeStop
ProcessBasic
ExportResults
```

## 六、错误信息对照

| 报错原文 | 含义 | 处理 |
|---|---|---|
| `Unknown command 'xxx'` | 命令名拼错 | 对照命令表 |
| `takes no arguments` / `requires 1 integer argument` | 参数个数不对 | 注释独占一行，不要行内注释 |
| `contains characters outside the allowed set` | 行内有中文或特殊符号 | 只用字母/数字/空格/下划线/点/减号 |
| `RECORDDILUTION requires integer argument` | 稀释度写了小数 | 改为整数 |
| `DELAY requires positive integer argument` | 延时为负或非整数 | 改为非负整数 |
| `CAPTURE requires positive integer argument` | 采集时长不对 | 改为正整数 |
| `DETECTTHRESHOLD requires non-negative integer argument` | 阈值写了负数或小数 | 改为非负整数 |
| `temperature must be ... at most one decimal place` | 温度格式不对 | 最多一位小数，如 `25.5` |
| `temperature must be between 0.0 and 50.0` | 温度越界 | 限制在 0.0-50.0 |
| `REPEAT requires positive integer argument` | 循环次数不对 | 改为 ≥ 1 |
| `Loop structure error` | 循环开始/结束数量不匹配 | 一一配对 |
| `Loop nesting is not allowed` | 嵌套循环 | 改成单层，或用 SetNumberCapture |
| `等待温度到达超时：目标 X℃，当前 Y℃` | WaitForTemperature 等满 600 秒未到温 | 确认温控已开、目标温度一致、升温速度够快 |

## 七、生成工作流

### 第一步：问清楚缺的信息
用户需求不完整时，先问（能合理推断的就推断并注明）：
- 测几个温度段？各多少度？
- 每段采集几秒？重复几次？
- 稀释度多少？介质是水还是其它？
- 注射泵要不要？流速多少？
- 升温用 WaitForTemperature 还是固定 Delay？（没说就用 WaitForTemperature）

### 第二步：按模板组装
从上面模板里挑，把数字替换进去。

### 第三步：自查（输出前必过）
- [ ] 每条命令参数个数正确
- [ ] `Capture` 后面都跟了 `Delay`
- [ ] `SetTemperature` 后面跟了 `WaitForTemperature` 或 `Delay`
- [ ] `Repeat N` 的 N 是"想要的次数 - 1"
- [ ] 没有行内注释、没有中文（命令行内）、没有全角符号
- [ ] 温度都是一位小数以内
- [ ] `DetectThreshold` 在 0-10 之间
- [ ] `SetNumberCapture` 在 1-11 之间
- [ ] 没有用 `CameraSettingsMsg`
- [ ] 循环体是单层

## 八、输出规范

- 输出完整 `.nta` 内容，用代码块包裹
- 注释用中文写在**单独一行**，说明这段在做什么
- 对用户没说清、你帮着填的数字，在脚本外说明："升温时间 X 秒是占位，请按你实际升温速度调整"
- 如果用户的需求现有语法表达不了（比如"根据上次结果决定下一步"），**明确告诉用户现在的语言做不到**，不要硬编不存在的命令

## 九、限制说明

- 循环只支持单层
- 不支持行内注释
- 无变量、无条件判断、无表达式
- 执行期间没有"停止脚本"按钮
- 跑完会恢复执行前的温控开关状态，但**目标温度不恢复**

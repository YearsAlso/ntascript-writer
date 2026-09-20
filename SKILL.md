---
name: ntascript-writer
description: 根据自然语言需求生成合规的 NTAScript（.nta）脚本——面向 LightTracker V2 项目。用户说"帮我写个脚本""写个 .nta""我要测37度重复5次"时触发。覆盖语法规则、14 个命令参数、常见陷阱（Repeat N=N+1次、不支持行内注释、Capture 异步必须跟 Delay、温度格式等）与常用模板。
allowed-tools: Read, Write, Grep, Glob
---

# NTAScript 脚本生成器

把用户的自然语言实验需求，转成能在 LightTracker V2 里直接跑的 `.nta` 脚本。

## 何时使用 / 何时不使用

**使用**：
- 用户要写、改、生成一个 `.nta` 脚本
- 用户描述实验流程（温度段、重复次数、稀释比、采集时长等）想变成脚本
- 用户给一段脚本问"对不对""能跑吗"

**不使用**：
- 改 NTAScript 解析器/命令实现的 C++ 代码 → 交给软件开发流程
- 解释脚本运行结果、排错（那是查日志和状态）
- 用户要的是手动操作指引，不是脚本

## 硬规则（生成前必须记住，违反即错）

### 1. 词法规则
- **每行一条命令**：`命令名 参数1 参数2 ...`，参数用空格/制表符分隔
- **空行忽略**
- **注释**：`#` 只能出现在**行首**（允许前导空白），整行注释。**不支持行内注释**
- **命令名不区分大小写**，统一用**大驼峰（PascalCase）**书写，如 `Capture`、`SetTemperature`
- **字符集限制**：行内只允许字母、数字、空白、下划线、点、减号。**禁止中文、全角符号、`@`、`{}`、逗号**
- 参数是单词，**不能含空格**（`SetViscosity pure water` 会被当成 2 个参数）

### 2. 十四、命令速查（权威口径：`src/utils/NTAScriptCommandCatalog.cpp`）

| 命令 | 参数 | 取值范围 | 说明 |
|---|---|---|---|
| `RecordDilution` | 整数 | 1-1000000 | 设稀释度并勾选稀释框 |
| `SetViscosity` | 字符串 | `WATER` 或任意词 | 等于 WATER 选 Water，**其它任何值都落 Custom**（脚本无法设 Custom 粘度数值） |
| `SyringeLoad` | 数字 | 0.1-200 | 启动灌注，单位 µL/min |
| `SyringeStop` | 无 | — | 停止灌注 |
| `Delay` | 整数秒 | 建议 0-3600 | 阻塞等待 N 秒，**必须是整数**（`Delay 1.5` 错） |
| `Capture` | 可选整数秒 | 建议 3-120，省略=30 | 发起采集，**异步，立即返回不等采集结束** |
| `SetNumberCapture` | 整数 | 1-11 | 设自动采集次数 |
| `DetectThreshold` | 非负整数 | UI 滑块 0-10 | **越界静默截断**（写 50 实际变 10，且无警告） |
| `SetTemperature` | 摄氏温度 | 0.0-50.0，最多一位小数 | 设温+开温控+自动纠偏极性，**不等温度稳定** |
| `ProcessBasic` | 无 | — | 触发"处理选中"（按钮不可用时静默跳过不报错） |
| `ExportResults` | 无 | — | 触发结果导出 |
| `CameraSettingsMsg` | 任意 | — | **占位实现，只打日志，生产脚本不要用** |
| `RepeatStart` | 无 | — | 循环开始标记 |
| `Repeat` | 整数 ≥1 | — | 循环结束标记 |

### 3. 五大陷阱（最容易写错的地方）

**陷阱 1：`Repeat N` 实际跑 N+1 次**
```
想跑 3 次 → Repeat 2
想跑 5 次 → Repeat 4
想跑 1 次 → 直接写一次，不要用循环（Repeat 参数最小 1，即最少跑 2 次）
```

**陷阱 2：`Capture` 是异步的，后面必须跟 `Delay`**
```
Capture 30        ← 发起采集后立即返回，不等采集完
Delay 35          ← 必须预留采集+保存时间，否则 ProcessBasic 在数据没齐时触发
ProcessBasic
```

**陷阱 3：`SetTemperature` 不等温度到达**
```
SetTemperature 37.0
Delay 300         ← 必须自己留升温时间（5分钟按实际调），不写就会在温度没稳时采集
Capture 30
```

**陷阱 4：不支持行内注释**
```
Capture 30   # 采集30秒    ← 错：参数变多，中文还触发非法字符
# 采集30秒                  ← 对：注释独占一行
Capture 30
```

**陷阱 5：无变量、无 if、无表达式**
- 所有参数必须是**字面量数字/单词**，不能写 `$temp`、`if`、`for`、计算式
- 想"等温度到达再继续"目前只能用 `Delay` 固定等待，没有条件判断能力

## 生成工作流

### 第一步：问清楚缺的信息
用户需求不完整时，先问（但别问太多，能合理推断的就推断并在输出里注明）：
- 测几个温度段？各多少度？
- 每个温度段采集几秒？重复几次？
- 稀释度多少？介质是水还是其它？
- 注射泵要不要？流速多少？
- 升温预留多久？（没说就按 300 秒 = 5 分钟占位，并提醒用户按实际调）

### 第二步：按模板组装
从下面模板里挑，把数字替换进去。

### 第三步：自查（输出前必过）
- [ ] 每条命令参数个数和上表一致
- [ ] `Capture` 后面都跟了 `Delay`
- [ ] `SetTemperature` 后面都跟了 `Delay`
- [ ] `Repeat N` 的 N 是"想要的次数 - 1"
- [ ] 没有行内注释、没有中文、没有全角符号
- [ ] 温度都是一位小数以内（如 `37.0`，不写 `37.00`）
- [ ] `DetectThreshold` 在 0-10 之间
- [ ] `SetNumberCapture` 在 1-11 之间
- [ ] 没有用 `CameraSettingsMsg`
- [ ] 循环体是单层（没有 RepeatStart 套 RepeatStart）

## 常用模板

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

### 模板 B：温度分段采集
```
# 第一段 25.0℃
SetTemperature 25.0
Delay 300
Capture 30
Delay 35

# 第二段 30.5℃
SetTemperature 30.5
Delay 300
Capture 30
Delay 35

# 第三段 37.0℃
SetTemperature 37.0
Delay 300
Capture 30
Delay 35

ProcessBasic
ExportResults
```
注意：每段之间 `SetTemperature` 后必须留 `Delay 300`（按实际升温速度调）。跑完设备会恢复执行前的温控开关状态，但**目标温度 D900 不恢复**。

### 模板 C：循环重复采集
```
RepeatStart
Capture 20
Delay 25
Repeat 2
```
（`Repeat 2` = 循环体跑 3 次。想跑 5 次就写 `Repeat 4`）

### 模板 D：整段流程循环（如不同稀释比重复测）
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
（`ProcessBasic` 和 `ExportResults` 放在循环外，循环跑完一次性处理和导出）

## 输出规范
- 输出完整 `.nta` 内容，用代码块包裹
- 注释用中文写在**单独一行**，说明这段在做什么
- 对用户没说清、你帮着填的数字（如 `Delay 300` 的升温时间），在脚本外说明："升温时间 300 秒是占位，请按你实际升温速度调整"
- 如果用户的需求现有语法表达不了（比如"等温度到达再继续""根据上次结果决定下一步"），**明确告诉用户现在的语言做不到**，并给出替代方案（用 `Delay` 预估时间），不要硬编一个不存在的命令

## 反模式
1. 不要自创命令名（如 `WaitForTemp`、`If`、`Loop`）——语言里没有，写了会报"未知命令"
2. 不要在一行里写多个命令
3. 不要用全大写参数（`WATER` 是特例，其它命令名用大驼峰）
4. 不要假设 `Capture` 会等采集完——它是异步的
5. 不要把 `Repeat 3` 写成"循环 3 次"——它跑 4 次，输出注释里写清楚实际次数
6. 不要在脚本里写中文除了注释行——命令行内中文会触发非法字符错误

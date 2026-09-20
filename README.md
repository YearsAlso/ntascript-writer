# ntascript-writer

给 AI 助手用的 Skill——根据自然语言需求自动生成 LightTracker 的自动化实验脚本（.nta 文件）。

## 这是什么

NTAScript 是 LightTracker 软件的自动化实验脚本语言。这个 Skill 让 AI 助手
（Claude Code、Cursor、豆包、Trae 等）学会写正确的 `.nta` 脚本。

**你不需要记命令名和参数格式**——直接用大白话告诉 AI 你要做什么实验，它会帮你生成能直接跑的脚本。

## 安装

在你的项目目录下执行：

```bash
npx skills add https://github.com/YearsAlso/ntascript-writer.git
```

安装后，AI 助手会自动识别并加载这个 Skill。

> 前提：你电脑上装了 Node.js（自带 npx）。没装的话先去 https://nodejs.org 下载安装。

## 怎么用：跟 AI 对话生成实验脚本

装好 Skill 后，打开你的 AI 助手，直接说你要做什么实验就行。

### 第一步：告诉 AI 你的实验需求

你只要用大白话说清楚这几件事：

| 你要告诉 AI 的 | 例子 |
|---|---|
| 测几个温度段、各多少度 | "测 25 度、30.5 度、37 度三个温度段" |
| 每段采集多久 | "每段采集 30 秒" |
| 重复几次 | "每个温度段重复测 3 次" |
| 稀释比多少 | "稀释比 100" |
| 介质是水还是其它 | "水介质" |
| 要不要注射泵、流速多少 | "注射泵流速 20" |

**没提到的参数 AI 会用默认值**（升温自动等待、采集 30 秒、阈值 6），生成后你可以自己改数字。

### 第二步：AI 自动生成脚本

**你说：**
> "帮我写个脚本，测 25 度、30.5 度、37 度三个温度段，每段采集 30 秒，稀释比 100，水介质，每个温度段重复测 3 次"

**AI 自动生成：**
```
# 温度分段重复测量脚本
# 25.0 → 30.5 → 37.0 ℃，每段重复 3 次

RecordDilution 100
SetViscosity WATER

# 第一段 25.0℃
SetTemperature 25.0
WaitForTemperature 25.0
RepeatStart
Capture 30
Delay 35
Repeat 2

# 第二段 30.5℃
SetTemperature 30.5
WaitForTemperature 30.5
RepeatStart
Capture 30
Delay 35
Repeat 2

# 第三段 37.0℃
SetTemperature 37.0
WaitForTemperature 37.0
RepeatStart
Capture 30
Delay 35
Repeat 2

ProcessBasic
ExportResults
```

### 第三步：把脚本用到 LightTracker 里

1. 把 AI 生成的内容复制，存成 `.nta` 文件（比如 `my_experiment.nta`）
2. 打开 LightTracker，进入 **Experiments → Script** 标签页
3. 点「加载脚本」，选你刚存的 `.nta` 文件
4. 确认状态显示「就绪」后，点「执行脚本」

## 常见对话示例

### 示例 1：单次测量
> "帮我写个单次测量脚本，稀释比 100，水介质，采集 30 秒"

AI 会生成标准单次测量脚本。

### 示例 2：温度分段对比
> "测 25 度和 37 度两段，每段采集 60 秒，水介质"

AI 会生成两段温度脚本，自动加 `WaitForTemperature` 等温度稳定。

### 示例 3：重复采集
> "我要重复采集 5 次，每次 20 秒"

AI 会生成循环脚本，注意 `Repeat 4` = 跑 5 次（AI 会自动处理这个陷阱）。

### 示例 4：修改现有脚本
> "这个脚本现在是测 25 度，帮我改成测 37 度，采集时间改成 60 秒"

AI 会帮你改。

## 支持的 AI 工具

任何支持 `npx skills add` 安装 Skill 的 AI 编程工具都能用：

- Claude Code
- Cursor
- 豆包编程助手
- Trae
- GitHub Copilot

## 注意事项

- 脚本命令名统一用大驼峰（如 `Capture`、`SetTemperature`）
- 不支持条件判断和变量——所有参数必须是具体数字
- `SetTemperature` 后会自动等温度到达（用 `WaitForTemperature`）
- 生成脚本后，建议先在 LightTracker 里试跑一次确认没问题再正式用

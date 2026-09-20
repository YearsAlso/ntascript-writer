# ntascript-writer

给 AI 助手用的 Skill——根据自然语言需求自动生成合规的 NTAScript（.nta）脚本。

## 这是什么

NTAScript 是 LightTracker 软件的自动化实验脚本语言。这个 Skill 让 AI 助手
（Claude Code、Cursor、豆包、Trae 等）学会写正确的 `.nta` 脚本，
你不用记命令名和参数格式，直接说人话描述实验需求就行。

## 安装

在你的项目目录下执行：

```bash
npx skills add https://github.com/YearsAlso/ntascript-writer.git
```

安装后，AI 助手会自动识别并加载这个 Skill。

> 前提：你电脑上装了 Node.js（自带 npx）。没装的话先去 https://nodejs.org 下载安装。

## 怎么用

装好之后，打开你的 AI 助手，直接说你要做什么实验就行。

### 示例对话

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
Delay 300
RepeatStart
Capture 30
Delay 35
Repeat 2

# 第二段 30.5℃
SetTemperature 30.5
Delay 300
RepeatStart
Capture 30
Delay 35
Repeat 2

# 第三段 37.0℃
SetTemperature 37.0
Delay 300
RepeatStart
Capture 30
Delay 35
Repeat 2

ProcessBasic
ExportResults
```

### 你只要告诉 AI 这几件事

| 你要告诉 AI 的 | 例子 |
|---|---|
| 测几个温度段、各多少度 | "测 25 度和 37 度两段" |
| 每段采集多久 | "每段采集 30 秒" |
| 重复几次 | "重复 3 次" |
| 稀释比多少 | "稀释比 100" |
| 介质是水还是其它 | "水介质" |
| 要不要注射泵、流速多少 | "注射泵流速 20" |

没提到的参数 AI 会用默认值（升温时间 5 分钟、采集 30 秒、阈值 6），
生成后你可以自己改数字。

## 支持的 AI 工具

任何支持 `npx skills add` 安装 Skill 的 AI 编程工具都能用，包括但不限于：

- Claude Code
- Cursor
- 豆包编程助手
- Trae
- GitHub Copilot

## 注意事项

- 脚本命令名统一用大驼峰（如 `Capture`、`SetTemperature`）
- 不支持条件判断和变量——所有参数必须是具体数字
- `SetTemperature` 不会自动等温度到达，必须用 `Delay` 留升温时间
- 生成脚本后，建议先在 LightTracker 里试跑一次确认没问题再正式用

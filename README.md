# Lazy AI Morning Workbench

> 勤劳的人有自己的工作台，懒人只需要定时工作流。

这是我每天早晨工作流的脱敏、可复用版本。它解决的不是“怎样做一个更复杂的效率系统”，而是一个更懒的问题：每天不想重新打开一堆页面、回忆项目进度、检查卡点，再决定先做什么。

到点以后，它会读取已有项目状态，生成一张很短的晨间工作台：

1. 今天最重要的 1—3 件事；
2. 卡点与需要本人拍板的事项；
3. 真实异常；
4. 可选的当日课程或学习材料链接。

自动化负责准备，人仍然负责判断、验收和拍板。

## 真实版本与公开版本

我目前使用的私人版本分三段运行：

- `09:20`：准备当天课程，并渲染为独立 HTML；
- `09:30`：读取项目看板，生成不超过 500 字的每日工作台；
- `10:00`：运行一个私人业务内容预览任务。

公开仓库只包含 09:30 工作台聚合器，以及一个可接收 09:20 上游课程结果的输入接口。私人课程生成器和 10:00 业务任务涉及个人学习状态、业务、账号和提示词，不公开真实配置。

公开版不会读取私人记忆、浏览器 Cookie、Codex 任务历史或用户主目录，也不会自动发布、发消息、删除文件或修改外部系统。

## 目录

```text
.
├── config.example.json
├── examples/
│   ├── anomalies.json
│   ├── board.json
│   └── course.json
├── prompts/
│   └── daily_workbench.md
├── schedulers/
│   ├── linux-cron.example
│   └── run-linux.sh
├── scripts/
│   └── morning_workbench.py
└── tests/
    └── test_morning_workbench.py
```

## 快速运行

默认配置要求 Python 3.11 或更高版本，不依赖第三方包。

```bash
cp config.example.json config.json
python scripts/morning_workbench.py --config config.json --dry-run
python scripts/morning_workbench.py --config config.json
```

默认示例只读取 `examples/`，并把结果写入 `output/daily-workbench-YYYY-MM-DD.md`。

`--dry-run` 只会打印将要读取和预期写入的仓库内相对路径，不会打印任务正文，也不会生成文件。若当天已有不同内容，它会显示将创建的带时间版本。第一次使用时建议先运行它。

## 输入合同

### 项目看板

`board.json` 的顶层为 `tasks` 数组。每条任务支持：

```json
{
  "title": "整理本周课程",
  "status": "in_progress",
  "next_action": "确认今天要学习的章节",
  "blocker": null,
  "approval_needed": null
}
```

`in_progress`、`blocked`，或不属于已完成、取消、归档状态且明确存在 `next_action` 的任务会进入工作台。`blocked` 优先于普通进行中任务。

### 课程

`course.json` 是可选输入：

```json
{
  "title": "今天的课程标题",
  "summary": "一句话说明今天学什么",
  "url": "https://example.com/course/today"
}
```

如果你的课程由另一个工作流生成，只要在 09:30 前更新这个文件即可。

### 异常

`anomalies.json` 只接受已经由监控或脚本确认的异常，不让模型猜测：

```json
{
  "items": ["昨晚备份任务失败，需要检查网络"]
}
```

## 两种使用方式

### 1. 确定性版本

直接运行 `morning_workbench.py`。它不会调用模型，适合作为安全默认值和测试基线。

### 2. 可选的 Agent 压缩版本

确定性脚本是安全默认值。如果需要更自然的表达，可以把字段白名单后的上下文交给一个没有工具、没有外部写权限的独立 Agent，并使用 `prompts/daily_workbench.md`。

提示词不是安全隔离。不要把这份上下文交给能够执行命令、读取凭据、发消息或修改外部系统的 Agent。仍需对 Agent 输出做人工检查。

## 定时执行

Linux cron 示例见 `schedulers/linux-cron.example` 和 `schedulers/run-linux.sh`。其它调度器请使用自己的受保护配置或 secret store；不要把会话标识、访问令牌或其它凭据写进仓库脚本。

示例中的 `CRON_TZ` 适用于支持该变量的 cron 实现；不支持时请把系统时区设好或按本机时区换算。脚本自身使用进程锁；若进程崩溃后留下 `output/.morning-workbench.lock`，先确认没有任务正在运行，再手工删除该锁文件。

Linux wrapper 默认调用 `python3`。如果解释器不在这个名称下，可在调度器的受保护环境中设置 `PYTHON_BIN`；不要把本机绝对路径提交进仓库。

调度器必须自行处理以下问题：

- 明确时区；
- 防止同一任务并发重入；
- 电脑休眠或关机后的补跑策略；
- 日志轮换与失败提醒；
- 外部发送动作默认关闭。

这个仓库只生成本地 Markdown，不负责发飞书、Slack、邮件或社交平台。

## 安全边界

- 配置中只允许项目内相对路径；脚本拒绝普通路径穿越和符号链接输出目录，并使用唯一临时文件与进程锁。
- 不读取环境变量集合、浏览器配置、凭据库或用户主目录。
- 不包含任何 API Key、Cookie、账号 ID、线程 ID、私有域名和本机绝对路径。
- 确定性脚本把第三方输入只当数据处理；可选 Agent 不构成安全边界，必须无工具、无外部写权限并接受人工复核。
- 高风险动作——发布、付款、删除、权限修改——必须留给人工确认。

脚本会转义 Markdown 字段，只接受没有内嵌账号密码的 HTTP/HTTPS 课程链接。可选 Agent 仍不是安全边界。

公开版会主动移除课程 URL 的全部 query 与 fragment，避免把签名链接、临时 token 或跟踪参数写入工作台。它防御常见误配置和普通符号链接路径，不声称隔离同一系统账号下主动篡改目录的恶意进程。

一句话：我懒得重复，但我不懒得验收。

## License

MIT，见 [LICENSE](LICENSE)。

## 平台说明

默认 Asia/Shanghai 在没有系统时区数据库的 Windows 环境中也能运行。若改用其它 IANA 时区，操作系统可能需要安装 tzdata。

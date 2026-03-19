---
name: birthday-reminder
description: 管理并计算生日提醒（阳历与农历），支持每条记录单独配置和全局默认值，支持当天提醒/提前 N 天/多次提醒和提醒时间配置，默认使用北京时间。用于需要生成或维护本地生日提醒方案、编写配置文件、验证提醒是否到期、以及在本机设置定时任务（cron/launchd）时。
---

# Birthday Reminder

## 概览

使用本技能生成一套完全本地运行的生日提醒能力：读取 JSON 配置，按北京时间计算阳历/农历生日提醒时点，输出到期提醒，便于接入通知渠道（终端、企业微信机器人、邮件脚本等）。

## 工作流

1. 创建或更新配置文件（建议复制 `assets/birthdays.example.json`）。
2. 运行 `scripts/birthday_reminder.py check` 验证提醒是否按预期触发。
3. 运行 `scripts/birthday_reminder.py list` 查看所有已配置提醒。
4. 设置本地定时任务，按固定频率执行脚本并处理输出。

## 用户示例请求

- “添加一条农历八月初八的生日，并提前 15 天和当天上午 9 点提醒。”
- “把妻子生日设为阳历 3 月 25 日，默认时区北京，提前 7 天和 1 天提醒。”

## 配置规则

配置文件为 JSON，顶层结构：

```json
{
  "defaults": {
    "calendar": "solar",
    "timezone": "Asia/Shanghai",
    "remind_at": "09:00",
    "offset_days": [7, 1, 0],
    "leap_strategy": "skip"
  },
  "people": [
    {
      "name": "妻子",
      "calendar": "solar",
      "month": 3,
      "day": 25
    },
    {
      "name": "父母",
      "calendar": "lunar",
      "month": 8,
      "day": 8,
      "offset_days": [15, 3, 0]
    }
  ]
}
```

字段说明：

- `defaults`：全局默认值。
- `people`：生日记录列表；每一条可覆盖任意默认字段。
- `calendar`：`solar` 或 `lunar`。
- `month/day`：生日月日。
- `offset_days`：提醒提前天数数组，`0` 表示当天提醒。
- `remind_at`：提醒时间，格式 `HH:MM`。
- `timezone`：IANA 时区名，默认 `Asia/Shanghai`。
- `leap_month`：仅农历使用，是否闰月生日。
- `leap_strategy`：闰月缺失年份处理策略。
- `skip`：该年跳过。
- `use-non-leap`：该年改用同月非闰月。

## 运行命令

```bash
python3 scripts/birthday_reminder.py check --config assets/birthdays.example.json
```

列出所有已配置提醒：

```bash
python3 scripts/birthday_reminder.py list --config assets/birthdays.example.json
```

测试指定时间：

```bash
python3 scripts/birthday_reminder.py check \
  --config assets/birthdays.example.json \
  --now "2026-03-25 09:00:00" \
  --output json
```

常用参数：

- `--window-minutes`：回看窗口（默认 `70` 分钟）。
- `--output text|json`：输出格式。
- `--now`：建议使用 `yyyy-MM-DD HH:mm:ss`，例如 `2026-03-25 09:00:00`。
- `list` 命令按“下一次生日”计算每条记录的所有提醒点（含多次提前提醒）。

## 如何检查（推荐顺序）

1. 先看你当前配置会生成哪些提醒：

```bash
python3 scripts/birthday_reminder.py list --config /绝对路径/birthdays.json
```

2. 再检查“现在是否有到期提醒”：

```bash
python3 scripts/birthday_reminder.py check --config /绝对路径/birthdays.json --output json
```

3. 最后预览通知发送内容（不真实发送）：

```bash
python3 scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json \
  --dry-run
```

## 本地定时方案（推荐）

优先使用系统自带任务调度器，不依赖第三方服务。

### macOS (`launchd`)

1. 创建 `~/Library/LaunchAgents/ai.clawhub.birthday-reminder.plist`，每 10 分钟执行一次脚本。
2. 命令中调用：

```bash
python3 /绝对路径/birthday-reminder/scripts/birthday_reminder.py check --config /绝对路径/your-birthdays.json --output json
```

3. 使用 `launchctl load -w` 加载任务。

### Linux (`cron`)

每 10 分钟执行一次：

```cron
*/10 * * * * /usr/bin/python3 /绝对路径/birthday-reminder/scripts/birthday_reminder.py check --config /绝对路径/your-birthdays.json --output json >> /tmp/birthday-reminder.log 2>&1
```

## 通知配置（支持多平台）

推荐使用 `scripts/notify_bridge.py` 做统一分发。它会先计算到期提醒，再按配置分发到一个或多个通知通道。

支持的通知类型：

- `console`：打印到终端（调试用）。
- `file`：写入文件日志。
- `webhook`：通用 Webhook（推荐，适配任意平台）。
- `feishu`：飞书机器人 Webhook。
- `dingtalk`：钉钉机器人 Webhook。
- `slack`：Slack Incoming Webhook。
- `telegram`：Telegram Bot。

示例通知配置：`assets/notify.example.json`。

消息风格：`notify.json` 顶层 `message_style` 可选 `warm|simple`，默认 `warm`（温馨版，含蛋糕图标）。

执行命令：

```bash
python3 scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json
```

仅测试不发送（预览结果）：

```bash
python3 scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json \
  --dry-run
```

建议：

- 优先用 `webhook` 做统一接入，这样能兼容飞书、企业微信、Discord、自建系统等。
- 平台切换时只改 `notify.json`，无需改生日计算逻辑。

### 示例：桐桐（阳历 3/18 晚上 19:00，Telegram 提醒）

`birthdays.json`：

```json
{
  "defaults": {
    "calendar": "solar",
    "timezone": "Asia/Shanghai",
    "remind_at": "09:00",
    "offset_days": [0],
    "leap_strategy": "skip"
  },
  "people": [
    {
      "name": "桐桐",
      "calendar": "solar",
      "month": 3,
      "day": 18,
      "remind_at": "19:00",
      "offset_days": [0]
    }
  ]
}
```

`notify.json`：

```json
{
  "channels": [
    {
      "type": "telegram",
      "enabled": true,
      "bot_token": "你的_bot_token",
      "chat_id": "你的_chat_id"
    }
  ]
}
```

测试发送：

```bash
python3 scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json \
  --now "2026-03-18 19:00:00"
```

## 问题反馈

- 代码仓库：`https://github.com/905583906/jeff-skills`
- 提交问题（Issues）：`https://github.com/905583906/jeff-skills/issues`

遇到问题时，建议在 Issue 里附上：

- 你的配置片段（隐藏敏感信息）
- 执行命令
- 报错信息或截图

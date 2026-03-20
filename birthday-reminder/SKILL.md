---
name: birthday-reminder
description: 在 OpenClaw 中管理并计算生日提醒（阳历与农历），支持每条记录单独配置和全局默认值，支持当天提醒、提前 N 天、多次提醒和多通道通知发送。
homepage: https://github.com/905583906/jeff-skills
metadata: {"openclaw":{"requires":{"anyBins":["python3","python"]},"homepage":"https://github.com/905583906/jeff-skills"}}
---

# Birthday Reminder

## 定位

- 这个 skill 仅面向 OpenClaw。
- 不需要兼容 Codex Automation 或其他平台的定时任务接口。

## 给 OpenClaw 的执行规则

- 这个 skill 的核心是两个本地脚本：
  - `{baseDir}/scripts/birthday_reminder.py`
  - `{baseDir}/scripts/notify_bridge.py`
- 执行 Python 时优先使用 `python3`；如果当前环境没有 `python3`，回退到 `python`。
- 需要真实发送提醒时，必须执行 `notify_bridge.py`；`check` 只检查，`list` 只预览。
- 用户没有提供配置路径时，不要猜测；要求用户提供 `birthdays.json` 和 `notify.json` 的绝对路径。
- 用户要求“定时执行”时，说明这个 skill 不会自动注册 Cron；需要由 OpenClaw Cron 单独创建作业。
- 作为 Cron 后台任务运行时，如果没有到期提醒，应安静结束，不额外发送解释性消息。

## 5分钟上手

### 1) 先决条件

- 已安装本技能。
- 环境可执行 `python3` 或 `python`（建议 3.9+）。
- 已准备通知通道凭据（如 Telegram 的 `bot_token` 和 `chat_id`）。

说明：

- 下文命令默认使用 `python3`。
- 如果当前环境没有 `python3`，请将命令中的 `python3` 改为 `python`。
- 最快起步方式是直接编辑：
  - `{baseDir}/quickstart/birthdays.json`
  - `{baseDir}/quickstart/notify.json`

### 2) 准备 `birthdays.json`

复制下面模板并按需修改：

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

关键字段：

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

### 3) 准备 `notify.json`

建议从 `assets/notify.example.json` 复制后修改。常见 Telegram 配置：

```json
{
  "message_style": "warm",
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

支持的通知类型：

- `console`
- `file`
- `webhook`
- `feishu`
- `dingtalk`
- `slack`
- `telegram`

### 4) 三步检查（推荐顺序）

先看会生成哪些提醒：

```bash
python3 {baseDir}/scripts/birthday_reminder.py list --config /绝对路径/birthdays.json
```

再查当前是否有到期提醒：

```bash
python3 {baseDir}/scripts/birthday_reminder.py check --config /绝对路径/birthdays.json --output json
```

最后预览发送内容（不真实发送）：

```bash
python3 {baseDir}/scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json \
  --dry-run
```

时间模拟（推荐格式 `yyyy-MM-DD HH:mm:ss`）：

```bash
python3 {baseDir}/scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json \
  --now "2026-03-18 19:00:00"
```

### 5) 自动运行（OpenClaw Cron，可选）

这个 skill 本身不会自动创建定时任务。要自动执行，请单独在 OpenClaw 里创建 Cron 作业，定时唤起 agent 使用此 skill。

推荐做法：

- 调度频率：每 10 分钟
- 时区：`Asia/Shanghai`
- 会话模式：`isolated`
- 投递模式：`--no-deliver`
  - 理由：真正的通知已经由 `notify_bridge.py` 发送，Cron 不需要再额外回传一份摘要

示例：

```bash
openclaw cron add \
  --name "Birthday Reminder" \
  --cron "*/10 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --no-deliver \
  --message "使用 birthday-reminder skill。读取 /绝对路径/birthdays.json 和 /绝对路径/notify.json，按 skill 说明执行一次生日提醒检查与发送；如果没有到期提醒则安静结束。"
```

手动排查时可用：

```bash
openclaw cron list
openclaw cron run <job-id> --force
openclaw cron runs --id <job-id> --limit 50
```

## 用户示例请求

- “添加一条农历八月初八的生日，并提前 15 天和当天上午 9 点提醒。”
- “把妻子生日设为阳历 3 月 25 日，默认时区北京，提前 7 天和 1 天提醒。”

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
python3 {baseDir}/scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthdays.json \
  --notify-config /绝对路径/notify.json \
  --now "2026-03-18 19:00:00"
```

## 常见问题

- 配置了但没收到提醒：
  - 先确认 OpenClaw Gateway 是否持续运行。
  - 再确认 Cron 作业是否真的执行到了这个 skill。
- `--now` 怎么写：
  - 推荐 `yyyy-MM-DD HH:mm:ss`，例如 `2026-03-25 09:00:00`。
- 只执行了 `check` 为什么没发消息：
  - `check` 只检查；真正发送需要执行 `notify_bridge.py`。
- OpenClaw Cron 会不会直接运行脚本：
  - 不会。Cron 调度的是 agent 任务；真正的脚本执行由 agent 按本 skill 的说明完成。

## 问题反馈

- 代码仓库：`https://github.com/905583906/jeff-skills`
- 提交问题（Issues）：`https://github.com/905583906/jeff-skills/issues`

遇到问题时，建议在 Issue 里附上：

- 你的配置片段（隐藏敏感信息）
- 执行命令
- 报错信息或截图

# birthday-reminder

一个仅面向 OpenClaw 的生日提醒 skill。它负责告诉 agent 如何读取生日配置、计算提醒时间，并调用本地 Python 脚本把到期提醒发到指定通道。

## 支持范围

- 支持：OpenClaw 手动调用
- 支持：OpenClaw Cron 周期执行
- 不承诺支持：Codex Automation
- 不承诺支持：Windows 任务计划、Linux cron、macOS launchd
- 不承诺支持：其他 Agent 平台的 skill/automation 机制

## 功能

- 支持阳历生日
- 支持农历生日
- 支持当天提醒、提前 N 天和多次提醒
- 支持每个人单独配置提醒时间、时区和偏移天数
- 支持多种通知通道

支持的通知通道：

- `console`
- `file`
- `webhook`
- `feishu`
- `dingtalk`
- `slack`
- `telegram`

## 前置条件

- 已安装 OpenClaw
- 当前环境可执行 `python3` 或 `python`
- 已准备 `birthdays.json`
- 已准备 `notify.json`

说明：

- 这个 skill 不会自动注册 Cron。
- 如果你需要自动提醒，请在 OpenClaw 中单独创建 Cron 作业。

## 安装

把当前目录复制到任一位置：

- `<workspace>/skills/birthday-reminder`
- `~/.openclaw/skills/birthday-reminder`

目录应包含：

```text
birthday-reminder/
  SKILL.md
  README.md
  assets/
  quickstart/
  references/
  scripts/
```

## 配置文件

最快开始方式：

- 直接编辑 [quickstart/birthdays.json](/D:/skills/jeff-skills/birthday-reminder/quickstart/birthdays.json)
- 直接编辑 [quickstart/notify.json](/D:/skills/jeff-skills/birthday-reminder/quickstart/notify.json)
- 对照说明看 [quickstart/README.md](/D:/skills/jeff-skills/birthday-reminder/quickstart/README.md)

这套 `quickstart/` 目录不需要复制或重命名文件，适合第一次安装时直接使用。

示例文件：

- [assets/birthdays.example.json](/D:/skills/jeff-skills/birthday-reminder/assets/birthdays.example.json)
- [assets/notify.example.json](/D:/skills/jeff-skills/birthday-reminder/assets/notify.example.json)

### birthdays.json

用途：

- 定义默认时区和默认提醒时间
- 定义每个人的生日信息
- 定义提前提醒的天数

最小示例：

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
    }
  ]
}
```

### notify.json

用途：

- 定义消息风格
- 定义通知通道与凭据

Telegram 示例：

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

## 手动验证

下面示例默认使用 `python3`。如果当前环境没有 `python3`，可改成 `python`。

### 1. 列出下一次会触发的提醒

```bash
python3 /绝对路径/birthday-reminder/scripts/birthday_reminder.py list --config /绝对路径/birthday-reminder/quickstart/birthdays.json
```

### 2. 检查当前窗口内是否有到期提醒

```bash
python3 /绝对路径/birthday-reminder/scripts/birthday_reminder.py check --config /绝对路径/birthday-reminder/quickstart/birthdays.json --output json
```

### 3. 预览发送内容

```bash
python3 /绝对路径/birthday-reminder/scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthday-reminder/quickstart/birthdays.json \
  --notify-config /绝对路径/birthday-reminder/quickstart/notify.json \
  --dry-run
```

### 4. 模拟某个具体时间

```bash
python3 /绝对路径/birthday-reminder/scripts/notify_bridge.py \
  --birthday-config /绝对路径/birthday-reminder/quickstart/birthdays.json \
  --notify-config /绝对路径/birthday-reminder/quickstart/notify.json \
  --now "2026-03-18 19:00:00"
```

## 在 OpenClaw 里自动执行

推荐每 10 分钟执行一次。

```bash
openclaw cron add \
  --name "Birthday Reminder" \
  --cron "*/10 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --no-deliver \
  --message "使用 birthday-reminder skill。读取 /绝对路径/birthday-reminder/quickstart/birthdays.json 和 /绝对路径/birthday-reminder/quickstart/notify.json，按 skill 说明执行一次生日提醒检查与发送；如果没有到期提醒则安静结束。"
```

推荐这样配置的原因：

- `notify_bridge.py` 已经会直接把提醒发到目标通道
- Cron 任务本身不需要再额外回传一份文本
- `isolated` 会话更适合后台周期任务

## 排查

常用命令：

```bash
openclaw cron list
openclaw cron run <job-id> --force
openclaw cron runs --id <job-id> --limit 50
```

排查顺序：

1. 先手动执行一次 `notify_bridge.py --dry-run`
2. 再用 `openclaw cron run <job-id> --force` 强制跑一次
3. 最后看 `openclaw cron runs --id <job-id> --limit 50` 的执行记录

如果没有收到提醒，通常检查这几项：

- OpenClaw Gateway 是否持续运行
- Python 是否能在当前环境里执行
- 配置文件路径是否正确
- 通知通道凭据是否有效
- 当前时间窗口内是否真的存在到期提醒
- `quickstart/notify.json` 里是否真的启用了至少一个真实通知通道

## 相关文件

- Agent 说明：[SKILL.md](/D:/skills/jeff-skills/birthday-reminder/SKILL.md)
- 配置参考：[references/config-schema.md](/D:/skills/jeff-skills/birthday-reminder/references/config-schema.md)
- 提醒计算脚本：[scripts/birthday_reminder.py](/D:/skills/jeff-skills/birthday-reminder/scripts/birthday_reminder.py)
- 通知发送脚本：[scripts/notify_bridge.py](/D:/skills/jeff-skills/birthday-reminder/scripts/notify_bridge.py)

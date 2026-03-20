# jeff-skills

仅支持 OpenClaw 的自定义 skills 仓库。当前包含一个可公开分发的生日提醒 skill。

## 仓库结构

```text
jeff-skills/
  README.md
  birthday-reminder/
    SKILL.md
    README.md
    assets/
    quickstart/
    references/
    scripts/
```

- 每个一级子目录都是一个独立 skill。
- 面向 agent 的说明放在 `SKILL.md`。
- 面向最终用户的安装与使用说明放在各 skill 目录下的 `README.md`。

## 当前技能

### birthday-reminder

功能：

- 管理阳历与农历生日
- 支持当天提醒、提前 N 天和多次提醒
- 支持 `console`、`file`、`webhook`、`feishu`、`dingtalk`、`slack`、`telegram`
- 可由 OpenClaw 手动调用，也可由 OpenClaw Cron 周期性唤起

支持范围：

- 支持：OpenClaw
- 不承诺支持：Codex Automation、系统级任务计划、其他 Agent 平台

文档入口：

- 用户说明：[birthday-reminder/README.md](/D:/skills/jeff-skills/birthday-reminder/README.md)
- Agent 说明：[birthday-reminder/SKILL.md](/D:/skills/jeff-skills/birthday-reminder/SKILL.md)

## 安装到 OpenClaw

有两种常见方式。

### 方式一：复制单个 skill

把 `birthday-reminder` 目录复制到：

- 工作区级：`<workspace>/skills/birthday-reminder`
- 全局级：`~/.openclaw/skills/birthday-reminder`

最终目录应类似：

```text
~/.openclaw/skills/
  birthday-reminder/
    SKILL.md
    README.md
    scripts/
```

### 方式二：把整个仓库作为 skills 包

如果你维护多个自定义 skill，可以把整个仓库保存在任意目录，然后把仓库根目录加入 OpenClaw 的 `skills.load.extraDirs`。

例如把仓库放在：

```text
D:\skills\jeff-skills
```

然后让 OpenClaw 额外扫描这个目录，把其中的一级子目录当作 skills。

## 快速开始

### 1. 准备配置文件

优先使用最小起步目录：

- [quickstart/birthdays.json](/D:/skills/jeff-skills/birthday-reminder/quickstart/birthdays.json)
- [quickstart/notify.json](/D:/skills/jeff-skills/birthday-reminder/quickstart/notify.json)
- [quickstart/README.md](/D:/skills/jeff-skills/birthday-reminder/quickstart/README.md)

这套起步目录不需要重命名文件，直接修改两个 JSON 即可。

`assets/` 下的文件仍保留为参考模板：

- [birthdays.example.json](/D:/skills/jeff-skills/birthday-reminder/assets/birthdays.example.json)
- [notify.example.json](/D:/skills/jeff-skills/birthday-reminder/assets/notify.example.json)

### 2. 手动验证脚本

下面示例默认使用 `python3`。如果当前环境没有 `python3`，可改成 `python`。

```bash
python3 /绝对路径/birthday-reminder/scripts/birthday_reminder.py list --config /绝对路径/birthday-reminder/quickstart/birthdays.json
python3 /绝对路径/birthday-reminder/scripts/birthday_reminder.py check --config /绝对路径/birthday-reminder/quickstart/birthdays.json --output json
python3 /绝对路径/birthday-reminder/scripts/notify_bridge.py --birthday-config /绝对路径/birthday-reminder/quickstart/birthdays.json --notify-config /绝对路径/birthday-reminder/quickstart/notify.json --dry-run
```

### 3. 创建 OpenClaw Cron

```bash
openclaw cron add \
  --name "Birthday Reminder" \
  --cron "*/10 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --no-deliver \
  --message "使用 birthday-reminder skill。读取 /绝对路径/birthday-reminder/quickstart/birthdays.json 和 /绝对路径/birthday-reminder/quickstart/notify.json，按 skill 说明执行一次生日提醒检查与发送；如果没有到期提醒则安静结束。"
```

说明：

- 这个 skill 本身不自动创建定时任务。
- 定时调度由 OpenClaw Cron 负责。
- 真正的提醒发送由 `notify_bridge.py` 完成。

## 排查命令

```bash
openclaw cron list
openclaw cron run <job-id> --force
openclaw cron runs --id <job-id> --limit 50
```

如果 Cron 没有生效，先确认：

- OpenClaw Gateway 正在持续运行
- Cron 功能已启用
- 作业时区与期望一致
- `quickstart/birthdays.json` 和 `quickstart/notify.json` 路径可被当前环境访问

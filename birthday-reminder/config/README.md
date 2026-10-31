# quickstart

这是 `birthday-reminder` 的最小起步目录。

目标：

- 不需要先复制或重命名示例文件
- 只需要修改两个 JSON
- 改完后就可以直接手动测试
- 再按同样路径创建 OpenClaw Cron

## 需要修改的文件

- [birthdays.json](birthday-reminder/quickstart/birthdays.json)
- [notify.json](birthday-reminder/quickstart/notify.json)

## 推荐步骤

1. 把 `birthdays.json` 里的示例联系人改成你自己的生日数据，支持农历（lunar）和阳历（solar）生日提醒
2. 在 `notify.json` 里至少启用一个真实通知通道
3. 先手动跑一次 `--dry-run`
4. 再创建 OpenClaw Cron

## 手动测试

```bash
python3 birthday-reminder/scripts/birthday_reminder.py list --config birthday-reminder/quickstart/birthdays.json
python3 birthday-reminder/scripts/birthday_reminder.py check --config birthday-reminder/quickstart/birthdays.json --now "2026-11-18 09:00:00" 
python3 birthday-reminder/scripts/notify_bridge.py --birthday-config birthday-reminder/quickstart/birthdays.json --notify-config birthday-reminder/quickstart/notify.json --window-minutes 180 --now "2026-11-18 09:00:00" --dry-run
```

## 创建 Cron

```bash
openclaw cron add \
  --name "Birthday Reminder" \
  --cron "*/10 * * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --no-deliver \
  --message "使用 birthday-reminder skill。读取 /绝对路径/birthday-reminder/quickstart/birthdays.json 和 /绝对路径/birthday-reminder/quickstart/notify.json，按 skill 说明执行一次生日提醒检查与发送；如果没有到期提醒则安静结束。"
```

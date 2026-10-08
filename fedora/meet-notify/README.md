# Meeting notifications with a Join button

**Problem:** KDE's calendar reminders don't show the Google Meet link. Google stores it in the event's description, which KDE leaves out of the reminder.

**Fix:** a small notifier pops up a **Join** button a minute before any meeting with a Google Meet, Zoom or Teams link. Clicking it opens the link.

## How it works

- Reads your calendars' private iCal addresses and re-downloads them every 10 minutes.
- Handles recurring meetings, including skipped and moved dates, and skips meetings you declined.
- After sleep, it still notifies up to 5 minutes after a meeting starts.
- Feeds are parsed in a short-lived child process, so the parser's memory is freed between refreshes.

KDE's own reminders keep working alongside it: they give the early heads-up, and this one gives the Join button.

## Requirements

```bash
sudo dnf install python3-icalendar python3-dateutil libnotify
```

## Install

```bash
install -Dm755 meet-notify ~/.local/bin/meet-notify
install -Dm644 meet-notify.service ~/.config/systemd/user/meet-notify.service
```

Add your calendar feeds, one address per line:

1. In Google Calendar, open Settings → (your calendar) → **Secret address in iCal format** and copy it.
2. Put it in `~/.config/meet-notify/urls` and make that file private:

```bash
mkdir -p ~/.config/meet-notify
nano ~/.config/meet-notify/urls
chmod 600 ~/.config/meet-notify/urls
```

> ⚠️ The secret address lets anyone read your calendar. Never commit or share it.

Then:

```bash
meet-notify --selftest     # checks the recurrence handling
meet-notify --list         # meetings with links in the next 24 hours
meet-notify --demo         # test notification
systemctl --user enable --now meet-notify
```

## Tuning

These constants are at the top of the script:

| Setting | Default | Meaning |
|---|---|---|
| `LEAD` | 1 minute | How early to notify |
| `GRACE` | 5 minutes | How late to still notify (after sleep) |
| `REFRESH` | 600 s | How often to re-download the feeds |

## Uninstall

```bash
systemctl --user disable --now meet-notify
rm -r ~/.local/bin/meet-notify ~/.config/systemd/user/meet-notify.service ~/.config/meet-notify ~/.cache/meet-notify
```

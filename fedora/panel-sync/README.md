# Same panel on every screen

KDE Plasma gives each screen its own panel and never keeps them in sync. When you connect a new monitor, it gets a fresh default panel, not a copy of yours. So a widget you added, an app you pinned or a setting you changed stays on the screen where you did it.

`sync-panels` copies the panel on your main screen to the panel on the same edge of every other screen: the widgets, their order and their settings.

## Use

```bash
./sync-panels --dry-run   # show what would change
./sync-panels
```

The main screen is the one marked primary in System Settings → Display & Monitor. To copy a different panel, pass its ID with `--from` (IDs are the `[Containments][N]` sections in the config file below, with `plugin=org.kde.panel`).

The script stops Plasma, edits `~/.config/plasma-org.kde.plasma.desktop-appletsrc`, and starts Plasma again, so your panels disappear for a few seconds. It has to: Plasma rewrites that file when it exits and would undo the change. Each run saves a timestamped backup next to the file.

Run it again whenever you change the main panel. Running it twice changes nothing.

## What it does and doesn't copy

- **Widgets** that both panels have keep their place and get the main panel's settings. Missing ones are added; extra ones are removed.
- **System tray:** each panel keeps its own. Its icons come from the running apps, so they're the same everywhere anyway.
- **Panel size, floating and auto-hide** aren't copied. Set those in each panel's Edit mode.
- **Accounts inside widgets are copied too.** For example, a Google sign-in in Event Calendar works on every screen without signing in again.
- **No repeated reminders:** [Event Calendar](../event-calendar/) would remind you of each meeting once per screen, so the copies get its reminders, invitation alerts and auto-join turned off. Only the main screen's widget alerts you. To quiet another widget like this, add it to `QUIET` at the top of the script.

## Undo

```bash
systemctl --user stop plasma-plasmashell
cp ~/.config/plasma-org.kde.plasma.desktop-appletsrc.bak-<time> ~/.config/plasma-org.kde.plasma.desktop-appletsrc
systemctl --user start plasma-plasmashell
```

Tested on Plasma 6.7.5, Fedora 44, with a laptop screen and two monitors.

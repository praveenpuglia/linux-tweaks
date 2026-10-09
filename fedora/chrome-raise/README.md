# Chrome comes to the front when you open a link

**Problem:** on KDE Plasma (Wayland), clicking a link in another app (Slack, a terminal, a mail client) opens it in Chrome, but Chrome stays behind. It only flashes in the taskbar, and you have to click it.

## Why

KWin's focus stealing prevention stops apps from jumping in front of the window you're using. An app is allowed to come forward only if it brings a valid activation token from the app you clicked in. Many apps don't pass one on when they open a link, so KWin treats Chrome's request as unwanted and only highlights it.

## Fix

A KWin window rule that turns focus stealing prevention off for Chrome only:

```bash
kwriteconfig6 --file kwinrulesrc --group chrome-raise --key Description "Chrome: come to the front when opening links"
kwriteconfig6 --file kwinrulesrc --group chrome-raise --key wmclass chrome
kwriteconfig6 --file kwinrulesrc --group chrome-raise --key wmclassmatch 2   # class contains "chrome"
kwriteconfig6 --file kwinrulesrc --group chrome-raise --key fsplevel 0       # prevention: None
kwriteconfig6 --file kwinrulesrc --group chrome-raise --key fsplevelrule 2   # Force

# list the rule, keeping any rules you already have
rules=$(kreadconfig6 --file kwinrulesrc --group General --key rules)
kwriteconfig6 --file kwinrulesrc --group General --key rules "${rules:+$rules,}chrome-raise"
kwriteconfig6 --file kwinrulesrc --group General --key count $(( $(kreadconfig6 --file kwinrulesrc --group General --key count --default 0) + 1 ))

busctl --user call org.kde.KWin /KWin org.kde.KWin reconfigure
```

**The list matters.** KWin only loads rules named under `[General] rules=`. A rule group that isn't listed there does nothing, with no error.

You can also make the same rule by hand: System Settings → Window Management → Window Rules → Add New. Match window class "chrome" (substring), then add "Focus stealing prevention", set to Force, None.

**Other browsers:** change `chrome` to a part of their window class, e.g. `firefox`. To find a window's class, open the Window Rules page, click "Detect Window Properties" and click the window.

## Check

Click into another window, then run:

```bash
xdg-open https://example.com
```

Chrome should come to the front with the new tab.

## Undo

Delete the rule in System Settings → Window Management → Window Rules.

# Per-display brightness keys

**Problem:** on Plasma 6.7, the brightness keys change every display at once. With a laptop panel and two external monitors, that's rarely what you want.

**Fix:** the brightness keys now change only the display you're working on: the one with the focused window, or the one under the mouse if you turn that option on.

Tested on Fedora 44, KDE Plasma 6.7 (Wayland), with a Dell Pro Max 16 Premium (OLED panel), a Dell U3225QE (DisplayPort) and an LG 32UD59 (HDMI).

## How it works

`brightness-focused up|down [percent]` (default step: 5%)

1. Asks KWin which display is active (`org.kde.KWin.activeOutputName`).
2. Finds that display among Plasma's brightness controls (`org.kde.ScreenBrightness`). The laptop panel is the one marked as internal. An external monitor is matched by the model name in its EDID, for example `DELL U3225QE`.
3. If Plasma controls that display, sets its brightness over D-Bus and shows Plasma's brightness popup on that screen only.
4. If Plasma can't control it (the LG on HDMI, for example), talks to the monitor directly over DDC/CI with `ddcutil`. A DDC read takes about a second, so the script remembers the last value for 60 seconds, and key presses are queued so fast repeats don't collide.

The `.desktop` files are hidden launchers that the shortcuts point to. They set `StartupNotify=false`: without it, Plasma treats every key press as an app launch and shows a bouncing "app is starting" cursor for several seconds.

## Requirements

```bash
sudo dnf install ddcutil   # only needed for monitors Plasma can't control
```

`ddcutil`'s udev rule gives the logged-in user access to `/dev/i2c-*`, so no group changes are needed. `busctl`, `kscreen-doctor` and `python3` are already on a Plasma install.

## Install

```bash
install -Dm755 brightness-focused ~/.local/bin/brightness-focused
for f in net.local.brightness-focused-{up,down}.desktop; do
  sed "s|@HOME@|$HOME|" "$f" > ~/.local/share/applications/"$f"
done
kbuildsycoca6
```

The launchers need the full path: the shortcut daemon runs inside KWin, and `~/.local/bin` isn't on its `PATH`.

Then bind the keys in **System Settings → Keyboard → Shortcuts**:

1. **Free the keys:** under **Power Management**, clear "Increase Screen Brightness" and "Decrease Screen Brightness".
2. **Bind the launchers:** assign the Monitor Brightness Up and Down keys to "Brightness up (focused display)" and "Brightness down (focused display)".
   - If those two entries aren't listed, use **Add New → Command or Script** with `~/.local/bin/brightness-focused up` (and `down`).
   - Then add `StartupNotify=false` to the `.desktop` files Plasma generates for them.

The result in `~/.config/kglobalshortcutsrc`:

```ini
[services][net.local.brightness-focused-down.desktop]
_launch=Monitor Brightness Down

[services][net.local.brightness-focused-up.desktop]
_launch=Monitor Brightness Up

[org_kde_powerdevil]
Decrease Screen Brightness=none,Monitor Brightness Down,Decrease Screen Brightness
Increase Screen Brightness=none,Monitor Brightness Up,Increase Screen Brightness
```

## Usage notes

- **Follow the mouse instead:** to have the active display follow the mouse pointer, turn on System Settings → Window Management → Window Behavior → "Active screen follows mouse".
- **Test a specific display:** `BRIGHTNESS_OUTPUT=HDMI-A-1 brightness-focused up` acts as if that display were active. `kscreen-doctor -o` lists the display names.
- **Shift + brightness keys** (1% steps) aren't rebound.

## Gotchas

- **Shortcuts registered over D-Bus:** set the `SetPresent` flag (2). Without it, real key presses are ignored.
- **The popup can stick to the wrong screen:** while Plasma's brightness popup is open, it adds every display it's told about to that popup. The script closes the popup first when you move to a different display.
- **Duplicate popup:** PowerDevil shows its own popup, which can land on the wrong screen. The script passes flag `1` to `SetBrightness` to suppress it and shows its own instead.

## Uninstall

Reassign the keys under System Settings → Keyboard → Shortcuts → Power Management, then:

```bash
rm ~/.local/bin/brightness-focused ~/.local/share/applications/net.local.brightness-focused-{up,down}.desktop
kbuildsycoca6
```

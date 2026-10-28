# Bitwarden Flatpak: tray icon that responds to clicks

**Problem:** on KDE Plasma (Wayland), the Bitwarden Flatpak's tray icon does nothing when clicked. System Tray Settings lists it as `bitwarden-app`.

## Why

Electron apps put their tray icon on D-Bus under a name like `org.freedesktop.StatusNotifierItem-<pid>-1`. The Flatpak is allowed to talk to the tray, but not to own that name. So Electron falls back to an old X11 tray icon, which Plasma shows through `xembedsniproxy`, and clicks on it don't reach Bitwarden.

Inside its sandbox Bitwarden is always PID 2, so the name is always `org.freedesktop.StatusNotifierItem-2-1`.

## Fix

```bash
flatpak override --user --own-name=org.freedesktop.StatusNotifierItem-2-1 com.bitwarden.desktop
flatpak kill com.bitwarden.desktop
```

Then start Bitwarden again. A plain restart from inside the app isn't enough; the sandbox has to restart to get the new permission. The override is in your user settings, so Flatpak updates keep it.

The same fix works for other Electron Flatpaks with a dead tray icon: check the app's PID inside its sandbox with `grep NSpid /proc/<pid>/status` (the second number).

## Check

```bash
busctl --user get-property org.kde.StatusNotifierWatcher /StatusNotifierWatcher \
    org.kde.StatusNotifierWatcher RegisteredStatusNotifierItems
```

It should list `org.freedesktop.StatusNotifierItem-2-1/StatusNotifierItem`. Before the fix, Bitwarden showed up as a `:1.xxx` name owned by `xembedsniproxy`.

## Undo

```bash
flatpak override --user --reset com.bitwarden.desktop
```

This removes all your user overrides for Bitwarden, not only this one.

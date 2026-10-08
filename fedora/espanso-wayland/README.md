# Espanso on Fedora KDE (Wayland)

A text expander: type a short trigger anywhere and it turns into longer text. For example, `imeet` becomes your Google Meet link. Fedora doesn't package Espanso, so it comes from the **Terra** repo, and getting it to work on KDE Wayland takes a few fixes.

## 1. Only take Espanso from Terra

Terra carries thousands of packages, and some replace Fedora or RPM Fusion ones. Without a limit, a routine update once swapped out a working webcam stack for Terra's builds.

So limit Terra to Espanso **before** adding the repo:

```bash
sudo install -Dm644 99-terra-espanso-only.repo /etc/dnf/repos.override.d/99-terra-espanso-only.repo
sudo dnf install --nogpgcheck --repofrompath 'terra,https://repos.fyralabs.com/terra$releasever' terra-release
sudo dnf install espanso-wayland
```

Keep the limit in `repos.override.d`, not in `/etc/yum.repos.d/terra.repo`. Updating or reinstalling `terra-release` overwrites `terra.repo` and silently removes the limit. The override file is yours, so updates never touch it.

When dnf asks to import Terra's signing key, check the fingerprint against the one Terra publishes.

## 2. Let it read and type on Wayland

```bash
sudo setcap cap_dac_override+p /usr/bin/espanso
espanso service register
espanso start
```

This lets Espanso read the keyboard (`/dev/input`) and type the replacement (`/dev/uinput`). **A package update can remove this capability.** If expansions stop working after an update, run `setcap` again.

## 3. Config

Copy the two files, then edit `base.yml` to add your own snippets:

```bash
install -Dm644 default.yml ~/.config/espanso/config/default.yml
install -Dm644 base.yml ~/.config/espanso/match/base.yml
```

- [`default.yml`](default.yml) sets the keyboard layout to `us`. Auto-detection fails on KDE Wayland, and expansions come out garbled or don't fire.
- [`default.yml`](default.yml) also turns off Espanso's start-up pop-ups.
- [`base.yml`](base.yml) holds the snippets.

## 4. Keyboards that connect later

Espanso only listens to keyboards that are connected when it starts. If you connect a Bluetooth keyboard afterwards, or it reconnects after sleep, Espanso doesn't see it. To fix that, a udev rule restarts Espanso whenever a keyboard appears:

```bash
sudo install -Dm644 90-espanso-keyboard-hotplug.rules /etc/udev/rules.d/90-espanso-keyboard-hotplug.rules
install -Dm644 espanso-rescan.service ~/.config/systemd/user/espanso-rescan.service
sudo udevadm control --reload
```

The rule ignores Espanso's own virtual keyboard, so it can't trigger itself.

## Not covered

App-specific snippets don't work on KDE Wayland without `kdotool`. Plain snippets work everywhere.

## Uninstall

```bash
espanso service unregister
sudo dnf remove espanso-wayland terra-release terra-gpg-keys
sudo rm /etc/dnf/repos.override.d/99-terra-espanso-only.repo /etc/udev/rules.d/90-espanso-keyboard-hotplug.rules
rm ~/.config/systemd/user/espanso-rescan.service
```

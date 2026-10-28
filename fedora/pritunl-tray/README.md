# Pritunl: dark tray icon and compact profiles

**Problem:** on a light KDE Plasma panel the Pritunl client's tray icon is white and hard to see. And its profile list stays expanded, with no way to collapse it.

## Why

- **Icon:** Pritunl ships light and dark tray icons for Linux (`tray_*_linux_light.png` and `tray_*_linux_dark.png`), but its code always picks the light (white) ones, whatever your theme.
- **Profiles:** the compact list only turns on with **more than 3** profiles and a window narrower than 864 px. With 3 or fewer there's no way to get it.

## Fix

```bash
./pritunl-fix
```

It copies Pritunl's own dark icons over the white ones and changes the profile rule to "more than 0". Then quit Pritunl from its tray menu and open it again. The VPN service keeps running separately, so restarting the window doesn't drop a connection.

Disconnected shows an outline "p", connected a filled one. Widen the window past 864 px and the full profile cards come back.

Tested with pritunl-client-electron 1.4.4791.77 on Fedora 44.

## Package updates undo it

Both changes edit files owned by the RPM, so every Pritunl update puts the originals back. Run `./pritunl-fix` again after updating. If the profile part fails, Pritunl changed that code; the icon part still works.

## Undo

```bash
sudo dnf reinstall pritunl-client-electron
```

# linux-tweaks

Fixes, workarounds and small tools from setting up Linux desktops, each written up so it can be done again.

## Fedora

| Tweak | What it fixes |
|---|---|
| [Intel IPU6 webcam (Dell)](fedora/ipu6-camera/) | MIPI webcam through Intel's camera HAL with the privacy LED working, plus a check script that tells you which fixes your machine needs |
| [Per-display brightness keys](fedora/per-display-brightness/) | KDE Plasma's brightness keys change every display at once; this changes only the active one, including monitors Plasma can't control (over DDC/CI) |
| [Swap that slows down instead of killing apps](fedora/swap-zram-oom/) | Out-of-memory kills under heavy dev load: bigger zram, a btrfs swap file behind it, and swappiness that survives the Performance profile |
| [Haruna HEVC](fedora/haruna-hevc/) | "missing video track" on phone videos, and why `libavcodec-freeworld` alone doesn't fix it |
| [Espanso on Wayland](fedora/espanso-wayland/) | A text expander from Terra without letting Terra replace other packages, plus Wayland and keyboard-hotplug fixes |
| [Meeting notifications](fedora/meet-notify/) | A Join button before Google Meet, Zoom and Teams meetings, which KDE's reminders don't show |
| [Chrome to the front on links](fedora/chrome-raise/) | Links opened from other apps load in Chrome, but Chrome only flashes in the taskbar instead of coming forward |

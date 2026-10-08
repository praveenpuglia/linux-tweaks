# Haruna: "missing video track" on HEVC videos

**Problem:** Haruna plays the audio of phone videos (HEVC/H.265) but shows "missing video track". Installing `libavcodec-freeworld`, the usual advice, doesn't help.

## Why

Fedora ships a reduced ffmpeg (`ffmpeg-free`) that can't decode HEVC. `libavcodec-freeworld` from RPM Fusion adds a full decoder in a separate folder, `/usr/lib64/ffmpeg/`, and most apps find it there.

Haruna doesn't. Its program file tells it to search `/usr/lib64` first, so it always loads Fedora's reduced library and never reaches the full one.

## Fix

Replace the reduced ffmpeg with RPM Fusion's full one. While you're at it, swap in Intel's full video driver so decoding runs on the GPU. Both need [RPM Fusion](https://rpmfusion.org/Configuration) (free and nonfree) enabled.

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
sudo dnf remove libavcodec-freeworld        # no longer needed
sudo dnf swap libva-intel-media-driver intel-media-driver --allowerasing
```

Then, in Haruna, go to Settings → Playback and set hardware decoding to `auto`.

This is the same ffmpeg swap that most "things to do after installing Fedora" guides include. If you've already done it, Haruna works from the start.

## Check

```bash
vainfo 2>/dev/null | grep -i hevc                       # the GPU can decode HEVC
grep -o '/usr/lib64[^ ]*libavcodec[^ ]*' /proc/$(pgrep -f haruna | head -1)/maps | sort -u
```

Check `/proc/<pid>/maps` to see which libraries an app *actually* loaded. `ldconfig` and `ldd` don't show what the app really picked.

**Result:** CPU use for a 4K HEVC 10-bit clip dropped from about 2.4 cores to about 17% of one core.

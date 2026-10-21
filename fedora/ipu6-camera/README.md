# Intel IPU6 webcam on Fedora (Dell laptops)

Recent Dell laptops with Intel Core Ultra chips use a **MIPI camera**: a bare image sensor wired to the Intel camera controller (IPU6) inside the CPU, instead of a self-contained USB webcam. Out of the box on Fedora, it either doesn't show up, or it shows up through libcamera with a poor, grainy image.

This guide sets it up through **Intel's camera HAL**, the same stack Dell ships on Ubuntu, with Intel's image tuning, at 2560×1440 / 30 fps, with the privacy LED working.

**Tested on:** Dell Pro Max 16 Premium (MA16250), Fedora 44 KDE, kernel 7.2.11, `ipu6-camera-hal` 0.0-30, Secure Boot on. Other Dell models with an IPU6 camera need the same core steps. The check script below tells you which of the extra fixes your machine needs.

## How the camera works

```
sensor ──MIPI──▶ [bridge chip] ──▶ IPU6 in the CPU ──▶ Intel HAL ──▶ v4l2-relayd ──▶ /dev/videoN ──▶ apps
(ov08x40)        (Intel CVS,        (kernel: isys        (image        (copies frames    ("Intel MIPI
                  optional)          + psys)              processing)   into a virtual    Camera")
                                                                        webcam)
```

- **Sensor:** turns light into raw numbers.
- **IPU6:** receives them (`isys`) and processes them (`psys`).
- **Intel HAL:** turns raw data into a good image, using Intel's tuning file for that sensor.
- **Apps:** most apps only speak the standard webcam API (V4L2), not the HAL's. So `v4l2-relayd` runs the HAL and feeds a virtual webcam (`v4l2loopback`) that every app can open. The relay only starts the camera while an app is using it.

## 0. Check your machine

```bash
./ipu6-camera-check
```

It only reads, never changes anything. Run it again after the setup to confirm everything is in place.

| The check says | What to do |
|---|---|
| IPU `8086:7d19` / `a75d` / `465d` / `9a19`… | Supported. Note the **HAL config folder** (e.g. `ipu6epmtl`). |
| IPU7 (Lunar Lake, Panther Lake) | Not covered here. |
| **Kernel sensor name**, e.g. `ov08x40 18-0036` | The kernel found the sensor. |
| No HAL sensor file for your sensor | The Intel HAL can't drive it. Use libcamera instead. |
| `Intel CVS` in the path | Needs `ipu6-camera-hal` from Sep 25, 2026 or later (RPM Fusion 0.0-30+). The check flags an older one. |
| USB-IO bridge `06cb:0701` | You need step 6 for the privacy LED. |

## 1. RPM Fusion and a module signing key

The IPU6 processing driver and the LED fix are kernel modules that aren't in Fedora's kernel. With Secure Boot on, they must be signed with a key your firmware trusts.

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install akmods
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

Reboot, and enroll the key in the blue MOK screen that appears.

## 2. Camera packages

```bash
sudo dnf install ipu6-camera-bins ipu6-camera-hal gstreamer1-plugins-icamerasrc \
                 v4l2-relayd v4l2loopback akmod-intel-ipu6
sudo dnf remove pipewire-plugin-libcamera
```

- **`akmod-intel-ipu6`:** you need it only for the `psys` driver. The rest is already in kernel 7.2.
- **Don't install `intel-vision`:** on kernel 7.2 its driver is in the kernel.
- **Removing libcamera's PipeWire plugin:** this stops apps from also offering a second, worse "camera" that goes through libcamera.

## 3. HAL config overlay for 2560×1440 (optional)

**Why:** the HAL only offers the sizes listed in the sensor's config file. For the ov08x40 that's 1280×720, 1920×1080 and 2560×1920 (4:3), with no 16:9 size above 1080p. If 1920×1080 is enough, skip this step and set that size in step 4.

**Fix:** a copy of the HAL's config folder made of symlinks, with one sensor file that adds 2560×1440 and 3840×2160. The package's own files stay untouched.

```bash
HAL=ipu6epmtl                 # HAL config folder from the check

SRC=/usr/share/defaults/etc/camera/$HAL; D=/etc/camera/$HAL
sudo mkdir -p $D/sensors
for f in $SRC/*; do [ "$(basename $f)" = sensors ] || sudo ln -sfn $f $D/; done
for f in $SRC/sensors/*; do [ "$(basename $f)" = ov08x40-uf.xml ] || sudo ln -sfn $f $D/sensors/; done
sed '/V4L2_PIX_FMT_NV12,1920x1080,0,0,/a\                                          V4L2_PIX_FMT_NV12,2560x1440,0,0,\n                                          V4L2_PIX_FMT_NV12,3840x2160,0,0,' \
  $SRC/sensors/ov08x40-uf.xml | sudo tee $D/sensors/ov08x40-uf.xml >/dev/null
```

**After a HAL update,** the copied sensor file doesn't follow it. Re-run the block above; it's safe to repeat. The check script tells you when the copy is out of date.

**Different sensor?** Use your sensor's `*-uf.xml` in place of `ov08x40-uf.xml`.

**Older HAL with `Intel CVS` in the path?** HAL snapshots before Sep 25, 2026 can't handle the Intel CVS chip, so update `ipu6-camera-hal` first. The old workaround is in [this folder's history](https://github.com/praveenpuglia/linux-tweaks/tree/c84bdd0/fedora/ipu6-camera).

## 4. Relay settings

Put the settings in your own file, [`v4l2-relayd-icamerasrc.env`](v4l2-relayd-icamerasrc.env), and load it through a systemd drop-in:

```bash
sudo install -m644 v4l2-relayd-icamerasrc.env /etc/v4l2-relayd-icamerasrc.env
sudo install -Dm644 50-local-settings.conf \
  /etc/systemd/system/v4l2-relayd@icamerasrc.service.d/50-local-settings.conf
```

- **Size and frame rate:** the file sets 2560×1440 at 30 fps, which needs step 3. Apps only get this one size. 3840×2160 also works with step 3. Without it, use 1920×1080.
- **Overlay path:** `CAMERA_CFG_PATH` points the HAL at the overlay from step 3. Change `ipu6epmtl` there if your HAL folder is different. If you skipped step 3, delete that line.
- **Don't edit `/etc/v4l2-relayd.d/icamerasrc.conf`:** it's a package-owned link. At every boot it's pointed back at the HAL's defaults (1280×720, no overlay), and package updates restore it. Your own file, loaded last, always wins.

Check that your file is listed last:

```bash
systemctl show v4l2-relayd@icamerasrc -p EnvironmentFiles
```

## 5. Make the relay wait for the processing device

**Why:** on some boots the relay started before the `psys` device existed. The relay's sandbox only allows that device, and systemd resolves the rule once, at start. If the device isn't there yet, the rule matches nothing, and the HAL fails with `Failed to open PSYS, error: Operation not permitted` until the next reboot.

```bash
sudo install -m644 70-ipu6-psys-systemd.rules /etc/udev/rules.d/
sudo install -Dm644 wait-psys.conf /etc/systemd/system/v4l2-relayd@icamerasrc.service.d/wait-psys.conf
```

## 6. Privacy LED (only for USB-IO bridge `06cb:0701`)

**Why:** for this Synaptics bridge, the kernel's `intel_cvs` driver tells the CVS chip "the computer drives the LED", but nothing on the computer does. So the LED stays off while the camera is on. Intel's own driver (used on Ubuntu) says the opposite, which is why the LED works there. As of October 2026 this isn't fixed in the kernel.

**Fix:** [`intel-cvs-led-build`](intel-cvs-led-build) does all of this:

- downloads the `intel_cvs` source that matches your kernel;
- removes that one flag;
- builds and signs the module with the akmods key;
- installs it to `/lib/modules/<kernel>/updates/`.

A kernel-install hook re-runs it for every new kernel. If a future kernel fixes the flag itself, the script removes its own module.

```bash
sudo dnf install kernel-devel-$(uname -r) gcc make
TAG=v$(uname -r | cut -d- -f1); TAG=${TAG%.0}
sudo mkdir -p /usr/local/src/intel-cvs-led          # offline fallback copy of the source
for f in core.c icvs.h v4l2.c; do
  sudo curl -sfL -o /usr/local/src/intel-cvs-led/$f \
    "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/media/i2c/cvs/$f?h=$TAG"
done
sudo install -m755 intel-cvs-led-build /usr/local/sbin/
sudo install -m755 95-intel-cvs-led.install /etc/kernel/install.d/
sudo /usr/local/sbin/intel-cvs-led-build
```

## 7. Reboot

Reboot instead of loading or unloading the camera modules by hand. ⚠️ Removing `v4l2loopback` or `intel_ipu6_psys` while the camera stack was running caused a kernel panic.

After the reboot:

- **Check:** run `./ipu6-camera-check`. Everything should be ✓, and the LED quirks should read `0x3a` (stock is `0x7a`).
- **Try it:** open "Intel MIPI Camera" in a browser or Kamoso. The first frame takes 2–3 seconds while the HAL starts, and the LED comes on.

## What can break it later

- **Another repo replacing the camera packages.** One repo, Terra, shipped its own builds of these packages with a higher version number. A routine `dnf upgrade` swapped them in, and the camera broke. Limit any third-party repo to the packages you want from it; see [Espanso on Wayland](../espanso-wayland/) for how. The check script shows which repo each package came from.
  - **Going back:** don't swap modules on a running system. Stage the change and apply it during a reboot:

    ```bash
    sudo dnf do --offline --allowerasing --action=downgrade <pkg-version>… --action=remove <leftovers>
    sudo dnf offline reboot
    ```
- **The akmod rebuilding during boot.** When `akmod-intel-ipu6` updates, the module is rebuilt early in the next boot, sometimes too late for that boot. If `/dev/ipu-psys0` is missing, reboot once more rather than running `modprobe`.
- **A new kernel without `kernel-devel`.** The LED module can't be built for it. The stock module loads instead, so the camera still works but the LED stays off. Install `kernel-devel` for that kernel and run `sudo intel-cvs-led-build <kernel version>`.

## Undo

```bash
sudo rm -r /etc/camera /etc/v4l2-relayd-icamerasrc.env \
  /etc/systemd/system/v4l2-relayd@icamerasrc.service.d \
  /etc/udev/rules.d/70-ipu6-psys-systemd.rules \
  /usr/local/sbin/intel-cvs-led-build /etc/kernel/install.d/95-intel-cvs-led.install \
  /lib/modules/*/updates/intel_cvs.ko /usr/local/src/intel-cvs-led
sudo depmod -a
sudo dnf remove ipu6-camera-bins ipu6-camera-hal gstreamer1-plugins-icamerasrc v4l2-relayd v4l2loopback akmod-intel-ipu6
```

Then reboot.

# Swap that slows down instead of killing apps

**Problem:** on a 32 GB laptop, the kernel kept killing Chrome during heavy dev work: Chrome, VS Code, a Next.js dev server, Gradle, an Android emulator and `tsc` all running at once. macOS and Ubuntu would just slow down at that point.

## Why it broke

Fedora's default swap is **zram only**: an 8 GB compressed area that lives **in RAM**, with no swap on disk.

- **Nowhere to overflow:** zram stretches RAM a bit, but it is still RAM. Once memory and zram filled up, there was no disk to spill onto, so the kernel's out-of-memory killer stepped in.
- **Chrome died first:** Chrome marks its tabs as the first thing to kill, so Chrome was always the victim. That held even when another program caused the shortage; once it was `tsc`.
- **The Performance profile made it worse:** the Performance power profile applies tuned's `throughput-performance` profile, which sets `vm.swappiness = 10`. That value suits slow disk swap. With zram, it made the kernel avoid swapping until far too late.

## Fix

Three layers:

1. **Bigger, better zram** (still the first, fast layer)
2. **A disk swap file** behind it as overflow (slow, but nothing gets killed)
3. **Swappiness kept high**, whatever the power profile

### 1. zram: half of RAM, zstd

[`zram-generator.conf`](zram-generator.conf) → `/etc/systemd/zram-generator.conf`

zstd packs more into the same space than Fedora's default, lzo-rle. This takes effect at the next boot.

### 2. 16 GB swap file on btrfs

```bash
sudo btrfs subvolume create /var/swap
sudo btrfs filesystem mkswapfile --size 16g --uuid clear /var/swap/swapfile
sudo semanage fcontext -a -t swapfile_t '/var/swap(/.*)?'
sudo restorecon -RF /var/swap
sudo swapon --priority 10 /var/swap/swapfile
echo '/var/swap/swapfile none swap defaults,pri=10 0 0' | sudo tee -a /etc/fstab
```

- **Own subvolume:** the file sits in a subvolume of its own so btrfs snapshots skip it.
- **SELinux label:** without `swapfile_t`, SELinux blocks swapping to the file.
- **Priority:** the file has priority 10 and zram has 100, so the kernel fills zram first and only then uses the disk.
- **Encryption:** if the disk uses LUKS, the swap file is encrypted with it.

### 3. Swappiness 150

[`99-zram-swappiness.conf`](99-zram-swappiness.conf) → `/etc/sysctl.d/`. Swapping to zram is cheap, so let the kernel use it early.

The Performance profile still overrides this, so add a copy of that profile that keeps swappiness at 150:

```bash
sudo install -Dm644 tuned/tuned.conf /etc/tuned/profiles/throughput-performance-zram/tuned.conf
```

Then, in `/etc/tuned/ppd.conf`, point the Performance slider at it:

```ini
[profiles]
performance=throughput-performance-zram
```

```bash
sudo systemctl restart tuned tuned-ppd
```

## Check

```bash
swapon --show                 # zram0 at priority 100, the swap file at 10
cat /proc/sys/vm/swappiness   # 150, in every power profile
```

## Undo

1. Turn off the swap file: `sudo swapoff /var/swap/swapfile`
2. Remove the swap file's line from `/etc/fstab`.
3. Delete the subvolume: `sudo btrfs subvolume delete /var/swap`
4. Remove `zram-generator.conf`, `99-zram-swappiness.conf` and the tuned profile.
5. Reset the `performance=` line in `ppd.conf` back to `throughput-performance`.

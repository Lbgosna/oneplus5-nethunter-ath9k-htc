# OnePlus 5/5T NetHunter ath9k_htc kernel

Custom Kali NetHunter kernel for OnePlus 5 / 5T with working Alfa AWUS036NHA support.

The short version: the official `oneplus5-los` NetHunter kernel already had the usual NetHunter stuff like HID, injection, RTL88XXAU and RTL8187, but it did not have Atheros enabled. That means Alfa AWUS036NHA / AR9271 did not work because `ath9k_htc` was missing from the kernel.

I built this because I am broke and don't have money to buy another wifi adapter.

---

## Device / ROM

| Item | Value |
|---|---|
| Device | OnePlus 5 / OnePlus 5T |
| Codenames | `cheeseburger`, `dumpling` |
| ROM | LineageOS 22.2 / Android 15 |
| NetHunter target | `oneplus5-los-fifteen` |
| Kernel base | Linux 4.4.302 |
| Root setup tested | Magisk + Kali NetHunter full |
| Adapter tested | Alfa AWUS036NHA |
| Chipset | Atheros AR9271 |
| Driver | `ath9k_htc` |

This is not official LineageOS, Kali, NetHunter, OnePlus, or anything else. It is just a working build from my device.

---

## What this adds

Built into the kernel:

```text
CONFIG_ATH_CARDS=y
CONFIG_ATH_COMMON=y
CONFIG_ATH9K_HW=y
CONFIG_ATH9K_COMMON=y
CONFIG_ATH9K_HTC=y
```

Already present in the base kernel:

```text
CONFIG_CFG80211=y
CONFIG_MAC80211=y
CONFIG_FW_LOADER=y
CONFIG_USB=y
CONFIG_MODULES=y
CONFIG_MODVERSIONS=y
```

I made `ath9k_htc` built-in instead of a `.ko` module because Android module paths are annoying here and the current NetHunter install did not have a clean `/lib/modules`, `/system/lib/modules`, or `/vendor/lib/modules` tree.

Firmware is provided by NetHunter:

```text
/system/etc/firmware/htc_9271.fw
/system/etc/firmware/ath9k_htc/htc_9271-1.4.0.fw
```

---

## Tested result

This is the important part. The adapter actually works.

Expected `dmesg`:

```text
ath9k_htc: Firmware ath9k_htc/htc_9271-1.4.0.fw requested
ath9k_htc: Transferred FW: ath9k_htc/htc_9271-1.4.0.fw, size: 51008
ath9k_htc: FW Version: 1.4
ieee80211 phy1: Atheros AR9271 Rev:1
```

Expected `iw dev`:

```text
phy#1
    Interface wlan2
        addr 00:c0:ca:...
        type managed
```

Monitor mode works:

```bash
ip link set wlan2 down
iw dev wlan2 set type monitor
ip link set wlan2 up
iw dev
```

Expected:

```text
Interface wlan2
type monitor
channel 1 (2412 MHz), width: 20 MHz
```

Scan works:

```bash
iw dev wlan2 scan | head
```

Expected:

```text
BSS ... on wlan2
```

---

## Files

Release assets:

| File | What it is |
|---|---|
| `boot-ath9k_htc.img` | Fastboot-ready boot image made from my current NetHunter boot image with only kernel/kernel_dtb replaced |
| `zero-0.6.4-dumplinger-los-4.4-ath9k_htc.zip` | AnyKernel3 flashable zip |
| `zero-0.6.4-dumplinger-los-4.4-ath9k_htc.config` | Kernel config used for the build |
| `zero-kernel-v0.6.4-ath9k_htc-build.diff` | My small builder/config patch |
| `SHA256SUMS.txt` | Checksums |

Checksums:

```text
d14762663f0f6f0c61e8f6cf9df00426e3412ed6a84442e46dd68c48a4303643  boot-ath9k_htc.img
21e1b86da7a3c3ce3f9ac8d2d17d34bdf7a2f3c32a72b0e8b07f1b6261fe202e  zero-0.6.4-dumplinger-los-4.4-ath9k_htc.zip
2d8e7b1493d97e2975469f262d37c3d786885e2542b8d409b22631a553b36921  zero-0.6.4-dumplinger-los-4.4-ath9k_htc.config
dab5eda7fadd3bdd7c69b0edc698a767ddf4df1f04d8e857e73fdf8b48fe5d94  zero-kernel-v0.6.4-ath9k_htc-build.diff
```

---

## Flashing

First make your own boot backup. Seriously.

```bash
adb shell su -c 'dd if=/dev/block/by-name/boot of=/sdcard/current-boot-before-ath9k.img bs=4M'
adb pull /sdcard/current-boot-before-ath9k.img .
```

Safer temporary boot:

```bash
adb reboot bootloader
fastboot boot boot-ath9k_htc.img
```

Permanent flash:

```bash
adb reboot bootloader
fastboot flash boot boot-ath9k_htc.img
fastboot reboot
```

Rollback:

```bash
adb reboot bootloader
fastboot flash boot current-boot-before-ath9k.img
fastboot reboot
```

Do not wipe data. This only touches the boot partition.

---

## Verify after boot

```bash
su
uname -a
zcat /proc/config.gz | grep -E "ATH9K|ATH_CARDS"
dmesg | grep -iE "ath9k|htc_9271|ar9271|firmware"
iw dev
ip link
```

`lsusb: unable to initialize usb spec` in Android terminal was not fatal in my test. `dmesg`, `iw dev`, and `ip link` are the useful checks.

---

## Source / GPL sanity

Kernel source base:

```text
https://github.com/LineageOS/android_kernel_oneplus_msm8998
branch: lineage-21
commit: 5d01a147
```

Builder:

```text
https://github.com/seppzer0/zero-kernel
tag: v0.6.4
```

My changes are in:

```text
build-files/zero-kernel-v0.6.4-ath9k_htc-build.diff
build-files/zero-0.6.4-dumplinger-los-4.4-ath9k_htc.config
```

The Linux kernel is GPL. If you mirror or modify this, keep the source/config/diff available. Do not just reupload a mystery `boot.img`.

---

## Build notes

Original zero-kernel command:

```bash
uv run zkb kernel --build-env=local --base=los --codename=cheeseburger --lkv=4.4
```

Wrapper patch was needed because:

- `zero-kernel v0.6.4` assumes Debian/apt in local mode
- the `kernel` command hit an argparse issue around `chroot`
- upstream cleaner uses `git reset --hard`, which I did not want near my working files
- path quoting was needed in a few places

---

## Notes

This repo is a public archive, not a universal copy-paste guide. It worked on my phone with my ROM/kernel/NetHunter state.

If it helps you, nice. If it bricks your boot partition, restore your backup.

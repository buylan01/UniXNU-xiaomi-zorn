# Redmi K80 (zorn) porting notes

Status: research / bring-up planning. This is **not** a bootable device configuration.

## Target

- Device: Redmi K80
- Android codename: `zorn`
- SoC: Qualcomm Snapdragon 8 Gen 3 / SM8650 family (verify exact board SKU from the device)
- Current baseline: HyperOS, bootloader unlocked
- Host available: Ubuntu 26.04; no local macOS/Xcode host

## Important constraints

1. Do not copy `UniXNU-sheng`'s `board.toml`, defconfig, kernelcache hashes, prelink addresses, DTB fixups, UEFI config, or boot arguments as if they applied to zorn. Sheng is Xiaomi Pad 6S Pro / SM8550 and has a device-specific boot chain and drivers.
2. The generic UniXNU repository is a build framework, not a Redmi K80 port. A real port needs a separate device tree/repository with `board.toml`, `configs/zorn_defconfig`, platform-specific scripts/drivers, and a compatible Mu-Silicium UEFI platform.
3. The existing macOS GitHub Actions workflow deliberately uploads text logs only. It does not provide a downloadable kernel, payload, disk image, kernelcache, or IPSW members. A successful CI run alone therefore cannot produce a flashable image for this device.
4. Do not flash or erase partitions during early bring-up. Keep a verified stock recovery path and backups before any device-side experiment.

## Bring-up sequence

### Phase 0 — collect facts (no writes to device)
- Record exact model/SKU, HyperOS build, Android version, bootloader state, and current partition layout.
- Save the stock boot-chain images and device tree from the exact installed build where legally/technically available.
- Identify the active boot image format, AVB/vbmeta chain, ABL behavior, DTB/DTBO arrangement, and whether the bootloader can launch a test UEFI image without replacing critical partitions.
- Obtain the matching Xiaomi open-source kernel/device-tree release and confirm its branch corresponds to this exact zorn build.

### Phase 1 — UEFI feasibility
- Establish a reproducible UEFI build for SM8650/zorn based on the Qualcomm platform support in Mu-Silicium.
- Validate serial/USB diagnostics and memory map before attempting any XNU kernel handoff.
- Keep the stock boot path untouched; use only a documented, reversible test method.

### Phase 2 — XNU kernel bring-up
- Add a dedicated device repository and defconfig only after the required board values and source inputs are known.
- Confirm the XNU target configuration, matching IPSW/kernelcache compatibility, prelink metadata, interrupt controller, timer, SMP, memory map, and device tree handoff.
- First milestone is diagnostic kernel execution/panic output, not a graphical iOS boot.

### Phase 3 — storage and display
- Add device-specific UFS/storage and display support, then touch/input and power management.
- Treat GPU acceleration, Wi-Fi, Bluetooth, audio, sensors, and cellular as separate later milestones.

### Phase 4 — iOS userspace
- Only after the kernel can reliably mount the expected APFS volumes and start essential services should the project attempt the iOS UI/SpringBoard path. A kernel boot does not imply a usable iOS system.

## First local diagnostics

Run from Ubuntu, with the phone booted normally and USB debugging enabled if available:

```sh
adb devices -l
adb shell getprop ro.product.device
adb shell getprop ro.product.vendor.device
adb shell getprop ro.build.fingerprint
adb shell getprop ro.boot.slot_suffix
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.flash.locked
adb shell uname -a
```

If `adb` is not installed, install Android platform-tools on Ubuntu first. These commands only read properties; they do not unlock, erase, or flash anything.

## Current blocker

No verified zorn-specific UniXNU device repository/UEFI platform has been identified yet. Do not run the example device configuration or substitute the sheng configuration. Next work should establish the exact device build and source/device-tree inputs, then build a minimal zorn UEFI port separately from the generic framework.

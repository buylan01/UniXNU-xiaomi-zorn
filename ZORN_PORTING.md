# Redmi K80 (zorn) porting notes

Status: research / bring-up planning. This is **not** a bootable device configuration.

## Target

- Device: Redmi K80
- Android codename: `zorn`
- SoC: Qualcomm SM8650 (`pineapple`)
- Current vendor baseline: HyperOS `OS4.0.0.13.XOKCN`, vendor Android 14 / SDK 34
- System build properties identify the generic system source as `missi`; this is expected and is not treated as a device mismatch.
- Kernel: `6.1.157-android14-11`, clang 17 build
- Bootloader: unlocked; current slot `_a`; A/B update enabled
- Host available: Ubuntu 26.04; no local macOS/Xcode host

## Porting guide audit

The repository's porting guide defines two device-specific repositories plus the shared UniXNU build framework:

1. `UniXNU-zorn`: `board.toml`, `configs/zorn_defconfig`, ADT generator, device scripts, device drivers/kits and package.
2. `uefi-zorn`: Mu-Silicium device platform, DSC/FDF, firmware configuration and preflight checks.
3. The shared `UniXNU` tree supplies the build driver, DarwinBoot, NativePrelink, generic drivers and shared XNU patches.

The documented bring-up order is UEFI -> UniXNU menu -> payload -> DarwinBoot -> XNU entry -> platform expert -> storage/root -> userland. A successful kernel compile or UEFI menu is not evidence that the next stage works.

The `sheng` reference is a method/example only. Its board file targets Xiaomi Pad 6S Pro / SM8550 and pins a specific iPadOS IPSW, KernelCache hash, prelink metadata, memory map, device drivers and UEFI platform. Do not copy those device-specific values to zorn.

## Confirmed read-only device observations

- `ro.product.device=zorn`; model `24117RK2CC`
- `ro.board.platform=pineapple`, `ro.soc.model=SM8650`, `ro.soc.manufacturer=QTI`
- System fingerprint: `Xiaomi/missi/missi:17/CP2A.260605.016/17OS4.0.260920.223526195.QCPECN.S:user/release-keys`
- Vendor fingerprint: `Redmi/zorn/zorn:14/UKQ1.240624.001/OS4.0.0.13.XOKCN:user/release-keys`
- Kernel command line includes `rootwait ro init=/init`; this is the stock Android kernel and does not establish an XNU boot path.
- Current slot is `_a`; A/B update is enabled.
- Readable block-name listing includes `boot_a/b`, `vendor_boot_a/b`, `init_boot_a/b`, `dtbo_a/b`, `vbmeta_a/b`, `uefi_a/b`, `super` and `userdata`. Presence of `uefi_a/b` alone does not prove they are safe or appropriate targets for a custom image.
- Direct reads of `/proc/device-tree/compatible` and `/sys/firmware/devicetree/base/{model,compatible}` were denied by Android permissions. Do not assume the nodes are absent.

## Important constraints

1. Do not copy `UniXNU-sheng`'s `board.toml`, defconfig, kernelcache hashes, prelink addresses, DTB fixups, UEFI config, or boot arguments as if they applied to zorn.
2. The generic UniXNU repository is a build framework, not a Redmi K80 port. A real port needs the dedicated device repository and a compatible Mu-Silicium UEFI platform.
3. The current `macos.yml` workflow uploads text logs only; it does not publish kernel/payload/boot-image artifacts. A successful CI run alone therefore cannot deliver a boot image.
4. The XNU kernel build step requires macOS/Xcode. Ubuntu can be used for source review and parts of the firmware/build preparation, but it is not a supported host for the kernel step.
5. Do not flash or erase partitions during early bring-up. Prefer a documented reversible RAM-boot test if the K80 bootloader supports it; verify device support and recovery procedure first.

## Same-SoC reference port

The UniXNU device index lists two community ports on SM8650, neither is a Redmi K80 port:

- OPPO Find X7 Ultra (`PHY120`): UEFI/menu works; kernel builds in CI, but payload discovery and prelink/device boot remain pending. This is useful as a UEFI/platform reference, not as a source of zorn constants.
- OnePlus Ace 5 (`giulia` / PKG110): its public project reports XNU entry and later userland/Setup progress on a different SM8650 board, with a Darwin 22 / iOS 16 target. Its `board.toml`, ADT generator, UFS and display details are board-specific. Use the architecture and staged diagnostic approach as a reference only; do not transplant memory maps, kernel arguments, UFS addresses, DT values or image pins.

This cross-reference means there is an SM8650 precedent worth studying before designing zorn from scratch. The remaining work is to compare its generic-vs-board-specific implementation and the official zorn device tree, then determine which UEFI components can be reused safely.

## Bring-up sequence

### Phase 0 — source and boot-chain research
- Match the public Xiaomi zorn kernel/device-tree source branch to the installed vendor build; do not infer an exact source match from the kernel's `android14` label alone.
- Identify the boot image header, AVB/vbmeta chain, ABL behavior, DTB/DTBO arrangement, UEFI partition role and a reversible test path.
- Extract or obtain stock boot-chain images and DT/DTBO from the exact installed build where technically and legally available.
- Identify a compatible Apple IPSW/kernelcache strategy for the UniXNU framework and document the ABI/prelink assumptions before writing a board configuration.

### Phase 1 — UEFI feasibility
- Establish a reproducible Mu-Silicium build for SM8650/zorn.
- Validate the boot image header, firmware base/size, GOP/framebuffer, storage visibility and serial/USB diagnostics.
- Keep the stock boot path untouched; use only a documented, reversible test method.

### Phase 2 — XNU kernel bring-up
- Add the zorn device repository and defconfig once the board values and source inputs are known.
- Confirm the XNU target configuration, compatible IPSW/kernelcache, prelink metadata, interrupt controller, timer, SMP, memory map and ADT handoff.
- First milestone is diagnostic kernel execution/panic output, not a graphical iOS boot.

### Phase 3 — storage and display
- Add device-specific UFS/storage and display support, then touch/input and power management.
- Treat GPU acceleration, Wi-Fi, Bluetooth, audio, sensors and cellular as separate later milestones.

### Phase 4 — iOS userspace
- Only after the kernel can reliably mount the expected APFS volumes and start essential services should the project attempt the iOS UI/SpringBoard path. A kernel boot does not imply a usable iOS system.

## First local diagnostics

Initial read-only ADB diagnostics are complete. Do not repeat them unless a new question requires it. Partition names are observations only; no partition writes have been performed.

## Current blocker

No verified zorn-specific UniXNU device repository/UEFI platform has been identified yet. Next work is source/boot-chain analysis and UEFI feasibility, not another round of generic property collection or a guessed `zorn_defconfig`.

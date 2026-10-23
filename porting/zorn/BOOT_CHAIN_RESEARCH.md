# zorn stock boot-chain research (read-only)

This note records what public Android device configuration tells us, and separates it from facts that still require the exact installed stock images. It is not a flash guide.

## Public device-tree / Android build evidence

Reviewed public device configuration files for Xiaomi `zorn`:

- Sakura-Devices `device_xiaomi_zorn/BoardConfig.mk` at commit `63cc02ced0ab7f1087ea4160f09796dae6066b38`: https://github.com/Sakura-Devices/device_xiaomi_zorn/blob/63cc02ced0ab7f1087ea4160f09796dae6066b38/BoardConfig.mk
- zorn-developement-sources `device_xiaomi_zorn/BoardConfig.mk` at commit `3e2738b19a8251e4c4ccdc3c0e54c6a35a59798e`: https://github.com/zorn-developement-sources/device_xiaomi_zorn/blob/3e2738b19a8251e4c4ccdc3c0e54c6a35a59798e/BoardConfig.mk
- AOSP device-tree dump `aosp-device-tree/BoardConfig.mk` at commit `a60e54dad830c23893593a2f7114569557760108`: https://github.com/Jiovanni-dump/redmi_zorn_dump/blob/a60e54dad830c23893593a2f7114569557760108/aosp-device-tree/BoardConfig.mk

These public configurations agree on the Android platform family and modern boot image conventions, but they are community ROM build configurations—not a forensic dump of the user's installed HyperOS build.

Useful signals in the ROM configs:

- Boot image header version 4 is configured.
- A/B slots and separate `init_boot`, `vendor_boot`, and `dtbo` partitions are used.
- `TARGET_USES_UEFI := true` appears in a community ROM configuration. This says that ROM's boot flow expects UEFI; it does **not** prove that the handset's `uefi_a/b` partition can be overwritten, that the public UEFI binary is compatible, or that fastboot RAM boot is supported.
- The Sakura config points at prebuilt DTB and DTBO assets and sets `BOARD_INCLUDE_DTB_IN_BOOTIMG := true`. The dump config also shows a DTB/DTBO arrangement. This is evidence that image packaging matters; it does not identify which DTB/DTBO is active on the currently installed build.
- The public ROM config contains partition sizes and AVB settings for building that ROM. Do not use those values to write partitions or infer the installed image's exact layout/signatures.

## What remains unknown

The following must be established from exact stock-build artifacts or a documented runtime handoff before constructing a test image:

1. Exact `boot`, `vendor_boot`, `init_boot`, `dtbo`, `vbmeta`, and any UEFI-related image versions for the installed OS build.
2. Boot image header fields, page size, kernel/ramdisk/DTB placement and vendor ramdisk table contents.
3. Which DTBO overlay is selected for this board/revision, and the final merged live device tree.
4. Whether the bootloader supports `fastboot boot` for a custom image on this model and what image formats it accepts.
5. How the existing `uefi_a/b` partitions are used by the actual boot chain, and whether the bootloader verifies or chains into them.
6. A verified return-to-stock/recovery procedure for any eventual RAM-boot experiment.

## Safe next acquisition steps (read-only)

On Ubuntu, first use the official full fastboot/Recovery package for the *exact* installed build if available, or a trusted local copy of the matching factory images. Keep the archive and record its checksum. Do not substitute an image from another HyperOS build or a custom ROM.

If the exact build's image files are available on the host, inspect copies offline with Android image tools (for example `unpack_bootimg.py` / `mkbootimg` from AOSP, `avbtool info_image`, and `dtc` for DTB/DTBO decoding). Save all extracted outputs in a new working directory and never overwrite the source files. Record image SHA-256 hashes and the full tool versions in the notes.

ADB partition names alone are not enough to safely dump raw partitions. Do not use `dd` against guessed block paths, and do not write to `boot_a/b`, `vendor_boot_a/b`, `dtbo_a/b`, `vbmeta_a/b`, or `uefi_a/b`. If no exact stock package is available, the next task is to identify a supported, read-only acquisition path; do not improvise with root-only paths or a flash operation.

## Implications for UniXNU

The Android kernel image and Android boot metadata cannot simply be reused as an XNU boot image. The future test image needs a compatible Mu-Silicium UEFI stage, a deliberately constructed Darwin payload, and a valid Apple Device Tree handoff. The Linux DTB is useful evidence for hardware discovery, but its properties are not automatically an Apple Device Tree.

The first hardware milestone should remain a reversible UEFI/XNU diagnostic entry with a captured serial/USB/panic log. Do not attempt root filesystem mounting or iOS UI until the entry and platform initialization are repeatable.

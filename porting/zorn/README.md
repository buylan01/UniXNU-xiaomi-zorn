# Redmi K80 (zorn) device-port staging area

This directory is a staging scaffold, **not a buildable device repository**. The UniXNU repository contract puts board-specific files in a separate `UniXNU-zorn` repository and firmware-platform files in `uefi-zorn`. These files are kept here temporarily so the first implementation can be reviewed before splitting/publishing those repositories.

## Confirmed source facts

- Target codename: `zorn`; SoC: Qualcomm SM8650 / `pineapple`.
- The Xiaomi public device-tree branch `bsp-zorn-v-oss` contains:
  - `qcom/zorn-sm8650-overlay.dts`
  - `qcom/zorn-sm8650.dtsi`
  - `qcom/zorn-pinctrl.dtsi`
  - `qcom/xiaomi-sm8650-common.dtsi`
- The official overlay identifies the model as `Qualcomm Technologies, Inc. Zorn based on SM8650`, compatible with `qcom,pineapple-mtp`, `qcom,pineapple`, `qcom,pineapplep-mtp`, `qcom,pineapplep`, and `qcom,mtp`; it carries `qcom,board-id = <8 0>` and `xiaomi,miboard-id = <0xd 0>`.
- The zorn DTSI includes Xiaomi's SM8650 common DTSI and zorn pinctrl. The file comments say it is for attribution and public platform attribution; it is not a complete standalone device-tree dump.
- Device runtime DT access through ordinary ADB shell is denied, so the source DTS identifiers are **public-source facts, not a verified readback from this handset**.

## Staging rules

1. Do not copy the memory window, kernelcache hash, prelink slide, boot arguments, or firmware FD base/size from `giulia`, `PHY120`, or `sheng`.
2. Do not enable UFS, display, touch, GPU, or multi-core startup in the first defconfig without the corresponding zorn evidence and implementation.
3. Keep all build-affecting board fields unresolved until a target IPSW/kernelcache and UEFI source/config are selected and checked.
4. Do not flash partitions as part of source bring-up. A RAM-boot path and recovery procedure must be established first.

## Files

- `board.toml.template`: schema-shaped checklist; intentionally contains no usable IPSW pin or UEFI FD address.
- `configs/zorn_defconfig.template`: conservative first-entry feature selection, not an active defconfig.
- The ADT generator is deliberately not created yet: a valid ADT requires measured/validated CPU, GIC, timer, memory handoff, and platform identity details. We must not relabel the giulia ADT as zorn.

## Next implementation gates

1. Determine the actual zorn DTB/DTBO and boot-image arrangement from the exact installed firmware.
2. Establish a zorn-specific Mu-Silicium platform package and identify the vendor firmware/DXE input path.
3. Select and pin a supported Apple IPSW/KernelCache and verify the XNU ABI/prelink path.
4. Promote the templates into a dedicated device repository only when the inputs are complete enough for reproducible builds.

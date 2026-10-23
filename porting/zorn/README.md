# Redmi K80 (zorn) device-port staging area

This is a staging device repository inside the framework fork, so the CLI can load an active board file and defconfig without inventing hardware parameters. The long-term repository layout remains a dedicated `UniXNU-zorn` repository plus a separate `uefi-zorn` Mu-Silicium platform repository. This staging folder is not yet a bootable port.

## Current active files

- `board.toml`: active SM8650 board identity; intentionally omits the unqualified `[kernel.prelink]` section.
- `configs/zorn_defconfig`: conservative baseline XNU configuration with native UFS disabled.
- `BUILD_NOW.md`: copyable commands, runtime DTB facts, and current blockers.
- `board.toml.template` and `configs/zorn_defconfig.template`: original template/checklist files.

## Confirmed source and runtime facts

- Target codename: `zorn`; SoC: Qualcomm SM8650 / `pineapple`.
- The Xiaomi public device-tree branch `bsp-zorn-v-oss` contains `qcom/zorn-sm8650-overlay.dts`, `qcom/zorn-sm8650.dtsi`, `qcom/zorn-pinctrl.dtsi`, and `qcom/xiaomi-sm8650-common.dtsi`.
- The official overlay identifies the model as `Qualcomm Technologies, Inc. Zorn based on SM8650`, compatible with `qcom,pineapple-mtp`, `qcom,pineapple`, `qcom,pineapplep-mtp`, `qcom,pineapplep`, and `qcom,mtp`; it carries `qcom,board-id = <8 0>` and `xiaomi,miboard-id = <0xd 0>`.
- The handset's runtime `/sys/firmware/fdt` readback confirms the Zorn model/compatible list, eight PSCI-enabled CPU nodes, PSCI via SMC, GICv3 at `0x17100000`, GIC ITS at `0x17140000`, and ARMv8 timer frequency property `0x124f800`.
- The runtime `chosen.stdout-path` points to GENI debug UART at `0x89c000`, but Linux cmdline contains `console=ttynull`; external UART access is not confirmed.
- Linux-visible `MemTotal` is `11367352 kB`. It is not the physical-RAM total and does not establish an Apple Device Tree memory map.

## Build commands

From the framework root:

```bash
./unixnu defconfig zorn_defconfig -D porting/zorn
./unixnu status -D porting/zorn
```

The XNU build step requires macOS and Xcode's iPhoneOS SDK. On a supported Mac, `./unixnu build kernel -D porting/zorn` can start the kernel build, but currently stops at prelink because no verified `[kernel.prelink]` section exists. See `BUILD_NOW.md`.

## Staging rules

1. Do not copy memory windows, KernelCache hashes, prelink slide, boot arguments, or firmware FD addresses from `giulia`, `PHY120`, or `sheng`.
2. Do not enable UFS, display, touch, GPU, or multi-core hardware drivers without Zorn-specific evidence and implementation.
3. Keep IPSW/KernelCache, prelink roots/slide, and UEFI FD/config unresolved until selected and checked.
4. Do not flash partitions during source bring-up. Establish a RAM-boot path and recovery procedure first.

## Next gates

1. Determine the selected DTB/DTBO and boot-image arrangement from the exact installed firmware.
2. Establish the Zorn Mu-Silicium platform package and vendor firmware/DXE input path.
3. Select and pin a supported Apple IPSW/KernelCache and verify XNU ABI/prelink compatibility.
4. Decode and validate the runtime memory map and Apple Device Tree handoff.
5. Implement and test the initial UEFI console and boot path before adding storage/display/touch support.

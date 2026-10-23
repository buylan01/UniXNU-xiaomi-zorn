# Zorn first-build handoff

This folder now contains an active `board.toml` and `configs/zorn_defconfig`. It is still a bring-up scaffold, not a bootable port.

## Select the device config

Run from the UniXNU framework checkout:

```bash
./unixnu defconfig zorn_defconfig -D porting/zorn
./unixnu status -D porting/zorn
```

Use the path relative to the framework root, or an absolute path if running elsewhere.

## Build limitation

The framework's XNU kernel step uses the Xcode iPhoneOS SDK and `xcodebuild`. Ubuntu can inspect sources, validate configuration, and prepare inputs, but the supported XNU build itself requires macOS with Xcode. On a supported Mac, the current command is:

```bash
./unixnu build kernel -D porting/zorn
```

The kernel compilation can begin, but the command will stop at prelink because `[kernel.prelink]` is deliberately not fabricated yet. A usable kernel artifact requires a compatible target IPSW and original KernelCache, verified prelink slide/root set, and the correct XNU source ABI. Do not treat a partial build as bootable.

## Verified runtime DTB facts

From the handset's `/sys/firmware/fdt` readback:

- Model: `Qualcomm Technologies, Inc. Zorn based on SM8650`.
- Compatible list includes `qcom,pineapple-mtp` and `qcom,pineapple`.
- Eight CPU nodes use `enable-method = "psci"`; PSCI method is `smc`.
- Interrupt controller is `arm,gic-v3` at `0x17100000`; GIC ITS at `0x17140000`.
- Architectural timer is `arm,armv8-timer`; the DT clock-frequency property is `0x124f800` (19.2 MHz).
- `chosen.stdout-path` references the GENI debug UART at `0x89c000`, but Linux command line has `console=ttynull`. External UART access is not established.
- Linux-visible memory reported `MemTotal: 11367352 kB`; this is not a physical-RAM total and is not a substitute for a validated Apple Device Tree memory map.

## Do not guess

Still unresolved: target IPSW / KernelCache compatibility, prelink slide, retained kernel roots, Zorn Mu-Silicium UEFI platform/config/FD, actual Android bootloader handoff and RAM-boot/recovery procedure, ADT memory map, UFS/display/touch/GPU support. Do not flash any partition based on this scaffold.

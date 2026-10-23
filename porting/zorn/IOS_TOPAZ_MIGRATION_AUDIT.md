# iOS-topaz → Redmi K80 (zorn / SM8650) migration audit

Status: source audit and migration staging only. No K80 image has been built or booted by this document. No device partitions have been written.

## Decision

Use `MakrSas/ios-topaz` as the implementation baseline for the iOS 16 / XNU `xnu-8796` path. Keep this UniXNU repository as the source of K80 research and hardware evidence; do not transplant the UniXNU `board.toml`, defconfig, prelink assumptions, or UEFI layout into ios-topaz. The two projects use different boot and build architectures.

Upstream: https://github.com/MakrSas/ios-topaz (branch `topaz`)
Original base: https://github.com/dipa-studio/ios-sweet
Related GPU/userland source credited by upstream: https://github.com/oosexgang/hackios-xnu-kernel and https://github.com/oosexgang/hackios-xnu-userland

## What the commit history teaches us

The useful pattern is **measure → isolate a boot blocker → make one small patch → verify a concrete milestone → keep the patch as a numbered delta**. The numbers below are upstream commit identifiers/patch series, not K80-ready patches.

| Upstream commit / patch | What changed on topaz | Lesson for zorn |
| --- | --- | --- |
| `ffc8e8b6` / patch `0080` | Retargeted shim, ADT, XNU topology and MMIO from sweet/SM7150 to topaz/SM6225; gated sweet-only touch/key/USB code because it poked the wrong SoC registers. | Keep common XNU code; explicitly gate or remove drivers that touch the old board's registers. Generate a zorn hardware layer from measured data. |
| `d3d155b9` / patch `0081` | Changed the build target to ARMv8.0, replaced LSE `casa` use, disabled PAN assumptions, emulated userland LSE/LDAPR, and worked around the target bootloader dropping the v4 ramdisk by embedding the payload after the shim image. | Treat CPU features and bootloader image handling as separate measured compatibility problems. The ARMv8.0 workaround and payload placement are topaz-specific, not defaults for SM8650. |
| `1e633b54` / patch `0082` | Booted XNU on Gold MPIDR `0x100` because the Silver A53 did not expose the 16 KiB translation granule required by that XNU configuration. | Do not copy the Gold MPIDR. Probe K80 ID registers on each core and establish the actual MPIDR/PSCI topology before enabling SMP. |
| `3f289a6a`, `f26e7a5f` | Found XNU hanging after `initCPU`; prepared GIC distributor/redistributor and SGI/PPI state for the boot and secondary Gold cores. | The DTB's GIC addresses alone are insufficient. Verify redistributor affinity, wake state, group, priority and timer/SGI delivery before debugging higher layers. |
| `0083`–`0112` | Brought up touch, keys, display scanout, cache maintenance, brightness and framebuffer behavior in small, diagnosable steps. | Reuse the staged diagnostic method, not the FocalTech/DSI/MDP register constants. K80 display/input drivers need their own evidence. |
| `0113`–`0130` | Added modem PAS, GLINK/QRTR, host services, WLFW and Wi-Fi scanning/association, based partly on the topaz-woa driver. | Treat this as an architectural reference only; SM6225 firmware, transport resources and WLAN implementation are not proof of SM8650 compatibility. |
| `0131`–`0139` | Iteratively fixed IOSurface/CoreAnimation client-memory lifetime, IDs, mapping and empty accelerator-service behavior. | These XNU/userland ABI fixes may be reusable only after checking the same `xnu-8796` base and applying/testing each patch independently. |
| `0140` + `0141` | Added a `TOPAZ_4K` experiment and AMP cluster handling so the topaz port could attempt to use all eight heterogeneous cores. The patch explicitly preserves 16 KiB user-task/surface alignment and relaxes 16 KiB shared-cache signature handling. | This is not a free switch for K80. Validate the 16 KiB/4 KiB VM and shared-cache assumptions against the exact iOS 16 userland before enabling it. It also disables effective code-signature page validation in this experiment. |
| `fe7625f1` (2026-10-22) | Published previously private GPU/audio/Wi-Fi/sandbox work after HackiOS published its source; added `CREDITS.md` and userland components. | Preserve upstream attribution and license boundaries. Adreno 610/RedwoodGPU code is not a K80 Adreno 750 driver. |
| `4093a61b`, `e4772911` | Fixed display mode reporting/surface reserve and a touch release edge case after the UI already worked. | Even late-stage regressions were fixed as small, evidence-driven patches rather than broad rewrites. |

Sources:
- https://github.com/MakrSas/ios-topaz/commit/ffc8e8b6e0
- https://github.com/MakrSas/ios-topaz/commit/d3d155b95d
- https://github.com/MakrSas/ios-topaz/commit/1e633b54c9
- https://github.com/MakrSas/ios-topaz/commit/3f289a6a74
- https://github.com/MakrSas/ios-topaz/commit/fe7625f1652933d402ff1600f80c440ddc4044de
- https://github.com/MakrSas/ios-topaz/commit/4093a61bae5899ae232acc2086d666724a392bc0

## File-by-file migration matrix

| ios-topaz area | Disposition for zorn | Notes |
| --- | --- | --- |
| `xnu/` source generation | **Reuse as baseline** | Prefer `xnu-8796`/iOS 16 lineage over the current UniXNU XNU 10063 / Darwin 23 baseline. Keep upstream patch history intact; add separately numbered zorn patches. |
| `.github/workflows/xnu.yml`, `tools/build-local.sh` | **Adapt build only** | macOS/Xcode 15-compatible CI is useful because the host here is Ubuntu 26.04. Keep kernel artifacts and logs as CI outputs. Review patch-apply/build results before treating a green job as a bootable image. |
| `shim/start.S`, `shim/loader.c`, payload parser | **Reuse concept; validate ABI** | The XNU boot_args/payload loader and entry sequence are valuable references. Do not assume K80 ABL passes the same image fields, uses the same boot image header, or places ramdisk/payload the same way. |
| `shim/shim.c`, `shim/mmu.c`, `shim/ufs.c`, `shim/console.*` | **Rewrite board-specific parts** | GIC/UFS/framebuffer/restart/PS_HOLD addresses and DDR windows are SoC/board-specific. Start with minimal read-only probes and panic/console path. |
| `tools/adt.py` | **Reuse parser/serializer after validation** | Generic Apple Device Tree parsing is useful; compare against known ADT samples and validate generated trees before boot. |
| `tools/mkadt.py` | **Fork and replace board data** | Keep the generation structure, but replace every CPU, memory, GIC, timer, UFS, framebuffer, product and security property based on verified zorn facts. Linux FDT is evidence, not a ready-made Apple ADT. |
| `tools/mkpayload.py` | **Reuse with checks** | Validate Mach-O segment/chained-fixup assumptions against the chosen kernel build and boot ABI. |
| `tools/mkiosroot.sh`, `tools/sweet-root-tweaks.py`, IPSW/userland extraction | **Reuse cautiously** | The iPhone X iOS 16.7.16 (20H392) userland is a candidate shared baseline. Do not run the root-install flow on the K80: upstream explicitly says its root install erases Android userdata. Build and inspect a host-side image first. |
| patches `0001`–`0079` (sweet base) | **Audit individually** | Many patches implement generic missing XNU pieces or the SWEET platform. Keep only patches required by the chosen configuration and verify patch context. |
| patches `0080`–`0082` | **Port concept, rewrite hardware assumptions** | CPU topology, CPU feature level and boot-CPU assumptions must be measured for SM8650. |
| patches `0083`–`0112` | **Mostly rewrite/skip initially** | Touch, keys, panel, display scanout, cache and brightness are topaz-specific. Bring up console and interrupt/timer before these. |
| patches `0113`–`0130` | **Research only initially** | Modem/WLAN transport and firmware assumptions are device-specific; postpone until basic kernel and storage are stable. |
| patches `0131`–`0139` | **Candidate reusable XNU fixes** | Verify each against upstream XNU base and iOS userland behavior. |
| patches `0140`–`0152`, `0900`–`0950`, `userland/gpu`, `userland/audio` | **Research only; no blind copy** | 4 KiB/AMP experiment needs independent validation; RedwoodGPU/Adreno 610, framebuffer path and audio codec are not K80 hardware support. Retain attribution/license notices for any derivative work. |
| `RECOVERY.md` | **Do not copy as a K80 runbook** | It contains topaz-specific partition layout and destructive flash/GPT repair commands. We need a separate zorn recovery plan based on K80 evidence. |

## Zorn facts already collected in the original project

These facts should be copied as *research records*, not treated as an XNU configuration:

- Device: Redmi K80, model `24117RK2CC`, codename `zorn`; SoC SM8650 / `pineapple`.
- Runtime Linux FDT model: `Qualcomm Technologies, Inc. Zorn based on SM8650`; compatible includes `qcom,pineapple-mtp`, `qcom,pineapple`, `qcom,pineapplep-mtp`, `qcom,pineapplep`, `qcom,mtp`.
- Runtime CPU topology reported in Linux DT: 2× Cortex-A520, 5× Cortex-A720, 1× Cortex-X4. Obtain the exact per-core MPIDR and feature-register values from a minimal shim before constructing the Apple ADT.
- Runtime GICv3 base `0x17100000`, ITS `0x17140000`; candidate timer property `0x124f800`; UART candidate `0x89c000` but console is `ttynull`, so UART access is unverified.
- Current Android boot images are Linux boot artifacts (boot/vendor_boot/init_boot v4); the extracted `boot_a` kernel is Linux, not XNU. Existence of `uefi_a/b` is not permission or evidence to overwrite them.
- The runtime DTB is `~/k80-xnu-research/unpacked/runtime.dtb` on the user's host. It is not included in this repository unless separately added by the user.

See `ZORN_PORTING.md` and `porting/zorn/BOOT_CHAIN_RESEARCH.md` for the longer evidence trail and image hashes.

## Proposed migration sequence

1. Fork `MakrSas/ios-topaz` under the user's account. Do not push changes to the upstream owner's branch.
2. Copy the original K80 research notes into `research/zorn/` in that fork (at minimum `ZORN_PORTING.md`, `porting/zorn/BOOT_CHAIN_RESEARCH.md`, and relevant board/DT notes). Keep them clearly labelled research, not active configuration.
3. Pin the exact upstream `topaz` commit used as the base and preserve upstream patches/credits. Create a branch such as `zorn/sm8650-bringup`.
4. Add a separate `tools/mkadt-zorn.py` or data-driven board file; do not replace topaz's generator in-place until the new one is tested offline.
5. Implement a minimal zorn shim diagnostic: early framebuffer/serial if confirmed, per-core MPIDR/ID registers, PSCI version/CPU_ON, GIC redistributor affinity, timer IRQ, memory map and safe reboot path. Avoid UFS/root mounting initially.
6. Generate and parse/diff an ADT offline; assert there are no topaz MMIO constants left in the zorn config.
7. Add a zorn-specific CI workflow that builds the same XNU base and uploads kernel/buildtree/log artifacts. A successful build is only a compile milestone.
8. Only after a documented recovery path exists, consider a reversible RAM-boot test if K80 fastboot supports it. Do not flash `boot`, `vendor_boot`, `dtbo`, `vbmeta`, or `uefi` partitions.
9. Do not install an iOS root into K80 `userdata` until the user has explicitly approved the destructive effect and a verified backup/recovery plan exists.

## Open questions before the first K80 payload

- Does K80's ABL accept `fastboot boot` for a custom boot-v4 image, and exactly what does it pass to the payload?
- Can the boot path execute a shim without touching persistent boot partitions?
- What are the exact MPIDRs and `ID_AA64MMFR0_EL1` granule fields for each A520/A720/X4 core?
- Which memory ranges are safe for the shim/XNU after subtracting secure-world, modem, GPU, camera, display and firmware carve-outs?
- Is the candidate timer a generic architectural timer source or only a DT property describing a platform timer block? Verify actual interrupt IDs and delivery.
- What UFS controller resources and storage boot path are active on this exact K80 firmware?
- Is the iPhone X 20H392 userland compatible with the selected XNU configuration once the kernel page-size and shared-cache assumptions are chosen?

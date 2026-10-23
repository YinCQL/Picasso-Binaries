## Firmware Info

- Device: Redmi K30 5G (picasso)
- SoC: Snapdragon 765G (SM7250 / lito)
- Region: China
- Version: `V13.0.5.0.SGICNXM` / `BOOT.XF.3.3.1-00049-SAIPANLAZ-1`
- Binaries extracted from the device's xbl.img via UEFIReader.

## Patches/Fixes

### ButtonsDxe

- Reason: Allows navigating/confirming in the UEFI menus. Power button acts as **ENTER**.
- Patch (file offsets are relative to the .efi, .text starts at 0x1000):
  - 0x3714: `51 20 80 52` (`mov w17, #0x102`) -> `b1 01 80 52` (`mov w17, #0xd`)
  - 0x3719: `33` -> `37` (`strh w17, [sp, #0x18]` -> `strh w17, [sp, #0x1a]`)
  - Effect: the key report now has UnicodeChar = 0x0D (CHAR_CARRIAGE_RETURN) instead of a
    ScanCode of 0x102, which is what EDK2 BootManagerMenuApp expects for "select".
- Patch Origin: identical fix used for miatoll (SM7125). The picasso function is byte-identical
  to the miatoll one in that region (verified by disassembly).
- Verified result:
  ```
  3714: mov  w17, #0xd
  3718: strh w17, [sp, #0x1a]
  ```

### UFSDxe

- Reason: The UFS IOMMU domain is **already created and the UFS0 device already attached by
  the previous firmware (XBL)**. UEFI's UFSDxe re-creates the domain, which crashes/hangs and
  ends in a watchdog reset. Description matches the miatoll (SM7125) case verbatim:
  "An MMU Domain is already setup by the previous firmware and gets re-set again, causing a crash."
- Patch: short-circuit the "UFS IOMMU domain create/attach" function (called only when the
  SharedIMEM boot-device-type reports UFS, i.e. == 8) so it returns EFI_SUCCESS immediately.
  - 0x24D8: `ff 03 01 d1 f4 4f 02 a9` (`sub sp,sp,#0x40; stp x20,x19,[sp,#0x20]`)
    -> `00 00 80 52 c0 03 5f d6` (`mov w0,#0; ret`)
- **Why this patch only works together with the SharedIMEM fix:** with a wrong
  SharedIMEMBaseAddr the boot-device-type lookup fails, UFSDxe bails out before the IOMMU
  call (no crash, but no BlockIo either). Once SharedIMEM is correct, the IOMMU path is
  actually reached and the redundant domain create must be skipped. Both changes are needed
  as a pair.
- Function identified by referenced strings "UFS IOMMU LocateProtocol failed" /
  "UFS IOMMU domain create failed" / "UFS IOMMU domain attach ARID 0x0 failed".
- Verified result:
  ```
  24d8: mov w0, #0
  24dc: ret
  ```

### PmicDxe

- Reason: On boot the PMIC power-key check validates the press duration of the key that
  powered the device on. On this device the PON timestamps come back as garbage
  (negative/large values), so the check fails and the driver calls
  `gRT->ResetSystem(EfiResetShutdown)` — this is the cause of the random "power off during
  boot" symptom. The log shows:
  ```
  KeyPressed for 150 msec was invalid, release_time -1905712870, press_time -1905712820
  Invalid pwrkey press. Shutting down!
  ```
- Patch: neutralise the single shutdown call so the invalid-press check only logs and boot
  continues.
  - 0x10148: `16 ff ff 97` (`bl 0xfda0`, the shutdown helper) -> `1f 20 03 d5` (`nop`)
  - The shutdown helper was confirmed to be `gRT->ResetSystem(EfiResetShutdown)`; it has
    exactly one caller in the whole driver.
- Patch style matches existing community PmicDxe patches (e.g. i005d / venus) that force
  boot to continue instead of dying in a PMIC error path.
- Verified result:
  ```
  10138: adrp x1, #0x1c000
  1013c: add  x1, x1, #0x19e      ; "Invalid pwrkey press. Shutting down!"
  10144: bl   #0x3ab0              ; debug print
  10148: nop                      ; <- was bl #0xfda0 (shutdown)
  1014c: b    #0xfff0              ; continue boot
  ```

## Configuration Sources (IMPORTANT)

The authoritative source for platform configuration is the device's own XBL config
`RawFiles/uefiplat.cfg` (extracted from this device's xbl.img) plus the device devicetree.
Do NOT copy values from other devices (e.g. miatoll) without checking these two sources.

### Fixed after UFS BlockIo enumeration failure

- `SharedIMEMBaseAddr`: **0x146AA000 -> 0x146AB000**.
  - Vendor `uefiplat.cfg`: `SharedIMEMBaseAddr = 0x146AB000`
  - Devicetree: `qcom,msm-imem@146ab000`
  - Read by 13+ drivers (UFSDxe, TzDxe, QcomBds, PILDxe, SdccDxe, VariableDxe, ...).
    Wrong value produced `ERROR: Failed to Get Shared Imem Boot Device type` in every driver
    and UFS never produced a BlockIo handle.
- Added `EnableUfsIOC = 0` and `UfsSmmuConfigForOtherBootDev = 1` (both read by UFSDxe;
  values taken from this device's uefiplat.cfg). 62 of 74 platforms declare these keys —
  picasso was one of the few missing them.
- Added `UefiMemUseThreshold = 0xC8`, `ShmBridgememSize = 0xA00000` (vendor values).
- Memory map IO regions corrected against vendor uefiplat.cfg, notably:
  - `UFS UFS REGS` 0x01D84000 -> **0x01D80000** (+0x20000): vendor range covers the UFS PHY/GDSC
    area below 0x1D84000 that was previously unmapped.
  - `SMMU` size 0x100000 -> **0xD0000** (vendor), `TCSR` 0x40000 -> 0x30000,
    `SECURITY CONTROL` 0x7000 -> 0x10000, `MDSS` 0x134000 -> 0x200000,
    `APSS_GIC500_GICD` 0x10000 -> 0x170000, `APSS_RSC_RSCCR` 0x30000 -> 0x40000,
    `USB30_PRIM` size -> 0x200000, `IMEM Base` size -> 0x2C000.
  - Added missing regions: IPC_ROUTER_TOP, MMCX_CPR3, GPU_CPR, USB30_SEC, TITAN_SS.

### Memory-map region names (critical)

Prebuilt Qualcomm drivers locate MMIO blocks **by name** through the "UEFI Config Lib" shim
(`LocateMemoryRegionByName`). The names must match the vendor `uefiplat.cfg` exactly,
otherwise the lookup fails and the driver either skips initialization or uses a zero address.
`DALSYSDxe` alone looks up 27 named regions.

Fixed names (were copied/renamed earlier and therefore not found):

| Driver lookup name | vendor address | previous (wrong) |
|---|---|---|
| `PSHOLD` | 0x0C264000 | "PSHOLD Reset" |
| `SLP_CNTR` | 0x0C221000 | "SLP_CNTR_TSENS" |
| `TSENS0` / `TSENS1` | 0x0C222000 / 0x0C223000 | single "TSENS SROT" |
| `TSENS0_TM` / `TSENS1_TM` | 0x0C263000 / 0x0C265000 | "TSENS TM0/1" |
| `RPMH_CPRF_CPRF` | 0x0C200000 (+0x10000) | "RPMH_CPRF" at 0x0B220000 |
| `TLMM_SOUTH/WEST/EAST` | 0x0F100000/0x0F500000/0x0F900000 (+0x300000) | single oversized "TLMM" |
| `IPC_ROUTER_TOP` | 0x00400000 (+0x100000) | missing |
| `GPU_CPR` | 0x03D9A000 (+0x4000) | missing |
| `GPU_GMU_CX_BLK` | 0x03D7D000 (+0xC000) | "GPU_GMU" overlapping GPU_CC |
| `CPU Vectors` | needed by EnvDxe (KEP) | missing |

## Known Pending Items


- **TsensDxe**: currently left disabled in the platform (not required to boot UEFI). The earlier
  data abort at FAR 0x0C223004 (TSENS sensor #1 registers) is fixed by the memory map covering
  both sensors; re-enabling is a separate controlled test.
- **DisplayDxe**: disabled in favour of SimpleFbDxe for stability.

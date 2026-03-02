# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-02
**Commit:** 1fea505
**Branch:** main

## OVERVIEW

OpenCore 1.0.6 EFI configuration for Hackintosh. Hardware: ROG STRIX Z790-I GAMING WIFI + Intel 14900K + AMD Radeon RX 6950 XT. SMBIOS spoofed as MacPro7,1.

## STRUCTURE

```
OC/
├── config.plist          # Main OpenCore config (THE critical file)
├── oldConfig.plist       # Backup config (pre-USBToolBox era, has Intel WiFi/BT kexts disabled)
├── OpenCore.efi          # Bootloader binary
├── ACPI/                 # Compiled SSDT patches (.aml)
├── Drivers/              # UEFI drivers (.efi)
├── Kexts/                # Kernel extensions
└── Resources/            # OpenCanopy GUI themes (Acidanthera: Chardonnay, GoldenGate, Syrah)
```

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Change boot behavior | `config.plist` → Misc/Boot | Picker timeout, show/hide, hotkeys |
| Fix sleep/wake | `config.plist` → ACPI/Patch + `ACPI/SSDT-GPRW.aml` | GPRW→XPRW rename fixes instant wake |
| GPU issues | `config.plist` → Kernel/Add (NootRX) + `ACPI/SSDT-RX6950XT.aml` | NootRX handles Navi 21 patching (replaced WhateverGreen) |
| USB mapping | `Kexts/UTBMap.kext/Contents/Info.plist` | Has variants: 0Info.plist, 1Info.plist, allInfo.plist for different port configs |
| CPU power mgmt | `Kexts/CPUFriendDataProvider.kext` + `ACPI/SSDT-PLUG-ALT.aml` | CpuTopologyRebuild also active |
| NVMe blocking | `ACPI/SSDT-BLOCK-NVME.aml` | Blocks specific NVMe from native Apple driver |
| Thunderbolt | `ACPI/SSDT-TB3.aml` + `ACPI/SSDT-DTPG.aml` | TB3 hotplug support |
| SMBIOS/serial | `config.plist` → PlatformInfo/Generic | MacPro7,1 with custom MLB/Serial/UUID |
| SIP toggle | Boot picker → ToggleSipEntry | `csr-active-config 0xA7F` (mostly disabled) |

## ACPI PATCHES (all enabled)

| SSDT | Purpose |
|------|---------|
| SSDT-AWAC | Disable AWAC clock, enable legacy RTC |
| SSDT-EC-USBX | Fake EC device + USB power properties |
| SSDT-PLUG-ALT | CPU power management plugin type |
| SSDT-GPRW | Fix instant wake (rename GPRW→XPRW) |
| SSDT-TB3 | Thunderbolt 3 hotplug |
| SSDT-DTPG | Device Tree Property Gateway for TB3 |
| SSDT-PMC | Power Management Controller (native NVRAM) |
| SSDT-RX6950XT | GPU-specific patches for RX 6950 XT |
| SSDT-BLOCK-NVME | Block specific NVMe device |

## KEXTS (load order matters — Lilu MUST load first)

### Enabled

| Kext | Version | Role |
|------|---------|------|
| Lilu | 1.7.2 | Patching framework — **dependency for NootRX, VirtualSMC, RestrictEvents** |
| VirtualSMC | 1.3.8 | SMC emulation (core) |
| NootRX | 1.0.0 | AMD Navi GPU driver (RX 6950 XT) — replaced WhateverGreen |
| SMCProcessor | 1.3.8 | CPU sensor data (optional, monitoring only) |
| SMCSuperIO | 1.3.8 | Fan/voltage sensor data (optional, monitoring only) |
| CPUFriend | 1.3.1 | CPU frequency/power management |
| CPUFriendDataProvider | 1.0.0 | CPU power data (machine-specific) |
| CpuTopologyRebuild | 2.0.2 | Fix CPU topology for P/E-core CPUs — **critical for 14th Gen** |
| NVMeFix | 1.1.4 | NVMe power management fix (optional, improves sleep) |
| RestrictEvents | 1.1.7 | Block unwanted system events/notifications + `revpatch=sbvmm` |
| USBToolBox | 1.2.0 | USB port mapping framework |
| UTBMap | 1.1 | USB port map data (machine-specific) |

### Disabled (in config.plist, still on disk)

| Kext | Version | Why disabled |
|------|---------|-------------|
| WhateverGreen | 1.7.1 | Replaced by NootRX for Navi 21 GPU. Kept as fallback (MaxKernel=24.99.99) |
| SMCRadeonGPU | 0.3.3 | GPU temp/fan monitoring — disabled, may conflict with NootRX |

### Not loaded (on disk only)

| Kext | Status |
|------|--------|
| USBPortsMap.kext | Legacy USB mapping — **do not enable**, conflicts with USBToolBox/UTBMap |

## DRIVERS

| Driver | Purpose |
|--------|---------|
| OpenRuntime.efi | Runtime services (essential) |
| OpenHfsPlus.efi | HFS+ filesystem support |
| OpenCanopy.efi | GUI boot picker |
| ResetNvramEntry.efi | NVRAM reset tool (--preserve-boot arg) |
| ToggleSipEntry.efi | SIP toggle (csr-active-config 0xA7F) |

## CONVENTIONS

- **Kext versions tracked** in config.plist Comment fields (e.g., `<string>V1.7.1</string>`) — update these when upgrading kexts
- **CPUID spoofing active**: Cpuid1Data/Mask emulates older CPU for macOS compatibility (MinKernel 20.0.0 = Big Sur+)
- **Boot args**: `-v debug=0x100 keepsyms=1` (verbose/debug), `-ctrsmt` (CpuTopologyRebuild SMT), `revpatch=sbvmm` (RestrictEvents VM bypass)
- **Language**: zh-Hans (Simplified Chinese)
- **Picker**: External mode (OpenCanopy GUI), 1s timeout, show picker enabled

## ANTI-PATTERNS

- **NEVER** edit kext binaries directly — update whole .kext bundle from Acidanthera releases
- **NEVER** change kext load order without understanding dependencies (Lilu must be first)
- **NEVER** update OpenCore.efi without also updating config.plist schema — breaking changes between versions
- **NEVER** modify PlatformInfo serials without regenerating via GenSMBIOS (iServices will break)
- **DO NOT** enable USBPortsMap.kext alongside USBToolBox/UTBMap — they conflict
- **DO NOT** remove SSDT-GPRW without also removing the GPRW→XPRW ACPI patch in config.plist
- **DO NOT** enable NootRX and WhateverGreen simultaneously — they conflict on Navi GPU patching
- **DO NOT** enable SMCRadeonGPU without verifying NootRX compatibility first
- Backup config.plist before ANY change (oldConfig.plist exists for this reason)

## UNIQUE DETAILS

- **UTBMap variants**: `0Info.plist`, `1Info.plist`, `allInfo.plist` in UTBMap.kext — different USB port configurations. Active one is `Info.plist`. Swap by renaming to test different mappings.
- **ResizeAppleGpuBars=0**: GPU BAR resizing enabled (BIOS must also have Resizable BAR enabled)
- **CustomSMBIOSGuid + Custom UpdateSMBIOSMode**: Required for ASUS boards to avoid BIOS OEM string injection issues
- **Kernel patches**: F1 Startup patch (AppleRTC) + Disable RTC wake scheduling — both target `com.apple.driver.AppleRTC`

## UPGRADE CHECKLIST

1. Download new OpenCore release from [Acidanthera](https://github.com/acidanthera/OpenCorePkg/releases)
2. Backup current EFI to `oldConfig.plist` / separate folder
3. Update OpenCore.efi, Drivers/, and compare config.plist with new Sample.plist
4. Update kexts from respective repos (check Lilu compatibility first)
5. Update Comment version strings in config.plist Kernel/Add entries
6. Test boot — if fail, swap back backup EFI
7. Commit with version tag (current pattern: `1.0.6`, `1.0.4`, `1.0.3`)

## GIT

- Versioned with tags (`1.0.6`, `1.0.4`, `1.0.3`)
- Commit messages: version bumps + kext upgrades + fixes
- No CI/CD, no scripts — manual EFI management

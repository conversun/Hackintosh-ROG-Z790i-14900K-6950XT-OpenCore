# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-02
**Last verified:** 2026-08-28 — OpenCore 1.0.7 (`REL-107-2026-03-20`) on macOS 26.6.2 Tahoe (25G83)
**Commit:** 1fea505
**Branch:** main

## OVERVIEW

OpenCore 1.0.7 EFI configuration for Hackintosh. Hardware: ROG STRIX Z790-I GAMING WIFI + Intel 14900K + AMD Radeon RX 6950 XT. SMBIOS spoofed as MacPro7,1.

## STRUCTURE

```
OC/
├── config.plist          # Main OpenCore config (THE critical file)
├── oldConfig.plist       # Backup config (pre-Bluetooth snapshot, 2026-07-17 — refreshed before adding Intel BT kexts)
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
| GPU issues | `config.plist` → Kernel/Add (WhateverGreen) + `ACPI/SSDT-RX6950XT.aml` | WhateverGreen 1.7.1 (dev) handles Navi 21 patching; **requires `agdpmod=pikera`** to prevent black screen at boot completion |
| USB mapping | `Kexts/UTBMap.kext/Contents/Info.plist` | 23 ports on `_SB.PC00.XHCI` (HS16 removed 2026-08-28, see ANTI-PATTERNS). Map exceeds 15 ports, so it **depends on `XhciPortLimit=true`** |
| CPU power mgmt | `Kexts/CPUFriendDataProvider.kext` + `ACPI/SSDT-PLUG-ALT.aml` | CpuTopologyRebuild also active |
| NVMe blocking | `ACPI/SSDT-BLOCK-NVME.aml` | Blocks specific NVMe from native Apple driver |
| Thunderbolt | `ACPI/SSDT-TB3.aml` + `ACPI/SSDT-DTPG.aml` | TB3 hotplug support |
| SMBIOS/serial | `config.plist` → PlatformInfo/Generic | MacPro7,1 with custom MLB/Serial/UUID |
| SIP toggle | Boot picker → ToggleSipEntry | Actual `csr-active-config` is `0x0A03` (`AwoAAA==`) = UNTRUSTED_KEXTS + UNRESTRICTED_FS + UNAPPROVED_KEXTS + UNAUTHENTICATED_ROOT |
| Bluetooth (Intel AX211) | **Currently DISABLED** — BIOS Bluetooth Controller off, and all 3 BT kexts `Enabled=false` | Kexts still on disk. NVRAM/Add keys restored 2026-08-28. To re-enable: BIOS BT on + flip the 3 kexts true. Open issue: enabling BT has caused intermittent total USB loss — mechanism NOT identified |

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
| Lilu | 1.7.2 | Patching framework — **dependency for WhateverGreen, VirtualSMC, RestrictEvents** |
| VirtualSMC | 1.3.8 | SMC emulation (core) |
| WhateverGreen | 1.7.1 (dev) | AMD Navi GPU driver (RX 6950 XT) — requires `agdpmod=pikera` boot-arg |
| SMCProcessor | 1.3.8 | CPU sensor data (optional, monitoring only) |
| SMCSuperIO | 1.3.8 | Fan/voltage sensor data (optional, monitoring only) |
| CPUFriend | 1.3.1 | CPU frequency/power management |
| CPUFriendDataProvider | 1.0.0 | CPU power data (machine-specific) |
| CpuTopologyRebuild | 2.0.2 | Fix CPU topology for P/E-core CPUs — **critical for 14th Gen** |
| NVMeFix | 1.1.4 | NVMe power management fix (optional, improves sleep) |
| RestrictEvents | 1.1.7 | Block unwanted system events/notifications |
| USBToolBox | 1.2.0 | USB port mapping framework |
| UTBMap | 1.1 | USB port map data (machine-specific) |

### Disabled (in config.plist, still on disk)

| Kext | Version | Why disabled |
|------|---------|-------------|
| SMCRadeonGPU | 0.3.3 | GPU temp/fan monitoring — disabled (optional, not currently used) |
| IntelBluetoothFirmware | 2.5.0 | Intel AX211 BT firmware loader (fork build; OpenIntelWireless official latest is 2.4.0) — disabled 2026-08-27, BIOS BT off so no hardware present |
| IntelBTPatcher | 2.5.0 | Patches `AppleBluetoothHCIController` so AX211 binds (fork build) — disabled with the rest of the BT stack |
| BlueToolFixup | 2.7.1 | macOS 12+ `bluetoothd` handshake fix (BrcmPatchRAM bundle), MinKernel=21.0.0, needs Lilu — disabled with the rest of the BT stack |

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
| ToggleSipEntry.efi | SIP toggle (current `csr-active-config` = `0x0A03`) |

## CONVENTIONS

- **Kext versions tracked** in config.plist Comment fields (e.g., `<string>V1.7.1</string>`) — update these when upgrading kexts
- **CPUID spoofing active**: Cpuid1Data/Mask emulates older CPU for macOS compatibility (MinKernel 20.0.0 = Big Sur+)
- **Boot args**: `keepsyms=1 agdpmod=pikera` — `agdpmod=pikera` swaps board-id for AGDP, **required** for Navi 21 GPUs to prevent black screen at boot completion. (`debug=0x100` was dropped at some point before 2026-08-28.)
- **Language**: zh-Hans (Simplified Chinese)
- **Picker**: External mode (OpenCanopy GUI), 1s timeout, show picker enabled

## ANTI-PATTERNS

- **NEVER** edit kext binaries directly — update whole .kext bundle from Acidanthera releases
- **NEVER** change kext load order without understanding dependencies (Lilu must be first)
- **NEVER** update OpenCore.efi without also updating config.plist schema — breaking changes between versions
- **NEVER** modify PlatformInfo serials without regenerating via GenSMBIOS (iServices will break)
- **DO NOT** enable USBPortsMap.kext alongside USBToolBox/UTBMap — they conflict
- **DO NOT** remove SSDT-GPRW without also removing the GPRW→XPRW ACPI patch in config.plist
- **DO NOT** enable NootRX alongside WhateverGreen — they conflict on Navi GPU patching
- **DO NOT** enable SMCRadeonGPU without verifying WhateverGreen compatibility on Navi 21
- Backup config.plist before ANY change (oldConfig.plist exists for this reason)
- **DO NOT** disable BlueToolFixup **while Intel BT hardware is active** on macOS 12+ — `bluetoothd` refuses to bind Intel BT without it (silent BT failure). Irrelevant while BIOS BT is off, which is the current state.
- **DO NOT** expect AirDrop / Handoff / iPhone-unlock on Intel AX211 BT — Continuity requires a Broadcom BT chip; swap to BCM94360 M.2 / USB BCM dongle for full Continuity
- **DO NOT** use upstream OpenIntelWireless `IntelBTPatcher` v2.4.0 on macOS Tahoe 26+ — Lilu plugin max kernel is Sonoma; use `lshbluesky` fork v2.5.1 or newer Tahoe-built release
- **DO NOT** remove `bluetoothExternalDongleFailed` + `bluetoothInternalControllerInfo` from NVRAM/**Add** — BlueToolFixup needs them zero-initialized to inject spoofed Broadcom controller info; without them System Report shows `Address: NULL`. They sat in NVRAM/Delete only (never re-added) until 2026-08-28; `nvram` confirmed both variables absent at runtime.
- **DO NOT** disable `XhciPortLimit` on this build. OpenCore 1.0.7 ships a Tahoe-specific patch selected for Darwin 25.0+ (PR #590, release note: *"Improved XhciPortLimit compatibility with macOS Tahoe"*). Dortania's "broken since macOS 11.3" guidance predates PR #471 (2023) and PR #590 (2025) and no longer applies. The 23-port map **requires** this quirk to be true.
- **DO NOT** map any port where `port & 0xF == 0` (e.g. port 16). locationID encodes the port number in 4 bits (`portNumber & 0xF` shifted left 20), so port 16 collapses to `0x14000000` — which is the XHCI root hub's own locationID. `HS16` was removed for exactly this reason on 2026-08-28. Ports 17–24 (SS01–SS08) are fine: they alias onto 1–8, which are their own USB2 companions and are never populated simultaneously.
- **NEVER** `launchctl disable` / `bootout` the Bluetooth daemons to silence idle BT processes. `com.apple.bluetoothd` registers 15 Mach services including `com.apple.BTAudioHALPlugin.xpc`, `com.apple.server.bluetooth.general.xpc` and `com.apple.bluetooth.xpc`. Removing them sends coreaudiod, ContextStoreAgent, WindowServer and audiomxd into a **no-backoff bootstrap lookup loop**: measured ~37,000 failed lookups/sec, launchd at 136% CPU, logd at 40%, coreaudiod leaking ~5 GB/h (peaked at 25.5 GB). Recovery needs `launchctl enable` **plus** `launchctl bootstrap` — `enable` alone leaves the job unloaded (`kickstart` fails with exit 113). Verified 2026-08-28.

## UNIQUE DETAILS

- **UTBMap variants**: `0Info.plist`, `1Info.plist`, `allInfo.plist` in UTBMap.kext — different USB port configurations. Active one is `Info.plist`. Swap by renaming to test different mappings. Note: all three variants omit HS14–HS16 entirely (HS14 is the Bluetooth port) and carry 21 ports on `_SB.PC00.XHCI`; only the active `Info.plist` has had HS16 surgically removed.
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
7. Commit with version tag (current pattern: `1.0.7`, `1.0.6`, `1.0.4`, `1.0.3`)

## GIT

- Versioned with tags (`1.0.7`, `1.0.6`, `1.0.4`, `1.0.3`)
- Commit messages: version bumps + kext upgrades + fixes
- No CI/CD, no scripts — manual EFI management

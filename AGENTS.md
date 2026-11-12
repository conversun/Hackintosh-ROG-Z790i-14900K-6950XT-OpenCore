# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-02
**Last verified:** 2026-11-12 — OpenCore 1.0.8 (`REL-108-2026-09-27`) on macOS 26.7.3 Tahoe (25G342)
**EFI baseline commit:** ab72ded
**Branch:** main

## OVERVIEW

OpenCore 1.0.8 EFI configuration for Hackintosh. Hardware: ROG STRIX Z790-I GAMING WIFI + Intel 14900K + AMD Radeon RX 6950 XT + Intel AX211. SMBIOS spoofed as MacPro7,1. Current wireless builds target Tahoe, not Sequoia.

## STRUCTURE

```
OC/
├── config.plist          # Main OpenCore config (THE critical file)
├── oldConfig.plist       # Historical config backup; not a full rollback EFI for the current kext set
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
| GPU issues | `config.plist` → Kernel/Add (WhateverGreen) + `ACPI/SSDT-RX6950XT.aml` | WhateverGreen 1.7.2 handles Navi 21 patching; **requires `agdpmod=pikera`** to prevent black screen at boot completion |
| USB mapping | `Kexts/UTBMap.kext/Contents/Info.plist` | 23 ports on `_SB.PC00.XHCI` (HS16 removed 2026-08-28, see ANTI-PATTERNS). Map exceeds 15 ports, so it **depends on `XhciPortLimit=true`** |
| CPU power mgmt | `Kexts/CPUFriendDataProvider.kext` + `ACPI/SSDT-PLUG-ALT.aml` | CpuTopologyRebuild also active |
| NVMe blocking | `ACPI/SSDT-BLOCK-NVME.aml` | Blocks specific NVMe from native Apple driver |
| Thunderbolt | `ACPI/SSDT-TB3.aml` + `ACPI/SSDT-DTPG.aml` | TB3 hotplug support |
| SMBIOS/serial | `config.plist` → PlatformInfo/Generic | MacPro7,1 with custom MLB/Serial/UUID |
| SIP toggle | Boot picker → ToggleSipEntry | Actual `csr-active-config` is `0x0A03` (`AwoAAA==`) = UNTRUSTED_KEXTS + UNRESTRICTED_FS + UNAPPROVED_KEXTS + UNAUTHENTICATED_ROOT |
| Bluetooth (Intel AX211) | **Active** — IntelBluetoothFirmware + BlueToolFixup + BTLEAdvFixup enabled | IntelBTPatcher is disabled because BTLEAdvFixup replaces its hook. Preserve the zero-initialized Bluetooth NVRAM/Add keys. Historical intermittent USB loss was not re-evaluated in this test. |
| Wi-Fi / AirDrop | ForbiddenItlwm 0.4.1-beta1 + BTLEAdvFixup 0.8.3, Tahoe builds | Bidirectional iPhone AirDrop confirmed 2026-11-12. Mac must stay associated with Wi-Fi for this version's AWDL listener. See README for flags, checks and channel limitations. |

## ACPI PATCHES (all enabled)

| SSDT | Purpose |
|------|---------|
| SSDT-RTCAWAC | Disable AWAC clock, enable legacy RTC |
| SSDT-EC | Fake EC device |
| SSDT-USBX | USB power properties |
| SSDT-PLUG-ALT | CPU power management plugin type |
| SSDT-GPRW | Fix instant wake (rename GPRW→XPRW) |
| SSDT-TB3 | Thunderbolt 3 hotplug |
| SSDT-DTPG | Device Tree Property Gateway for TB3 |
| SSDT-SBUS | SMBus device |
| SSDT-RX6950XT | GPU-specific patches for RX 6950 XT |
| SSDT-BLOCK-NVME | Block specific NVMe device |

## KEXTS (load order matters — Lilu MUST load first)

### Enabled

| Kext | Version | Role |
|------|---------|------|
| Lilu | 1.7.3 | Patching framework — **dependency for WhateverGreen, VirtualSMC, RestrictEvents** |
| VirtualSMC | 1.3.9 | SMC emulation (core) |
| WhateverGreen | 1.7.2 | AMD Navi GPU driver (RX 6950 XT) — requires `agdpmod=pikera` boot-arg |
| SMCProcessor | 1.3.9 | CPU sensor data (optional, monitoring only) |
| SMCSuperIO | 1.3.9 | Fan/voltage sensor data (optional, monitoring only) |
| CPUFriend | 1.3.1 | CPU frequency/power management |
| CPUFriendDataProvider | 1.0.0 | CPU power data (machine-specific) |
| CpuTopologyRebuild | 2.0.3 | Fix CPU topology for P/E-core CPUs — **critical for 14th Gen** |
| NVMeFix | 1.1.4 | NVMe power management fix (optional, improves sleep) |
| RestrictEvents | 1.1.7 | Block unwanted system events/notifications |
| USBToolBox | 1.2.0 | USB port mapping framework |
| UTBMap | 1.1 | USB port map data (machine-specific) |
| DisplayBrightness | 0.0.42 | Display brightness support, Darwin 25.x only |
| IntelBluetoothFirmware | 2.5.1 | Intel AX211 Bluetooth firmware |
| BlueToolFixup | 2.7.2 | macOS Bluetooth compatibility, MinKernel=21.0.0, needs Lilu |
| BTLEAdvFixup | 0.8.3 | Bluetooth LE advertising and IntelBTPatcher replacement; Tahoe build, Darwin 25.x only |
| ForbiddenItlwm | 0.4.1-beta1 | Wi-Fi and AWDL; Tahoe build, Darwin 25.x only; CFBundleVersion=0.4.1 |

### Disabled (in config.plist, still on disk)

| Kext | Version | Why disabled |
|------|---------|-------------|
| IntelBTPatcher | 2.5.1 | Required to stay disabled while BTLEAdvFixup is enabled: both hook the same function |

### Not part of the current configuration

AirportItlwm, itlwm, SMCRadeonGPU and the legacy USBPortsMap are not in the current Kernel/Add list or active kext directory. Do not reintroduce conflicting Wi-Fi or USB mapping drivers.

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
- **Boot args**: `keepsyms=1 agdpmod=pikera` plus the complete ForbiddenItlwm/BTLEAdvFixup set documented in README. `agdpmod=pikera` is **required** for Navi 21 GPUs to prevent black screen at boot completion. `boot-args` is also in NVRAM/Delete so the new value replaces the old one.
- **Language**: zh-Hans (Simplified Chinese)
- **Picker**: External mode (OpenCanopy GUI), 3s timeout, show picker enabled

## ANTI-PATTERNS

- **NEVER** edit kext binaries directly — update whole .kext bundles from their respective project releases
- **NEVER** change kext load order without understanding dependencies (Lilu must be first)
- **NEVER** update OpenCore.efi without also updating config.plist schema — breaking changes between versions
- **NEVER** modify PlatformInfo serials without regenerating via GenSMBIOS (iServices will break)
- **DO NOT** enable USBPortsMap.kext alongside USBToolBox/UTBMap — they conflict
- **DO NOT** remove SSDT-GPRW without also removing the GPRW→XPRW ACPI patch in config.plist
- **DO NOT** enable NootRX alongside WhateverGreen — they conflict on Navi GPU patching
- **DO NOT** enable SMCRadeonGPU without verifying WhateverGreen compatibility on Navi 21
- Backup config.plist before ANY change (oldConfig.plist exists for this reason)
- **DO NOT** disable BlueToolFixup **while Intel BT hardware is active** on macOS 12+ — `bluetoothd` refuses to bind Intel BT without it (silent BT failure).
- **DO NOT** load AirportItlwm or itlwm alongside ForbiddenItlwm, or IntelBTPatcher alongside BTLEAdvFixup. BTLEAdvFixup replaces IntelBTPatcher's hook; loading both can panic.
- **DO NOT** assume Intel AX211 cannot do AirDrop: ForbiddenItlwm adds AWDL, and bidirectional transfer was confirmed here. Handoff and Universal Clipboard are upstream-reported, not locally verified; do not claim all Continuity features work.
- **DO NOT** treat Wi-Fi power-on as sufficient for this version: the Mac must be associated with a Wi-Fi network for the AWDL listener. Ethernet can remain connected.
- **DO NOT** force channel 64 by changing a boot argument: it is excluded by this release's AWDL selection and fixed-schedule whitelist. Firmware permissions and peer interoperability need separate validation. Normal access-point use of channel 64 is a different mode.
- **Before macOS updates**, boot once without `-btleadv` or disable BTLEAdvFixup, then check the new OS build before re-enabling. Use the native macOS Wi-Fi stack, without OCLP Modern Wireless root patches.
- If reverting to AirportItlwm, disable BTLEAdvFixup before restoring a Tahoe-compatible IntelBTPatcher. Do not use upstream IntelBTPatcher v2.4.0 on Tahoe.
- **DO NOT** remove `bluetoothExternalDongleFailed` + `bluetoothInternalControllerInfo` from NVRAM/**Add** — BlueToolFixup needs them zero-initialized to inject spoofed Broadcom controller info; without them System Report shows `Address: NULL`. They sat in NVRAM/Delete only (never re-added) until 2026-08-28; `nvram` confirmed both variables absent at runtime.
- **DO NOT** disable `XhciPortLimit` on this build. OpenCore 1.0.7 ships a Tahoe-specific patch selected for Darwin 25.0+ (PR #590, release note: *"Improved XhciPortLimit compatibility with macOS Tahoe"*). Dortania's "broken since macOS 11.3" guidance predates PR #471 (2023) and PR #590 (2025) and no longer applies. The 23-port map **requires** this quirk to be true.
- **DO NOT** map any port where `port & 0xF == 0` (e.g. port 16). locationID encodes the port number in 4 bits (`portNumber & 0xF` shifted left 20), so port 16 collapses to `0x14000000` — which is the XHCI root hub's own locationID. `HS16` was removed for exactly this reason on 2026-08-28. Ports 17–24 (SS01–SS08) are fine: they alias onto 1–8, which are their own USB2 companions and are never populated simultaneously.
- **NEVER** `launchctl disable` / `bootout` the Bluetooth daemons to silence idle BT processes. `com.apple.bluetoothd` registers 15 Mach services including `com.apple.BTAudioHALPlugin.xpc`, `com.apple.server.bluetooth.general.xpc` and `com.apple.bluetooth.xpc`. Removing them sends coreaudiod, ContextStoreAgent, WindowServer and audiomxd into a **no-backoff bootstrap lookup loop**: measured ~37,000 failed lookups/sec, launchd at 136% CPU, logd at 40%, coreaudiod leaking ~5 GB/h (peaked at 25.5 GB). Recovery needs `launchctl enable` **plus** `launchctl bootstrap` — `enable` alone leaves the job unloaded (`kickstart` fails with exit 113). Verified 2026-08-28.

## UNIQUE DETAILS

- **Wireless evidence (2026-11-12)**: ForbiddenItlwm 0.4.1, BTLEAdvFixup 0.8.3, IntelBluetoothFirmware 2.5.1 and BlueToolFixup 2.7.2 loaded; `ItlwmAWDLBuildCheck=verified`, `BTLEAdvHookRouted=1`. Joining Wi-Fi changed `ITLAWDLCoexAdded` from No to Yes, with `MultiState=own-channel`, channel 6, width 20 MHz and `CountryLearned=Yes`; the user then confirmed sending and receiving with an iPhone. The cause of channel 149 being unavailable is not established. Sleep/wake with this wireless stack was not tested. Surge was present during diagnosis, but disabling it did not resolve the failure according to the user; do not record Surge as the proven cause.
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
7. When a commit/release is requested, use the OpenCore version as the release tag; do not overwrite existing tags.

## GIT

- Releases use OpenCore version tags (previous published release: `1.0.6`; current bootloader: `1.0.8`)
- Commit messages: version bumps + kext upgrades + fixes
- No CI/CD, no scripts — manual EFI management

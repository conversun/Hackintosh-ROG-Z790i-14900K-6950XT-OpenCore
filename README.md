# Hackintosh EFI — ROG STRIX Z790-I + i9-14900K + RX 6950 XT

OpenCore 1.0.7 EFI for macOS Sequoia / Tahoe on ASUS ROG STRIX Z790-I GAMING WIFI.

## Hardware

| Component | Model |
|-----------|-------|
| Motherboard | ASUS ROG STRIX Z790-I GAMING WIFI |
| CPU | Intel Core i9-14900K |
| GPU | AMD Radeon RX 6950 XT |
| SMBIOS | MacPro7,1 |

## What Works

- macOS Sequoia (15.x) and Tahoe (26.x)
- GPU acceleration (WhateverGreen)
- USB mapping (USBToolBox + UTBMap, 23 ports — requires `XhciPortLimit=true`)
- CPU power management (CPUFriend + CpuTopologyRebuild for P/E-cores)
- Sleep/Wake (GPRW patch)
- Thunderbolt 3 hotplug
- NVMe drives
- OpenCanopy GUI boot picker

**Not enabled:** Bluetooth. The Intel AX211 BT stack (IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup) is on disk but disabled, and Bluetooth is turned off in BIOS. See Notes.

## ACPI

| SSDT | Purpose |
|------|---------|
| SSDT-RTCAWAC | Disable AWAC clock, enable legacy RTC |
| SSDT-EC | Fake Embedded Controller |
| SSDT-USBX | USB power properties |
| SSDT-PLUG-ALT | CPU power management plugin type |
| SSDT-GPRW | Fix instant wake (GPRW→XPRW rename) |
| SSDT-TB3 | Thunderbolt 3 hotplug |
| SSDT-DTPG | Device Tree Property Gateway for TB3 |
| SSDT-RX6950XT | GPU-specific patches |
| SSDT-SBUS | SMBus device |
| SSDT-BLOCK-NVME | Block specific NVMe from native driver |

## Kexts

| Kext | Version | Role |
|------|---------|------|
| Lilu | 1.7.2 | Patching framework (must load first) |
| VirtualSMC | 1.3.8 | SMC emulation |
| WhateverGreen | 1.7.1 (dev) | AMD GPU driver (RX 6950 XT, Navi 21) — requires `agdpmod=pikera` |
| SMCProcessor | 1.3.8 | CPU sensor data |
| SMCSuperIO | 1.3.8 | Fan/voltage sensor data |
| CPUFriend | 1.3.1 | CPU frequency/power management |
| CPUFriendDataProvider | 1.0.0 | CPU power data (machine-specific) |
| CpuTopologyRebuild | 2.0.2 | P/E-core topology fix (14th Gen) |
| NVMeFix | 1.1.4 | NVMe power management |
| RestrictEvents | 1.1.7 | Block unwanted system events |
| USBToolBox | 1.2.0 | USB port mapping framework |
| UTBMap | 1.1 | USB port map (machine-specific) |

## Drivers

| Driver | Purpose |
|--------|---------|
| OpenRuntime.efi | Runtime services |
| OpenCanopy.efi | GUI boot picker |
| HfsPlus.efi | HFS+ filesystem |
| ResetNvramEntry.efi | NVRAM reset tool |
| ToggleSipEntry.efi | SIP toggle |

## Usage

1. **Generate your own SMBIOS** — Do not use my serials. Use [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) to generate `MacPro7,1` values and update `config.plist → PlatformInfo/Generic`.
2. Copy the `BOOT` and `OC` folders to your EFI partition.
3. Verify BIOS settings match [Dortania's guide](https://dortania.github.io/OpenCore-Install-Guide/) for Z790.

## Notes

- `oldConfig.plist` is a backup config kept for rollback.
- `UTBMap.kext` contains alternate port maps (`0Info.plist`, `1Info.plist`, `allInfo.plist`) — swap with `Info.plist` to test different USB configurations. The active map has `HS16` removed: port 16 yields locationID `0x14000000`, which collides with the XHCI root hub (locationID encodes the port in only 4 bits).
- WhateverGreen is the active GPU driver. `agdpmod=pikera` boot-arg is **required** for Navi 21 (without it: black screen at boot completion). NootRX is an alternative driver — do not enable both simultaneously if you switch.
- Boot args: `keepsyms=1 agdpmod=pikera`
- `XhciPortLimit` is **true** and must stay true — the port map exceeds 15 ports. OpenCore 1.0.7 carries a Tahoe-specific (Darwin 25.0+) patch for this quirk; the widely-repeated "XhciPortLimit is broken since macOS 11.3" advice predates OpenCorePkg PR #471 and PR #590.
- Bluetooth is disabled in BIOS and all three BT kexts are `Enabled=false`. Do **not** try to silence the idle `bluetoothd` / `BlueTool` processes with `launchctl disable` — `bluetoothd` owns the `com.apple.BTAudioHALPlugin.xpc` and `com.apple.server.bluetooth.general.xpc` Mach services, and removing them puts coreaudiod and ContextStoreAgent into a no-backoff lookup loop that pins several cores and leaks gigabytes per hour.

## Credits

- [Acidanthera](https://github.com/acidanthera) — OpenCore, Lilu, VirtualSMC, WhateverGreen, and more
- [Dortania](https://dortania.github.io/OpenCore-Install-Guide/) — OpenCore Install Guide
- [USBToolBox](https://github.com/USBToolBox) — USB mapping

# Hackintosh EFI — ROG STRIX Z790-I + i9-14900K + RX 6950 XT

OpenCore 1.0.6 EFI for macOS Sequoia / Tahoe on ASUS ROG STRIX Z790-I GAMING WIFI.

## Hardware

| Component | Model |
|-----------|-------|
| Motherboard | ASUS ROG STRIX Z790-I GAMING WIFI |
| CPU | Intel Core i9-14900K |
| GPU | AMD Radeon RX 6950 XT |
| SMBIOS | MacPro7,1 |

## What Works

- macOS Sequoia (15.x) and Tahoe (26.x)
- GPU acceleration (NootRX)
- USB mapping (USBToolBox + UTBMap)
- CPU power management (CPUFriend + CpuTopologyRebuild for P/E-cores)
- Sleep/Wake (GPRW patch)
- Thunderbolt 3 hotplug
- NVMe drives
- OpenCanopy GUI boot picker

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
| NootRX | 1.0.0 | AMD Navi 21 GPU driver (RX 6950 XT) |
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
- `UTBMap.kext` contains alternate port maps (`0Info.plist`, `1Info.plist`, `allInfo.plist`) — swap with `Info.plist` to test different USB configurations.
- WhateverGreen is disabled in favor of NootRX. Do not enable both simultaneously.
- Boot args: `-v debug=0x100 keepsyms=1 -ctrsmt revpatch=sbvmm`

## Credits

- [Acidanthera](https://github.com/acidanthera) — OpenCore, Lilu, VirtualSMC, NootRX, and more
- [Dortania](https://dortania.github.io/OpenCore-Install-Guide/) — OpenCore Install Guide
- [USBToolBox](https://github.com/USBToolBox) — USB mapping

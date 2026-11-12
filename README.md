# Hackintosh EFI — ROG STRIX Z790-I + i9-14900K + RX 6950 XT

OpenCore 1.0.8 EFI for ASUS ROG STRIX Z790-I GAMING WIFI.

Last verified on **macOS Tahoe 26.7.3 (25G342)**, 2026-11-12. The bundled ForbiddenItlwm and BTLEAdvFixup are the **Tahoe builds**, limited to Darwin 25.x. This wireless configuration has not been tested on Sequoia.

## Hardware

| Component | Model |
|-----------|-------|
| Motherboard | ASUS ROG STRIX Z790-I GAMING WIFI |
| CPU | Intel Core i9-14900K |
| GPU | AMD Radeon RX 6950 XT |
| Wi-Fi / Bluetooth | Intel AX211 |
| SMBIOS | MacPro7,1 |

## What Works

- macOS Tahoe 26.7.3 boot
- GPU acceleration (WhateverGreen)
- USB mapping (USBToolBox + UTBMap, 23 ports — requires `XhciPortLimit=true`)
- CPU power management (CPUFriend + CpuTopologyRebuild for P/E-cores)
- Sleep/Wake fix (GPRW patch; not revalidated with the current wireless stack)
- Thunderbolt 3 hotplug
- NVMe drives
- OpenCanopy GUI boot picker
- Intel AX211 Wi-Fi and Bluetooth
- AirDrop to and from an iPhone, confirmed on this machine with the Mac connected to Wi-Fi

**Wireless is experimental:** ForbiddenItlwm 0.4.1-beta1 and BTLEAdvFixup 0.8.3 replace AirportItlwm and IntelBTPatcher. See [Wi-Fi, Bluetooth and AirDrop](#wi-fi-bluetooth-and-airdrop) for requirements and limitations.

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
| Lilu | 1.7.3 | Patching framework (must load first) |
| VirtualSMC | 1.3.9 | SMC emulation |
| WhateverGreen | 1.7.2 | AMD GPU driver (RX 6950 XT, Navi 21) — requires `agdpmod=pikera` |
| SMCProcessor | 1.3.9 | CPU sensor data |
| SMCSuperIO | 1.3.9 | Fan/voltage sensor data |
| CPUFriend | 1.3.1 | CPU frequency/power management |
| CPUFriendDataProvider | 1.0.0 | CPU power data (machine-specific) |
| CpuTopologyRebuild | 2.0.3 | P/E-core topology fix (14th Gen) |
| NVMeFix | 1.1.4 | NVMe power management |
| RestrictEvents | 1.1.7 | Block unwanted system events |
| USBToolBox | 1.2.0 | USB port mapping framework |
| UTBMap | 1.1 | USB port map (machine-specific) |
| DisplayBrightness | 0.0.42 | Display brightness support; Darwin 25.x only |
| IntelBluetoothFirmware | 2.5.1 | Intel Bluetooth firmware loader |
| BlueToolFixup | 2.7.2 | Bluetooth compatibility on macOS 12+ |
| BTLEAdvFixup | 0.8.3 | Intel Bluetooth LE advertising and IntelBTPatcher replacement; Tahoe build |
| ForbiddenItlwm | 0.4.1-beta1 | Native Wi-Fi and AWDL for AirDrop; Tahoe build (`CFBundleVersion` 0.4.1) |

`IntelBTPatcher.kext` 2.5.1 remains on disk with `Enabled=false`. Do not enable it alongside BTLEAdvFixup: both hook the same function. AirportItlwm and itlwm are not enabled.

## Drivers

| Driver | Purpose |
|--------|---------|
| OpenRuntime.efi | Runtime services |
| OpenCanopy.efi | GUI boot picker |
| OpenHfsPlus.efi | HFS+ filesystem |
| ResetNvramEntry.efi | NVRAM reset tool |
| ToggleSipEntry.efi | SIP toggle |

## Usage

1. **Generate your own SMBIOS** — Do not use my serials. Use [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) to generate `MacPro7,1` values and update `config.plist → PlatformInfo/Generic`.
2. Copy the `BOOT` and `OC` folders to your EFI partition.
3. Verify BIOS settings match [Dortania's guide](https://dortania.github.io/OpenCore-Install-Guide/) for Z790.
4. Test from a USB EFI first and keep a separate, known-working EFI for rollback. `oldConfig.plist` is a historical config, not a complete backup of the current driver set.

## Wi-Fi, Bluetooth and AirDrop

The enabled wireless kexts load in this relative order:

```text
Lilu -> IntelBluetoothFirmware -> BlueToolFixup -> BTLEAdvFixup -> ForbiddenItlwm
```

- **Keep the Mac associated with a Wi-Fi network.** In this version, the AWDL listener is added when the ordinary Wi-Fi connection enters its running state and removed when it leaves that state. Wi-Fi being switched on is not enough. On this machine, both directions stayed on "Waiting" while disconnected; connecting to Wi-Fi restored the listener and both transfers succeeded.
- Ethernet can remain connected. AirDrop uses a separate AWDL peer-to-peer link; the Mac and iPhone do not need to join the same access point.
- AirDrop was confirmed in both directions on this machine. Handoff and Universal Clipboard are reported working by the upstream project but were not tested here. Instant Hotspot, Continuity Camera and iPhone Mirroring are not supported by the current upstream release; Sidecar is untested.
- Do not load AirportItlwm or itlwm alongside ForbiddenItlwm, or IntelBTPatcher alongside BTLEAdvFixup.
- The driver expects the system's native Wi-Fi stack. Revert OCLP "Modern Wireless" root patches before using it.
- **Before updating macOS, boot once without `-btleadv`** (or with BTLEAdvFixup disabled). Its in-memory `bluetoothd` patches can interfere with incremental updates. Check the new macOS build against upstream compatibility notes before re-enabling it.

### Channels

Normal Wi-Fi and AWDL can use different channels. With the included `-itlawdlcoexmulti` configuration, the driver prefers **channel 149 at 80 MHz**, requiring firmware permission for all four subchannels (149, 153, 157 and 161), and otherwise falls back to **channel 6 at 20 MHz**.

The observed AWDL state after Wi-Fi association was `Added=Yes`, `CountryLearned=Yes`, channel 6 at 20 MHz. AirDrop then worked in both directions. The reason channel 149 was unavailable has not been established; learning a country does not by itself guarantee permission to use it.

Channel **64 can be used for an ordinary access-point connection**, but it is not supported as an independent AWDL channel in this release. It is absent from both the listener's channel choices and the fixed-schedule channel whitelist. The send path also rejects firmware DFS/passive/no-ad-hoc restrictions. Supporting 64 would require driver changes and firmware/iPhone interoperability tests, not a boot-argument change. No channel-64 support is included in this EFI.

### Boot arguments and checks

The complete configured `boot-args` value is:

```text
keepsyms=1 agdpmod=pikera -btleadv -itlawdlcoexprobe -itlawdliface -itlawdlnativeup -itlawdlcoexrx -itlawdlcoextx -itlawdlcoexmulti -itlawdlcoexdata -itlawdlmitx -itlawdlelicit -itlawdlcaps -itlawdlsvcmerge -itlawdlawsync itlawdlintcoal=4 -itlawdlcoexbatx -itlawdlcoexpeerpause -itlawdlcoexbatrace itlawdltxqueue=4096
```

Keep `boot-args` in `NVRAM/Delete` under the same Apple GUID as `NVRAM/Add`, so changes replace the previous value. After booting and joining Wi-Fi:

```bash
ioreg -l -w0 -d1 | grep '"BTLEAdvHookRouted"'
ioreg -l -w0 | grep -E '"ItlwmAWDL(BuildCheck|TransportReady)"|"ITLAWDLCoex(Added|Channel|OwnWidth|CountryLearned)"'
```

`BTLEAdvHookRouted` should be 1 and `BuildCheck` should be `verified` on the tested OS. If a transfer stays on "Waiting", check the Wi-Fi association and `ITLAWDLCoexAdded` first. Bluetooth discovery alone does not prove that the AWDL data link is ready.

Upstream: [ForbiddenItlwm release](https://github.com/retropineaple/ForbiddenItlwm/releases/tag/v0.4.1-beta1), [installation](https://github.com/retropineaple/ForbiddenItlwm/blob/v0.4.1-beta1/docs/INSTALL.md), [troubleshooting](https://github.com/retropineaple/ForbiddenItlwm/blob/v0.4.1-beta1/docs/TROUBLESHOOTING.md).

## Notes

- `oldConfig.plist` is a historical backup; keep a full EFI backup before changing drivers.
- `UTBMap.kext` contains alternate port maps (`0Info.plist`, `1Info.plist`, `allInfo.plist`) — swap with `Info.plist` to test different USB configurations. The active map has `HS16` removed: port 16 yields locationID `0x14000000`, which collides with the XHCI root hub (locationID encodes the port in only 4 bits).
- WhateverGreen is the active GPU driver. `agdpmod=pikera` boot-arg is **required** for Navi 21 (without it: black screen at boot completion). NootRX is an alternative driver — do not enable both simultaneously if you switch.
- `XhciPortLimit` is **true** and must stay true — the port map exceeds 15 ports. OpenCore 1.0.7 introduced a Tahoe-specific (Darwin 25.0+) patch, also present in 1.0.8; the widely-repeated "XhciPortLimit is broken since macOS 11.3" advice predates OpenCorePkg PR #471 and PR #590.
- Bluetooth is active with IntelBluetoothFirmware, BlueToolFixup and BTLEAdvFixup. Do **not** try to silence `bluetoothd` / `BlueTool` with `launchctl disable` — `bluetoothd` owns the `com.apple.BTAudioHALPlugin.xpc` and `com.apple.server.bluetooth.general.xpc` Mach services, and removing them puts coreaudiod and ContextStoreAgent into a no-backoff lookup loop that pins several cores and leaks gigabytes per hour.

## Credits

- [Acidanthera](https://github.com/acidanthera) — OpenCore, Lilu, VirtualSMC, WhateverGreen, and more
- [Dortania](https://dortania.github.io/OpenCore-Install-Guide/) — OpenCore Install Guide
- [USBToolBox](https://github.com/USBToolBox) — USB mapping
- [OpenIntelWireless](https://github.com/OpenIntelWireless) — Intel Wi-Fi and Bluetooth drivers
- [ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm) — Intel AWDL and Bluetooth LE advertising support

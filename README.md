# HP EliteBook 840 G7 — macOS Tahoe OpenCore EFI

OpenCore EFI for the **HP EliteBook 840 G7** running **macOS Tahoe 26** (tested on 26.6, build 25G72). The same EFI also covers Sequoia 15. Every included kext supports both versions, and the OS-specific Wi-Fi kexts are limited with `MinKernel`.

## Hardware

| Component | Details |
|---|---|
| CPU | Intel Core i5-10310U (Comet Lake) |
| iGPU | Intel UHD 620 (`0x9B41`), spoofed to `0x3E9B`, platform-id `0x3EA50009` |
| USB | Comet Lake PCH-LP XHCI (`0x02ED`) + Titan Ridge Thunderbolt XHCI (`0x15EC`) |
| Wi-Fi / BT | Intel (AirportItlwm / IntelBluetoothFirmware) |
| Audio | `layout-id` 21 |
| SMBIOS | `MacBookPro16,2` |

## Software

- **OpenCore 1.0.8** (RELEASE)
- Lilu 1.7.2, VirtualSMC 1.3.8, WhateverGreen 1.7.1, AppleALC 1.9.8, RestrictEvents 1.1.6
- VoodooPS2Controller 2.3.8, VoodooI2C 2.9.1 + VoodooI2CHID
- AirportItlwm 2.3.0 + IO80211FamilyLegacy + IOSkywalkFamily (downgrade), AMFIPass 1.4.1
- IntelBluetoothFirmware / IntelBTPatcher 2.5.0, BlueToolFixup 2.7.2
- ECEnabler, BrightnessKeys, NVMeFix, SMCBatteryManager, SMCProcessor, SMCLightSensor
- USBMap.kext: a USBToolBox map using the native `AppleUSBHostMergeProperties` class
- SIP partly disabled (`csr-active-config` = `0x00000A03`) and `SecureBootModel` = `Disabled`. The Wi-Fi root patch needs both.

## Before you boot: generate your own SMBIOS

The serial number, MLB, ROM and system UUID in `config.plist` are **intentionally blank**. Generate a `MacBookPro16,2` SMBIOS with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS), then fill in `PlatformInfo → Generic`:

- `SystemSerialNumber`, `MLB`, `SystemUUID`
- `ROM`: your network adapter's MAC address (6 bytes)

## Tahoe-specific notes

### USB map keys (the main fix)
macOS Tahoe identifies USB ports with **new property keys**. A map that has only the old keys is ignored. Because `SSDT-USB-Reset` disables `XHC.RHUB`, the result is no USB ports at all and **"Still waiting for root device"** in the installer.

Each port in `USBMap.kext` therefore carries both key sets:

| Older macOS | Tahoe | Type |
|---|---|---|
| `port` | `usb-port-number` | Data |
| `UsbConnector` | `usb-port-type` | Number |

Keeping both lets the same map work on Sequoia and earlier. If you remap with an older tool, add the Tahoe keys back.

### Other changes
- `XHCI-unsupported.kext` is **disabled**. The Comet Lake XHCI (`0x02ED`) is matched natively.
- The TXHC (Thunderbolt) personality matches on `pcidebug 5:0:0`, which can change with Thunderbolt state. To boot an installer or external drive, use the **USB-A ports**, which are on the main XHC.

### Installing
- **Use a full USB installer** made with `createinstallmedia`. Running `Install macOS Tahoe.app` from inside Sequoia to install onto another volume failed at the RAM-disk stage. `patchd` logged `MobileSoftwareUpdateErrorDomain 1015/1013 – could not retrieve update path` and the laptop rebooted every time.
- **Reset NVRAM** from the OpenCore picker before installing if earlier attempts failed. Stale installer state in NVRAM was flagged as "inconsistent" by the installer.
- Expect one to three automatic reboots into "macOS Installer".

## Status

| Feature | Status |
|---|---|
| Boots to Tahoe desktop | ✅ |
| Installer from USB | ✅ |
| Wi-Fi | ⚠️ Needs a root patch for `IO80211FamilyLegacy` after install, same as on Sequoia |
| Graphics acceleration / Metal | Not yet verified |
| Bluetooth, audio, sleep, trackpad gestures | Not yet verified on Tahoe |

## Debugging

For logs, swap in the OpenCore **DEBUG** binaries (OpenCore.efi, BOOTx64.efi and the OpenCore drivers, from the same release). Then set:

- `Misc → Debug → Target` = `67`, `AppleDebug` / `ApplePanic` = `true`
- add `-v debug=0x100` to boot-args

Logs are written to the root of the EFI partition (`opencore-*.txt`).

## Credits

[Acidanthera](https://github.com/acidanthera) (OpenCore, Lilu and plugins), [OpenIntelWireless](https://github.com/OpenIntelWireless), [VoodooI2C](https://github.com/VoodooI2C), [USBToolBox](https://github.com/USBToolBox), [CorpNewt](https://github.com/corpnewt) (SSDTTime, GenSMBIOS), and the [Dortania guide](https://dortania.github.io/OpenCore-Install-Guide/).

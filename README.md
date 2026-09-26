# MSI GF63 9RCX — macOS Sequoia Hackintosh (OpenCore)

OpenCore EFI for the **MSI GF63 9RCX**, running **macOS Sequoia (15)**.

| | |
|---|---|
| **CPU** | Intel Core i7-9750H (Coffee Lake Refresh) |
| **iGPU** | Intel UHD Graphics 630 |
| **dGPU** | Nvidia (hardware-disabled — not usable, see notes) |
| **WiFi / BT** | Intel Wireless-AC 9560 |
| **Ethernet** | Realtek RTL8111 |
| **Audio** | Realtek ALC codec |
| **SMBIOS** | `MacBookPro16,4` |
| **macOS tested** | Sequoia 15.x |

> ⚠️ **Status: Sequoia support is newly built and pending full verification.** This EFI carries over a hardware layer that was extensively tested and confirmed working on Ventura (13) first. Sequoia-specific items are marked below and will be updated as testing completes — please open an issue if you test this and can confirm/deny an item.

## ✅ Confirmed working (carried over from tested Ventura build)

| Feature | Status | Notes |
|---|---|---|
| Boot / display output | ✅ Working | Required specific boot-args, see below |
| CPU power management | ✅ Working | Real per-core P-state table via `CPUFriend`, idles correctly instead of floor-locked at base clock |
| Internal webcam | ✅ Working | Required a manual USB port-map fix — port `0x0b` (internal, no SuperSpeed pair) was missing from auto-generated maps |
| Fan control | ✅ Working | Custom fan curve applied at boot via `SSDT-FAN-MOD.aml` |
| USB ports | ✅ Working | Hand-verified physical port map, not auto-generated |
| WiFi | ✅ Working | `AirportItlwm` — shows in native WiFi menu, no companion app needed |
| Trackpad / keyboard | ✅ Working | VoodooI2C + VoodooPS2 |

## ⏳ Expected to work, pending Sequoia-specific confirmation

| Feature | Status | Notes |
|---|---|---|
| Internal audio (speakers/headphone jack) | ⏳ Untested | Sequoia still has `AppleHDA.kext` (unlike Tahoe, where it's removed entirely), so this should be fixable via `AppleALC` — not yet confirmed correct on this exact codec |
| Bluetooth pairing | ⏳ Untested | Works on Ventura; Sequoia not yet verified |
| OTA software updates | ⏳ Untested | Requires `RestrictEvents.kext` + `revpatch=sbvmm` boot-arg + `SecureBootModel: Disabled` (all present) — mechanism confirmed correct, an actual OTA update hasn't been tested yet |
| Sleep / wake | ⏳ Retest pending | Working on Ventura after disabling `SSDT-DDGPU`/`SSDT-WAK` (see Known Issues) — needs reconfirmation on Sequoia |

## ❌ Known broken / not fixable on this hardware

| Feature | Status | Why |
|---|---|---|
| Safari / native app hardware DRM (Netflix, Prime Video, Apple TV+, iTunes movies) | ❌ Broken, no fix exists | iGPU-only systems lack Apple's Management Engine certificate required for hardware DRM. Broken since macOS 10.12.3, applies to every macOS version, not specific to this EFI. **Workaround: use Chrome, Firefox, or Edge instead of Safari** — they use software DRM (Widevine) and work normally. |
| Discrete Nvidia GPU | ❌ Not usable | Hardware-disabled prior to acquiring this unit (prior repair). Not a macOS/EFI limitation. |

## 🔧 Boot-args explained

```
-v debug=0x100 keepsyms=1 -amfipassbeta -igfxblt igfxfw=2 igfxonln=1 igfxonlnfbs=0x01 -noDC9 igfxagdc=0 revpatch=sbvmm
```

| Flag | Purpose |
|---|---|
| `-v debug=0x100 keepsyms=1` | Verbose boot + debug symbols (diagnostic, safe to remove once stable) |
| `-amfipassbeta` | Pairs with `AMFIPass.kext` |
| `-igfxblt` | Fixes a Coffee Lake/Comet Lake backlight bug (macOS 13.4+) |
| `igfxfw=2` | Forces Apple's GuC firmware load for the iGPU — **required to fix a boot-time black screen on this hardware** |
| `igfxonln=1 igfxonlnfbs=0x01` | Fixes a Coffee Lake/Comet Lake display-wake-after-sleep bug (macOS 10.15.4+) |
| `-noDC9` | Prevents the iGPU entering its deepest display-power-off state — part of the black screen fix |
| `igfxagdc=0` | Disables Apple's automatic display-reconfiguration manager (AGDC), another black-screen contributor on this iGPU |
| `revpatch=sbvmm` | Required for OTA software updates on Sonoma 14.4+/Sequoia when Secure Boot is disabled |

**⚠️ Syntax matters:** `igfxfw=2`, `igfxonln=1`, and `igfxagdc=0` are key=value arguments — they must **not** have a leading `-`. A single misplaced dash silently breaks the fix with no error, and was the actual cause of an intermittent black screen during development of this EFI that looked like a hardware/timing issue but wasn't.

Also set: `ExitBootServicesDelay = 5000000` (5 sec) in `Misc → Boot` — added to resolve a random/intermittent black screen at boot, a real timing race condition separate from the syntax issue above.

## Why macOS Sequoia and not Tahoe

Tahoe (macOS 26) introduced a system-wide visual redesign ("Liquid Glass") with much heavier transparency/shadow compositing. On this hardware it produced a drop-shadow rendering bug, plausibly because Apple no longer meaningfully optimizes iGPU drivers for this CPU generation. Tahoe also completely removes `AppleHDA.kext` (no internal audio fix possible at all) and requires an additional `-ibtcompatbeta` boot-arg for Bluetooth. Sequoia predates the Liquid Glass redesign, still has `AppleHDA`, and is still within Apple's active security-patch window (unlike older versions such as Ventura/Monterey) — the best currently-known balance of stability and support for this exact hardware.

## Known issues / do not enable

- **`SSDT-DDGPU.aml`, `SSDT-WAK.aml`, and the `_WAK`→`ZWAK` ACPI rename patch are present in this EFI but intentionally left disabled.** Enabling them causes a sleep regression (display turns off but the machine doesn't actually enter S3 sleep) — confirmed via testing. Since the Nvidia dGPU is already hardware-disabled, `SSDT-DDGPU` serves no purpose anyway; there's no reason to re-enable this set.
- `SSDT-FAN-MOD.aml` is enabled standalone and does *not* depend on the above — safe as-is.

## Credits

- Hardware-specific ACPI/SMBIOS/USB-port-mapping baseline originally derived from an existing Catalina-era EFI built specifically for this exact laptop unit.
- `AirportItlwm.kext` version sourced from [rahulhingve/Hackintosh-MSI-GF63-Thin-9SCXR](https://github.com/rahulhingve/Hackintosh-MSI-GF63-Thin-9SCXR) (verified working with the same CPU/WiFi combo on Ventura before reuse here — WiFi drivers match by PCI hardware ID, not per-unit tuning, so this is safe to share across boards).
- Acidanthera and OpenIntelWireless projects for OpenCore, Lilu, WhateverGreen, VirtualSMC, AppleALC, CPUFriend, and itlwm/AirportItlwm.

## Disclaimer

This is a personal, hardware-specific configuration shared as-is. It is unlikely to work correctly on any GF63 9RCX unit other than the one it was built for without adjustments (USB port map and SMBIOS serials especially). Use at your own risk; back up your data before attempting any installation.


# HyperOS GSI for Samsung Galaxy A03

![Device](https://img.shields.io/badge/device-Samsung%20Galaxy%20A03-blue)
![Model](https://img.shields.io/badge/model-SM--A035F-blue)
![Platform](https://img.shields.io/badge/platform-Unisoc%20T606-purple)
![Android](https://img.shields.io/badge/Android-14-green)
![HyperOS](https://img.shields.io/badge/HyperOS-1.x-orange)
![Status](https://img.shields.io/badge/status-experimental-red)

A long-running compatibility, reverse-engineering and porting project focused on
running **Xiaomi HyperOS 1 (Android 14)** on the **Samsung Galaxy A03
(SM-A035F / UMS9230)**.

The project started as a simple GSI boot attempt and evolved into a full
userspace compatibility effort involving framework patching, vendor/HAL
compatibility, Binder/service shims, camera integration, Xiaomi Account
compatibility and the MIUI Theme system.

> **This is an experimental port.**
> It is not an official Xiaomi or Samsung build.

---

## Device

| Component | Value |
|---|---|
| Device | Samsung Galaxy A03 |
| Model | SM-A035F |
| SoC | Unisoc UMS9230 / T606 |
| Vendor base | Samsung Android 11 |
| Kernel | Linux 4.14.x |
| Target | HyperOS 1 / Android 14 |
| ROM type | GSI / cross-platform port |

---

# Project Goals

The primary goal is not simply to make HyperOS boot.

The goal is to make HyperOS behave like a usable daily Android system on
hardware it was never designed for.

Current work includes:

- Boot and init compatibility
- HyperOS framework compatibility
- Android 14 userspace on a legacy Android 11 vendor
- Camera HAL compatibility
- Xiaomi Account compatibility
- ThemeManager compatibility
- Local `.mtz` theme support (but not work well)
- Xiaomi-specific service dependency removal
- Runtime binary and Smali patching
- Reproducible image generation

---

# Why HyperOS is difficult on the A03

The Galaxy A03 and the HyperOS source device have fundamentally different
platform assumptions.

HyperOS was designed around a newer Xiaomi/MediaTek environment while the
A03 uses:

- Unisoc hardware
- a legacy 4.14 kernel
- Samsung Android 11 vendor components
- HIDL-based vendor interfaces
- different hardware services
- different device-specific framework assumptions

As a result, HyperOS may boot successfully but still expect Xiaomi- or
MediaTek-specific services that do not exist on the A03.

The project therefore uses a **first-failure / minimal-patch methodology**:

# ~~Edited: Accidently found an unwanted bug in every ThemeManager Patch: cannot click "Wallpaper & Personalization" in Settings, will fix it later.~~ Fixed it on ver T1.11! 


```
Boot
  ↓
observe
  ↓
identify first fatal event
  ↓
reverse engineer
  ↓
patch only the broken dependency
  ↓
runtime test
  ↓
preserve baseline
````

---

# Milestone History

## T1.5 — Initial HyperOS 1 Boot Compatibility

The first usable HyperOS 1 baseline required several isolated framework fixes.

### MiuiLightsService

Samsung's A03 does not expose the Xiaomi notification light expected by
HyperOS.

The original framework dereferenced a missing light object during startup,
causing a `NullPointerException`.

A null-safe fallback was added to `miui-services.jar`.

### File Descriptor Allowlist

HyperOS keeps framework resources under paths that differ from the AOSP
allowlist assumptions.

The affected `libandroid_runtime.so` logic was patched so valid framework
resources could remain open during the Zygote lifecycle.

After these fixes, HyperOS 1 was able to reach the graphical environment.

---

# T1.6 — Camera Compatibility

The LineageOS Camera application was used as a neutral test client.

The camera initially required approximately **30 seconds** to initialize.

The camera itself was functional, including:

* rear camera
* flash / torch
* camera provider
* camera HAL

The delay was traced to a Xiaomi-specific MediaTek ATMs service lookup:

```text
CameraImpl
    ↓
adjCameraPriority()
    ↓
vendor.mediatek.hardware.camera.atms.IATMs/default
    ↓
service does not exist on UMS9230
    ↓
repeated timeout
    ↓
~30 second delay
```

The compatibility patch makes the affected priority-management path return
without waiting for the unavailable service.

Runtime result:

```text
Before: ~29–30 s
After:  ~0.5 s
```

The camera HAL itself remained untouched.

---

# T1.7 — Xiaomi Account Compatibility

Mi Account login initially stalled during authentication.

The root cause was another Xiaomi-specific hardware dependency:

```
MIUIXiaomiAccount
    ↓
Xiaomi Passport
    ↓
MTD hardware service lookup
    ↓
service does not exist on Galaxy A03
    ↓
repeated wait
    ↓
"Checking password..."
```

The affected compatibility path was changed so the missing hardware service
does not block authentication.

Additional compatibility work included:

* account authenticator integration
* Google sign-in bridge
* FindDevice Android 14 receiver compatibility

The objective was to preserve the normal Xiaomi authentication flow instead
of faking successful authentication.
# Hold up, after this ver, all later ver make the mi login cannot be launch because the signature after patch have problems, will fix it later.
---

# T1.8 — ThemeManager Stability

The Themes / Wallpaper & Personalization UI initially crashed because of
incompatible assumptions in the ThemeManager activity stack.

Null-safe compatibility changes were introduced in the relevant ThemeManager
Smali classes.

After the fix:

```
Settings
  ↓
Wallpaper & Personalization
  ↓
Themes
  ↓
Theme list
```

could be opened without the original crash.

A separate issue then became visible:

```
ThemeDetailActivity
    ↓
Xiaomi Theme API
    ↓
empty / rejected response
    ↓
"Can't load themes."
```

This was treated as a separate online-theme problem rather than mixing it with
the original crash fix.

---

# Local `.mtz` Theme Support

HyperOS ThemeManager supports local theme import, but custom `.mtz` files are
subject to Xiaomi's theme validation system.

The tested flow is:

```
.mtz
  ↓
ThemeManager import
  ↓
theme assets
  ↓
/data/system/theme/
  ↓
theme validation
  ↓
SystemUI / Launcher reload
```

The custom theme used during development was successfully unpacked and its
assets were applied to the system.

The project also investigated Xiaomi's local theme rights / validation layer,
which checks theme asset hashes against the installed rights database.

---

# T1.10 / Theme Forensics

The theme stack was reverse engineered down to:

* ThemeManager APK
* ThemeManager Smali
* local theme database
* theme metadata
* `.mrm` metadata
* `.mrc` content
* preview assets
* `/data/system/theme/`
* `miui.drm.DrmManager`
* theme validation tasks

This work allows local themes to be treated as a normal compatibility problem
instead of depending entirely on Xiaomi's online Theme Store.

But why I still upload and recommended T1.11 before T1.12->later release? Cuz the theme patching (T1.8-T1.10) have a bug that cannot open theme (and Wallpaper & Personalization) apps through setting and else. T1.11 fixed it correctly.
---

# T1.12 — Generic AOSP RIL Compatibility

T1.12 was the first successful attempt to make HyperOS use the standard Android
radio stack provided by the Samsung/Unisoc platform instead of the missing
MediaTek-specific radio services.

### Original failure

```text
MtkTeleService
    ↓
MtkRIL / MtkRadioExProxy
    ↓
vendor.mediatek.hardware.radio
    ↓
NoSuchElementException
```

The A03 does not provide the MediaTek radio service expected by HyperOS.

### Patch

The first live solution made:

```text
com.android.internal.telephony.TelephonyComponentFactory
    └── injectTheComponentFactory(XmlResourceParser)
            ↓
        return-void
```

This disabled HyperOS' MTK component-factory injection and allowed the existing
generic AOSP RIL in `telephony-common.jar` to initialize.

The generic RIL successfully connected to:

```text
android.hardware.radio@1.5::IRadio/slot1
```

### Result

Verified on the live device:

- PhoneFactory initialization
- Generic RIL initialization
- `setResponseFunctions`
- SIM detection
- LTE registration
- signal reporting
- Samsung/Unisoc RIL communication

The original `MtkRadioExProxy` / MediaTek HAL failure path was eliminated.

### Limitation

Disabling the whole component-factory injection introduced a second problem:
HyperOS-specific code expected `MtkUiccController`, while the framework now
created the generic AOSP `UiccController`.

This became the basis for T1.13.

---

# T1.13 — Hybrid MTK Components + Generic AOSP RIL

T1.13 refined T1.12 instead of replacing the whole telephony architecture.

### Problem

With T1.12, HyperOS components such as:

```text
MtkPhoneInterfaceManagerEx
MtkTelephonyManagerEx
TelecomAccountRegistry
```

could receive the generic:

```text
UiccController
```

instead of the expected:

```text
MtkUiccController
```

This caused a runtime:

```text
ClassCastException
UiccController cannot be cast to MtkUiccController
```

which made `com.android.phone` restart and caused
**Settings → Mobile Networks** to crash.

### Architecture

T1.13 restores the MediaTek component factory, but changes only the RIL
creation path:

```text
MtkTelephonyComponentFactory
    ├── MtkUiccController
    ├── MTK SIM / phonebook components
    ├── other HyperOS-required MTK components
    │
    └── makeRil()
            ↓
        Generic AOSP RIL
            ↓
        android.hardware.radio@1.5
            ↓
        Samsung / Unisoc rild
```

The important design change is:

> Preserve MTK framework components that HyperOS still depends on, while
> replacing only the hardware-incompatible MTK RIL client.

### Result

T1.13 restored:

- stable `com.android.phone`
- `MtkUiccController`
- SIM-related framework functionality
- Settings → Mobile Networks
- LTE registration
- generic AOSP RIL operation

This became the hybrid telephony architecture used for the next milestone.

---

# T1.14 — SIM Power-State Compatibility / Voice Call Fix

After T1.13, the telephony framework and Mobile Networks UI were stable, but
normal outgoing calls still failed.

The Dialer displayed:

```text
Mobile network not available
```

### Root Cause

The call was failing **before it reached the RIL**.

HyperOS `TelecomAccountRegistry` relies on:

```text
MtkTelephonyManagerEx.getSimOnOffState()
```

and expects the MTK SIM power-state value:

```text
SIM_POWER_STATE_SIM_ON = 0x0B
```

The compatibility path was returning:

```text
0x01
```

instead.

Because the expected value did not match, the real SIM PhoneAccount was not
registered correctly. The call therefore fell back to:

```text
subId = -1
phone_id = -1
Phone = null
```

and failed before `RIL.dial()`.

### Patch

Only the return constant of:

```text
com.mediatek.telephony.MtkTelephonyManagerEx
    └── getSimOnOffState(int)
```

was changed:

```smali
- const/4 v0, 0x1
+ const/16 v0, 0xb
  return v0
```

No change was made to:

- Generic RIL
- Radio HAL
- vendor RIL
- modem firmware
- kernel
- telephony APK signatures

### Result

The SIM PhoneAccount could be registered using the value expected by the
HyperOS telephony layer, allowing the normal call path to continue through:

```text
PhoneAccount
    ↓
GsmCdmaPhone
    ↓
GsmCdmaCallTracker
    ↓
RIL.dial()
    ↓
android.hardware.radio@1.5
```

T1.14 therefore completed the transition from:

```text
HyperOS MTK RIL
```

to a hybrid:

```text
HyperOS MTK framework components
            +
Generic AOSP RIL
            +
Samsung / Unisoc radio HAL
```

---

# Telephony Milestone Summary

| Milestone | Main change | Main result |
|---|---|---|
| **T1.12** | Disabled global MTK component-factory injection | Generic AOSP RIL reached `IRadio@1.5` |
| **T1.13** | Restored MTK factory, changed only RIL creation | MTK UICC compatibility + Generic AOSP RIL |
| **T1.14** | Fixed `SIM_POWER_STATE_SIM_ON` compatibility value | Normal SIM PhoneAccount / call path restored |

# Forensics Methodology

The project relies heavily on reproducible runtime evidence.

Typical workflow:

```text
1. Reproduce
2. Clear logcat
3. Capture crash / binder / service state
4. Identify first fatal event
5. Locate implementation
6. Disassemble / decompile
7. Patch minimum code path
8. Bind-mount live test
9. Verify runtime behavior
10. Package final image
```

Useful tools include:

* `adb`
* `fastboot`
* `debugfs`
* `e2fsck`
* `llvm-objdump`
* `capstone`
* `baksmali`
* `smali`
* `readelf`
* `strings`
* `sqlite3`
* Python
* Android framework / AOSP sources

---

# Live Patching

During development, most userspace experiments are first tested using
temporary bind mounts rather than immediately rebuilding and flashing the
whole image.

Example:

```bash
adb push patched.apk /data/local/tmp/
adb shell mount -o bind \
    /data/local/tmp/patched.apk \
    /system/product/app/Target/Target.apk
```

After runtime verification, the proven modification can be integrated into
the final system image.

This greatly reduces iteration time and keeps the known-good baseline intact.

---

# Image Milestones

Current artifacts follow a sequential milestone model:

```
T1.5  → HyperOS 1 boot / framework compatibility
T1.6  → Camera compatibility
T1.7  → Xiaomi Account compatibility
T1.8  → ThemeManager stability
T1.10  → Local MTZ theme investigationTheme-packaged system image + Local MTZ theme investigation
T1.11 → Theme UID / stability baseline
T1.12 → Generic AOSP RIL compatibility
T1.13 → Hybrid MTK components + Generic AOSP RIL
T1.14 → SIM power-state / voice call compatibility
```

Each milestone is intended to be derived from the previous verified baseline
rather than rebuilding the ROM from scratch.

---

# Important Design Principle

The project intentionally avoids broad compatibility hacks whenever possible.

Preferred:

```text
missing service
    ↓
return gracefully
```

instead of:

```text
disable entire subsystem
```

Preferred:

```text
one failing binary branch
    ↓
minimal patch
```

instead of:

```text
replace entire framework
```

Preferred:

```text
preserve vendor HAL
```

instead of:

```text
replace vendor camera stack
```

This keeps the amount of platform-specific modification as small as possible.

---

# Current Status

## Working

* HyperOS 1 graphical boot
* SystemUI
* Launcher
* Display
* Touch
* Camera
* Flash / Torch (only work when use it in camera app)
* Xiaomi Account compatibility work
* ThemeManager startup
* Local theme import investigation (or not?)

## Experimental

* Online Xiaomi Theme Store
* Full local `.mtz` integration
* Persistent custom-theme packaging
* Additional Xiaomi services

## Not a Goal Yet

* Full Xiaomi vendor stack
* Complete Xiaomi hardware parity
* Production OTA compatibility
* Official Samsung / Xiaomi certification

---

# Repository Structure

A future source layout may look like:

```text
.
├── patches/
│   ├── framework/
│   ├── camera/
│   ├── account/
│   └── themes/
├── images/
├── forensic/
│   ├── logs/
│   ├── disassembly/
│   └── reports/
├── scripts/
├── tools/
└── README.md
```

---

# Credits

This project builds on work from:
* MysticGSI on Sourceforge.net
* Android Open Source Project
* Samsung Open Source releases
* Xiaomi HyperOS / MIUI components
* Android community reverse-engineering tools
* GSI / Treble community
* the developers and maintainers whose device trees, kernels and tools made
  cross-device experimentation possible

Special thanks to everyone involved in reverse-engineering, testing and
debugging this port.

---

# Disclaimer

This is an independent community project.

**HyperOS**, **MIUI**, **Samsung**, **Galaxy**, **Xiaomi**, and related names
are trademarks of their respective owners.

This project is intended for research, development and interoperability
testing.

Flashing modified system software can brick a device or cause data loss.
Always keep a known-good stock firmware and recovery path available.

---

# Project Philosophy

> Make it boot.
>
> Then make it work.
>
> Then find out why it didn't work in the first place.

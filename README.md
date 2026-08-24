
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

But why I still upload and recommended T11.1? Cuz the theme patching (T1.8-T1.10) have a bug that cannot open theme (and Wallpaper & Personalization) apps through setting and else. T1.11 fixed it correctly.
---

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
T1.9  → Local MTZ theme investigation
T1.10 → Theme-packaged system image
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


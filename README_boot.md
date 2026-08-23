## Boot Image Lineage

The HyperOS 1 port went through several isolated boot-image experiments
before reaching the working hybrid boot architecture.

| Milestone | Artifact | Achievement |
|---|---|---|
| T1 | `T1_BOOT_FSTAB_NOVERITY.img` | First removal of `avb=vbmeta_system` from `/system` |
| T1.1 | `T1_1_BOOT_FSTAB_NOVERITY_NOAVBKEYS.img` | Removed both `avb` and `avb_keys`, eliminating the remaining `/system` AVB path |
| T1.2 | `T1.2_BOOT_FSTAB_NOVERITY_VENDOR.img` | Investigated secondary AVB activation originating from Samsung vendor fstab |
| T1.2-P | `T1.2-P_BOOT_SELINUX_PERMISSIVE.img` | Controlled A/B experiment isolating SELinux enforcement as an independent compatibility variable |

### Final HyperOS 1 Boot Architecture

The final HyperOS 1 boot architecture preserves the Samsung A03 hardware
foundation:

- Samsung Galaxy A03 Linux 4.14.199 kernel
- Samsung A03 DTB
- Samsung first-stage `/init`
- Samsung `fstab.ums9230_25c10`
- HyperOS Android 14 as second-stage userspace

This hybrid architecture allows HyperOS userspace to run on the A03's
native Unisoc hardware stack without replacing the device's fundamental
hardware initialization layer.

> **Note:** T2/T3 boot artifacts belong to the separate HyperOS 2 / Android 15
> investigation and are intentionally excluded from the HyperOS 1 lineage.

# 1. Match the device and establish a baseline

[Guide index](../README.md) · [Next: memory and root](02-memory-and-root.md)

## Exact tested profile

| Property | Tested value |
|---|---|
| Retail model | Fengmi / Formovie V10, M146FCN |
| Optical engine / platform | FMP6 / target1006 |
| Hardware profile | ID 4, revision 5 |
| Android | 9, API 28, VNDK 28 |
| Firmware | FengOS / BSP 2.1.5, build 3936 |
| Build fingerprint | `Formovie/nemo/nemo:9/PPR2.180905.006.A1/2.1.5.3936:user/release-keys` |
| Reported security patch | 2021-01-05 |
| Kernel | AArch64 Linux 4.9.118+, build dated 2025-02-10 |
| Android userspace | ARM32 / `armeabi-v7a`, binder64 |
| Partition model | A-only, legacy system-as-root |
| Mali driver | Bifrost `r16p0-01rel0.c96e22d.20220226` |
| Mali userspace/kernel interface | 11.13 |
| Baseline shell | UID 2000, SELinux enforcing |
| Baseline verified boot | Locked / green |

The tested `mali_kbase.ko` SHA-256 is:

```text
e064691a543a294fa196c8a1361e2f5ed1204373081df8a8ed71d14b703a3a68
```

Treat the full profile as the compatibility condition. Re-derive addresses, structure layouts and loader behavior when any relevant component differs. The codename `nemo` also occurs on other Formovie products; retail model and firmware identity matter.

## Establish network ADB and record state privately

Use the projector's authorized ADB connection. Substitute its address locally:

```sh
adb connect <PROJECTOR_IP>:5555
adb -s <PROJECTOR_IP>:5555 shell getprop ro.build.fingerprint
adb -s <PROJECTOR_IP>:5555 shell uname -a
adb -s <PROJECTOR_IP>:5555 shell id
adb -s <PROJECTOR_IP>:5555 shell getenforce
adb -s <PROJECTOR_IP>:5555 shell getprop ro.boot.vbmeta.device_state
adb -s <PROJECTOR_IP>:5555 shell getprop ro.boot.verifiedbootstate
adb -s <PROJECTOR_IP>:5555 shell sha256sum /vendor/lib/modules/mali_kbase.ko
```

Keep the complete property set, boot identity, private network information and raw logs in private local records. Capture enough state to establish whether a later operation rebooted the device or altered its security state. Apply timeouts to every device command.

The helpers in this investigation were standalone Rust `no_std` AArch64 executables using Linux syscalls. That permits native 64-bit kernel-facing code even though the Android application environment is 32-bit. Read-only GPU-property qualification was also demonstrated with an ARMv7 caller.

## Preserve the projector's working interface

Record working HDMI picture, audio, Bluetooth-remote input, held volume, Home, focus, keystone and optical controls before changes. Preserve the vendor services needed for those functions.

Optional launcher cleanup can be performed independently of root. The demonstrated FengOS arrangement retains the system Home package and uses a CleanMyTV/CleanYourTV-derived accessibility service to observe Home-window changes and explicitly open the replacement launcher. Preserve normal key handling: the working service observes windows rather than intercepting volume keys. User-level package removal with `pm uninstall -k --user 0 <package>` retains the system APK and is paired with an exact per-package restore inventory.

Credit for this preparation belongs to the [CleanYourTV community and FengOS-specific adaptation](../CREDITS.md#projector-preparation). It is not a prerequisite for the kernel mechanism below.

## Useful vendor inspection

Two vendor interfaces were useful for establishing the platform before memory access:

- The MediaTek `/dev/cli` synchronous command interface accepts ioctl request 1 with a complete, initialized, NUL-terminated **256-byte** buffer. Read-only help, command listing and a print-only control confirm the interface and its debug-level kernel forwarding.
- A read-only FengmiPlatform Binder bridge can call the normal boot-environment getter with fixed key names. The boot recipes establish how the stock loader reads BOOT, recovery and DTB.

The measured recipes are:

```text
bootcmd=mmc read.p 0x23000000 dtb 0x00100000; mmc read.p 0x25000000 boot 0x02000000; bootm 0x25000000
recoverycmd=mmc read.p 0x23000000 dtb 0x00100000; mmc read.p 0x25000000 recovery 0x02000000; bootm 0x25000000
```

Interpret these using exact MBOOT evidence. The environment's `verify` setting belongs to the legacy-image checking path; determine AVB authorization separately as described in [the backup and trust-chain chapter](03-backups-and-avb.md).

The earlier Philips work by **yath**, **Said Sürücü** and the published MStar sources provided protocol and bootloader context; see [credits](../CREDITS.md#vendor-interface-and-bootloader-research).
